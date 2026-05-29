# AWS Networking Interview Questions & Answers

---

**Q: How does AWS Load Balancer route traffic?**

**A:** AWS ALB routes at Layer 7 (HTTP/HTTPS):
1. Client sends a request to the ALB DNS name (which resolves to one of the ALB's IP addresses per AZ).
2. ALB terminates TLS, reads the HTTP request headers.
3. **Listener rules** are evaluated in priority order: host-based (`host: api.example.com`), path-based (`/api/*`), header-based, query string, or source IP conditions.
4. The matching rule forwards the request to a **target group**.
5. The target group selects a target (EC2, ECS task, Lambda, IP) using the configured algorithm: round-robin (default), least outstanding requests, or weighted random.
6. The ALB forwards the request to the target, adding `X-Forwarded-For` (original client IP) and `X-Forwarded-Proto` headers.
7. The target's response flows back through the ALB to the client.

NLB routes at Layer 4: pure connection-level hash based on source IP, source port, destination IP, destination port, and protocol. No HTTP awareness.

---

**Q: What happens internally when you hit a CloudFront URL?**

**A:**
1. **DNS resolution**: the domain (e.g., `d1234.cloudfront.net`) resolves to a CloudFront PoP (Point of Presence) edge server IP geographically closest to the client via Anycast routing.
2. **Edge cache check**: the edge server checks if a cached response exists for the request (based on the cache key: URL path + configured headers/cookies/query strings). If cache hit → return cached response immediately with `X-Cache: Hit from cloudfront`.
3. **Cache miss**: the edge server forwards the request to the **origin** (S3 bucket, ALB, API Gateway, EC2) over the AWS backbone network. The request includes the original `Host` header and optionally the `X-Forwarded-For` header.
4. **Origin response**: the origin processes the request and returns the response with cache control headers (`Cache-Control: max-age=86400`).
5. **Edge caches the response**: CloudFront stores it in the edge cache for the TTL duration.
6. **Response to client**: the edge server sends the response to the client, adding CloudFront headers (`X-Cache: Miss from cloudfront`, `Via`, `X-Amz-Cf-Id`).

CloudFront can also run **Lambda@Edge** or **CloudFront Functions** at steps 2/3/5 to modify requests/responses without hitting the origin.

---

**Q: Netflix runs multi-cloud. Describe your approach to cross-cloud routing, IAM, and secret syncing.**

**A:**

**Cross-cloud routing**:
- Use a global **Anycast DNS** solution (Route53 + Cloud DNS + latency-based routing) or a third-party GSLB (Akamai, Cloudflare) to route users to the nearest healthy cloud region.
- For service-to-service cross-cloud: establish **VPN tunnels or Dedicated Interconnect/Direct Connect** between GCP VPC and AWS VPC; use a consistent CIDR plan to avoid overlaps.
- **Service mesh federation**: Istio multi-cluster setup (or Consul Connect) with a shared root CA; services in AWS can call services in GCP via the mesh with mTLS — routing is handled by the mesh control plane which has endpoints from both clouds.

**Cross-cloud IAM**:
- Use **Workload Identity Federation** (GCP) and **OIDC provider** (AWS): GCP workloads get a short-lived token from GCP's OIDC endpoint; AWS STS accepts it and issues AWS credentials. No static cross-cloud keys.
- Centralize identity in an enterprise IdP (Okta, Azure AD) federated to both AWS IAM Identity Center and GCP Cloud Identity.

**Secret syncing**:
- Avoid bidirectional sync — it creates conflicts and complexity.
- Prefer **secret references**: store the canonical secret in one vault (HashiCorp Vault or AWS Secrets Manager) and have all clouds fetch from that single source using OIDC/Workload Identity.
- If dual storage is required: use Terraform or a GitOps tool to write secrets to both AWS Secrets Manager and GCP Secret Manager from a single pipeline step; rotate in one place, propagate automatically.

---

**Q: How do you handle DNS-level outages inside a service mesh without a full app redeploy?**

**A:** In Istio/Envoy-based service meshes, service-to-service communication uses Envoy's Endpoint Discovery Service (EDS) — it knows pod IPs directly and doesn't rely on kube-dns at runtime for each request. So DNS outages in the cluster don't immediately break mesh-internal traffic.

**But for external DNS dependencies and initial discovery**:
- **Envoy's `dns_refresh_rate`**: tune how often Envoy re-resolves external hostnames; reduce TTL reliance.
- **CoreDNS resilience**: run CoreDNS with at least 2 replicas + a PodDisruptionBudget. Use `autopath` and caching plugins to reduce upstream DNS load.
- **Fallback**: configure Envoy `ServiceEntry` for external services with both DNS and static IP fallback:
  ```yaml
  spec:
    resolution: DNS
    endpoints:
      - address: "api.external.com"  # primary
    trafficPolicy:
      connectionPool:
        tcp:
          connectTimeout: 5s
  ```
- **Circuit breakers in Envoy**: if external DNS is failing, circuit breakers in `DestinationRule` prevent cascading failures while DNS recovers.
- **Ops response**: during a DNS outage, use `kubectl edit configmap coredns -n kube-system` to add static host overrides as a stopgap; coreDNS picks up the change within seconds.

---

**Q: What is the purpose of a NAT Gateway?**

**A:** A NAT (Network Address Translation) Gateway allows instances in **private subnets** to initiate outbound internet traffic (e.g., downloading packages, calling external APIs) **without being directly reachable from the internet**.

How it works:
- The private instance sends traffic with its private IP as the source.
- The NAT Gateway (in a public subnet, with an Elastic IP) rewrites the source IP to its own EIP using SNAT.
- The response comes back to the EIP; the NAT Gateway translates it back to the private instance's IP.
- Inbound connections initiated from the internet cannot reach the private instance (no entry in the NAT table).

**Cost**: NAT Gateway charges per hour + per GB processed. Alternatives for lower cost: VPC endpoints for AWS services (free for S3/DynamoDB), NAT instances (EC2, cheaper but less reliable), or AWS PrivateLink.

---

**Q: How would you set up geolocation-based routing using AWS services?**

**A:** Using **Route 53 Geolocation Routing**:
1. Create a hosted zone for your domain in Route 53.
2. Create A/AAAA records with geolocation routing policies:
   ```
   example.com → EU-ALB (for European traffic)
   example.com → US-ALB (for North American traffic)
   example.com → Default-ALB (for all other traffic)
   ```
3. Route 53 evaluates the source IP of the DNS query against GeoIP databases and returns the appropriate record.

**Enhanced with CloudFront + Lambda@Edge**:
- CloudFront injects `CloudFront-Viewer-Country` header.
- Lambda@Edge reads the header and redirects or rewrites the origin path for country-specific content.

**Use cases**: regulatory compliance (data residency), latency optimization, localized content.

**Alternative**: **Geoproximity routing** in Route 53 Traffic Flow (with bias) routes based on distance to resources, not just continent/country.

---

**Q: How do you implement Auto Scaling with proper health checks?**

**A:**
```bash
# Create launch template (reference existing)
aws ec2 create-launch-template --launch-template-name myapp-lt ...

# Create ASG
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name myapp-asg \
  --launch-template "LaunchTemplateName=myapp-lt,Version=$Latest" \
  --min-size 2 --max-size 10 --desired-capacity 3 \
  --vpc-zone-identifier "subnet-aaa,subnet-bbb" \
  --target-group-arns arn:aws:elasticloadbalancing:...:targetgroup/myapp/xxxx \
  --health-check-type ELB \           # use ALB health checks (not just EC2)
  --health-check-grace-period 120     # wait 2 min before checking (allow app startup)

# Add target tracking scaling policy (maintain 50% CPU)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name myapp-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ASGAverageCPUUtilization"},
    "TargetValue": 50.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

**Key health check considerations**:
- Use **ELB health checks** (not EC2) — ALB actively pings your app's `/health` endpoint; unhealthy instances are terminated and replaced.
- Set `health-check-grace-period` long enough for your application to fully start before health checks begin.
- Configure the ALB target group health check path, interval, threshold, and timeout to match your application's startup and response characteristics.

---

**Q: One Availability Zone suddenly goes down. How should the application behave?**

**A:**
A well-architected multi-AZ application should survive an AZ failure with minimal or zero user impact. Here is what should happen at each layer and what to verify if it doesn't:

**Expected behavior by layer**:

| Layer | Expected behavior during AZ failure |
|---|---|
| ALB / NLB | Automatically stops routing to targets in the failed AZ; continues sending traffic to healthy AZs |
| Auto Scaling Group | Detects unhealthy instances in the failed AZ, launches replacements in the remaining AZs |
| RDS Multi-AZ | Automatic failover to standby in a healthy AZ (60–120 seconds); application reconnects via the same endpoint |
| ElastiCache (Multi-AZ) | Replica in healthy AZ promoted to primary automatically |
| ECS / EKS | Tasks/pods in the failed AZ are rescheduled onto nodes in healthy AZs |

**Pre-requisites for this to work**:

1. **Subnets in multiple AZs**: the ASG must span at least 2–3 AZs (`--vpc-zone-identifier "subnet-az1,subnet-az2,subnet-az3"`).

2. **ALB cross-zone load balancing**: enabled by default on ALB; verify it is on for NLB (disabled by default, costs extra per GB).

3. **Min capacity**: `min-size` must be >= 2 so that if the AZ with the only running instance fails, a new one launches immediately.

4. **Stateless application tier**: EC2/ECS/EKS instances must be stateless — session state stored in ElastiCache or DynamoDB, not in local memory.

5. **RDS Multi-AZ enabled**: single-AZ RDS has no automatic failover. Verify:
   ```bash
   aws rds describe-db-instances \
     --db-instance-identifier prod-db \
     --query 'DBInstances[].MultiAZ'
   ```

6. **DNS TTL / connection retry**: RDS Multi-AZ failover changes the IP behind the DNS endpoint. Applications must handle reconnect (short `connect_timeout`, retry logic, or use RDS Proxy which absorbs the failover transparently).

**What to monitor during an AZ event**:
```bash
# Check ASG activity for replacements
aws autoscaling describe-scaling-activities --auto-scaling-group-name myapp-asg

# Check ALB target health across AZs
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/myapp/xxxx

# Check RDS failover events
aws rds describe-events \
  --source-identifier prod-db \
  --source-type db-instance \
  --duration 60
```

**Testing**: run regular **AZ failure drills** using AWS Fault Injection Simulator (FIS) — terminate all instances in one AZ, stop network traffic to a subnet — to verify your architecture actually survives before a real failure.

---
