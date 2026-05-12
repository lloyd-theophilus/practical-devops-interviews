# SRE Interview Questions & Answers

---

**Q: How do you build a culture where latency SLOs are enforced like uptime SLAs in a streaming org?**

**A:** The gap between uptime SLAs (binary — up or down) and latency SLOs (percentile-based, nuanced) is mostly cultural, not technical. Closing it requires three things: visibility, accountability, and consequence.

**1. Make latency as visible as uptime**:
- Every team dashboard has p50/p95/p99 latency for their service front-and-center alongside error rate and availability. If you have to dig for latency, it won't get fixed.
- Define SLOs in terms that resonate with the business: "95% of stream start times < 2s" lands harder than "p95 < 2000ms."
- In a streaming org specifically: track stream start time, buffering ratio, and rebuffering events — these are the metrics viewers feel. Wire them to Grafana alerts alongside uptime.

**2. Burn rate alerts tied to SLO budgets**:
- Calculate an error budget per SLO: if the p99 SLO is 500ms over a 30-day window, how many requests can exceed 500ms before the budget is exhausted?
- Alert when burn rate > 2× normal (budget burning too fast). This makes latency degradation page the on-call before users notice — same urgency as an outage.

**3. Embed latency in the deployment gate**:
- Load tests in CI measure p99 latency against the SLO target. A deploy that regresses p99 by > 10% is rejected automatically — no manual override without VP approval.
- Canary deployments (Argo Rollouts or Flagger) check latency SLOs on the canary slice before promoting to 100%.

**4. Ownership and consequence**:
- Each service has a named team that owns its SLO. SLO compliance (not just uptime) is reviewed in quarterly engineering reviews.
- Error budget policies: if budget is < 10% remaining, the team freezes feature deploys and focuses only on reliability work. Leadership sees this on the same dashboard as revenue metrics.

**5. Blameless retrospectives for latency SLO breaches** — same rigor as post-mortems for outages. The cultural message is: latency regressions are production incidents.

---

**Q: A batch job missed SLA by 14 minutes due to downstream API throttling. How would you implement an adaptive backoff + SLA-aware retry policy?**

**A:** Standard exponential backoff ignores time remaining in the SLA window. An SLA-aware retry policy has to be deadline-conscious.

**Design**:

```python
import time
import random
from datetime import datetime, timezone

def sla_aware_retry(fn, sla_deadline: datetime, max_attempts: int = 10):
    """
    Retries fn() with adaptive backoff that accelerates as the SLA deadline approaches.
    """
    attempt = 0
    base_delay = 1.0   # seconds
    max_delay  = 60.0

    while attempt < max_attempts:
        try:
            return fn()
        except ThrottlingError as e:
            attempt += 1
            now = datetime.now(timezone.utc)
            time_remaining = (sla_deadline - now).total_seconds()

            if time_remaining <= 0:
                raise SLAExceeded(f"SLA deadline exceeded after {attempt} attempts") from e

            # Exponential backoff with jitter, but capped by time remaining
            delay = min(
                base_delay * (2 ** attempt) + random.uniform(0, 1),
                max_delay,
                time_remaining * 0.5   # never wait more than half remaining budget
            )

            if delay <= 0:
                raise SLAExceeded("Insufficient time remaining to retry") from e

            time.sleep(delay)
        except NonRetriableError:
            raise   # don't retry 4xx auth errors, etc.

    raise MaxAttemptsExceeded()
```

**Key principles**:
- **Deadline propagation**: pass the SLA deadline through every layer of the call stack (use context deadlines in Go, `asyncio` timeouts in Python). The downstream call timeout shrinks as the deadline approaches.
- **Adaptive backoff ceiling**: `min(exponential_backoff, remaining_time * 0.5)` prevents wasting the last few seconds on a long sleep.
- **Jitter**: `random.uniform(0, 1)` spreads retries across time to avoid synchronized thundering herd when many jobs are throttled simultaneously.
- **Circuit breaker**: wrap the retry loop in a circuit breaker (using `pybreaker` or Resilience4j). If the downstream API is throttling at the account level, continuing to retry burns your quota faster. Open the circuit after N consecutive throttle responses; let it half-open after a configurable timeout.
- **Observability**: emit metrics on `retry_attempt_count`, `delay_applied_ms`, `sla_headroom_remaining_s` so you can detect systematic throttling patterns and negotiate higher quotas proactively.

**Infrastructure-level complement**: use an SQS queue with a dead-letter queue and visibility timeout tuned to the SLA window. If the job can be made async, the queue handles retry scheduling without the job process blocking.

---

**Q: How do you define "SLOs that actually matter" in financial systems, where failure isn't an error, it's a loss?**

