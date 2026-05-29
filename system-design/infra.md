# System Design / Infrastructure Interview Questions & Answers

---

**Q: How do you design infrastructure that empowers devs without giving them footguns?**

**A:** The core principle is **self-service within guardrails** — developers can move fast, but the platform prevents them from accidentally breaking things.

**Concrete mechanisms**:

1. **Internal Developer Platform (IDP)**: expose curated, opinionated abstractions (Backstage, Humanitec, or a custom portal). Developers click "deploy to staging" and don't need to know Terraform or kubectl. The platform handles the how; developers own the what.

2. **Golden paths, not mandates**: provide well-maintained reference architectures (Terraform modules, Helm chart library, GitHub Actions reusable workflows) that make the right thing easy. Don't block devs from going off the path — just make the path so good they rarely need to.

3. **Policy as code**: OPA/Gatekeeper admission policies in Kubernetes, Checkov in CI pipelines, and SCPs in AWS Organizations enforce security and compliance rules automatically — devs get a clear error message rather than a surprise in prod.
   - Example OPA policy: "No container may run as root; all images must come from our internal registry."

4. **Blast radius isolation**: 
   - Separate AWS accounts per environment (prod, staging, dev) — a dev mistake in dev literally cannot affect prod.
   - Kubernetes RBAC: devs have `edit` role in their namespace, not `cluster-admin`.
   - Resource quotas prevent one team from monopolizing the cluster.

5. **Fast feedback loops**: integrate `terraform plan` output into PRs so devs see infrastructure changes before merge. Pre-commit hooks run `hadolint`, `checkov`, and `trivy` so issues surface in the IDE, not in prod.

6. **Observability by default**: new services get pre-wired dashboards, alerts, and log forwarding automatically via a platform sidecar or Helm chart default values. Devs don't have to think about observability setup.

7. **Break-glass procedures**: for genuine emergencies, devs have a documented, audited path to elevated access (e.g., assume a prod-write role via SSO with auto-expiry and mandatory justification). Escape hatches exist, but they're tracked.

---

**Q: Describe a hybrid cloud routing architecture between GCP and AWS. Where do you enforce boundaries?**

**A:**

**Architecture overview**:
```
[AWS VPC - us-east-1]          [GCP VPC - us-central1]
  10.0.0.0/8                     172.16.0.0/12
       │                               │
  AWS Transit Gateway ◄──── Cloud VPN / Dedicated Interconnect + Partner Interconnect ────► GCP Cloud Router
       │                           (BGP over IPsec)                                             │
  AWS Direct Connect                                                                     GCP Cloud Interconnect
       │                                                                                         │
  On-premises DC ──────────────────────────────────────────────────────────────────────────────►
```

**Routing**:
- **BGP over VPN tunnels** (or Interconnect): both clouds advertise their CIDRs to each other. AWS Transit Gateway acts as the hub for multiple AWS VPCs; GCP Cloud Router handles dynamic route propagation on the GCP side.
- **Non-overlapping CIDRs**: critical — plan IP address space across all clouds and on-prem before day one. Use a CMDB for IP allocation.
- **Traffic flow**: AWS app → Transit Gateway → VPN tunnel → GCP Cloud Router → GCP VPC. Latency: typically 30–70ms cross-cloud (vs < 1ms within AZ).

**Where boundaries are enforced**:
1. **Network layer**: AWS Network Firewall / GCP Firewall Policies define which subnets/services can talk cross-cloud. Not everything should cross the boundary — only explicitly approved flows.
2. **Identity layer**: AWS IAM + GCP IAM are separate systems. Use Workload Identity Federation (OIDC) so GCP workloads can assume AWS roles without static keys, and vice versa. Central IdP (Okta) manages human identities for both.
3. **Data boundary**: classify data; PII and regulated data may not leave a specific cloud/region. Enforce via DLP policies and network egress rules.
4. **API boundary**: prefer REST/gRPC APIs (exposed via API Gateway on each cloud) over direct database connections across clouds — adds an authorization layer and decouples the two clouds' internal implementations.
5. **Encryption**: all cross-cloud traffic is encrypted in transit (IPsec for VPN tunnels; TLS 1.3 for application traffic). No unencrypted traffic leaves a VPC boundary.
6. **Audit**: CloudTrail (AWS) + Cloud Audit Logs (GCP) both feed into a centralized SIEM. Cross-cloud flows are logged at the Network Firewall / VPC Flow Log level.

