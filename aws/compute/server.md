# AWS Compute Interview Questions & Answers

---

**Q: Your application running on EC2 suddenly starts failing health checks behind an ALB. How would you troubleshoot the issue step by step?**

**A:**
1. **Check ALB target group**: AWS Console → EC2 → Target Groups → select the TG → Targets tab. Look at the health status reason for each instance (e.g., "Request timed out", "Connection refused", "HTTP 5xx").
2. **Check ALB access logs**: enable and review logs in S3 for `target_status_code` and `target_processing_time` on the failing target.
3. **CloudWatch metrics**: check `UnHealthyHostCount`, `TargetResponseTime`, and `HTTPCode_Target_5XX_Count`.
4. **SSH to the EC2 instance**: test the health check endpoint locally:
   ```bash
   curl -v http://localhost:<port>/health
   ```
   If it succeeds locally but fails from ALB, the issue is network-level (security group, NACL). If it fails locally, the application is down.
5. **Application logs**: `journalctl -u myapp -n 200` or check `/var/log/app/`. Look for OOM, startup errors, DB connection failures.
6. **Security groups**: verify the ALB security group is allowed as an inbound source in the EC2 security group on the health check port.
7. **Resource exhaustion**: `top`, `df -h`, `free -m` — check CPU, disk, and memory. A full disk or OOM can cause the app to crash.
8. **Recent changes**: check deploy history, AMI change, user-data script changes, or security group rule modifications.

---

**Q: If an S3 bucket accidentally gets deleted, how would you recover the data?**

**A:**
- **Versioning enabled**: S3 versioning adds a delete marker on deletion. Recover by removing the delete marker:
  ```bash
  aws s3api delete-object --bucket mybucket --key myfile.txt --version-id <delete-marker-version-id>
  ```
  Or restore all objects by removing all delete markers with a script.
- **S3 Replication**: if cross-region or same-region replication is configured, the data exists in the destination bucket.
- **AWS Backup**: if the bucket was backed up via AWS Backup, restore from the recovery point.
- **Versioning + MFA Delete**: the strongest protection — deletion of versioned objects requires MFA. If MFA Delete was enabled, the bucket can't have been deleted without MFA.
- **If no protection was in place**: file an AWS Support ticket immediately (Premium Support can sometimes recover recently deleted S3 data within hours). No guarantees.
- **Prevention**: enable versioning + MFA Delete, use an S3 bucket policy denying `s3:DeleteBucket` and `s3:DeleteObject` except for specific roles, enable Object Lock (WORM) for critical data.

---

**Q: Your Lambda function is timing out frequently. How do you troubleshoot and optimize it?**

**A:**
**Troubleshoot**:
1. CloudWatch Logs: look for "Task timed out after X seconds" — check what the function was doing at that point.
2. X-Ray tracing: enable active tracing (`aws lambda update-function-configuration --tracing-config Mode=Active`); view the trace to see which segment (DB call, external API, S3) is slow.
3. CloudWatch metrics: `Duration`, `Throttles`, `ConcurrentExecutions`.

**Common causes & fixes**:
- **Database connection**: Lambda creates a new DB connection on cold start. Use RDS Proxy to pool connections. Set `connect_timeout` lower than the Lambda timeout.
- **External API slow**: add a per-call timeout in your HTTP client shorter than the Lambda timeout; implement retries with exponential backoff.
- **Cold start**: increase memory (also increases CPU), use Provisioned Concurrency for latency-sensitive functions, use Lambda SnapStart for Java.
- **VPC cold start**: Lambda in a VPC now uses pre-warmed ENIs (fixed in 2019), but verify you're in a subnet with available IPs and the security group allows outbound.
- **Timeout too low**: simply increase `timeout` in the Lambda config (max 15 minutes) if the work genuinely takes longer.
- **Synchronous downstream calls**: parallelize independent calls using `asyncio.gather()` (Python) or `Promise.all()` (Node.js).

---

**Q: How do you securely store secrets for applications running on EC2 or Lambda?**

**A:**
- **AWS Secrets Manager**: store database credentials, API keys. Rotate automatically. Retrieve at runtime:
  ```python
  import boto3, json
  client = boto3.client('secretsmanager')
  secret = json.loads(client.get_secret_value(SecretId='prod/myapp/db')['SecretString'])
  ```
  Grant access via the EC2 instance profile IAM role or Lambda execution role — no credentials in code.
