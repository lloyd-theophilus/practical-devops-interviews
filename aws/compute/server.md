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
