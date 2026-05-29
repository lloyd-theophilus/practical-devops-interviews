# Load Balancer Interview Questions & Answers

---

**Q: A new AWS ALB config caused TLS handshakes to fail intermittently. Walk through your full RCA path.**

**A:**
1. **Reproduce and scope**: check if failures are intermittent (some requests succeed) or total (all fail). Use `curl -v https://example.com` to capture the TLS handshake details — look for the cipher suite and certificate presented.
2. **ALB access logs**: enable ALB access logs to S3; filter for `ssl_cipher` and `ssl_protocol` fields to see which TLS sessions are failing and from which client IPs.
3. **CloudWatch metrics**: check `ClientTLSNegotiationErrorCount` metric on the ALB — spike confirms TLS-specific failures, not app-level errors.
4. **Security policy change**: review recent ALB changes. A new SSL security policy (e.g., `ELBSecurityPolicy-TLS13-1-2-2021-06`) may have dropped cipher suites that some clients (old browsers, legacy services) were using.
5. **Certificate validity**: `openssl s_client -connect alb-dns:443` — verify the certificate chain is complete, cert is not expired, and the SAN matches the hostname.
6. **ACM certificate**: if using ACM, check if the cert was recently renewed and the ALB has the new cert ARN configured.
7. **Intermediate certificate issue**: ALB must present the full chain. Check if the intermediate cert is included.
8. **Fix**: if cipher policy is the issue, revert to a broader policy or coordinate client updates. If cert chain is broken, re-import the full chain to ACM.

---

**Q: Autoscaling isn't kicking in despite the CPU crossing the threshold. What's broken — metrics, HPA, or API server?**

**A:** Systematic isolation:

1. **Check HPA status**:
   ```bash
   kubectl describe hpa myapp
   ```
   Look for: `Unable to fetch metrics`, `unknown`, or the current/target metric values.

2. **Metrics Server health**:
   ```bash
   kubectl top pods -n kube-system   # should work
   kubectl get apiservice v1beta1.metrics.k8s.io -o yaml   # check Available: true
   ```
   If Metrics Server is unavailable, HPA can't get CPU data.

3. **HPA logs and events**: `kubectl get events | grep HPA`.

4. **CPU metrics vs limits**: HPA uses `cpu usage / cpu request` ratio, NOT raw CPU. If pods have no `resources.requests.cpu` set, HPA can't calculate utilization percentage — configure requests.

5. **Cooldown / stabilization window**: HPA has a 5-minute scale-down stabilization; for scale-up it should be immediate. Check `scaleUp.stabilizationWindowSeconds`.

6. **API server**: `kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes` — if this times out, the aggregation layer or Metrics Server is broken.

7. **Fix**: ensure Metrics Server is running, pods have resource requests set, and the HPA threshold is realistic (not set to 0 or 100%).

---

**Q: Prod users reporting 504s, but ELB health checks are green. Explain your isolation + triage process.**

**A:** Green health checks mean the backend is accepting connections, but 504 = upstream timeout. The issue is between the ALB and the application response time.

**Triage**:
1. **ALB access logs**: filter for `504` status codes. Check `target_processing_time` — if it's close to or exceeds the ALB idle timeout (default 60s), that's the culprit.
2. **Target response time**: check ALB CloudWatch metric `TargetResponseTime` — is it spiking?
3. **Connection draining**: if a deploy is happening, old targets may be draining while new ones are starting.
4. **Backend application**: 504 from ALB means the target didn't respond within `idle_timeout.timeout_seconds`. Check application logs for slow queries, DB connection pool exhaustion, or deadlocks.
5. **Database**: check RDS CloudWatch for `DatabaseConnections`, `ReadLatency`, `WriteLatency`. A slow DB query can cascade to 504s.
6. **Thread pool / connection pool exhaustion**: if all application threads are busy (e.g., waiting for a slow downstream service), new requests time out.
7. **Downstream API timeout**: if your app calls an external API that's slow, increase your internal timeout or implement circuit breakers.
8. **Fix**: increase ALB idle timeout if processing is inherently slow, optimize slow queries, scale application horizontally, implement async processing for long-running tasks.

---

**Q: Difference between ALB and NLB and when to use which?**

**A:**

| | ALB (Application Load Balancer) | NLB (Network Load Balancer) |
|---|---|---|
| OSI Layer | Layer 7 (HTTP/HTTPS/gRPC/WebSocket) | Layer 4 (TCP/UDP/TLS) |
| Routing | Content-based: host header, path, query string, headers | Connection-based: IP, port |
| TLS termination | Yes (offloads TLS) | Yes (pass-through or terminate) |
| Latency | Higher (L7 processing) | Ultra-low (millions of req/s, microseconds) |
| Target types | EC2, containers, Lambda, IP | EC2, IP, ALB |
| Static IP | No (use NLB in front) | Yes (one static IP per AZ) |
| WebSocket | Yes | Yes |
| Use case | Web apps, microservices, API Gateway, host-based routing | Real-time apps, gaming, IoT, TCP services, static IP requirement, PrivateLink |