- **AWS Systems Manager Parameter Store**: for non-sensitive config (SecureString type for secrets, encrypted with KMS). Lower cost than Secrets Manager for high-volume reads.
- **IAM roles**: EC2 instances get credentials from the instance metadata service (IMDS); Lambda gets credentials from the execution role. Never hardcode `AWS_ACCESS_KEY_ID` in code.
- **Never**: put secrets in environment variables visible in the console (use Secrets Manager references instead), in source code, AMI baked-in, or CloudFormation templates without `NoEcho`.
- **IMDSv2**: enforce IMDSv2 (`HttpTokens=required`) on EC2 to prevent SSRF attacks from stealing instance credentials.

---

**Q: Explain how you'd secure cross-region S3 replication and validate data integrity at scale.**

**A:**
**Securing replication**:
- Enable S3 Replication with SSE-KMS using a customer-managed KMS key in each region. The replication IAM role needs `kms:GenerateDataKey` and `kms:Decrypt` on both source and destination keys.
- Source bucket policy: deny `s3:PutObject` without SSE-KMS encryption header — ensures all objects are encrypted before replication.
- Destination bucket: Block Public Access enabled; separate bucket policy restricting access to specific roles.
- Replicate delete markers selectively — consider NOT replicating delete markers to avoid accidental deletion propagation.

**Validating data integrity**:
- **S3 Checksum**: enable object checksum (SHA-256, CRC32C) at upload time. S3 stores and validates it on retrieval:
  ```bash
  aws s3api put-object --bucket mybucket --key myfile --checksum-algorithm SHA256 --body myfile
  ```
- **S3 Replication metrics**: enable replication metrics and events; CloudWatch alarms on `ReplicationLatency` and `BytesPendingReplication`.
- **S3 Inventory + Batch Operations**: generate S3 Inventory reports for both source and destination; use a Lambda/Athena job to compare ETags and object counts to detect missing or corrupted objects.
- **S3 Object Lock**: for regulatory compliance, enable WORM protection on the destination bucket.

---

**Q: How to troubleshoot SSH issues in an EC2 instance?**

**A:**
```bash
# Step 1: Verify the instance is running
aws ec2 describe-instance-status --instance-ids i-xxxxx

# Step 2: Check security group allows port 22 from your IP
aws ec2 describe-security-groups --group-ids sg-xxxxx

# Step 3: Verify the NACL allows SSH (port 22) inbound and ephemeral ports (1024-65535) outbound

# Step 4: Check if the instance has a public IP or use bastion/SSM

# Step 5: Try SSH with verbose output
ssh -vvv -i ~/.ssh/mykey.pem ec2-user@<ip>

# Step 6: Check instance system log for boot errors
aws ec2 get-console-output --instance-id i-xxxxx --latest
```

**If you can't SSH at all, use AWS Systems Manager Session Manager** (no port 22 needed):
```bash
aws ssm start-session --target i-xxxxx
```

Common SSH issues: wrong key pair, wrong username (`ec2-user` for Amazon Linux, `ubuntu` for Ubuntu, `centos` for CentOS), security group missing port 22 inbound, instance in private subnet without bastion/VPN, SSHd crashed (check console output).

---

**Q: What is the difference between EBS, S3, and EFS?**

**A:**

| | EBS | S3 | EFS |
|---|---|---|---|
| Type | Block storage | Object storage | Network file system (NFS) |
| Access | Single EC2 (or multi-attach for io1/io2) | Any HTTP client; multi-region | Multiple EC2 instances simultaneously |
| Protocol | Block device (ext4, xfs) | REST API (PUT/GET/DELETE) | NFS v4.1 |
| Persistence | Attached to instance lifecycle (can persist after instance stop) | Independent, highly durable (11 9s) | Independent, auto-scales |
| Performance | High IOPS (io2: 64,000 IOPS) | High throughput, higher latency than block | Throughput scales with storage size |
| Use case | OS boot volumes, databases, single-instance workloads | Backups, static assets, data lake, logs | Shared home dirs, CMS, lift-and-shift NFS workloads |
| Cost model | Provisioned (GB/month) | Pay per GB stored + requests | Pay per GB stored (no provisioning) |