---

**Q: How to troubleshoot high latency in API Gateway?**

**A:** Systematic isolation across the full request path:

**1. Identify where latency is introduced**:
- AWS API Gateway provides `IntegrationLatency` (time waiting for backend) and `Latency` (total end-to-end at API GW level) CloudWatch metrics.
- If `IntegrationLatency` ≈ `Latency` → the problem is in your backend (Lambda, ALB, EC2).
- If `Latency` >> `IntegrationLatency` → the problem is in API Gateway itself (throttling, auth, request validation).

**2. Check for throttling**:
- CloudWatch: `4XXError` with count spike → check `429` errors (throttled).
- API Gateway has account-level (10,000 req/s default) and per-stage limits. Use usage plans and API keys to manage throttling per client.
- Request: `aws apigateway get-stage --rest-api-id <id> --stage-name prod` → check `defaultRouteThrottlingBurstLimit`.

**3. Lambda cold starts** (if Lambda backend):
- CloudWatch Logs Insights: query `@initDuration` to find cold start latency.
- Fix: enable Provisioned Concurrency for latency-sensitive functions; use Lambda SnapStart for Java.

**4. VPC configuration** (if Lambda in VPC):
- Lambda in VPC used to have cold start ENI attachment delay (now fixed with pre-warmed ENIs post-2019). Verify you're using the new Hyperplane ENI model (Lambda function created after Aug 2019).

**5. Payload size**:
- API Gateway has a 10 MB payload limit. Large request/response bodies add processing time. Use S3 pre-signed URLs for large payloads.

**6. Caching**:
- Enable API Gateway caching for GET endpoints with stable responses: reduces backend invocations and latency significantly. Configure cache TTL and invalidation appropriately.

**7. X-Ray tracing**:
```bash
aws apigateway update-stage \
  --rest-api-id <id> \
  --stage-name prod \
  --patch-operations op=replace,path=/tracingEnabled,value=true
```
X-Ray traces show the exact breakdown: API Gateway overhead, Lambda init, Lambda execution, downstream calls.

**8. Regional vs Edge-optimized**:
- Edge-optimized API Gateway routes through CloudFront PoPs — adds latency for same-region clients. If your clients are in a known region, use a **Regional** endpoint and put CloudFront in front yourself for more control.

**Q: You’re asked to ship a multi-region failover for live events in 2 weeks with no DNS-based routing allowed. What’s your plan?**

**A:** No DNS routing means no Route53 latency/failover records or GeoDNS. Traffic routing must happen below DNS — at the network/anycast level or the application layer.

**Option 1 — AWS Global Accelerator (fastest to ship)**:
- Assigns two static anycast IPs routed over AWS’s backbone to the nearest healthy endpoint.
- Configure endpoint groups in each region (ALB, NLB, or EC2 IPs).
- Failover is automatic: if the primary region’s health check fails, Global Accelerator routes to the secondary within ~30 seconds. No DNS TTL involved.

```bash
aws globalaccelerator create-accelerator --name live-events-ga --ip-address-type IPV4
aws globalaccelerator create-listener --accelerator-arn <arn> --protocol TCP --port-ranges FromPort=443,ToPort=443
aws globalaccelerator create-endpoint-group --listener-arn <arn> \
  --endpoint-group-region us-east-1 \
  --endpoint-configurations EndpointId=<alb-arn>,Weight=100,ClientIPPreservationEnabled=true \
  --traffic-dial-percentage 100
# Add secondary region with traffic-dial-percentage 0 (standby)
```

**2-week delivery plan**:
- Week 1: deploy application stack to secondary region (Terraform module reuse), set up Aurora Global Database, configure Global Accelerator.
- Week 2: load test failover, validate replication lag, write the failover runbook, chaos test with AWS FIS.

**Data layer**: Aurora Global Database provides < 1s replication lag. On failover, promote the secondary cluster in ~60 seconds.

