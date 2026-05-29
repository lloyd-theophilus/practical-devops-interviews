# Terraform Interview Questions & Answers

---

**Q: How does Terraform dependency graph (DAG) work internally?**

**A:** Terraform builds a Directed Acyclic Graph (DAG) of all resources defined in configuration files. Edges in the DAG represent dependencies:
- **Explicit**: `depends_on` meta-argument.
- **Implicit**: reference to another resource's attribute (e.g., `aws_subnet.main.id` in an EC2 resource).

During `terraform plan`/`apply`:
1. Terraform parses all `.tf` files and builds the resource graph.
2. It performs a topological sort — resources with no dependencies are processed first (in parallel up to `-parallelism=N`, default 10).
3. Resources downstream in the graph wait for their dependencies to be created/updated first.
4. For `destroy`, the graph is traversed in reverse order.

`terraform graph | dot -Tsvg > graph.svg` renders the full dependency graph visually.

---

**Q: How does Terraform handle state locking and consistency?**

**A:** State locking prevents concurrent `plan`/`apply` operations from corrupting the state file.

- **S3 backend + DynamoDB**: S3 stores the state file; DynamoDB provides locking via a `LockID` item. Before any write operation, Terraform attempts to create/update the lock item. If it exists, the operation fails with a lock error (includes the lock owner's identity and timestamp).
- **Terraform Cloud/Enterprise**: uses its own locking mechanism with run queues.
- **Azure Blob**: uses blob lease-based locking.

If a lock is orphaned (crash mid-apply), use `terraform force-unlock <LOCK_ID>` — only after confirming no other operation is running.

---

**Q: Terraform remote_state backend suddenly times out. What's your recovery and damage containment strategy?**

**A:**
1. **Immediate**: Stop any running `apply` to avoid partial state writes. Check if an apply is in progress via DynamoDB lock table or Terraform Cloud run queue.
2. **Diagnose**: Check S3 bucket availability, DynamoDB endpoint reachability, VPC endpoint health (if using private endpoints), and IAM permissions.
3. **Fallback**: Switch to a local backend temporarily:
   ```hcl
   terraform init -reconfigure -backend-config="path=./terraform.tfstate"
   ```
   Pull the last known state: `terraform state pull > terraform.tfstate`.
4. **Restore**: Fix the backend issue (S3 throttling, DynamoDB capacity, network), re-init with the remote backend.
5. **Prevent recurrence**: Enable S3 versioning for state recovery, set DynamoDB on-demand capacity to avoid throttling, use VPC endpoints for private access.

---

**Q: Your Terraform state got corrupted during a backend migration. Rebuild strategy?**

**A:**
1. **Retrieve the last good state** from S3 versioning: `aws s3api list-object-versions --bucket tf-state-bucket --prefix myapp/terraform.tfstate`.
2. Restore: `aws s3api get-object --bucket tf-state-bucket --key myapp/terraform.tfstate --version-id <good-version> terraform.tfstate`.
3. Push the restored state: `terraform state push terraform.tfstate`.
4. Run `terraform plan` to verify the diff between the restored state and actual infrastructure.
5. If state is unrecoverable: use `terraform import` for each resource to rebuild the state from scratch by importing existing infrastructure.
6. **Lesson**: always enable S3 versioning and test state migrations in a lower environment first using `terraform state mv` rather than raw backend changes.

---

**Q: How does Terraform maintain the state of resources?**

**A:** Terraform uses a JSON state file (`terraform.tfstate`) to map configuration resources to real-world infrastructure objects. The state contains:
- Each resource's provider, type, name, and unique ID.
- All attributes returned by the provider after creation.
- Dependency metadata.

On each `plan`, Terraform refreshes the state (reads current attributes from the provider API), compares them to desired configuration, and computes a diff. On `apply`, it updates the state after each resource operation.

Remote backends (S3, Terraform Cloud) store state centrally for team use.

---

**Q: What are Terraform modules?**

**A:** Modules are reusable, encapsulated units of Terraform configuration. A module is any directory with `.tf` files. There are two types:
- **Root module**: the working directory where you run Terraform.
- **Child module**: called from the root (or another module) via a `module {}` block.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"
  cidr    = "10.0.0.0/16"
  azs     = ["us-east-1a", "us-east-1b"]
}
```

Modules abstract complexity, enforce standards, and are published to the Terraform Registry. Internal company modules are stored in private registries or Git repos.

---

**Q: How to manage sensitive variables in Terraform?**

**A:**
- Mark variables as `sensitive = true` — Terraform redacts them in plan output and logs.
- Store secrets in environment variables: `export TF_VAR_db_password=secret`.
- Use AWS Secrets Manager or HashiCorp Vault with the Vault provider — fetch secrets at apply time, never hardcode.
- **Never** commit `terraform.tfvars` containing secrets to version control. Add `*.tfvars` to `.gitignore`.
- Use SOPS-encrypted vars files for GitOps workflows.

---

**Q: What is the purpose of terraform validate and terraform fmt?**

**A:**
- `terraform validate`: checks the configuration for syntactic correctness and internal consistency (e.g., invalid argument names, missing required attributes) **without** contacting the provider API. Fast and useful in CI before `plan`.
- `terraform fmt`: formats `.tf` files to the canonical HCL style (indentation, alignment of `=` signs). `-check` flag returns a non-zero exit code if any file needs formatting — use in CI to enforce style consistency.

Both are lightweight and should run in every pipeline before `terraform plan`.

---

**Q: How do you handle provisioning in different environments (dev/stage/prod)?**

**A:** Common approaches:
1. **Workspaces**: `terraform workspace new dev/staging/prod` — separate state per workspace; single config with `terraform.workspace` conditionals. Good for small teams; state files are in the same bucket prefix.
2. **Directory-per-environment**: `environments/dev/`, `environments/staging/`, `environments/prod/` each with their own `main.tf` and `terraform.tfvars`, calling shared modules. Cleaner separation, preferred for complex setups.
3. **Terragrunt**: DRY wrapper that generates backend config and calls modules with environment-specific inputs from `terragrunt.hcl` files.

Use separate AWS accounts per environment (AWS Organizations) with separate state buckets for strong blast-radius isolation.

---

**Q: You need to import an existing AWS VPC into Terraform. What are the steps?**

**A:**
1. Write the resource block in your config:
   ```hcl
   resource "aws_vpc" "main" {
     cidr_block = "10.0.0.0/16"
   }
   ```
2. Run the import command:
   ```bash
   terraform import aws_vpc.main vpc-0abc123def456
   ```
3. Terraform updates the state file with the VPC's current attributes.
4. Run `terraform plan` — if there are diffs (e.g., tags not in config), add them to the config until the plan shows no changes.
5. Repeat for all associated resources (subnets, route tables, IGW, security groups).

For bulk imports (Terraform 1.5+), use `import {}` blocks in configuration for declarative imports, and `terraform plan` will preview the import.

---

**Q: How do you manage secrets in Terraform without hardcoding them?**

**A:**
- **Environment variables**: `TF_VAR_<name>` — Terraform picks them up without any code change.
- **AWS Secrets Manager** (via data source):
  ```hcl
  data "aws_secretsmanager_secret_version" "db" {
    secret_id = "prod/myapp/db"
  }
  locals {
    db_password = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)["password"]
  }
  ```
- **HashiCorp Vault provider**: fetch dynamic credentials at apply time.
- **SOPS + age/KMS**: encrypt `secrets.tfvars`; decrypt at CI runtime.
- Mark all sensitive outputs and variables with `sensitive = true`.

---

**Q: How would you implement cross-account resource provisioning using Terraform?**

**A:** Use multiple provider configurations with `assume_role`:
```hcl
provider "aws" {
  alias  = "account_a"
  region = "us-east-1"
}