---

**Q: How does IAM policy differ from IAM role?**

**A:**
- **IAM Policy**: a JSON document that defines permissions (Allow/Deny actions on resources). Policies are attached to identities (users, groups, roles) or resources (S3 bucket policy).
- **IAM Role**: an identity (like a user) that doesn't have permanent credentials. It's assumed by trusted entities (EC2, Lambda, another AWS account, OIDC identity provider) temporarily. When assumed, it provides short-lived credentials (STS).

**Relationship**: a role has policies attached that define what the role can do. An EC2 instance assumes a role via an instance profile — the application on the instance then has the permissions from the role's attached policies without any hardcoded keys.

---

**Q: Difference between security group and NACL?**

**A:**

| | Security Group | NACL |
|---|---|---|
| Level | Instance (ENI) level | Subnet level |
| State | Stateful (return traffic auto-allowed) | Stateless (must explicitly allow both directions) |
| Rules | Allow only (no explicit deny) | Allow and Deny rules |
| Rule evaluation | All rules evaluated; most permissive wins | Rules evaluated in order (lowest number first); first match wins |
| Default | Deny all inbound, allow all outbound | Allow all inbound and outbound |
| Scope | Specific instances | All instances in the subnet |

Use security groups as the primary access control; use NACLs for subnet-wide blacklisting (e.g., blocking a malicious IP range).

---

**Q: How to set up auto-scaling for an application?**

**A:**
**EC2 Auto Scaling**:
1. Create a Launch Template (AMI, instance type, security group, IAM profile, user data).
2. Create an Auto Scaling Group (ASG) with the launch template; specify min/max/desired capacity and VPC subnets.
3. Attach a scaling policy:
   - **Target tracking**: "maintain average CPU at 50%" — AWS manages scale-out/in automatically.
   - **Step scaling**: scale out by 2 instances if CPU > 70% for 5 min; scale in by 1 if < 30%.
   - **Scheduled**: scale up at 8 AM, scale down at 8 PM for predictable patterns.
4. Attach the ASG to an ALB target group for load balancing.
5. Set proper health check (ELB health checks > EC2 health checks) and grace period.

**For Kubernetes**: use Cluster Autoscaler (scales EC2 nodes) + HPA (scales pods).

---

**Q: How do you manage and connect services like DBs, EC2, EKS, or ECS? Include the command to connect to ECS.**

**A:**
- **EC2 → RDS**: use Security Groups (allow port 5432/3306 from EC2 SG to RDS SG), connect via the RDS endpoint. Use IAM authentication for passwordless connection.
- **EKS → RDS**: use IRSA + IAM authentication, or mount credentials from Secrets Manager via the CSI driver.
- **ECS → RDS**: ECS task IAM role + Secrets Manager.

**Connect to ECS container**:
```bash
# ECS Exec (requires SSM agent in container and ECS Exec enabled)
aws ecs execute-command \
  --cluster my-cluster \
  --task <task-id> \
  --container my-container \
  --interactive \
  --command "/bin/sh"
```

Enable ECS Exec on the service: `aws ecs update-service --cluster my-cluster --service my-service --enable-execute-command`.

---

**Q: How do you create AWS Lambda functions and manage the artifacts for deployment?**

**A:**
```bash
# Package
zip function.zip lambda_function.py

# Deploy via S3 (recommended for large functions)
aws s3 cp function.zip s3://my-artifacts-bucket/functions/myapp/v1.2.0.zip
aws lambda update-function-code \
  --function-name my-function \
  --s3-bucket my-artifacts-bucket \
  --s3-key functions/myapp/v1.2.0.zip

# Or directly (< 50MB)
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# Container image (for large runtimes)
docker push my-account.dkr.ecr.us-east-1.amazonaws.com/my-lambda:v1.2.0
aws lambda update-function-code \
  --function-name my-function \
  --image-uri my-account.dkr.ecr.us-east-1.amazonaws.com/my-lambda:v1.2.0
```