---

**Q: How would you simulate chaos in a streaming pipeline without risking real user impact?**

**A:** The key principle: inject failures into a shadow copy of the pipeline processing real traffic whose output is discarded, or use a dark-launch environment fed by replayed production data.

**Approach 1 — Shadow environment with traffic mirroring**:
- Use Kafka MirrorMaker 2 to replicate production topics to a staging pipeline.
- Run chaos experiments on the shadow pipeline: kill brokers, introduce consumer lag, inject network latency into the stream processor.
- Monitor shadow outputs vs production outputs for divergence — zero user impact.

**Approach 2 — AWS FIS in production with blast radius controls**:
```json
{
  "targets": { "kafka-brokers": { "resourceType": "aws:ec2:instance",
    "resourceTags": { "Role": "kafka-broker" }, "selectionMode": "COUNT(1)" } },
  "actions": { "inject-latency": { "actionId": "aws:ssm:send-command", "parameters": { "duration": "PT5M" } } },
  "stopConditions": [{ "source": "aws:cloudwatch:alarm", "value": "arn:...consumer-lag-alarm" }]
}
```
`stopConditions` auto-halt the experiment if consumer lag or error rate exceeds a threshold — a kill switch that limits blast radius.

**Approach 3 — Feature-flag–gated fault injection**:
- Wrap a `ChaosFaultInjector` middleware behind a feature flag (LaunchDarkly/Flagsmith).
- Enable for 1% of traffic in production; disable via flag if error rate rises.

**What to test in a streaming pipeline**:
- Consumer lag recovery after a broker restart.
- Exactly-once semantics under producer retries.
- Schema Registry availability loss (fail open or closed?).
- Dead-letter queue behavior on deserialization failures.

---

**Q: You're managing multi-region deployments using a single Kubernetes control plane. What architectural considerations must you address?**

**A:** A single control plane spanning multiple regions is an anti-pattern for production. The API server and etcd incur 50–150ms RTT between regions, slowing scheduling and heartbeats. A control plane region failure affects all regions simultaneously.

**The right architecture**: separate control planes per region, managed by a GitOps layer (Argo CD multi-cluster, Fleet) for unified deployment.

**If a single control plane is unavoidable**:
1. **Control plane placement**: host the control plane in one region. Workers in other regions remain functional during a control plane outage (existing pods keep running; no new scheduling).
2. **etcd latency**: Raft quorum requires round-trips. Cross-region etcd members add 2× RTT to every write. Evaluate if this is acceptable for your control plane write frequency.
3. **Data plane isolation**: separate VPCs per region with VPC peering/Transit Gateway. Don't route application traffic cross-region through the control plane.
4. **Node affinity**: use `topologySpreadConstraints` and affinity rules to prevent the scheduler from placing pods cross-region unintentionally.
5. **Network partition risk**: if the cross-region link fails, workers enter NotReady after `node-monitor-grace-period` (40s default). No new scheduling in the disconnected region.

**Strongly prefer separate clusters per region** — managed by Argo CD App of Apps for declarative, GitOps-driven multi-region delivery.

---

**Q: API latency spikes every morning between 9:25–9:35 AM (market open). Metrics show stable CPU/mem. What invisible infra factor are you missing?**

**A:** Stable CPU and memory with time-bound latency spikes means the bottleneck is **waiting**, not compute.

**Most likely causes at market open**:

1. **DB connection pool exhaustion**: sudden request burst exhausts the pool. New requests queue for a free connection — CPU is idle, latency spikes. Check: `pg_stat_activity` count vs `max_connections`; pool wait time metrics (HikariCP, pgBouncer).

2. **Cache cold start (thundering herd)**: a cache cleared overnight. At 9:25 AM all requests miss cache simultaneously and hammer the DB. Check: cache hit rate drops to 0% at that time.

3. **Scheduled job collision**: a daily report/aggregation/index rebuild runs at 9:25 AM, acquiring DB locks or consuming all I/O. Check: cron schedule, DB slow query log, `pg_locks`.

