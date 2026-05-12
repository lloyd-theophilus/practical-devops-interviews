# AWS VPC Interview Questions & Answers

---

**Q: How do you secure your VPC workloads at scale?**

**A:** A defense-in-depth approach across multiple layers:

**1. Network segmentation**:
- Separate public, private, and data subnets across multiple AZs.
- Place internet-facing resources only in public subnets (ALBs); compute and databases in private subnets with no direct internet access.
- Use separate VPCs per environment (prod/staging/dev) with VPC peering or AWS Transit Gateway for controlled inter-VPC connectivity.

**2. Security Groups (stateful, instance-level)**:
- Default-deny all inbound; explicitly allow only required ports and sources.
- Reference other security groups (not IP CIDRs) for internal traffic — this survives IP changes automatically.
- Apply the principle of least privilege: each tier has its own SG (ALB SG → App SG → DB SG).
- Audit unused SG rules regularly (`aws ec2 describe-security-groups` + scripts to find unused rules).

**3. NACLs (stateless, subnet-level)**:
- Use NACLs for coarse-grained controls: block known malicious CIDR ranges, restrict inbound to expected traffic types.
- Remember NACLs are stateless — explicitly allow ephemeral return ports (1024–65535) for responses.

**4. VPC Flow Logs**:
- Enable VPC Flow Logs to CloudWatch Logs or S3 for all VPCs. Use them for security analysis, traffic baselining, and incident investigation.
- Send to Amazon Security Lake or SIEM for alerting on anomalous patterns (unexpected outbound connections, port scanning).

**5. AWS Network Firewall**:
- Deploy AWS Network Firewall in the inspection VPC (or inline in a centralized hub VPC via Transit Gateway) for stateful L7 filtering, IDS/IPS signatures, and domain-based filtering (block traffic to known malicious domains).
- Enforce TLS inspection for encrypted egress if required by compliance.

**6. AWS WAF + Shield**:
- Attach WAF to CloudFront and ALBs: block OWASP Top 10, rate limit, block by geo or IP reputation.
- Enable AWS Shield Advanced for DDoS protection on critical endpoints.

**7. PrivateLink and VPC Endpoints**:
- Use VPC Interface Endpoints for AWS services (S3, Secrets Manager, ECR, SSM) — traffic stays within the AWS backbone, never traverses the internet.
- Use PrivateLink to expose internal services to other VPCs/accounts without peering (no network-level access to the entire VPC).

**8. IAM + SCPs**:
- Service Control Policies in AWS Organizations: prevent disabling Flow Logs, CloudTrail, or creating Internet Gateways in sensitive accounts.
- Block `ec2:AuthorizeSecurityGroupIngress` for port 22/3389 from `0.0.0.0/0` via SCP.

**9. GuardDuty + Security Hub**:
- Enable GuardDuty in all accounts/regions: uses VPC Flow Logs, DNS logs, and CloudTrail to detect threats (crypto mining, C2 communication, credential exfiltration).
- Centralize findings in Security Hub for a unified view across the organization.

**10. Patch and AMI management**:
- Use AWS Systems Manager Patch Manager to keep all EC2 instances patched; no direct SSH — use Session Manager.
- Use golden AMIs built and scanned via Packer + Inspector on a weekly schedule.

**At scale — automation**:
- Use AWS Config rules to enforce: VPC Flow Logs enabled, no public subnets with 0.0.0.0/0 route from EC2 instances, all EBS volumes encrypted, no SGs with open 22/3389.
- AWS Security Hub standards (CIS AWS Foundations Benchmark, AWS Foundational Security Best Practices) provide continuous compliance scoring.
- Use Terraform + policy-as-code (Checkov, OPA) to prevent non-compliant VPC configurations from being applied.