In CI/CD: use AWS SAM CLI (`sam deploy`) or the Serverless Framework for infrastructure-as-code deployment of Lambda functions including event source mappings, IAM roles, and layers.

---

**Q: You have an application in Account A that needs to access an S3 bucket in Account B. How would you configure this?**

**A:**
1. **In Account B — S3 bucket policy**: grant Account A's role access:
   ```json
   {
     "Effect": "Allow",
     "Principal": {
       "AWS": "arn:aws:iam::ACCOUNT_A_ID:role/MyAppRole"
     },
     "Action": ["s3:GetObject", "s3:PutObject"],
     "Resource": "arn:aws:s3:::account-b-bucket/*"
   }
   ```
2. **In Account A — IAM role**: `MyAppRole` must also have an IAM policy allowing `s3:GetObject`/`s3:PutObject` on the Account B bucket ARN (both bucket policy AND IAM policy must allow for cross-account access).
3. The EC2/Lambda/ECS in Account A assumes `MyAppRole` via instance profile or execution role — it can now access the Account B bucket using its temporary credentials.
4. **KMS**: if the bucket uses SSE-KMS with an Account B key, add Account A's role to the key policy as well.

---

**Q: Your EC2 instance in a private subnet needs to download packages without NAT Gateway. What alternatives exist?**

**A:**
- **VPC Endpoints (Gateway type)**: free for S3 and DynamoDB. Route table entry directs S3 traffic through the endpoint without leaving the AWS network.
- **VPC Interface Endpoints (PrivateLink)**: for other AWS services (EC2 API, ECR, Secrets Manager, SSM). Creates an ENI in your subnet; traffic stays within VPC. Costs per hour + per GB.
- **AWS Systems Manager (SSM)**: for package installation via `yum`/`apt` on Amazon Linux instances — SSM can run commands without internet access if the instance has SSM, EC2Messages, and SSMMessages interface endpoints.
- **S3-hosted yum/apt mirror**: create an S3 bucket with yum/apt packages; configure the instance to use it as a local mirror.
- **AWS CodeArtifact**: hosted package repository for npm, pip, Maven — accessible via VPC endpoint.
- **Self-hosted Nexus/Artifactory**: in a VPC-accessible subnet, proxies package repositories internally.

---

**Q: Your EC2 instance is unreachable — how do you diagnose?**

**A:**
```bash
# Step 1: Check instance state
aws ec2 describe-instances --instance-ids i-xxxxx --query 'Reservations[].Instances[].State'

# Step 2: Check instance status checks
aws ec2 describe-instance-status --instance-ids i-xxxxx

# Step 3: Get console output (boot log)
aws ec2 get-console-output --instance-id i-xxxxx --latest

# Step 4: Check security groups and NACLs for port 22 / 443 (SSM)

# Step 5: Use SSM Session Manager (bypasses SSH entirely)
aws ssm start-session --target i-xxxxx

# Step 6: If SSM fails, use EC2 Serial Console (Nitro instances)
aws ec2 enable-serial-console-access
aws ec2 send-serial-console-ssh-public-key --instance-id i-xxxxx --serial-port 0 --ssh-public-key file://mykey.pub
```

Common causes: instance status check failed (hardware issue → stop/start to move to new host), OS crash (check console output for kernel panic), security group/NACL blocking access, application consuming 100% CPU (instance unresponsive), disk full causing SSH daemon to fail, SSHd service crashed.

---

**Q: You launched an EC2 instance but cannot access it. What will you check first?**

**A:**
Check in this order:

1. **Instance state**: confirm the instance is in `running` state (not `pending`, `stopped`, or `impaired`):
   ```bash
   aws ec2 describe-instance-status --instance-ids i-xxxxx --include-all-instances
   ```
   Both **System Status Check** and **Instance Status Check** must be passing.

2. **Security group**: verify port 22 (SSH) or 3389 (RDP) is open inbound from your IP:
   ```bash
   aws ec2 describe-security-groups --group-ids sg-xxxxx \
     --query 'SecurityGroups[].IpPermissions'
   ```

3. **Public IP / DNS**: check the instance has a public IP (or Elastic IP if in a public subnet). Instances in private subnets require a bastion or VPN.