4. **JVM GC warm-up**: after a quiet night, JIT hasn't optimized hot paths and caches are cold. First burst triggers GC pressure. Check: GC pause histogram at 9:25 AM.

5. **Auto Scaling not pre-warmed**: ASG scaled down overnight; instances at 9:25 AM are still in grace period. Traffic hits a small fleet until new instances are healthy. Add scheduled scaling to pre-warm before market open.

6. **CloudFront / CDN TTL expiry**: cache TTL set to overnight causes a cache-miss storm on first morning request.

**Confirm**: plot p99 latency heatmap by hour over 1 week. Cross-reference with scheduled job inventory and connection pool exhaustion metrics specifically between 9:20–9:40 AM.

---

**Q: How would you design audit-grade logging for all infra actions while keeping overhead minimal and meeting financial compliance?**

**A:**

**Mandatory layers**:

1. **AWS CloudTrail** (org-wide): captures every API call. Store in a dedicated audit account S3 bucket with S3 Object Lock (WORM, 7-year retention), SSE-KMS, and a bucket policy that denies all deletes including from root — except a break-glass role.

2. **IaC audit trail**: all `terraform apply` runs only through CI/CD. Tag every Terraform-managed resource with `Pipeline=run-1234`, `Operator=user@company.com`. Pipeline logs stored in S3 with 1-year retention.

3. **Kubernetes API audit logs** (EKS): enable EKS audit logging to CloudWatch Logs. Every `kubectl` action is logged with who, what, when, and result.

4. **Minimize cost**:
   - Log **management plane events only** (state-changing API calls), not data plane events (S3 GETs on every object read).
   - Use S3 Intelligent-Tiering: hot logs in S3 Standard → auto-tiered to Glacier after 30–90 days.
   - **Athena** for compliance queries: query CloudTrail JSON directly from S3 — no Elasticsearch required.
   - 90 days in CloudWatch Logs; export older logs to S3.

5. **Meeting financial compliance (SOC2, PCI-DSS)**:
   - SIEM (Splunk, Panther, AWS Security Hub) normalizes CloudTrail + VPC Flow Logs + EKS audit logs.
   - Quarterly evidence exports: "all changes to production security groups in Q3" via Athena → CSV.
   - Alert on privileged API calls (IAM policy changes, security group modifications) outside the CI/CD role ARN.

---

**Q: APIs suddenly start returning timeout errors. What could be happening?**

**A:** A timeout means the server accepted the connection but didn't respond in time. The bottleneck is in the processing path.

**Triage by layer**:

1. **LB vs app**: check ALB `TargetResponseTime` metric. If it spikes → the app/DB is slow. ALB 504 (`HTTPCode_ELB_5XX_Count`) = target didn't respond within `idle_timeout` (default 60s).

2. **Database slowdown**: most timeout cascades start here. `SELECT * FROM pg_stat_activity WHERE state='active';` — long-running queries blocking others? Check for lock waits, full table scans, or a runaway analytics query.

3. **Thread/connection pool exhausted**: all app threads are waiting for DB connections or a slow downstream HTTP call. New requests queue until timeout.

4. **External dependency hung**: third-party API (payment, auth) is slow or unresponsive. Implement circuit breakers and per-call timeouts shorter than your overall API timeout.

5. **JVM GC full pause**: stop-the-world GC pauses of 5–30s timeout all in-flight requests. Check GC pause histogram.

6. **Lambda cold start**: provisioned concurrency insufficient for a traffic spike. Cold start + DB init > timeout threshold.

7. **Kubernetes CNI / network issue**: dropped packets causing TCP retransmission delays that masquerade as application timeouts. Check VPC Flow Logs for drops.

**Quick remediation**: temporarily increase timeout thresholds; add circuit breakers; scale the DB if connection-starved; rollback the last deploy.

---

**Q: Users report slowness but server resources look healthy. What could be missing?**

**A:** Healthy CPU, memory, and pods mean the compute layer is fine. The app is **waiting** — blocked on I/O or a dependency — not computing.

**Check these**:

1. **Database query latency**: the app waits for slow queries; CPU is idle during the wait. Check: Postgres `pg_stat_statements`, RDS Performance Insights `ReadLatency`/`WriteLatency`.

