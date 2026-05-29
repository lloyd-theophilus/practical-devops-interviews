# AWS Serverless Interview Questions & Answers

---

**Q: Lambda execution time suddenly increases. What could cause this?**

**A:**
Lambda duration increases fall into three categories: cold starts, downstream dependency slowness, and resource/concurrency constraints. Work through each:

**1. Check the metrics first**:
```bash
# Get average and p99 duration over the last hour
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=my-function \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average Maximum
```
Also check `Throttles`, `ConcurrentExecutions`, `InitDuration` (cold start time), and `Errors` in CloudWatch.

**2. Cold start increase**:
- `InitDuration` appearing in CloudWatch Logs (`REPORT` line) indicates a cold start. A spike in cold starts happens when:
  - Traffic increases rapidly and Lambda needs to scale out many new containers.
  - Function was redeployed (all existing containers are recycled).
  - Lambda was idle for a period and containers were reclaimed.
- **Diagnosis**: filter CloudWatch Logs Insights for `InitDuration`:
  ```
  filter @type = "REPORT"
  | stats avg(@initDuration), count(@initDuration) by bin(5m)
  ```
- **Fix**: enable **Provisioned Concurrency** for latency-sensitive functions; use **Lambda SnapStart** (Java); increase memory (larger memory = larger container class = sometimes faster init).

**3. Downstream dependency slowness**:
This is the most common cause of increased duration on previously stable functions:
- **Database**: RDS or Aurora connection latency increased. Symptoms: function logs show long DB query times. Fix: use RDS Proxy to avoid connection overhead; check RDS `ReadLatency`/`WriteLatency` metrics.
- **External API**: third-party API response time degraded. Add a per-request timeout in your HTTP client shorter than the Lambda timeout to fail fast.
- **S3 / DynamoDB**: check service health dashboard (`https://health.aws.amazon.com/`) for regional degradations.
- **Secrets Manager**: `GetSecretValue` called on every invocation instead of caching the secret in the Lambda execution environment (outside the handler):
  ```python
  # Bad: called on every invocation
  def handler(event, context):
      secret = get_secret()   # network call every time

  # Good: cached across warm invocations
  secret = get_secret()       # called once per container lifetime
  def handler(event, context):
      use(secret)
  ```

**4. Memory and CPU exhaustion**:
- Lambda CPU allocation is proportional to memory. A function at 128 MB has very limited CPU. If the workload grew (larger payloads, more data to process), the same memory setting will take longer.
- Check `max memory used` in the `REPORT` log line. If it is close to the configured memory limit, increase memory.
  ```
  filter @type = "REPORT"
  | stats max(@maxMemoryUsed / 1024 / 1024) as maxMB, avg(@duration) by bin(5m)
  ```

**5. Concurrency throttling causing queuing**:
- If `Throttles` metric is > 0, invocations are queuing. Synchronous callers (API Gateway) get `429 TooManyRequests`; async callers (SQS, EventBridge) retry with backoff, increasing effective latency.
- **Fix**: request a concurrency limit increase via AWS Support; implement reserved concurrency per function to protect critical paths; use SQS as a buffer to smooth traffic spikes.

**6. VPC configuration**:
- Lambda in a VPC must create an ENI to communicate with VPC resources. As of 2019, AWS pre-warms ENIs, but if the function's subnet runs out of available IP addresses, new containers stall waiting for an ENI.
  ```bash
  aws ec2 describe-subnets --subnet-ids subnet-xxxxx \
    --query 'Subnets[].AvailableIpAddressCount'
  ```
  If `AvailableIpAddressCount` is 0 or near 0, the subnet is exhausted. Add a larger CIDR or move the function to a subnet with more IPs.

**7. Runtime or dependency changes**:
- A recent deployment that added a heavier library (e.g., replacing a lightweight HTTP client with a full SDK) increases init time and per-invocation overhead.
- Review the deployment history and compare `InitDuration` before and after the deploy.

**Summary of common causes and fixes**:

| Cause | Signal | Fix |
|---|---|---|
| Cold start spike | `InitDuration` in logs | Provisioned Concurrency, SnapStart, keep-warm |
| DB connection latency | Long DB call in logs | RDS Proxy, connection reuse |
| External API slow | HTTP call duration in logs | Per-call timeout, circuit breaker |
| Memory too low / CPU starved | `maxMemoryUsed` near limit | Increase memory allocation |
| Throttling | `Throttles > 0` | Increase concurrency limit; use SQS buffer |
| Subnet IP exhaustion | ENI creation stall | Expand subnet CIDR or use a larger subnet |
| Heavier deployment | `InitDuration` increased post-deploy | Audit new dependencies; use Lambda layers |

---