provider "aws" {
  alias  = "account_b"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformCrossAccountRole"
  }
}

resource "aws_s3_bucket" "shared" {
  provider = aws.account_b
  bucket   = "cross-account-shared-bucket"
}
```

The `TerraformCrossAccountRole` in Account B must have a trust policy allowing Account A's Terraform execution role to assume it. CI/CD runners in Account A assume this role via STS.

---

**Q: An S3 bucket was created via Terraform, but someone manually added a policy. How do you handle this drift?**

**A:**
1. Run `terraform plan` — it will detect the manual policy as drift and plan to remove/replace it.
2. **Option A (Terraform wins)**: apply the plan to restore the desired state. Educate the team that manual changes to Terraform-managed resources are not allowed.
3. **Option B (Manual change wins)**: add the policy to the Terraform config, then apply — no drift, no resource destruction.
4. **Option C (Ignore)**: use `lifecycle { ignore_changes = [policy] }` if this attribute should be managed outside Terraform.
5. **Prevention**: use SCPs (Service Control Policies) or IAM permission boundaries to restrict manual changes to production resources. Set up AWS Config rules to detect drift and alert.

---

**Q: How do you recover from a deleted Terraform state file?**

**A:**
1. **Restore from S3 versioning** (if enabled — and it always should be):
   ```bash
   aws s3api list-object-versions --bucket tf-state --prefix app/terraform.tfstate
   aws s3api get-object --bucket tf-state --key app/terraform.tfstate --version-id <id> terraform.tfstate
   terraform state push terraform.tfstate
   ```
2. **If no backup exists**: rebuild the state file by importing each resource:
   ```bash
   terraform import aws_instance.web i-0abc123
   terraform import aws_vpc.main vpc-0def456
   ```
   This is tedious for large configs but restores Terraform's awareness of existing resources.
3. **Prevention**: enable S3 versioning + MFA Delete; restrict `s3:DeleteObject` and `s3:DeleteObjectVersion` via S3 bucket policy.

---

**Q: How do you create 50 EC2 instances with different configurations (dynamic blocks)?**

**A:**
```hcl
variable "instances" {
  type = map(object({
    instance_type = string
    ami           = string
    tags          = map(string)
  }))
}