**When to use ALB**: web applications that need path-based routing, host-based routing, authentication (Cognito), or WAF integration.

**When to use NLB**: TCP/UDP workloads, ultra-low latency requirements, static IP for whitelisting, services behind AWS PrivateLink.

---

**Q: Explain how you’d use Envoy + Istio to route low-latency live streams differently from VOD without service restarts.**

**A:** Istio’s traffic management is fully dynamic — `VirtualService` and `DestinationRule` changes are pushed to Envoy sidecars via xDS without any pod restart.

**Design**:
- Tag pods with a label: `stream-type: live` vs `stream-type: vod`.
- Create two `DestinationRule` subsets:
  ```yaml
  apiVersion: networking.istio.io/v1beta1
  kind: DestinationRule
  metadata:
    name: streaming-service
  spec:
    host: streaming-service
    subsets:
      - name: live
        labels:
          stream-type: live
        trafficPolicy:
          connectionPool:
            tcp:
              connectTimeout: 50ms     # aggressive timeout for live
            http:
              http2MaxRequests: 10000
          outlierDetection:
            consecutive5xxErrors: 1
            interval: 5s
            baseEjectionTime: 10s     # fast failover for live streams
      - name: vod
        labels:
          stream-type: vod
        trafficPolicy:
          connectionPool:
            tcp:
              connectTimeout: 500ms   # tolerant for VOD buffered delivery
  ```
- Route by HTTP header or URL prefix with a `VirtualService`:
  ```yaml
  spec:
    http:
      - match:
          - headers:
              x-stream-type:
                exact: live
        route:
          - destination:
              host: streaming-service
              subset: live
        retries:
          attempts: 1                 # no retries for live (stale frames)
        timeout: 200ms
      - route:
          - destination:
              host: streaming-service
              subset: vod
        retries:
          attempts: 3
        timeout: 10s
  ```

Applying this `VirtualService` is a `kubectl apply` — Istiod pushes the xDS update to all relevant Envoy sidecars in seconds, zero restarts.

---

**Q: HPA refuses to scale in a critical podset even though Prometheus shows CPU > 90%. Root cause & fix?**

**A:** The key insight: **HPA does not read from Prometheus directly**. By default, HPA queries the `metrics.k8s.io` API (served by Metrics Server), not Prometheus. A discrepancy between what Prometheus shows and what HPA acts on is a common trap.

**Root causes and fixes**:

1. **HPA is using Metrics Server, but you’re watching Prometheus**: Metrics Server and Prometheus collect CPU differently (cAdvisor data vs direct cgroups scraping with different scrape intervals). Verify what HPA is actually seeing:
   ```bash
   kubectl get hpa myapp -o yaml        # check currentMetrics
   kubectl describe hpa myapp           # shows actual observed value
   ```

2. **HPA is configured with a custom metric from Prometheus Adapter — but the adapter is misconfigured**: the metric name in the HPA spec doesn’t match the series name in the Prometheus Adapter config. Check:
   ```bash
   kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq ‘.resources[].name’
   ```
   If your metric isn’t listed, the Prometheus Adapter isn’t exposing it.

3. **Prometheus is reporting a different metric than what resources.requests.cpu reflects**: HPA calculates `utilization = actual_cpu / requested_cpu`. If `resources.requests.cpu` is set very high (e.g., `4000m`) but actual usage is `3600m`, utilization is only 90% of a very large number — which may still be below the HPA threshold.

4. **`minReplicas` == `maxReplicas`**: a misconfigured HPA with equal min/max simply can’t scale.

5. **Scale-up stabilization window** — check `behavior.scaleUp.stabilizationWindowSeconds`; a non-zero value delays scale-up.

**Fix**: if you want HPA to act on Prometheus metrics, deploy the [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter), configure `rules` mapping your Prometheus query to the custom metrics API, then reference `type: Pods` or `type: Object` in the HPA spec.

---

**Q: A network policy is blocking traffic between services in different namespaces. How would you design and debug the policy to allow only specific communication paths?**

**A:**
**Debug first**:
```bash
# Check existing policies in both namespaces
kubectl get networkpolicies -n namespace-a
kubectl get networkpolicies -n namespace-b

# Test connectivity from a debug pod
kubectl run test -n namespace-a --image=busybox --rm -it -- \
  wget -qO- --timeout=3 http://myservice.namespace-b.svc.cluster.local:8080

# Check if CNI supports NetworkPolicy (vanilla kubenet does NOT)
kubectl get pods -n kube-system | grep -E ‘calico|cilium|weave|flannel’
```

