# AWS Cost Optimization Interview Questions & Answers

---

**Q: Your company wants to reduce AWS cost for idle resources. What strategies would you implement?**

**A:**
1. **Identify idle resources**: use AWS Cost Explorer, Trusted Advisor, and Compute Optimizer to find:
   - EC2 instances with < 5% average CPU over 14 days.
   - Unattached EBS volumes (`aws ec2 describe-volumes --filters Name=status,Values=available`).
   - Unused Elastic IPs (incur charges when not attached).
   - Idle NAT Gateways, load balancers, and RDS instances with zero connections.
   - Lambda functions with zero invocations in 30 days.

2. **EC2 right-sizing**: use AWS Compute Optimizer recommendations; downsize over-provisioned instances.

3. **Reserved Instances / Savings Plans**: convert predictable workload spend to Compute Savings Plans (up to 66% discount) or EC2 Reserved Instances.

4. **Auto Scaling**: implement ASGs with target tracking so instances scale in during off-hours. Use scheduled scaling for predictable patterns (e.g., scale to 1 at midnight, scale to 10 at 8 AM).

5. **Spot Instances**: use Spot for batch jobs, CI/CD agents, development environments (up to 90% discount). Use `aws ec2 describe-spot-price-history` to pick cost-effective pools.

6. **Dev/test environment scheduling**: use AWS Instance Scheduler or Lambda + EventBridge to stop non-prod instances outside business hours.

7. **Storage tiering**: move infrequently accessed S3 objects to S3-IA or Glacier using lifecycle policies; use gp3 instead of gp2 EBS (20% cheaper, better performance).

8. **Delete orphaned resources**: automate weekly sweeps with AWS Config rules or a Lambda function that tags and terminates resources older than X days without owner tags.

---

**Q: How would you implement centralized logging for multiple AWS accounts?**

**A:** A common multi-account logging architecture:

**Architecture**:
```
Account A (App) ──► CloudWatch Logs ──► Kinesis Data Firehose ──┐
Account B (App) ──► CloudWatch Logs ──► Kinesis Data Firehose ──┼──► S3 (Log Archive Account)
Account C (App) ──► CloudWatch Logs ──► Kinesis Data Firehose ──┘        │
                                                                           ▼
                                                                    Athena / OpenSearch
```

**Implementation steps**:
1. **Log Archive Account**: create a dedicated AWS account in AWS Organizations for log storage. Create an S3 bucket with:
   - Object Lock (WORM) for tamper protection.
   - Block Public Access.
   - Bucket policy allowing `s3:PutObject` from a cross-account Firehose delivery role only.

2. **Per-account**: deploy a CloudWatch Logs subscription filter that sends log groups to Kinesis Data Firehose (or Kinesis Data Streams for more control).
   ```bash
   aws logs put-subscription-filter \
     --log-group-name /aws/lambda/myfunction \
     --filter-name AllLogs \
     --filter-pattern "" \
     --destination-arn arn:aws:firehose:us-east-1:LOG_ARCHIVE_ACCOUNT:deliverystream/centralized-logs
   ```

3. **Cross-account delivery role**: Firehose uses a role in the Log Archive account; each app account's CloudWatch Logs has permission to put records into this stream.

4. **AWS Organizations + CloudTrail**: enable an organization-level CloudTrail that automatically logs all accounts to the central S3 bucket.

5. **Query**: use Amazon Athena (with Glue crawler) or OpenSearch Service for log analysis. Partition S3 data by account/region/date for cost-efficient querying.

6. **Alternatives**: use Fluent Bit DaemonSets (EKS) → Amazon OpenSearch Service in the central account, or AWS Security Lake for security-focused centralized logging.

---

**Q: Your company's cloud costs are increasing rapidly. How would you approach cost optimization without impacting performance?**

**A:** Structured cost optimization approach:

**Phase 1 — Visibility (first 2 weeks)**:
- Enable AWS Cost Explorer with hourly granularity and resource-level tags.
- Set up AWS Budgets with alerts at 80% and 100% of budget.
- Tag all resources with `Project`, `Environment`, `Owner`, `CostCenter`. Enforce via SCPs.
- Use the Cost & Usage Report (CUR) in S3 + Athena for detailed analysis.
- Run AWS Trusted Advisor and Compute Optimizer for immediate recommendations.

