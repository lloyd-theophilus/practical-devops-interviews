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
- You’re asked to ship a multi-region failover for live events in 2 weeks with no DNS-based routing allowed. What’s your plan?
- ⁠How would you simulate chaos in a streaming pipeline without risking real user impact?
- You're managing multi-region deployments using a single Kubernetes control plane. What architectural considerations must you address to avoid cross-region latency and single points of failure?
- Debug this: API latency spikes every morning between 9:25–9:35 AM (market open). Metrics show stable CPU/mem. What invisible infra factor are you missing?
- How would you design audit-grade logging for all infra actions while keeping overhead minimal and still meeting financial compliance?