2. **External API latency**: a downstream service your app depends on is slow. Use distributed tracing to find the slow span.

3. **Lock contention**: DB row locks or application-level mutexes. Threads blocked while waiting consume no CPU. Check: `pg_locks`, thread dumps.

4. **DNS resolution delays**: `ndots:5` in Kubernetes adds extra search domain queries. Each `nslookup` may try 5 FQDNs before resolving. Check CoreDNS metrics for resolution time and errors.

5. **JVM GC pauses**: not reflected in CPU averages but cause stop-the-world request stalls. Check GC pause histogram.

6. **Network I/O**: cross-AZ latency, TCP retransmissions, packet loss. Check VPC Flow Logs and EC2 network metrics.

7. **Cache miss storm**: cold caches after a restart or TTL expiry → all requests hit the DB.

---

**Q: What steps do you take to prevent accidental infrastructure deletion?**

**A:**

1. **Terraform lifecycle protection**:
   ```hcl
   lifecycle { prevent_destroy = true }
   ```
   Any `terraform destroy` or resource removal fails with an explicit error.

2. **AWS resource-level protections**:
   - EC2: `--disable-api-termination`
   - RDS: `--deletion-protection`
   - S3: Object Lock (WORM) for compliance data
   - EKS: tag with `DeletionProtection=true` + enforce via SCP

3. **IAM / SCP deny rules** (applied at AWS Organizations level):
   ```json
   { "Effect": "Deny", "Action": ["ec2:TerminateInstances", "rds:DeleteDBInstance", "s3:DeleteBucket", "eks:DeleteCluster"],
     "Resource": "*",
     "Condition": { "StringNotEquals": { "aws:PrincipalArn": "arn:aws:iam::123:role/BreakGlassRole" } } }
   ```
   Only a specific break-glass role (with MFA requirement) can delete production resources.

4. **CI/CD plan review gate**: any `terraform plan` showing `destroy` requires explicit human approval before `apply` proceeds. Atlantis posts the plan to the PR and blocks auto-apply.

5. **Backups as a safety net**: S3 versioning, RDS automated backups (7-day retention), EBS snapshots via AWS Backup.

6. **AWS Config rule**: alert when a resource tagged `Protected=true` is deleted.

---

**Q: How do you design a disaster recovery process for zero-downtime applications?**

**A:**

**Step 1 — Define RTO and RPO**:
- **RTO** (Recovery Time Objective): max acceptable downtime (e.g., < 5 min).
- **RPO** (Recovery Point Objective): max acceptable data loss (e.g., < 1 min).

**DR strategies by tier**:

| Strategy | RTO | RPO | Cost |
|---|---|---|---|
| Backup & Restore | Hours | Hours | Low |
| Pilot Light | 10–30 min | Minutes | Medium |
| Warm Standby | 1–5 min | Seconds | High |
| Active-Active | < 1 min | Near-zero | Very High |

**For zero-downtime — Active-Active with Global Accelerator**:
- Application runs at full capacity in 2+ regions simultaneously.
- Aurora Global Database replicates < 1s cross-region.
- Global Accelerator routes users to the nearest healthy region via anycast.
- On region failure: traffic shifts automatically — no manual failover.

**Warm Standby failover runbook** (if Active-Active is cost-prohibitive):
1. `aws rds failover-global-cluster` → promote secondary.
2. Update Global Accelerator endpoint weights: 0% primary, 100% secondary.
3. Scale up DR region ASG to production capacity.
4. Verify health checks pass. Total time: 3–5 minutes.

**Test it**: run quarterly chaos drills with AWS FIS to simulate region failure. Measure actual RTO/RPO against targets. Update the runbook based on results.

---

**Q: How do you identify whether an issue is from the application, database, or network?**

**A:** Isolate each layer with direct tests, then use metrics and tracing to confirm.

**Step 1 — Distributed tracing first**: if you have Jaeger/X-Ray, open the trace for a slow request. The span breakdown immediately shows where time is spent: app code, DB query, or inter-service network call.