**A:** In financial systems, the standard "availability and latency" SLO framing misses the point. A payment that succeeds in 200ms but debits the wrong amount is infinitely worse than a payment that fails cleanly with a retryable error. The SLO framework must capture **correctness** alongside performance.

**Framework for financially meaningful SLOs**:

**1. Identify the user journeys that generate or protect money**:
- Payment initiation → settlement: every millisecond of latency has a cost (cash drag, FX exposure).
- Fraud detection: false negative rate (missed fraud) is a direct loss; false positive rate is customer experience loss.
- Reconciliation: late or incorrect reconciliation = regulatory risk and potential fines.

**2. Define SLOs in terms of the business invariant, not technical metrics**:

| Journey | Technical SLO ❌ | Business SLO ✅ |
|---|---|---|
| Payment processing | API p99 < 500ms | ≥ 99.95% of submitted payments reach terminal state (success/failure) within 30s |
| Fraud scoring | Service availability > 99.9% | ≤ 0.01% of transactions processed without a fraud score |
| Daily settlement | Batch completes by T+1 09:00 | Settlement file delivered to counterparty by 08:45 with zero row-count discrepancy |

**3. "Failure modes" are not all equal — weight them**:
- A silent data corruption (wrong amount processed) is a severity-1 SLO breach even if the system returned HTTP 200.
- Implement **correctness SLOs**: idempotency key collision rate < 1 in 10⁸; debit/credit balance mismatch rate = 0.
- Use end-to-end synthetic monitors that simulate real transactions and validate the outcome, not just the HTTP status code.

**4. Error budget policy tied to financial risk**:
- When the correctness SLO error budget falls below 20%, freeze all changes to the payment processing path.
- When it hits 0%, activate the incident runbook: pause new payment acceptance, engage the on-call CFO/risk officer.

**5. Regulatory dimension**:
- SLOs must reflect regulatory SLAs (PCI-DSS, SOX, MiFID II): audit log write latency < 1s, encryption key rotation completed within 24h of detection of compromise. These are non-negotiable SLOs with external accountability.

---

**Q: What's your framework for change management in infra that moves billions, balancing agility with auditability?**

**A:** The core tension: finance teams need auditability and control; engineering teams need velocity. The framework resolves this by making the audit trail automatic, not a manual tax on engineers.

**1. Everything is code, every change is a PR**:
- All infrastructure (Terraform, Helm, Kubernetes manifests) lives in Git. A change to production infrastructure = a merged PR. No exceptions, including hotfixes (hotfixes get a post-hoc PR within 24h).
- The PR is the audit artifact: it records who, what, when, why (linked to a ticket), and who approved.

**2. Tiered change classification**:

| Tier | Examples | Controls |
|---|---|---|
| Standard | Config value change, replica count | Automated CI gate + 1 peer review |
| Significant | New service, network route change, IAM policy | 2 peer reviews + security review + change advisory board (CAB) async notification |
| Emergency | Production incident hotfix | 1 reviewer (senior SRE), post-incident PR within 24h, incident ticket linked |
| Prohibited | Direct console changes to prod | Blocked by SCP + IAM policy; exceptions require CISO approval |