resource "aws_instance" "servers" {
  for_each      = var.instances
  ami           = each.value.ami
  instance_type = each.value.instance_type
  tags          = merge(each.value.tags, { Name = each.key })

  dynamic "ebs_block_device" {
    for_each = lookup(each.value, "extra_volumes", [])
    content {
      device_name = ebs_block_device.value.device_name
      volume_size = ebs_block_device.value.size
    }
  }
}
```

For uniform instances with count:
```hcl
resource "aws_instance" "workers" {
  count         = 50
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.medium"
  tags          = { Name = "worker-${count.index + 1}" }
}
```

`for_each` is preferred over `count` when instances have unique configurations because adding/removing one instance doesn't shift the indexes of others.

---

**Q: Share your screen and write a terraform code to create a VPC with all its components and create an RDS database in VPC.**

**A:**
```hcl
# variables.tf
variable "region"      { default = "us-east-1" }
variable "db_password" { sensitive = true }

# main.tf
provider "aws" { region = var.region }

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "main-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "main-igw" }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "public-${count.index}" }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "private-${count.index}" }
}

resource "aws_eip" "nat" { domain = "vpc" }

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  tags          = { Name = "main-nat" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}

resource "aws_security_group" "rds" {
  vpc_id = aws_vpc.main.id
  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id
}

resource "aws_db_instance" "postgres" {
  identifier             = "main-postgres"
  engine                 = "postgres"
  engine_version         = "15.4"
  instance_class         = "db.t3.medium"
  allocated_storage      = 20
  db_name                = "appdb"
  username               = "dbadmin"
  password               = var.db_password
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  skip_final_snapshot    = false
  multi_az               = true
  storage_encrypted      = true
}