**Design — allow namespace-a frontend → namespace-b backend on port 8080 only**:
```yaml
# Applied in namespace-b (protects the backend)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-namespace-a
  namespace: namespace-b
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: namespace-a   # namespace label (auto-set in k8s 1.21+)
          podSelector:
            matchLabels:
              app: frontend           # AND: must also be this pod
      ports:
        - protocol: TCP
          port: 8080
```

**Key nuance**: `namespaceSelector` + `podSelector` in the same `from` list element means **AND** (both must match). Two separate elements would mean **OR**.

For Cilium, use `CiliumNetworkPolicy` for L7 (HTTP path/method) rules across namespaces with the same pattern but richer match fields.

---

**Q: Your service mesh sidecar (e.g., Istio Envoy) is consuming more resources than the app itself. How do you analyze and optimize this setup?**

**A:**
**Analyze**:
```bash
# Compare app vs sidecar resource usage
kubectl top pods -n mynamespace --containers | grep -E ‘APP|istio-proxy’

# Envoy admin API — check active listeners, clusters, stats
kubectl exec -it <pod> -c istio-proxy -- curl http://localhost:15000/stats/prometheus | \
  grep -E ‘envoy_server_memory_allocated|envoy_cluster_upstream_rq_total’

# Check how many services Envoy knows about (xDS payload size)
kubectl exec -it <pod> -c istio-proxy -- curl http://localhost:15000/clusters | wc -l
```

**Common causes and fixes**:

1. **Too many xDS endpoints**: Envoy receives configuration for every service in the mesh. At 100+ services, this bloats memory.
   - **Fix**: use `Sidecar` resources to scope each proxy’s discovery:
     ```yaml
     apiVersion: networking.istio.io/v1beta1
     kind: Sidecar
     metadata:
       name: frontend-sidecar
       namespace: frontend-ns
     spec:
       egress:
         - hosts:
             - "backend-ns/backend-service"  # only what frontend calls
             - "istio-system/*"
     ```
   - This alone can reduce Envoy memory by 60–80% in large meshes.

2. **High access log volume**: Envoy logging every request at INFO level is CPU-intensive.
   - **Fix**: set `meshConfig.accessLogFile: ""` to disable access logs (use Zipkin/Jaeger sampling instead), or set sampling to 1%.

3. **CPU spikes from tracing**: 100% trace sampling floods the Jaeger collector and burns CPU.
   - **Fix**: set `meshConfig.defaultConfig.tracing.sampling: 1.0` (1%).

4. **Concurrency setting too high**: `proxy.concurrency: 0` uses all cores. Cap it:
   - Set `proxy.concurrency: 2` in `istio-operator` or pod annotation `proxy.istio.io/config: ‘{"concurrencyPolicy": 2}’`.

5. **Consider Istio Ambient Mesh**: moves the sidecar to a per-node `ztunnel` proxy — eliminates the per-pod overhead entirely for services that don’t need L7 features.

---

**Q: During peak traffic, your ingress controller fails to route requests efficiently. How would you diagnose and scale ingress resources effectively under heavy load?**

**A:**
**Diagnose**:
```bash
# Check ingress controller pod metrics
kubectl top pods -n ingress-nginx

# NGINX ingress: check active connections and request rate
kubectl exec -n ingress-nginx <pod> -- curl http://localhost:10246/nginx_status

# Check for errors in ingress controller logs
kubectl logs -n ingress-nginx <pod> --tail=200 | grep -E ‘error|upstream|timeout’

# Prometheus metrics (if NGINX ingress metrics are enabled)
# nginx_ingress_controller_requests (rate)
# nginx_ingress_controller_request_duration_seconds (latency)
# nginx_ingress_controller_nginx_process_connections (active connections)
```

**Common failure modes and fixes**:

1. **Too few ingress controller replicas**: ingress controller is a single point of failure if running one replica.
   ```bash
   kubectl scale deployment ingress-nginx-controller -n ingress-nginx --replicas=5
   ```
   Set an HPA on the ingress controller deployment targeting CPU at 60%.

2. **NGINX worker processes**: default is `auto` (= CPU cores). On a small pod this is 1–2.
   - Annotation: `nginx.ingress.kubernetes.io/worker-processes: "8"` (or set in the ConfigMap).

3. **Connection backlog / keepalive**: tune NGINX `worker_connections` and upstream keepalive in the ingress ConfigMap:
   ```yaml
   worker-connections: "65536"
   upstream-keepalive-connections: "200"
   upstream-keepalive-requests: "10000"
   ```