4. **Key pair**: confirm you are using the correct `.pem` file and the right OS username (`ec2-user` for Amazon Linux, `ubuntu` for Ubuntu, `admin` for Debian, `centos` for CentOS).

5. **NACL**: network ACLs are stateless — verify both inbound (port 22) and outbound (ephemeral ports 1024–65535) rules allow traffic.

6. **Route table**: ensure the public subnet's route table has `0.0.0.0/0 → igw-xxxxx` (Internet Gateway). Missing this is a common oversight.

7. **Console output**: if all networking looks correct, check if the OS booted successfully:
   ```bash
   aws ec2 get-console-output --instance-id i-xxxxx --latest
   ```
   Look for kernel panics, SSH daemon errors, or disk full messages.

8. **SSM fallback**: if SSH is blocked or broken, use Session Manager — no port 22 needed:
   ```bash
   aws ssm start-session --target i-xxxxx
   ```

---

**Q: Auto Scaling is creating instances repeatedly. What could be happening?**

**A:**
This is a **scale-out loop** — new instances are launched, fail health checks, get terminated, and trigger another scale-out. Common causes:

1. **Instances failing health checks immediately after launch**:
   - ALB health check path (`/health`) returns non-2xx because the application hasn't started yet → increase `health-check-grace-period` to allow full app startup.
   - Wrong health check port or path configured on the target group.
   - Application crashes on startup (missing env vars, secrets, DB connection failure) → check `/var/log` and `journalctl`.

2. **Min capacity set too high with a resource constraint**: if desired > available capacity in the subnet (no IP addresses left, instance type capacity not available in the AZ) → ASG keeps retrying.

3. **Launch template / AMI misconfiguration**: the user-data script fails, leaving the instance in a broken state → check `cloud-init-output.log` on the instance.

4. **Termination protection disabled + unhealthy threshold too low**: instances marked unhealthy too aggressively (e.g., 2 consecutive failures with a 10-second interval = 20 seconds) before the app can warm up.

5. **Scale-out cooldown too short**: a previous scale-out fires, instances spin up, CPU temporarily spikes again before stabilization → increase `ScaleOutCooldown`.

**Diagnosis**:
```bash
# Check ASG activity history — shows why instances were terminated
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name my-asg \
  --max-items 20

# Check target group health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/myapp/xxxx
```

**Fix**: set `health-check-grace-period` to at least 2x your app startup time; fix the root cause in the application; temporarily set `min-size 0` during debugging if the loop is costing money.

---

**Q: CPU utilization suddenly reaches 95% in production. What will you investigate?**

**A:**

**Immediate triage (first 5 minutes)**:

1. **Check which instance(s)**: CloudWatch → EC2 → per-instance `CPUUtilization`. Is it one instance or the whole fleet? If one instance, isolate it.

2. **Check for traffic spike**: ALB → `RequestCount` and `ActiveConnectionCount` metrics. A sudden traffic surge is the most common cause.

3. **SSH/SSM onto the hot instance** and identify the offending process:
   ```bash
   top -b -n 1 | head -20          # top CPU processes
   ps aux --sort=-%cpu | head -10  # sorted by CPU
   pidstat -u 1 5                  # per-process CPU over time
   ```

4. **Application logs**: look for error storms (e.g., repeated exception stack traces in a loop), slow queries generating retries, or a cron job that fired unexpectedly.

5. **Memory pressure causing swap**: high swap usage forces the kernel to do extra I/O which drives up CPU (soft interrupt). Check `free -m` and `vmstat 1 5`.

**Common root causes**:
- Sudden traffic increase (organic or bot/DDoS) — verify with WAF and ALB access logs.
- Memory leak causing GC pressure (Java/Go) — check heap metrics in APM.
- Runaway process or cron job (e.g., a backup/compression job running at peak hours).
- Inefficient database query doing a full table scan — check RDS `SlowQueryLog` or Performance Insights.
- Dependency slowness causing threads to pile up waiting — check thread/connection pool metrics.

**Response**:
- If traffic spike: scale out the ASG immediately (`aws autoscaling set-desired-capacity ...`).
- If runaway process: `kill -15 <pid>` gracefully; investigate before restarting.
- Long-term: set a CloudWatch alarm at 80% CPU; use target-tracking Auto Scaling to scale before saturation.