**3. Automated enforcement**:
- SCPs block direct AWS console changes to production accounts.
- Terraform Sentinel or OPA policies reject plans that violate security rules (open SGs, unencrypted storage) before they reach the `apply` stage.
- All `terraform apply` runs happen in CI/CD (not on engineers' laptops) — provides a tamper-evident audit log in the CI system.

**4. Immutable audit log**:
- CloudTrail + S3 Object Lock: every AWS API call recorded and immutable for 7 years (regulatory requirement).
- GitHub PR history + protected main branch: no force-push, no branch deletion. The full change history is permanent.
- All CI/CD pipeline runs logged to a central SIEM (Splunk/Security Lake) and retained per compliance requirement.

**5. Deployment windows and blast radius limits**:
- High-value payment flows: changes only during a defined maintenance window (e.g., Sunday 02:00–04:00 UTC, lowest transaction volume). Automated deploy scheduler enforces this.
- Canary deployments: max 5% of payment traffic on the new version for 30 minutes before full rollout. Automatic rollback if error rate or correctness SLO degrades.

**6. Change advisory board (CAB) — async, not a bottleneck**:
- Significant changes are posted to a Slack channel 48h before deployment. Stakeholders (risk, compliance, operations) have 24h to object. Silence = approval. This preserves agility while satisfying regulatory requirements for documented change approval.

---

**Q: How do you convince leadership to fund observability when "everything looks stable"?**

**A:** "Everything looks stable" is exactly the argument observability helps you make — or disprove. The pitch isn't technical, it's economic.

**1. Quantify the cost of the last incident we didn't see coming**:
- Find one incident in the past 12 months where time-to-detect (TTD) or time-to-resolve (TTR) was extended because you lacked data. Calculate the cost: `revenue_lost_per_minute × (actual_TTR - target_TTR)`.
- Example: "Last quarter's payment processing outage lasted 47 minutes. With distributed tracing, the similar incident at [competitor/peer company] was resolved in 8 minutes. The difference was $1.4M in delayed settlements and $200K in SLA penalties."

**2. Reframe "stable" as "blind"**:
- Show the dashboards you have today: uptime and basic CPU/memory. Then show a demo of what observability looks like at a mature org (flame graphs, distributed traces, SLO burn rate dashboards).
- Ask: "Do we actually know if our p99 latency for payment initiation is meeting the contractual SLA right now?" If the answer is "we think so" rather than "yes, here's the dashboard," that's the gap.

**3. Tie observability to regulatory and audit requirements**:
- In financial services, regulators (FCA, SEC, RBI) increasingly require evidence that you can detect and respond to incidents within defined timeframes. Observability is the evidence trail.
- "We are one audit finding away from being asked to demonstrate 30-second MTTD for fraud-related anomalies. We can't do that today."

**4. Show the ROI model**:
- Observability investment: $X/year (tooling + headcount).
- Expected MTTD improvement: from 45 min to 5 min (based on industry data).
- Expected MTTR improvement: from 90 min to 20 min.
- At our transaction volume and revenue rate, each avoided 60-minute outage saves $Y. We need N avoided incidents per year to break even.
- Frame it as insurance with a known premium, not a cost center.

**5. Start with a funded proof of concept**:
- Propose a 90-day PoC: instrument the top-3 revenue-critical services with distributed tracing and SLO dashboards. Track before/after MTTD and MTTR. Let the data make the case for the full rollout.

---

**Q: What's your approach to onboarding SREs into a culture of blameless RCA with regulatory rigor?**

**A:** The tension here is real: blameless culture requires psychological safety and forward-looking improvement, while regulatory environments require documented root causes, named accountable parties, and evidence of corrective action. Done poorly, they undermine each other. Done well, they reinforce each other.

**1. Separate "accountability for systems" from "blame for people"**:
- In training and post-mortems, consistently use language like "the system allowed this" rather than "person X caused this."
- The regulatory requirement for a "root cause" can be satisfied with a systemic root cause (e.g., "lack of input validation in the payment gateway allowed malformed data to propagate") — this doesn't require naming an individual as the cause.
- What regulators actually want: evidence that the organization identified what failed, why, and what was done to prevent recurrence. A blameless post-mortem delivers exactly this.

**2. Structured post-mortem template that satisfies both goals**:
```
Incident Summary: [what happened, impact in business terms]
Timeline: [factual, minute-by-minute, no language like "mistakenly" or "failed to"]
Contributing Factors: [5-whys or Fishbone, systemic causes only]
What Went Well: [explicitly required — acknowledges that people made good decisions too]
Corrective Actions: [each with an owner, ticket, and deadline — this is the regulatory artifact]
Lessons Learned: [shared with the wider team]
```
The "Corrective Actions" section is what regulators audit. It is specific, owned, and time-bound — satisfying rigor without assigning blame in the narrative.

**3. Onboarding curriculum**:
- Week 1: shadow 2–3 post-mortems before writing one. New SREs see the format and tone modeled by experienced practitioners.
- Week 2: lead a post-mortem for a low-severity incident with a senior SRE as coach.
- Week 4: solo post-mortem for a medium incident, reviewed by the SRE lead.
- Ongoing: monthly "post-mortem review" sessions where the team reads 3 post-mortems together (including external ones from the Google SRE book, AWS blog) — normalizes the practice and the language.

**4. Leadership signals matter most**:
- When leadership responds to a post-mortem by asking "who was responsible?" instead of "what can we improve?" — the blameless culture dies immediately.
- Coach leadership explicitly: post-mortems are not disciplinary documents. If an individual is performing poorly, that's a separate HR conversation, not a post-mortem finding.
- Senior SREs and the SRE manager must write their own post-mortems for incidents they were involved in — models that even senior people make mistakes and that's safe to admit.

**5. Regulatory documentation layer**:
- Maintain a separate, shorter "incident summary for compliance" document that extracts only the regulatory-required fields from the post-mortem: incident ID, date/time, impact classification, root cause (systemic), corrective actions with owners and dates.
- This satisfies audit requirements without turning the post-mortem itself into a legal document — keeping it safe for honest, open discussion.