data "aws_availability_zones" "available" { state = "available" }
```

---

**Q: Explain how you’d detect drift in IaC-managed infrastructure (Terraform, Ansible) before it affects trading or portfolio systems.**

**A:** Drift in financial infrastructure is a compliance and stability risk — a manually changed security group rule or a misconfigured instance can trigger a trading halt or an audit finding. The goal is to catch drift continuously, not just at the next deploy.

**Terraform drift detection**:

1. **Scheduled `terraform plan` in CI (drift detection pipeline)**:
   ```hcl
   # .github/workflows/drift-detect.yml (runs every 30 minutes)
   - name: Detect drift
     run: |
       terraform init -backend-config=prod.hccl
       terraform plan -detailed-exitcode -out=plan.tfplan
       # Exit code 0 = no changes, 1 = error, 2 = drift detected
     continue-on-error: true

   - name: Alert on drift
     if: steps.detect.outputs.exitCode == ‘2’
     run: |
       # Post to Slack/PagerDuty with the plan diff
       terraform show -no-color plan.tfplan | \
         curl -X POST $SLACK_WEBHOOK -d @- ...
   ```

2. **`terraform plan -refresh-only`**: refreshes state from the provider API and shows only real-world vs state diffs (not config vs state). Faster for large configs where you only want to detect out-of-band changes.

3. **AWS Config Rules + Custom Rules**:
   - AWS Config continuously evaluates resource configurations against rules. Write custom Config rules that validate critical attributes (e.g., "all security groups on trading instances must only allow port 443 from known CIDR ranges").
   - A drift triggers a Config compliance finding → CloudWatch Events → Lambda → PagerDuty/Jira ticket.

4. **Terraform Sentinel (Enterprise) / OPA (Open Source)**:
   - Policy-as-code prevents drift from being introduced via Terraform itself (e.g., no security group with `0.0.0.0/0` on port 22 can ever be applied).

5. **AWS CloudTrail + Config Aggregator**:
   - CloudTrail records every API call that modifies infrastructure. Set EventBridge rules to alert on `AuthorizeSecurityGroupIngress`, `ModifyDBInstance`, `PutBucketPolicy` outside of the expected CI/CD role ARN.
   - For trading systems specifically: alert on any change to VPC route tables, security groups, or IAM policies outside of the change window.

**Ansible drift detection**:

1. **Ansible `--check` mode (dry-run)**:
   ```bash
   ansible-playbook site.yml --check --diff -i inventory/prod
   # Shows what would change without making changes
   # Schedule this as a cron job or CI pipeline
   ```

2. **Ansible Tower / AWX**: schedule a "drift detection" job template that runs the playbook in `--check` mode daily; reports show configuration deviations per host.

3. **`ansible-lint` + custom checks**: lint playbooks for non-idempotent tasks that could cause false drift readings.

**For trading/portfolio systems specifically**:
- Define a **critical resource list**: the specific RDS instances, EC2 instances, security groups, IAM roles, and VPC route tables that, if changed, could affect trading operations.
- Run drift detection on this subset every 5 minutes (not just daily).
- Any detected drift on this list triggers an immediate P1 alert — not just a Slack notification — regardless of business hours.
- Integrate drift findings with your change management system (ServiceNow, Jira): a drift finding that doesn’t match an open change ticket is automatically escalated to the security team as a potential unauthorized change.

**Q: Terraform apply changes unexpected resources. What will you review?**

**A:**

1. **Re-read the plan output carefully**: `terraform plan` shows exactly what will change. If the plan shows changes you didn't intend, the cause is in the diff between your config and the current state.

2. **Provider version drift**: a provider upgrade may have changed the default value of an attribute, causing Terraform to "want" to update a resource even though your config hasn't changed. Pin providers in `versions.tf`:
   ```hcl
   terraform {
     required_providers {
       aws = { source = "hashicorp/aws", version = "~> 5.0" }
     }
   }
   ```

3. **Out-of-band manual changes (drift)**: someone changed the resource in the console. `terraform plan -refresh-only` shows only the drift. Decide: accept it into config, or let Terraform overwrite it.

4. **`for_each` / `count` index shift**: adding or removing an item in the middle of a `count`-indexed list shifts all subsequent resource addresses — Terraform plans to delete and recreate them. Switch to `for_each` with a stable key to avoid this.

5. **`ignore_changes` removed**: if `lifecycle { ignore_changes = [...] }` was removed from a resource that has out-of-sync attributes, Terraform will now plan to "fix" them.

6. **Data source refreshed with new results**: a `data` source returned different results (e.g., latest AMI ID changed), causing a resource referencing it to be replaced.

7. **Workspace mismatch**: you're in the wrong workspace and seeing a different environment's state.

8. **Module version bump**: a module upgrade changed default attribute values or resource configurations.

**Safe workflow**: always run `terraform plan -out=planfile`, review the plan output line by line, and only `terraform apply planfile` after explicit review.

---

**Q: Write a Terraform configuration to create an EC2 instance.**

**A:**
```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# main.tf
provider "aws" {
  region = var.region
}