4. **Upstream pod count**: ingress controller can route fast, but if backend pods are overwhelmed, requests queue. Scale the backend deployment or enable HPA.

5. **Affinity / session stickiness**: if `nginx.ingress.kubernetes.io/affinity: cookie` is set, load distribution is uneven. Remove it or switch to least-connections load balancing.

6. **Rate limiting too strict**: if the ingress has rate limiting annotations, peak traffic hits the limit. Tune `nginx.ingress.kubernetes.io/limit-rps`.

7. **For very high scale**: move from NGINX ingress to a purpose-built gateway (AWS ALB + AWS Load Balancer Controller, Kong, or Envoy Gateway) which offloads TLS termination and load balancing to purpose-built infrastructure.

**Q: ALB shows healthy targets but users still face downtime. What could be wrong?**

**A:** Healthy ALB targets mean the health check endpoint is responding, but that doesn't guarantee the full application is functional. Common causes:

1. **Health check path is too shallow**: the `/health` endpoint returns 200 even when the DB is down or a critical dependency is broken. Fix: make the health check endpoint verify real dependencies (DB connection, cache reachability).

2. **Sticky sessions routing to a bad instance**: if session affinity (sticky sessions) is enabled, some users are pinned to a degraded instance that passes the simple health check but fails for real traffic. Disable stickiness or fix the degraded instance.

3. **Connection draining in progress**: a deploy is draining old targets; in-flight requests on those targets are failing. Check ALB target group "deregistration delay" — if set too low, requests are dropped mid-flight.

4. **SSL/TLS mismatch at the application layer**: ALB terminates TLS and forwards plain HTTP to the target. If the app incorrectly expects HTTPS internally, it may fail requests while the TCP health check passes.

5. **Application-level errors not caught by health check**: 500 errors in the business logic won't affect the health check endpoint. Check ALB access logs for `HTTPCode_Target_5XX_Count`.

6. **WAF or Security Group blocking real user traffic**: the health check source IP is allowed, but user IP ranges are blocked by a WAF rule or Security Group change. Check WAF sampled requests and Security Group rules.

7. **Route53 DNS TTL caching old IPs**: if a DNS change was made, old cached IPs may point to decommissioned resources. TTL expiry is required before full propagation.

8. **Cross-zone load balancing disabled**: one AZ has no healthy targets, but cross-zone is off so traffic to that AZ fails. Enable cross-zone load balancing or ensure equal target distribution.

```bash
# Check for 5XX errors from targets
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=<alb-arn-suffix> \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Sum
```

---

**Q: Ingress works internally but fails externally. What will you check?**

**A:** When traffic works inside the cluster but fails from the internet, the break is in the path between the external client and the Ingress controller.

**Systematic checks**:

1. **DNS resolution**: does the domain resolve to the correct external IP/ALB?
   ```bash
   dig myapp.example.com
   nslookup myapp.example.com 8.8.8.8
   ```
   Confirm the A/CNAME record points to the Ingress controller's external IP or ALB DNS name.

2. **TLS/HTTPS certificate**: is the certificate valid and matching the domain?
   ```bash
   openssl s_client -connect myapp.example.com:443 -servername myapp.example.com
   ```
   Check for expired cert, self-signed cert not trusted by browsers, or missing SAN (Subject Alternative Name).

3. **Security Group / Firewall rules**: the LoadBalancer Service or ALB must allow inbound 80/443 from `0.0.0.0/0`.
   ```bash
   aws ec2 describe-security-groups --group-ids <sg-id> \
     --query 'SecurityGroups[].IpPermissions'
   ```

4. **Ingress controller Service type**: confirm the Ingress controller Service is `type: LoadBalancer` (not `ClusterIP`). Check it has an `EXTERNAL-IP` assigned:
   ```bash
   kubectl get svc -n ingress-nginx
   ```

5. **Ingress class annotation**: the Ingress resource must reference the correct class. Wrong class = the controller ignores it.
   ```bash
   kubectl describe ingress myapp -n production
   # Check: "kubernetes.io/ingress.class" or "ingressClassName"
   ```

6. **Cloud LB provisioning failure**: the LoadBalancer may be stuck in "Pending" state.
   ```bash
   kubectl describe svc ingress-nginx-controller -n ingress-nginx
   # Look for events like "Error creating load balancer"
   ```
   Common causes: IAM permissions missing, subnet not tagged with `kubernetes.io/role/elb: 1`, no available EIPs.

7. **NACL blocking external traffic**: Security Groups are checked but NACLs at the subnet level may block port 443 from external IPs (especially if a NACL deny rule was added).

8. **ALB annotation misconfiguration** (AWS Load Balancer Controller): verify `alb.ingress.kubernetes.io/scheme: internet-facing` is set. Without it, the ALB is internal-only.

---