**Step 2 — Direct component tests**:
```bash
# Test app directly (bypass LB)
kubectl exec -it <pod> -- curl http://localhost:8080/api/endpoint

# Test DB directly (bypass app)
time psql -h <rds-endpoint> -U appuser -c "SELECT 1"

# Test network (TCP reachability and latency)
kubectl exec -it <pod> -- nc -zv postgres-svc 5432
kubectl exec -it <pod> -- traceroute postgres-svc
```

**Step 3 — Interpret results**:

| Observation | Layer |
|---|---|
| App slow even for a trivial DB query (`SELECT 1` is fast) | Application (GC, thread pool, business logic) |
| `SELECT 1` fast, complex query slow | Database (missing index, lock contention, bad plan) |
| App and DB healthy but inter-service calls slow | Network (cross-AZ latency, packet loss, MTU mismatch) |
| Consistent latency regardless of load | Network baseline or DNS resolution delay |
| Latency only under load | Resource contention (connection pool, CPU throttling) |

**Per-layer metrics**:
- **Application**: GC pause histogram, thread pool queue depth, error rate per endpoint.
- **Database**: `pg_stat_statements` for slow queries, `pg_locks` for lock waits, RDS `ReadLatency`/`WriteLatency`.
- **Network**: VPC Flow Logs for drops, `ping`/`traceroute` RTT, CloudWatch network saturation metrics.

---

**Q: When would you prefer Virtual Machines over containers?**

**A:**

| Scenario | Why VMs | Example |
|---|---|---|
| **Kernel-level requirements** | Containers share the host kernel; VMs have an isolated kernel. Required for custom kernel modules or specific kernel versions. | Security tools that patch the kernel |
| **Strong security isolation** | VM hypervisor provides stronger isolation than container namespaces. A container escape can compromise the host. | PCI-DSS workloads, running untrusted third-party code, multi-tenant SaaS |
| **Stateful legacy applications** | Apps designed for bare-metal that assume full OS control, specific filesystem layouts, or don't containerize cleanly. | Oracle DB, SAP, legacy Java EE apps |
| **Windows workloads on Linux hosts** | Can't run a Windows container on a Linux kernel. | .NET Framework apps not yet on .NET Core |
| **GPU-intensive ML training** | GPU pass-through to VMs (SR-IOV) can be more straightforward for some GPU types. | Training large models on bare-metal GPUs |
| **Strict licensing** | Some software licenses require a dedicated VM/physical server. | Per-core licensed commercial software |

**In practice**: containers for all stateless services, APIs, and batch jobs. VMs for databases needing guaranteed IOPS and OS tuning, legacy apps that resist containerization, and any workload requiring a different kernel.

---

**Q: Where do you store and manage secrets in your project?**

**A:** Secrets are never in code, Git, Docker images, or environment variables visible in the console. They live in a dedicated secrets manager and are injected at runtime.

**My stack**:

1. **AWS Secrets Manager** (primary for application secrets): DB credentials, API keys, OAuth client secrets. Auto-rotation for RDS credentials built in. Applications fetch at startup via SDK; cached in memory, refreshed before expiry.

2. **Secrets Store CSI Driver** (for Kubernetes): mounts secrets from Secrets Manager as files or env vars in pods — never stored in Kubernetes Secrets/etcd.
   ```yaml
   volumes:
     - name: db-creds
       csi:
         driver: secrets-store.csi.k8s.io
         readOnly: true
         volumeAttributes:
           secretProviderClass: aws-prod-secrets
   ```

3. **AWS Systems Manager Parameter Store** (for non-sensitive config): feature flags, environment-specific config. SecureString type (KMS-encrypted) for mildly sensitive values.

4. **SOPS + KMS** (for GitOps/IaC secrets): Terraform variable files with sensitive values are encrypted with SOPS before committing. Decrypted in CI via KMS — plaintext never touches the repository.

5. **Access control**: IRSA for pod-level IAM in EKS — each workload has a minimal-scope IAM role. Secrets Manager resource policies restrict `GetSecretValue` to specific roles.

**What I never do**: secrets in `docker build --build-arg` (visible in image history), Kubernetes Secrets without external secrets management (stored base64-encoded in etcd by default), `.env` files committed to Git, or rotating secrets by redeploying the application.

---