variable "region"        { default = "us-east-1" }
variable "instance_type" { default = "t3.micro" }
variable "key_name"      { description = "EC2 key pair name" }

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTP and SSH"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]   # tighten to your IP in production
  }
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  key_name               = var.key_name
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    yum install -y httpd
    systemctl enable --now httpd
    echo "<h1>Hello from Terraform</h1>" > /var/www/html/index.html
  EOF

  tags = { Name = "web-server", ManagedBy = "terraform" }
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

---

**Q: What would you do if the Terraform state file is corrupted?**

**A:** State corruption (as opposed to deletion) usually means the JSON file is malformed — a partial write during a crash, a truncated file, or a binary overwrite.

**Step 1 — Don't run any Terraform commands yet.** A corrupted state + `terraform apply` = potential infrastructure destruction.

**Step 2 — Restore from S3 versioning** (the standard recovery path):
```bash
# List versions
aws s3api list-object-versions \
  --bucket my-tf-state-bucket \
  --prefix myapp/terraform.tfstate \
  --query 'Versions[].{VersionId:VersionId,LastModified:LastModified}' \
  --output table

# Download the last known good version
aws s3api get-object \
  --bucket my-tf-state-bucket \
  --key myapp/terraform.tfstate \
  --version-id <GOOD_VERSION_ID> \
  recovered.tfstate

# Validate it's valid JSON
python3 -m json.tool recovered.tfstate > /dev/null && echo "Valid JSON"

# Push it back
terraform state push recovered.tfstate
```

**Step 3 — If the file is partially corrupt (valid JSON but wrong data)**:
```bash
# Inspect the raw state
terraform show -json | jq '.values.root_module.resources[].address'

# If specific resources are corrupt, remove and re-import them
terraform state rm aws_instance.web
terraform import aws_instance.web i-0abc123def456
```

**Step 4 — If no version history exists** (this should never happen in a well-run repo):
- Run `terraform plan` to see the diff between a blank state and your config.
- Use `terraform import` for each existing resource to rebuild state.
- This is tedious for large configs — treat "no S3 versioning" as a P1 risk to fix immediately.

**Prevention**:
- Enable S3 versioning and MFA Delete on the state bucket.
- Use DynamoDB locking to prevent concurrent writes.
- Regularly test state restoration in a lower environment.

---

**Q: How do you handle infrastructure created outside Terraform?**

**A:** Infrastructure that exists in AWS but isn't in Terraform state is called "unmanaged" or "out-of-band" infrastructure. Options depend on the goal:

**Option 1 — Import into Terraform (bring it under management)**:
```bash
# Write the resource block in your .tf file first, then import
resource "aws_s3_bucket" "legacy" {
  bucket = "my-legacy-bucket"
}

terraform import aws_s3_bucket.legacy my-legacy-bucket
terraform plan   # should show no changes if config matches reality
```

For bulk imports (Terraform 1.5+), use declarative `import {}` blocks:
```hcl
import {
  to = aws_s3_bucket.legacy
  id = "my-legacy-bucket"
}
```

**Option 2 — Reference without managing** (read-only, via data source):
```hcl
data "aws_s3_bucket" "existing" {
  bucket = "my-legacy-bucket"
}
# Use data.aws_s3_bucket.existing.arn in other resources
```
This lets Terraform use the resource without owning its lifecycle.