---

**Q: Application works with IP but fails using domain name. What will you verify?**

**A:**
This is a DNS resolution issue. Work through it systematically:

1. **Verify DNS resolution from the failing host**:
   ```bash
   nslookup myapp.example.com
   dig myapp.example.com +short
   # Compare the resolved IP to the expected IP
   ```

2. **Check Route 53 / DNS provider record**:
   - Is the A/CNAME record pointing to the correct IP, ALB DNS name, or CloudFront distribution?
   - Is the TTL very high? Old cached records from before an IP change can persist.
   ```bash
   aws route53 list-resource-record-sets \
     --hosted-zone-id ZXXXXX \
     --query "ResourceRecordSets[?Name=='myapp.example.com.']"
   ```

3. **DNS propagation**: after a DNS change, old records may be cached at recursive resolvers. Check from multiple locations:
   ```bash
   dig myapp.example.com @8.8.8.8     # Google's resolver
   dig myapp.example.com @1.1.1.1     # Cloudflare's resolver
   ```

4. **SSL/TLS certificate**: if the domain resolves correctly but HTTPS fails, the cert may not cover the domain (check `curl -v https://myapp.example.com` and inspect the `CN`/`SAN` fields).

5. **VPC DNS settings** (for internal services): ensure `enableDnsSupport` and `enableDnsHostnames` are `true` on the VPC. Private hosted zone must be associated with the VPC.

6. **Hosts file override**: check if `/etc/hosts` on the application server has a stale override entry for the domain.

7. **ALB or CloudFront alias record**: if using Route 53 alias to an ALB, verify the ALB is in the correct region and the alias target matches. A deleted/recreated ALB gets a new DNS name that the alias must be updated to point to.

8. **Security groups on port 443/80**: confirm that when accessing via domain (which may resolve to a different IP than the direct EC2 IP), traffic is not hitting a different resource or being blocked by a SG rule.

---

**Q: CloudWatch alarms are not triggering. What will you verify?**

**A:**
Work through the alarm pipeline end-to-end:

1. **Alarm state**: check the alarm's current state — `OK`, `ALARM`, or `INSUFFICIENT_DATA`.
   ```bash
   aws cloudwatch describe-alarms --alarm-names "my-cpu-alarm"
   ```
   `INSUFFICIENT_DATA` means CloudWatch isn't receiving metric data — the metric itself is missing, not a threshold issue.

2. **Metric data is flowing**:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/EC2 \
     --metric-name CPUUtilization \
     --dimensions Name=InstanceId,Value=i-xxxxx \
     --start-time 2024-01-01T00:00:00Z \
     --end-time 2024-01-01T01:00:00Z \
     --period 300 \
     --statistics Average
   ```
   If no data points are returned, the metric is not being reported (e.g., instance stopped, CloudWatch agent not running for custom metrics).

3. **Alarm configuration**:
   - **Period and evaluation periods**: if the period is 5 minutes and `evaluation-periods` is 3, the condition must persist for 15 minutes before triggering. Verify this matches your expectation.
   - **Threshold**: confirm the threshold and comparison operator are set correctly (`GreaterThanOrEqualToThreshold`, not `GreaterThanThreshold`).
   - **Treat missing data**: if set to `missing` (default), a missing data point is not treated as a breach. Change to `breaching` if you want missing data to trigger.

4. **SNS topic / action**:
   - Verify the alarm has an action (`AlarmActions`, `OKActions`) configured.
   - Check the SNS topic exists and has an active subscription (the email address confirmed the subscription link).
   - Check SNS delivery status in CloudWatch: `NumberOfNotificationsFailed` on the SNS topic.

5. **IAM permissions**: CloudWatch must be able to publish to the SNS topic. The SNS topic policy must allow `sns:Publish` from `cloudwatch.amazonaws.com`.

6. **Custom metrics (CloudWatch agent)**: if the alarm is on a custom metric, verify the CloudWatch agent is running and configured correctly on the EC2 instance:
   ```bash
   systemctl status amazon-cloudwatch-agent
   cat /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
   ```

7. **Cross-account or cross-region**: alarms on cross-account or cross-region metrics require CloudWatch cross-account observability to be configured.

---