**Phase 2 — Quick wins (weeks 2–4)**:
- Delete idle resources (unused EBS, EIPs, NAT GWs, old snapshots).
- Rightsize over-provisioned EC2 and RDS instances per Compute Optimizer.
- Switch EBS gp2 volumes to gp3 (always cheaper, better baseline IOPS).
- Move S3 to Intelligent-Tiering or add lifecycle policies.
- Stop non-prod environments outside business hours (saves ~65% on dev costs).

**Phase 3 — Commitment discounts (month 2–3)**:
- Purchase Compute Savings Plans for the baseline (steady-state) workload (1-year, no upfront = 17–37% off; 3-year, all upfront = up to 66% off).
- Purchase RDS Reserved Instances for production databases.
- Use Spot Instances for CI runners, batch jobs, ML training.

**Phase 4 — Architecture optimization (ongoing)**:
- Move suitable workloads to serverless (Lambda, Fargate) — pay per invocation, not idle time.
- Implement S3 Select and Athena instead of running query servers 24/7.
- Use CloudFront to reduce EC2/origin load and data transfer costs.
- Optimize data transfer: keep traffic within the same AZ/region where possible (inter-AZ data transfer is charged).
- Use Graviton (ARM-based) instances: up to 40% better price/performance for compatible workloads.

**Measurement**: track cost per unit of business value (cost per API request, cost per user) rather than just absolute cost — allows you to optimize without sacrificing growth.

---

**Q: How do you justify infra costs for pre-warmed scaling capacity to executives before a major sports event?**

**A:** This is a risk-vs-cost conversation. Executives understand business risk better than infrastructure mechanics, so the justification must be framed in business terms.

**Build the cost-of-failure case first**:

Quantify what a degraded or down platform costs during peak:
- **Revenue at risk**: peak concurrent users × average revenue per user per minute × estimated downtime probability without pre-warming. Example: "100,000 concurrent users × $0.10 ARPU/min × 10-minute degradation = $100,000 potential revenue loss."
- **SLA penalties**: if you have contractual uptime guarantees with broadcast partners or enterprise customers, calculate the penalty clauses for a peak-event outage.
- **Reputational cost**: during a live event, social media reaction to buffering or downtime is amplified. Quantify as customer churn or brand equity impact if your org has historical data.
- **Historical precedent**: if you've had a scaling event before (or a competitor has), use it. "During the 2023 final, our platform buffered for 8 minutes during the first goal — we lost 12% of concurrent viewers and saw 3× normal churn that week."

**Frame pre-warming as insurance with a known premium**:

| | Without pre-warming | With pre-warming |
|---|---|---|
| Scaling response time | 3–7 minutes (autoscaler + instance boot) | Immediate (capacity already warm) |
| Risk of degradation | ~15% (based on load test results) | < 1% |
| Infrastructure cost | $X (baseline) | $X + $Y (warm capacity, 4-hour window) |
| Expected cost of degradation | 0.15 × $100K loss = $15K expected loss | 0.01 × $100K = $1K expected loss |
| **Net position** | — | **Save $14K in expected loss, cost $Y to pre-warm** |

If $Y (pre-warming cost) < $14K (expected loss reduction), the ROI is positive.

**Operational justification**:
- Pre-warming is time-bounded (4–6 hours around the event) — not a permanent cost increase.
- Spot/Reserved instances used for warm capacity reduce the cost vs on-demand.
- Load tests validate the exact capacity needed — you're not guessing, you're provisioning based on evidence.

**The ask**:
"For a 4-hour window around the event, we need an additional $X in EC2/CDN capacity. In exchange, we reduce the probability of a revenue-impacting incident from 15% to < 1%. This is the same principle as buying event cancellation insurance — the premium is small relative to the protected value."

Present this with a one-page brief: event date, peak user projection (based on ticket sales, historical viewership, marketing reach), load test results, cost of pre-warming, and cost of not pre-warming. Executives approve insurance; they just need to see the math.