**Option 3 — Ignore it**: if the resource is truly one-off and shouldn't be in Terraform (e.g., a manually created test resource), leave it out and document why.

**Prevention**:
- Use SCPs or IAM boundaries to require all production infrastructure to be created via Terraform CI/CD (not the console or manual CLI).
- Enable AWS Config rules to detect untagged or Terraform-unmanaged resources.
- Run `terraform plan -refresh-only` on a schedule to surface drift automatically.

---

**Q: How do you prevent multiple team members from running Terraform simultaneously?**

**A:** Use **state locking** — Terraform's built-in mechanism to prevent concurrent operations.

**DynamoDB locking (S3 backend)**:
```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"   # locking table
    encrypt        = true
  }
}
```

Create the DynamoDB table:
```bash
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

When a `plan`/`apply` starts, Terraform writes a lock item to DynamoDB. Any other `apply` attempt sees the lock and fails with a message showing who holds the lock and when it started.

**Force-unlock** (only when lock is orphaned after a crash):
```bash
terraform force-unlock <LOCK_ID>   # LOCK_ID is shown in the error message
```

**Additional controls**:
- Run Terraform exclusively from CI/CD (GitHub Actions, Jenkins) — block direct `terraform apply` from developer machines via IAM policy.
- **Terraform Cloud / Atlantis**: provides a PR-based workflow where `apply` only happens via merge, with a built-in run queue so no two applies overlap.
- **Serialized pipeline stages**: in Jenkins, use `lock('terraform-prod') { ... }` to serialize concurrent pipeline runs.

---

**Q: What is the Terraform state file and how do you troubleshoot state mismatch issues?**

**A:**

**What the state file is**:
The state file (`terraform.tfstate`) is a JSON document that records Terraform's understanding of what real-world infrastructure exists. It maps each `resource "type" "name"` in your config to the actual cloud resource's ID and current attributes.

```json
{
  "resources": [{
    "type": "aws_instance",
    "name": "web",
    "instances": [{
      "attributes": {
        "id": "i-0abc123def456",
        "instance_type": "t3.micro",
        ...
      }
    }]
  }]
}
```

Terraform uses the state to:
- Determine what already exists (no need to describe every resource on every plan).
- Compute diffs between desired config and current state.
- Manage resource dependencies.

**Troubleshooting state mismatches**:

| Symptom | Cause | Fix |
|---|---|---|
| `terraform plan` wants to create a resource that already exists | Resource exists in cloud but not in state | `terraform import <resource> <id>` |
| `terraform plan` wants to delete a resource that shouldn't be deleted | State has a resource that no longer exists in cloud | `terraform state rm <resource>` |
| `terraform plan` shows constant diffs for an attribute | Provider computes/normalizes the attribute differently | Use `lifecycle { ignore_changes = [<attr>] }` |
| Wrong resource being modified | State uses wrong resource ID | `terraform state show <resource>`, `terraform state mv` to rename |
| Two states diverged after a team merge conflict | Two `terraform apply` ran with the same state version | Restore the canonical state from S3 versioning; replay the missing changes |

```bash
# Inspect state
terraform state list              # all resources in state
terraform state show aws_instance.web   # attributes of a specific resource

# Reconcile drift
terraform plan -refresh-only      # show cloud vs state diff without config diff

# Fix a renamed resource without destroying and recreating
terraform state mv aws_instance.old_name aws_instance.new_name
```

---

**Q: Write a simple Terraform configuration to deploy an S3 bucket with versioning and encryption.**

**A:**
```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" { region = "us-east-1" }

resource "aws_s3_bucket" "data" {
  bucket = "my-app-data-bucket-20240101"
  tags   = { Environment = "production", ManagedBy = "terraform" }
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
    bucket_key_enabled = true   # reduces KMS API call costs
  }
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

output "bucket_arn" {
  value = aws_s3_bucket.data.arn
}
```

---



