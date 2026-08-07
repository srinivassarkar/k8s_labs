# 🏗️ Terraform + AWS — 30-Day Rapid Revision Guide

> **Production-Ready · Interview-Optimised · One Page Per Day**
>
> DevOps · Kubernetes · GitOps · Platform Engineering

---

## Day 01: Infrastructure as Code + Terraform Basics
*What IaC is, why it exists, and the Terraform workflow*

### Core Concept

IaC = provision/manage infrastructure via code files, not manual clicks. Version-controlled, repeatable, auditable.

### Why IaC Matters (Interview Answer)

| Problem (Manual) | IaC Solution | Keyword |
|---|---|---|
| Config drift across envs | Identical code → identical infra | Consistency |
| Hours to spin up env | terraform apply completes in minutes | Speed |
| No audit trail | Git history is your change log | Version Control |
| Human error | Declarative config removes guesswork | Reliability |
| Hard to replicate prod | Same code → any env | Parity |

### Terraform Core Workflow

| Command | Description |
|---|---|
| `terraform init` | Download providers, set up backend, initialise working dir |
| `terraform validate` | Check HCL syntax (no API calls) |
| `terraform plan` | Diff desired vs actual state → execution plan (read-only) |
| `terraform apply` | Execute plan → create/update/delete resources |
| `terraform destroy` | Tear down all managed resources |

### How Terraform Works (Architecture)

HCL files → Terraform Core → Provider Plugin (AWS, GCP…) → Cloud API

Provider is a separate binary that wraps cloud APIs. Core never calls AWS directly.

### Quick Install

```bash
brew install hashicorp/tap/terraform   # macOS
sudo apt install terraform              # Ubuntu
alias tf=terraform && tf -version
```

> **Interview Tip:** Explain: Terraform is declarative (you describe desired state). Ansible is imperative (you describe steps). Terraform manages state; Ansible does not.

---

## Day 02: Terraform Providers
*Plugins, versioning, and constraints*

### What is a Provider?

A provider is a downloadable plugin that translates Terraform resources into API calls for a specific platform (AWS, Azure, GCP, Kubernetes…). Terraform Core + Provider = complete toolchain.

### Core vs Provider Versioning

| | Terraform Core | Provider (e.g. aws) |
|---|---|---|
| What it does | Parse HCL, manage state, plan/apply | Call AWS APIs, resource CRUD |
| Version pin | `required_version = ">= 1.9"` | `version = "~> 5.0"` |
| Released by | HashiCorp | HashiCorp / community |

### Version Constraint Operators

| Constraint | Meaning |
|---|---|
| `= 5.0.0` | Exact version only |
| `>= 5.0` | Any version at or above 5.0 |
| `~> 5.0` | Pessimistic: allow patch, not minor (5.0.x only) |
| `~> 5.0.0` | Pessimistic: 5.0.0 and higher patch releases only |
| `>= 5.0, < 6` | Range: between 5.x and 6.x exclusive |

### Canonical Provider Config

```hcl
terraform {
  required_version = ">= 1.10"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" { region = "us-east-1" }
```

### Best Practices

- Always pin provider versions – new providers can have breaking changes
- Run `terraform providers lock` → commits exact hashes to `.terraform.lock.hcl`
- Test provider upgrades in dev before applying to prod

> **Gotcha:** The `.terraform.lock.hcl` file should be committed to Git. It ensures all team members use identical provider binaries.

---

## Day 03: AWS Auth + First Resource (S3 Bucket)
*Credential methods, IAM, and creating your first real resource*

### Authentication Methods (ordered by precedence)

| Method | Details |
|---|---|
| Env vars | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` (CI/CD pipelines) |
| AWS CLI / profile | `~/.aws/credentials` (local dev, `aws configure`) |
| IAM Role (EC2) | Instance profile – no static keys needed (production) |
| IAM Role (ECS/EKS) | Task/pod role – same concept, container scope |

### Security Best Practice

Never hardcode credentials in `.tf` files. Use IAM roles in production. For local dev, use `aws configure` or `aws-vault`.

### Implicit Dependencies (Key Terraform Concept)

When resource B references resource A's attribute (e.g., `aws_vpc.main.id`), Terraform automatically creates A before B. You don't need `depends_on`.

```hcl
resource "aws_vpc" "main" { cidr_block = "10.0.0.0/16" }
resource "aws_subnet" "pub" {
  vpc_id     = aws_vpc.main.id   # implicit dependency
  cidr_block = "10.0.1.0/24"
}
```

### S3 Bucket Key Facts

| Fact | Details |
|---|---|
| Globally unique name | Bucket names are unique across all AWS accounts globally |
| Naming rules | 3-63 chars, lowercase, numbers, hyphens only |
| Resources needed | `aws_s3_bucket` + `aws_s3_bucket_versioning` + `aws_s3_bucket_server_side_encryption_configuration` |
| Block public access | `aws_s3_bucket_public_access_block` (all 4 flags = true for private buckets) |

### Troubleshooting Auth

```bash
aws sts get-caller-identity    # confirm which identity Terraform will use
aws configure list             # show current config source
```

> **Interview Tip:** Explain the difference: IAM User has long-term credentials (key pair). IAM Role has temporary credentials via STS AssumeRole. Always prefer roles in production.

---

## Day 04: Terraform State & Remote Backend
*State file mechanics, S3 backend, S3 native locking (Terraform ≥ 1.10)*

### What is State?

`terraform.tfstate` is a JSON file mapping your HCL resources to real cloud resource IDs. Terraform compares state vs desired config to compute a diff (plan).

### Local vs Remote State

| Concern | Local (.tfstate) | Remote (S3) |
|---|---|---|
| Team collaboration | ❌ Only one person can work | ✅ Shared, centralised |
| Concurrent changes | ❌ Race conditions / corruption | ✅ State locking prevents this |
| Disaster recovery | ❌ Lost if laptop dies | ✅ S3 versioning = rollback |
| Sensitive data | ❌ Plaintext on disk | ✅ S3 encryption at rest |

### S3 Backend Config (Terraform ≥ 1.10 – no DynamoDB needed)

```hcl
terraform {
  backend "s3" {
    bucket       = "my-tf-state-bucket"
    key          = "env/dev/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true   # S3 native locking via conditional writes
    encrypt      = true
  }
}
```

### How S3 Native Locking Works

Terraform writes a `.tflock` object using the S3 `If-None-Match` header (conditional write). If another process already holds the lock, S3 returns HTTP 412 → Terraform fails safely. Lock file deleted on completion.

> **Important:** S3 versioning MUST be enabled for S3 native locking to function. DynamoDB locking is now deprecated — use `use_lockfile = true`.

### Key State Commands

| Command | Description |
|---|---|
| `terraform state list` | List all resources tracked in state |
| `terraform state show <resource>` | Show full attributes of one resource |
| `terraform state rm <resource>` | Remove resource from state (resource not destroyed) |
| `terraform state mv <src> <dst>` | Rename resource in state without recreating it |
| `terraform state pull` | Print raw state JSON |
| `terraform force-unlock <id>` | Release stuck lock (use after crash) |

### State Best Practices

- Separate state per environment (dev/staging/prod use different key paths)
- Never edit state manually — use `terraform state` commands
- State contains secrets in plaintext → restrict S3 bucket access via IAM
- Enable S3 access logging + CloudTrail for audit

> **Interview Tip:** `terraform import` vs `terraform state mv`: import brings an existing resource under Terraform management. `mv` renames/moves a resource within state without touching real infrastructure.

---

## Day 05: Terraform Variables
*Input, local, and output variables — precedence and patterns*

### Three Variable Types

| Type | Defined in | Purpose |
|---|---|---|
| Input (`var.*`) | variables.tf | Parameterise config — like function arguments |
| Local (`local.*`) | locals.tf | Computed/derived values — like local variables |
| Output | outputs.tf | Expose values post-apply — like return values |

### Input Variable Precedence (highest wins)

| Priority | Source |
|---|---|
| 1 – CLI flag | `terraform apply -var="env=prod"` |
| 2 – `*.auto.tfvars` | Auto-loaded in alphabetical order |
| 3 – `terraform.tfvars` | Auto-loaded if present |
| 4 – `TF_VAR_name` env | `export TF_VAR_environment=staging` |
| 5 – Default value | `default = "staging"` in variable block |

### Input Variable with Validation

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
  validation {
    condition     = contains(["dev","staging","prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}
```

### Locals — The Power Pattern

```hcl
locals {
  name_prefix = "${var.project}-${var.env}"
  common_tags = merge(var.extra_tags, {
    Environment = var.env
    ManagedBy   = "Terraform"
  })
}
```

### Outputs

```hcl
output "bucket_arn" {
  description = "S3 bucket ARN"
  value       = aws_s3_bucket.main.arn
  sensitive   = false
}
```

Access outputs: `terraform output` / `terraform output -json` / `terraform output -raw bucket_arn`

> **Interview Tip:** `locals{}` is evaluated at apply time and can reference other locals, vars, and resource attributes. Use them to avoid repeating expressions — DRY principle for Terraform.

---

## Day 06: Terraform File Structure & Organisation
*Canonical layout, multi-env patterns, best practices*

### Standard File Layout

| File | Purpose |
|---|---|
| `provider.tf` | Provider declarations + required_providers |
| `backend.tf` | Remote state backend config |
| `variables.tf` | All input variable declarations |
| `locals.tf` | Local value definitions |
| `main.tf` | Core resources OR module calls |
| `vpc.tf` / `compute.tf` | Service-specific resources (scale with project size) |
| `outputs.tf` | All output declarations |
| `terraform.tfvars` | Default variable values (committed to git) |
| `*.auto.tfvars` | Environment-specific overrides |

### Multi-Environment Pattern

```
environments/
  dev/    → backend.tf + terraform.tfvars + main.tf
  staging/ → backend.tf + terraform.tfvars + main.tf
  prod/   → backend.tf + terraform.tfvars + main.tf
modules/vpc/ modules/compute/   # shared reusable modules
```

### Key Rules

- All `.tf` files in a directory are loaded and merged — order doesn't matter
- Files are merged in lexicographical order (alphabetical)
- Keep files under 500 lines — split by service/function beyond that
- Variable defined in one `.tf` file is accessible in all others in same dir

### Validation + Format Commands

```bash
terraform validate    # syntax check only
terraform fmt -recursive   # auto-format all .tf files (run in CI)
terraform plan -out=tfplan   # save plan for apply later (CI pattern)
terraform apply tfplan
```

> **Interview Tip:** In monorepos with multiple Terraform roots, each directory with `.tf` files is an independent "root module" with its own state. Modules are reusable components called from root modules.

---

## Day 07: Type Constraints
*Primitive, collection, and structural types in Terraform*

### Primitive Types

| Type | Description |
|---|---|
| `string` | Text — `"us-east-1"`, `"t2.micro"` |
| `number` | Integer or float — `3`, `1.5` |
| `bool` | `true` or `false` |
| `any` | Disables type checking (avoid in prod) |

### Collection Types

| Type | Description |
|---|---|
| `list(string)` | Ordered, allows duplicates — `["a","b","a"]` |
| `set(string)` | Unordered, unique values — `{"a","b"}` (use for `for_each`) |
| `map(string)` | Key-value pairs, string keys — `{"env":"prod"}` |

### Structural Types

```hcl
variable "instance_config" {
  type = object({
    instance_type     = string
    count             = number
    enable_monitoring = bool
  })
}
```

### Tuple (fixed-length, mixed types)

```hcl
variable "port_protocol" {
  type    = tuple([number, string])
  default = [8080, "tcp"]
}
```

### Type Conversion Functions

| Function | Description |
|---|---|
| `toset(list)` | Convert list → set (removes duplicates, enables `for_each`) |
| `tolist(set)` | Convert set → list |
| `tonumber(str)` | `"42"` → `42` |
| `tostring(num)` | `42` → `"42"` |

> **Interview Tip:** When to use `set` vs `list`: use `set` when you need `for_each` (requires map or set) and don't care about ordering. Use `list` when order matters or you need index access via `count.index`.

---

## Day 08: Meta-Arguments
*count, for_each, depends_on, lifecycle, provider — all 5 in one page*

### count vs for_each — The Critical Choice

| Feature | count | for_each |
|---|---|---|
| Input | Integer or list | Map or set |
| Addressing | `resource[0]`, `resource[1]` | `resource["key"]` |
| Remove middle item | Recreates subsequent items ⚠️ | Only removes that specific key ✅ |
| Use in prod | Simple / identical resources | Preferred — stable addressing |

### count Example

```hcl
resource "aws_s3_bucket" "logs" {
  count  = 3
  bucket = "logs-bucket-${count.index}"
}
```

### for_each Example (preferred)

```hcl
resource "aws_s3_bucket" "env" {
  for_each = toset(["dev", "staging", "prod"])
  bucket   = "app-${each.value}-bucket"
}
```

### depends_on — Explicit Ordering

```hcl
resource "aws_s3_bucket" "dependent" {
  depends_on = [aws_iam_role_policy.bucket_access]
}
```

Use only when Terraform can't infer the dependency — e.g., policy must exist before IAM role is assumed, but no attribute reference exists.

### lifecycle Block — All Options

```hcl
lifecycle {
  create_before_destroy = true  # zero-downtime replace
  prevent_destroy       = true  # block terraform destroy
  ignore_changes        = [tags, desired_capacity]  # drift immunity
  replace_triggered_by  = [aws_security_group.app.id]
}
```

### provider Meta-Argument — Multi-Region

```hcl
provider "aws" { alias = "west"; region = "us-west-2" }
resource "aws_s3_bucket" "dr" { provider = aws.west }
```

> **Interview Tip:** Explain `for_each` stability: if you have 3 buckets [a,b,c] and remove b with `count`, Terraform must shift c down to index [1] → it plans to destroy and recreate c. With `for_each`, removing b only destroys `bucket["b"]` — c is untouched.

---

## Day 09: Lifecycle Meta-Arguments (Deep Dive)
*create_before_destroy, prevent_destroy, ignore_changes, precondition/postcondition*

### Lifecycle Rules Reference

| Rule | When to Use | Gotcha |
|---|---|---|
| `create_before_destroy` | ALB target group changes, cert rotation, zero-downtime | May briefly run 2x resources → cost spike |
| `prevent_destroy` | Production DB, state bucket, KMS keys | Must comment out to destroy — not skippable with `-target` |
| `ignore_changes` | ASG `desired_capacity` (auto-scaled), tags added by monitoring tools | Hides real drift — document why you're ignoring |
| `replace_triggered_by` | Force EC2 replace when launch template changes | Causes downtime unless paired with `create_before_destroy` |
| `precondition` | Validate allowed region / env before plan | Errors at plan time — fast feedback |
| `postcondition` | Verify required tags exist after resource creation | Errors at apply time after resource is created |

### precondition Example

```hcl
lifecycle {
  precondition {
    condition     = contains(["us-east-1","eu-west-1"], data.aws_region.current.name)
    error_message = "Deployment only allowed in approved regions."
  }
}
```

### postcondition Example

```hcl
lifecycle {
  postcondition {
    condition     = contains(keys(self.tags), "Owner")
    error_message = "Resource must have an Owner tag."
  }
}
```

### Production Database Pattern

```hcl
resource "aws_db_instance" "prod" {
  lifecycle {
    prevent_destroy       = true
    create_before_destroy = true
    ignore_changes        = [password]  # managed by Secrets Manager
  }
}
```

### ASG Ignore Capacity Pattern

```hcl
resource "aws_autoscaling_group" "app" {
  desired_capacity = 2
  lifecycle { ignore_changes = [desired_capacity, load_balancers] }
}
```

> **Interview Tip:** Explain `precondition` vs `validation` block: variable validation runs when the variable is evaluated. `precondition` runs at plan/apply time and can reference data sources and other resources — much more powerful for runtime checks.

---

## Day 10: Dynamic Blocks, Conditionals & Splat Expressions
*Eliminate repetition, branch logic, extract collections*

### Conditional Expression (Ternary)

```hcl
instance_type = var.env == "prod" ? "t3.large" : "t2.micro"
count = var.enable_monitoring ? 1 : 0
```

### Dynamic Block — Security Group Rules

```hcl
variable "ingress_rules" {
  type = list(object({ port = number, cidr = string }))
}
resource "aws_security_group" "web" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
    }
  }
}
```

### Splat Expression `[*]`

```hcl
output "all_instance_ids" {
  value = aws_instance.web[*].id
}
output "all_subnet_ids" {
  value = aws_subnet.public[*].id
}
```

### for Expression (transform collections)

```hcl
# List to map
local.instance_map = { for inst in aws_instance.web : inst.id => inst.public_ip }

# Filter
local.prod_ids = [for i in var.instances : i.id if i.env == "prod"]
```

### Practical Patterns

| Use Case | Pattern |
|---|---|
| Security group rules | Dynamic ingress/egress blocks from `list(object(...))` variable |
| EBS volumes | Dynamic `ebs_block_device` from list |
| IAM policy statements | Dynamic `statement` block from list |
| Route table routes | Dynamic `route` block from map |

> **Interview Tip:** Don't nest dynamic blocks more than 2 levels — readability collapses. Instead, flatten the data in `locals{}` first, then use a single dynamic block.

---

## Day 11–12: Terraform Built-in Functions
*String, numeric, collection, file, date, validation, and lookup functions*

### Function Categories Reference

| Category | Key Functions | Common Use |
|---|---|---|
| String | `lower`, `upper`, `replace`, `substr`, `split`, `join`, `trim`, `chomp` | Sanitise names, build ARNs |
| Numeric | `abs`, `max`, `min`, `ceil`, `floor`, `sum` | Cost calc, sizing |
| Collection | `length`, `concat`, `merge`, `reverse`, `toset`, `tolist`, `flatten` | Combine lists, tags, deduplicate |
| Type | `tonumber`, `tostring`, `tobool`, `toset`, `tolist` | Type coercion |
| File | `file`, `fileexists`, `dirname`, `basename` | Read configs, check paths |
| Date/Time | `timestamp`, `formatdate`, `timeadd` | Expiry tags, rotation |
| Validation | `can`, `try`, `regex`, `contains`, `startswith`, `endswith` | Safe type checks |
| Lookup | `lookup`, `element`, `index`, `keys`, `values` | Map access, list ops |
| Encoding | `jsonencode`, `jsondecode`, `base64encode`, `base64decode` | Secrets, inline policies |

### Critical Functions for Production IaC

```hcl
# Sanitise bucket name
lower(replace("My Project Name", " ", "-"))  → "my-project-name"

# Safe environment lookup with default
lookup(var.instance_sizes, var.env, "t2.micro")

# Merge tag maps
merge(local.common_tags, { Name = "my-resource" })

# Read JSON config file
jsondecode(file("${path.module}/config.json"))

# Timestamp tag
formatdate("YYYY-MM-DD", timestamp())

# Validate with regex (can = safe, won't throw)
can(regex("^t[2-3]\\.(micro|small|medium)", var.instance_type))
```

### Sensitive Data

```hcl
output "db_password" {
  value     = random_password.db.result
  sensitive = true   # masks in CLI output, still in state
}
```

> **Interview Tip:** Use `terraform console` to experiment with functions interactively — no cloud calls needed. Great for debugging expressions: `> merge({"a":1},{"b":2})`

---

## Day 13: Terraform Data Sources
*Read existing infrastructure without managing it*

### What is a Data Source?

A data source reads existing infrastructure (created outside Terraform or in another state) and makes its attributes available. It's read-only — Terraform won't modify or destroy the referenced resource.

### Syntax

```hcl
data "aws_vpc" "shared" {
  filter {
    name   = "tag:Name"
    values = ["shared-network-vpc"]
  }
}

# Reference: data.<type>.<name>.<attribute>
resource "aws_instance" "app" {
  subnet_id = data.aws_subnet.shared.id
}
```

### Essential Data Sources (Memorise These)

| Data Source | Use |
|---|---|
| `aws_ami` | Find latest AMI by owner/filter — avoid hardcoding AMI IDs |
| `aws_vpc` | Reference existing VPC by tag or ID |
| `aws_subnet` | Find subnet by VPC + AZ + tag |
| `aws_region` | Current region: `data.aws_region.current.name` |
| `aws_caller_identity` | Account ID: `data.aws_caller_identity.current.account_id` |
| `aws_availability_zones` | List AZs in current region |
| `aws_secretsmanager_secret_version` | Read secrets from Secrets Manager |
| `aws_iam_policy_document` | Build IAMolicy JSON programmatically |

### Latest AMI Pattern (Never Hardcode AMI IDs)

```hcl
data "aws_ami" "al2" {
  most_recent = true
  owners      = ["amazon"]
  filter { name = "name"; values = ["amzn2-ami-hvm-*-x86_64-gp2"] }
}
resource "aws_instance" "web" { ami = data.aws_ami.al2.id }
```

### Cross-Stack Reference Pattern

Stack A outputs `vpc_id` → Stack B reads it via data source or `terraform_remote_state`

```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config  = { bucket = "tf-state", key = "vpc/terraform.tfstate", region = "us-east-1" }
}
vpc_id = data.terraform_remote_state.vpc.outputs.vpc_id
```

> **Interview Tip:** Data sources evaluate at plan time, not just apply. If the referenced resource doesn't exist, `terraform plan` fails. This is useful for enforcing that shared infra exists before deploying app infra.

---

## Day 14: Static Website: S3 + CloudFront (Mini Project 1)
*CDN-backed static site with correct security posture*

### Architecture

Internet → CloudFront Distribution (HTTPS) → S3 Bucket (private, OAC) → Static files

### Key Resources

| Resource | Purpose |
|---|---|
| `aws_s3_bucket` | Origin bucket (NOT public website hosting) |
| `aws_s3_bucket_public_access_block` | All 4 flags = true (private bucket) |
| `aws_cloudfront_origin_access_control` | OAC — lets CloudFront read private S3 |
| `aws_cloudfront_distribution` | CDN with HTTPS, custom cache behaviour |
| `aws_s3_bucket_policy` | Allow `cloudfront.amazonaws.com` to `GetObject` |
| `aws_s3_object` (for_each) | Upload HTML/CSS/JS with correct `content_type` |

### Gotchas

- S3 website hosting (`aws_s3_bucket_website`) is NOT needed when using CloudFront OAC — and is less secure
- CloudFront distributions can take 10–15 min to deploy globally
- MIME types must be set on each object (`content_type`) or browsers won't render correctly
- HTTP to HTTPS redirect configured at CloudFront viewer protocol policy, not S3

### Cache Invalidation

```bash
aws cloudfront create-invalidation \
  --distribution-id E1ABCDEF \
  --paths '/*'
```

### Interview Architecture Question — Explain This Pattern

| Component | Role | Why not S3 direct? |
|---|---|---|
| S3 | Object storage, private | No HTTPS, no global edge |
| CloudFront | CDN + HTTPS termination + caching | Low latency globally, free TLS |
| OAC | Auth: CF → S3 without public bucket | Security: bucket stays private |
| ACM Cert | Custom domain TLS (must be us-east-1) | Required for custom domain |

> **Interview Tip:** OAC (Origin Access Control) replaced OAI (Origin Access Identity). OAC supports SSE-KMS encryption and POST methods. Always use OAC for new deployments.

---

## Day 15: VPC Peering (Mini Project 2)
*Cross-region VPC peering with multi-provider Terraform*

### Architecture

VPC-A (us-east-1, 10.0.0.0/16) ←→ VPC-B (us-west-2, 10.1.0.0/16) via VPC Peering Connection

### VPC Peering Rules (Critical — Frequently Asked)

| Rule | Detail |
|---|---|
| CIDR overlap? | VPCs must have non-overlapping CIDR blocks — peering fails otherwise |
| Transitive? | NOT transitive — A↔B + B↔C does NOT give A↔C |
| Edge-to-edge? | NOT supported — VGW/IGW/NAT GW routes don't transit through peering |
| Max per VPC | 125 peering connections per VPC |
| Cross-account? | Yes — requires `AccepterVpcInfo` + explicit acceptance |
| Cross-region? | Yes — supported, slightly higher latency + data transfer cost |

### Terraform Multi-Region Pattern

```hcl
provider "aws" { alias = "east"; region = "us-east-1" }
provider "aws" { alias = "west"; region = "us-west-2" }

resource "aws_vpc" "east"  { provider = aws.east; cidr_block = "10.0.0.0/16" }
resource "aws_vpc" "west"  { provider = aws.west; cidr_block = "10.1.0.0/16" }

resource "aws_vpc_peering_connection" "peer" {
  provider    = aws.east
  vpc_id      = aws_vpc.east.id
  peer_vpc_id = aws_vpc.west.id
  peer_region = "us-west-2"
  auto_accept = false
}
resource "aws_vpc_peering_connection_accepter" "peer" {
  provider                  = aws.west
  vpc_peering_connection_id = aws_vpc_peering_connection.peer.id
  auto_accept               = true
}
```

### Routing (Must Configure Both Sides)

Each VPC's route table needs a route to the other VPC's CIDR via peering connection. Security groups must allow traffic from the peered CIDR (not just the SG).

### Transit Gateway vs VPC Peering

| | VPC Peering | Transit Gateway |
|---|---|---|
| Topology | Point-to-point (mesh) | Hub and spoke |
| Transitive routing | ❌ No | ✅ Yes Cost | Data transfer only | Per attachment + data |
| Scale | 125 connections max | Thousands of VPCs |
| When to use | Few VPCs, simple | Large org, many VPCs |

---

## Day 16: IAM User Management from CSV (Mini Project 3)
*Data-driven IAM with csvdecode, for_each, dynamic group membership*

### Pattern: CSV as Source of Truth

```hcl
locals { users = csvdecode(file("users.csv")) }

resource "aws_iam_user" "users" {
  for_each = { for u in local.users : u.first_name => u }
  name = lower("${substr(each.value.first_name,0,1)}${each.value.last_name}")
  tags = { DisplayName = "${each.value.first_name} ${each.value.last_name}" }
}
```

### Dynamic Group Membership

```hcl
resource "aws_iam_group_membership" "managers" {
  name  = "managers-membership"
  group = aws_iam_group.managers.name
  users = [
    for user in aws_iam_user.users : user.name
    if contains(["Manager","CEO"], user.tags["JobTitle"])
  ]
}
```

### IAM Best Practices

| Practice | Detail |
|---|---|
| Never use root | Create an admin IAM user; lock root with hardware MFA |
| Least privilege | Grant only permissions needed for the specific job |
| Groups > Users | Attach policies to groups, not individual users |
| Roles > Users | Use IAM roles for services (EC2, Lambda, ECS) |
| MFA everywhere | Enforce MFA for console access via IAM password policy |
| AWS SSO / Identity Center | Preferred for human users in production — no long-term keys |
| Rotate keys | Rotate access keys every 90 days; audit with IAM Access Advisor |

### Import Existi Users

```bash
terraform import 'aws_iam_user.users["Michael"]' mscott
```

> **Interview Tip:** Explain IAM evaluation logic: Deny > Allow. An explicit Deny in any policy wins over any Allow. Implicit Deny = default if no Allow exists. SCPs (in AWS Orgs) set the maximum permission boundary.

---

## Day 17: Blue-Green Deployments with Elastic Beanstalk (Mini Project 4)
*Zero-downtime deployment strategy, CNAME swap pattern*

### Blue-Green Concept

Maintain two identical production environments. Route 100% traffic to Blue (current prod). Deploy new version to Green (staging). Validate Green. Swap traffic instantly. Blue becomes rollback standby.

### Deployment Strategies Compared

| Strategy | Downtime | Rollback | Cost | Risk |
|---|---|---|---|---|
| Blue-Green | Zero (CNAME/DNS swap) | Instant — swap back | 2x infra cost temporarily | Low |
| Rolling | None (gradual) | Complex | Normal | Medium |
| In-place | Possible | Redeploy | Cheapest | Highest |
| Canary | None | Redirect traffic | Slight overhe | Very Low |

### Elastic Beanstalk CNAME Swap

```bash
aws elasticbeanstalk swap-environment-cnames \
  --source-environment-name app-blue \
  --destination-environment-name app-green
```

Swap completes in ~1-2 minutes. DNS TTL is managed by EB. Old environment stays alive for instant rollback.

### create_before_destroy for Zero-Downtime Terraform Changes

```hcl
resource "aws_lb_target_group" "app" {
  name     = "app-tg-${random_id.suffix.hex}"  # must be unique
  lifecycle { create_before_destroy = true }
}
```

### Kubernetes Equivalent

In K8s: use Deployment with `RollingUpdate` strategy, or create a new Deployment and switch Service selector. ArgoCD supports blue-green via Argo Rollouts CRD.

> **Interview Tip:** Blue-green requires that database schema changes are backward-compatible during the transition window (both versions running simultaneously). Use expand-contract migrations: add column first, deploy, then remove old column in next cycle.

---

## Day 18: Serverless Image Processing: Lambda + S3 (Mini Project 5)
*Event-driven architecture, Lambda layers, IAM least privilege*

### Architecture

Upload S3 → S3 ObjectCreated Event → Lambda trigger → Pillow processing → 5 variants → Processed S3

### Key Terraform Resources

| Resource | Purpose |
|---|---|
| `aws_lambda_function` | Function with handler, runtime, role, env vars, layers |
| `aws_lambda_layer_version` | Pillow library as reusable layer (shared across functions) |
| `aws_s3_bucket_notification` | `trigger = { events=["s3:ted:*"], lambda_arn }` |
| `aws_lambda_permission` | Allow S3 to invoke Lambda (`principal=s3.amazonaws.com`) |
| `aws_iam_role_policy` | Least privilege: only `GetObject` from upload bucket + `PutObject` to processed bucket |

### Lambda Key Concepts

| Concept | Value | Notes |
|---|---|---|
| Cold start | ~470ms with Pillow layer | Warm execution ~300-600ms |
| Max timeout | 900s (15 min) | Set per function; image proc uses 60s |
| Memory | 128MB–10GB | More memory = more CPU + cost |
| Concurrency | 10 per account default | Request increase via support |
| Layers | Up to 5 per function | Each layer ≤ 250MB unzipped |

### Monitoring Lambda

```bash
# Live log tail
aws logs tail /aws/lambda/image-processor --follow

# Check invocations via CloudWatch
aws cloudwatch get-metric-statistics --namespace AWS/Lambda \
  --metric-name Invocations --dimensions Name=FunctionName,Value=image-processor \
  --period 300 --statistics Sum --start-time ... --end-time ...
```

### Security Posture

All buckets private.SE-AES256 at rest. Lambda role has minimal S3 permissions (no wildcard `*`). VPC optional for Lambda if private DB access needed.

> **Interview Tip:** Lambda concurrency = number of simultaneous executions. Reserved concurrency caps a function. Provisioned concurrency eliminates cold starts (pre-warms). Use provisioned for latency-sensitive prod functions.

---

## Day 19: Terraform Provisioners
*local-exec, remote-exec, file — and why to avoid them*

### Three Provisioner Types

| Type | Runs On | Conneion Required | Use Case |
|---|---|---|---|
| `local-exec` | Machine running Terraform | No | Trigger webhook, update inventory, Slack notification |
| `remote-exec` | Remote instance via SSH | Yes | Install packages, run bootstrap script |
| `file` | Copies local → remote | Yes | Upload config, certs, scripts before remote-exec |

### Critical Behaviour

| Behaviour | Detail |
|---|---|
| Run timing | ONLY on resource creation (not updates) — use `taint` to re-run |
| Failure | Resource marked 'tainteddestroyed and recreated on next apply |
| `on_failure` | `= continue` skips the error; `= fail` (default) taints resource |
| destroy-time | `when = destroy` runs before resource deletion |

### Re-Run a Provisioner

```bash
terraform taint aws_instance.demo    # mark for recreation
terraform apply -replace=aws_instance.demo    # Terraform 0.15.2+ equivalent
```

### Alternatives (Always Prefer These)

| Alternative | Details |
|---|---|
| `user_data` / cloud-init | EC2 bootstrap — native, no SSH, runs onvery start |
| Packer | Pre-bake AMIs with all config — immutable infrastructure |
| Ansible / Chef / Puppet | Config management — idempotent, mature tooling |
| AWS Systems Manager | No SSH required, works in private subnets, audit trail |
| Container images | Bake config into Docker image — most portable |

### Connection Block

```hcl
connection {
  type        = "ssh"
  user        = "ec2-user"
  private_key = file("~/.ssh/key.pem")
  host        = self.public_ip
  timeout     = "5m"
}
```

> **Inw Tip:** Provisioners are a code smell in modern IaC. If asked why you'd use them: legacy migrations, quick POC, or when cloud-init isn't available. Always explain the preferred alternative.

---

## Day 20: EKS Cluster with Terraform Modules (Real-time Project 1)
*Production EKS, Secrets Manager, modular architecture*

### Architecture

VPC (public + private subnets) → EKS Cluster (managed node groups) → Secrets Manager (DB creds + API keys) → IAM OIDC for IRSA

### Module Structure

```
modules/
  e          → cluster, node groups, OIDC provider
  secrets-manager/ → KMS key, secrets for DB/API/app config
  vpc/             → VPC, subnets, IGW, NAT, route tables
```

### EKS Key Concepts for Interviews

| Concept | Details |
|---|---|
| Managed Node Groups | AWS manages EC2 lifecycle, patching, draining during updates |
| Fargate Profiles | Serverless pods — no node management; use for burst workloads |
| IRSA | IAM Roles for Service Accounts — pod-level AWS permissions (no node IAM) |
| OIDC| Enables IRSA — must be configured on EKS cluster |
| aws-auth ConfigMap | Maps IAM users/roles to K8s RBAC groups (replaced by Access Entries in v21+) |
| Access Entries (v21+) | Native IAM-to-K8s mapping — no more aws-auth ConfigMap hacks |
| Cluster Autoscaler | Scales node groups based on pending pods; requires ASG tags |
| Karpenter | Modern replacement for CA — provisions nodes directly, faster |

### Terraform EKS Module v21 Syntax Changes

```hcl
module "eks" {
  source             = "terrafo-modules/eks/aws"
  version            = "~> 21.0"
  name               = "prod-cluster"        # was: cluster_name
  kubernetes_version = "1.30"               # was: cluster_version
  endpoint_public_access = true             # was: cluster_endpoint_public_access
}
```

### kubectl Essential Commands

```bash
aws eks update-kubeconfig --region us-east-1 --name my-cluster
kubectl get nodes -o wide
kubectl get pods -A
kubectl describe node <node-name>
```

> **Interview Tip:** IRSA vs node IAM: IRSA (pod-level role) is preferred — least privilege. With node IAM, ALL pods on that node share the same permissions. IRSA scopes permissions to the specific pod/service account.

---

## Day 21: AWS Policy & Governance (Mini Project 7)
*IAM policies, AWS Config, compliance monitoring as code*

### AWS Config — 7 Essential Rules

| Rule | What it checks |
|---|---|
| `s3-bucket-public-write-prohibited` | Blocks public write on S3 buckets |
| `s3-server-side-encryption-enabled` | Ensures SSE on all S3 buckets |
| `scket-public-read-prohibited` | Blocks public read on S3 buckets |
| `encrypted-volumes` | All EBS volumes must be encrypted |
| `required-tags` | Resources must have Environment + Owner tags |
| `iam-password-policy` | Enforces strong password requirements |
| `root-account-mfa-enabled` | Root account must have MFA |

### IAM Policy Patterns

```json
// MFA Delete — deny S3 delete without MFA
{ "Effect": "Deny", "Action": "s3:DeleteObject",
  "Condition": { "BoolIfExists": { "aws:MultiFactorAuthPresent": lse } } }

// Encryption in transit — deny HTTP
{ "Effect": "Deny", "Principal": "*", "Action": "s3:*",
  "Condition": { "Bool": { "aws:SecureTransport": false } } }

// Required tags on EC2
{ "Effect": "Deny", "Action": "ec2:RunInstances",
  "Condition": { "StringNotEquals": { "aws:RequestedRegion": ["us-east-1"] } } }
```

### AWS Config Architecture

Configuration Recorder (enabled) → Delivery Channel (S3 bucket) → Config Rules (evaluate) → Findings → SNS/EventBridge → Remediation Lambda

###Hierarchy

| Layer | Scope |
|---|---|
| SCPs (AWS Orgs) | Max permission boundary — applies to all accounts in OU |
| Permission Boundaries | Max permissions for a specific IAM entity (role/user) |
| IAM Policies | Actual permissions granted |
| Resource Policies | S3 bucket policy, KMS key policy — cross-account access |
| Session Policies | Passed when assuming a role — further restricts temporary creds |

> **Interview Tip:** AWS Config is detective (finds non-compliance after the fact). SCPs are tive (block actions before they happen). Use both: SCPs for guardrails, Config for compliance reporting and drift detection.

---

## Day 22: RDS Database + Modular Architecture (Mini Project 8)
*Module composition, VPC + RDS + EC2, security groups*

### Modular Architecture Pattern

```
root/main.tf calls:
  module "vpc"   → outputs vpc_id, subnet_ids
  module "sg"    → inputs vpc_id → outputs sg_ids
  module "rds"   → inputs vpc_id, subnet_ids, sg_id → outputs endpoint
  module "ec2"   → input sg_id, rds_endpoint → outputs public_ip
```

### RDS Security Architecture

EC2 (public subnet) → Security Group allows MySQL:3306 from EC2 SG only → RDS (private subnets, multi-AZ) → DB Subnet Group spans 2+ AZs

### RDS Key Terraform Resources

| Resource | Purpose |
|---|---|
| `aws_db_instance` | Core RDS resource — engine, version, storage, `multi_az`, `backup_retention` |
| `aws_db_subnet_group` | Points to 2+ private subnets — required for RDS |
| `aws_db_parameter_group` | MySQL/Postgreslow_query_log`, `max_connections` |
| `aws_secretsmanager_secret` | Store credentials; reference in `aws_db_instance` password |

### RDS Production Settings

```hcl
resource "aws_db_instance" "prod" {
  engine                      = "mysql"
  engine_version              = "8.0"
  multi_az                    = true        # automatic failover
  backup_retention_period     = 7           # days
  storage_encrypted           = true        # KMS encryption
  deletion_protection         = true        # prevent accidental delete
  skip_final_snapshot         = false       # snapshot on destroy
  performance_insights_enabled = true       # monitoring
}
```

### Module Best Practices

- Module = reusable set of resources with clear inputs/outputs interface
- Never use hardcoded values inside modules — all config via variables
- Module outputs expose only what callers need — hide internal resource IDs
- Use module source versioning: `source = "./modules/rds"` or registry with version

> **Interview Tip:** Explain Multi-AZ vs Read Replica: Multi-AZ is synchronous replication for HA/failover (same region, same data). Read Replica is async replication for read scaling or cross-region DR. Aurora uses shared storage across 6 copies in 3 AZs.

---

## Day 23: S3 Security & Monitoring (Mini Project 9)
*CloudTrail + CloudWatch + Metric Filters + Alarms + SNS*

### Monitoring Stack Architecture

S3 Bucket → CloudTrail (data events) → CloudWatch Logs → Metric Filters → CloudWatch Alarms → SNS → Email/PagerDuty

# Resources

| Resource | Purpose |
|---|---|
| `aws_cloudtrail` | Enable S3 data event logging → CloudWatch Logs |
| `aws_cloudwatch_log_group` | Log destination for CloudTrail |
| `aws_cloudwatch_log_metric_filter` | Pattern match: AccessDenied, `private/*` access |
| `aws_cloudwatch_metric_alarm` | Alarm threshold = 1 (any hit triggers alert) |
| `aws_sns_topic` + `aws_sns_topic_subscription` | Email/HTTPS notification endpoint |

### Key Metric Filter Patterns

```
# Access Denied
filterPattern = "{ ($rrorCode = AccessDenied) || ($.errorCode = 403) }"

# Restricted prefix access
filterPattern = "{ $.requestParameters.key = private/* }"

# Root account usage
filterPattern = '{ $.userIdentity.type = "Root" }'
```

### CloudTrail Management vs Data Events

| | Management Events | Data Events |
|---|---|---|
| What | IAM, VPC, EC2 create/delete | S3 GetObject, PutObject, Lambda invoke |
| Cost | Free (1 trail) | Extra charge per 100K events |
| Enabled by default | ✅ Yes (in Console) | ❌ No — must enablicitly |
| Volume | Low | Can be very high — filter by bucket |

### Alert Pipeline Test

```bash
BUCKET=$(terraform output -raw monitored_bucket_name)
aws s3 cp s3://$BUCKET/private/secret.txt ./  # triggers alarm
aws s3 cp s3://$BUCKET/ghost.txt .            # 403 triggers denied alarm
```

> **Interview Tip:** CloudTrail has 5-15 min lag before events appear in CloudWatch Logs. Not real-time. For real-time security events use GuardDuty (ML-based threat detection) or EventBridge with S3 event notificatns.

---

## Day 24: HA + Auto-Scaling Infrastructure (Mini Project 10)
*ALB + ASG + multi-AZ + NAT Gateway — production pattern*

### Architecture

Internet → ALB (public subnets, 2 AZs) → ASG EC2 (private subnets, min:1, desired:2, max:5) → NAT Gateway (outbound) → Django Docker on port 8000

### Terraform Files

| File | Contents |
|---|---|
| `vpc.tf` | VPC, 2 public + 2 private subnets, IGW, 2 NAT GWs, route tables |
| `security_groups.tf` | ALB SG (0.0.0.0:80), EC2 SG (ALB SG only) |
| `alb._lb`, `aws_lb_target_group`, `aws_lb_listener` |
| `asg.tf` | `aws_launch_template`, `aws_autoscaling_group`, scaling policies |
| `s3.tf` | Optional: static assets, ALB access logs |

### Key Scaling Settings

```hcl
resource "aws_autoscaling_policy" "scale_up" {
  scaling_adjustment = 1
  adjustment_type    = "ChangeInCapacity"
  cooldown           = 300
}
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  metric_name   = "CPUUtilization"
  threshold     = 70
  alarm_actions = [aws_autoscaling_policy.scale_up.arn]
}
```

### NAT Gateway — HA Pattern

One NAT GW per AZ (2x cost but no single-AZ failure). Route each private subnet's table to its local NAT GW. Losing one AZ doesn't affect the other.

### ALB Health Check + Zero-Downtime Deploy

```hcl
health_check { path = "/health"; interval = 30; healthy_threshold = 2 }
```

During ASG instance refresh: new instances must pass health check before old ones are terminated. Set `min_healthy_percentage = 90` for rolling zero-downtime.

> **Interview Tip:** B vs NLB: ALB is Layer 7 (HTTP/HTTPS path/host routing, WAF integration). NLB is Layer 4 (TCP/UDP, static IP, lowest latency, preserves client IP). Use ALB for web apps; NLB for high-performance TCP or when you need static IPs.

---

## Day 25: Terraform Import (Real-time Project 2)
*Bring existing AWS resources under Terraform management*

### When to Use Import

Resources were created manually (console/CLI) and you want IaC going forward. Migrations from other IaC tools. Disaster recovery — state file lt, need to re-adopt existing infra.

### Import Methods

| Method | How | When |
|---|---|---|
| CLI import | `terraform import <resource> <aws_id>` | One-off, quick |
| Import block (≥1.5) | Declarative in `.tf` file — plan shows import | Preferred in modern Terraform |
| `terraform-import` | Third-party tool that generates tf code | Large migrations |

### CLI Import

```bash
# Syntax: terraform import <resource_address> <cloud_resource_id>
terraform import aws_s3_bucket.main my-existing-bucket-name
tform import aws_vpc.main vpc-0abc123def
terraform import 'aws_iam_user.users["alice"]' alice
```

### Import Block (Terraform ≥ 1.5 — Preferred)

```hcl
import {
  to = aws_s3_bucket.main
  id = "my-existing-bucket-name"
}

# Then: terraform plan shows "will import"
# terraform apply performs the import
# terraform generate-config-out=generated.tf  # auto-generate resource block
```

### Import Workflow

1. Write empty resource block (or use `generate-config-out`)
2. `terraform import` / import block
3.rraform plan` — should show 0 changes (resource already in desired state)
4. If plan shows changes, update your `.tf` to match existing state
5. Commit `.tf` + state to source control

### Common Import IDs

| Resource | ID Format |
|---|---|
| `aws_s3_bucket` | bucket name |
| `aws_vpc` | vpc-id |
| `aws_security_group` | sg-id |
| `aws_iam_user` | username |
| `aws_instance` | instance-id (i-xxx) |
| `aws_route53_record` | `zone_id_name_type` |

> **Interview Tip:** `terraform import` only updates state it does NOT generate `.tf` code (unless you use `generate-config-out`). After import, plan must show no changes before your config is truly in sync.

---

## Day 26: Terraform Cloud & Workspaces
*Remote state, remote runs, workspace management*

### Terraform Cloud vs Local

| Feature | Local Terraform | Terraform Cloud |
|---|---|---|
| State storage | Local file or S3 | Managed, versioned, encrypted |
| State locking | DynamoDB/S3 | Built-in |
| Runs | Local machine or CI/CD | Remote runs in HCP infra |
| Secrets/vars | tfvars / env vars | Workspace variables (sensitive) |
| RBAC | IAM only | Teams + workspace permissions |
| Audit log | N/A | Full run history + who applied |
| VCS integration | Manual | Auto-trigger on PR/push |

### Workspace Concepts

A workspace = isolated state + variable set + run history. Use one workspace per environment (dev/staging/prod) per component.

### CLI Workspaces (local) — Multi-Env Pattern

```bash
terraform workspace new dev
terraform workspace new prod
terraform worpace select dev
terraform workspace list

# Reference in config
locals { env = terraform.workspace }  # "dev" or "prod"
resource "aws_s3_bucket" "main" { bucket = "app-${local.env}-bucket" }
```

### Terraform Cloud Backend Config

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces { name = "prod-infra" }
  }
}
```

### Variable Scoping

| Scope | Details |
|---|---|
| Workspace variables | Set per workspace in UI/API — override tfvars |
| Variable sets | Shared across workspaces — ider credentials, common tags |
| Sensitive variables | Encrypted at rest, masked in logs |

> **Interview Tip:** Terraform Cloud workspaces ≠ CLI workspaces. CLI workspaces share the same backend config and just prefix the state key. TF Cloud workspaces are independent environments with separate credentials, runs, and access controls.

---

## Day 27: Production CI/CD for Terraform Infrastructure
*GitHub Actions pipeline, separation of concerns, infra repo patterns*

### Infrastructure Repository PatternKeep application code and infrastructure code in SEPARATE repos. Infra repo only manages AWS resources. App repo builds Docker images / AMIs. They communicate via image tags or AMI IDs.

### GitHub Actions Terraform Workflow

```yaml
on:
  pull_request:  # plan + security scan
  push:          # apply on branch merge

jobs:
  plan:
    - uses: aws-actions/configure-aws-credentials
    - run: terraform init && terraform plan -out=tfplan
    - uses: actions/upload-artifact  # save plan for apply job

  apply:  # only on push to main
    needs: [plan]
    - uses: actions/download-artifact
    - run: terraform apply tfplan
```

### Security Checks in Pipeline

| Tool | Purpose |
|---|---|
| `tflint` | Terraform linter — catches bad practices, provider errors |
| `checkov` | Security/compliance scanner — CIS benchmarks, CVEs |
| `tfsec` | Security scanner for Terraform code |
| `infracost` | Cost estimation in PRs — shows $ impact of changes |
| `terraform fmt -check` | Fail pipeline if code isn't formatted  OIDC Auth for GitHub Actions → AWS (No Static Keys)

```bash
# 1. Create OIDC Provider in AWS (token.actions.githubusercontent.com)
# 2. Create IAM Role with trust policy for specific repo/branch
# 3. In workflow:
permissions: { id-token: write }
uses: aws-actions/configure-aws-credentials@v4
with: { role-to-assume: arn:aws:iam::123:role/GitHubActionsRole }
```

### Branch Strategy for Infra

| Branch | Behaviour |
|---|---|
| `feature/*` | Plan only — PR comment shows changes + cost estimate |
| `mainAuto-apply to dev environment |
| `release/*` | Manual approval → apply to staging → apply to prod |

> **Interview Tip:** OIDC federation eliminates long-term AWS credentials in GitHub. The workflow exchanges a GitHub-signed JWT for temporary AWS credentials via STS `AssumeRoleWithWebIdentity`. Zero secrets stored in GitHub.

---

## Day 28: 3-Tier AWS Production Application
*Frontend + Backend + DB with HA, autoscaling, bastion, Secrets Manager*

### Architecture

Internet → Public ALB → Frontend e.js, public subnets) → Internal ALB → Backend ASG (Go, backend subnets) → RDS PostgreSQL (private subnets) + Secrets Manager

### Subnet Layout

| Subnet | Contents |
|---|---|
| Public subnets (2 AZs) | ALB, NAT Gateway, Bastion host |
| Frontend subnets (2 AZs) | Frontend EC2/ASG — can reach internet via NAT |
| Backend subnets (2 AZs) | Backend EC2/ASG — no direct internet |
| Database subnets (2 AZs) | RDS PostgreSQL — only backend SG allowed |

### Key Security Architecture

- Bastion hostubnet — SSH jump server for private instances
- Frontend EC2 only accessible from public ALB SG
- Backend EC2 only accessible from internal ALB SG
- RDS only accessible from backend ASG SG — no internet path
- Secrets Manager stores DB password — EC2 instances read via IMDS role

### Instance Refresh for Rolling Deploys

```hcl
resource "aws_autoscaling_group" "frontend" {
  instance_refresh {
    strategy    = "Rolling"
    preferences { min_healthy_percentage = 90 }
  }
}

# Trigger refresh after nee push:
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name prod-frontend-asg
```

### Secrets Manager Pattern

```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/credentials"
}
locals { db_creds = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string) }
# access: local.db_creds.username / local.db_creds.password
```

> **Interview Tip:** For truly zero-downtime upgrades: decouple frontend from backend API versioning. Use API versioning (v1, v2). Both versions active during rolling deploy. Frontend speaks v1 until all backend pods are on v2.

---

## Day 29: GitOps with ArgoCD + EKS + Kustomize
*Full GitOps workflow: Git → ArgoCD → Kubernetes*

### GitOps Principles

1. Git is the single source of truth for desired cluster state
2. All changes go through Git (PRs, not `kubectl apply`)
3. Approved changes are automatically applied to cluster
4. Cluster continuously converges toward Git state (self-healing)

### ArgoCD Architecture

ArgoCD watches Gito → compares to cluster state → syncs differences → reports health

### Terraform Provisions

```
VPC + EKS cluster (terraform-aws-modules/eks/aws v21)
ArgoCD installed via Helm provider
ArgoCD Application CR deployed via kubectl/Terraform
```

### Kustomize Concepts

| Concept | Purpose |
|---|---|
| `kustomization.yaml` | Entry point — lists resources, patches, image overrides |
| Base | Core manifests shared across environments |
| Overlay | Environment-specific patches (dev/staging/prod) |
| patodify specific fields without duplicating full manifests |
| images | Override container image tags in one place |
| replicas | Control replica counts per environment |

### ArgoCD App Manifest

```yaml
spec:
  source:
    repoURL: https://github.com/myorg/gitops-manifests
    path: environments/dev
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true        # delete resources removed from Git
      selfHeal: true     # revert manual kubectl changes
```

### Troubleshooting ArgoCD

```bash
kubectl get application -n argocd
kubectl describe application myapp -n argocd
argocd app sync myapp
argocd app logs myapp
```

> **Interview Tip:** ArgoCD sync vs auto-sync: manual sync requires operator approval. auto-sync (prune+selfHeal) is full GitOps but can auto-delete resources if you're not careful. Use automated for dev, manual approval for prod.

---

## Day 30: Terraform Drift Detection & Auto-Remediation
*Scheduled GitHub Actions, detect + auto-fix + notify*

### What is Drift?

Drift = difference between Terraform state and actual cloud infrastructure. Causes: manual console changes, auto-scaling events, external tooling. Unmanaged drift leads to failed deployments and security gaps.

### Detection Flow

GitHub Actions cron → `terraform plan` → parse exit code → exit 0 = no drift, exit 2 = drift detected → auto-apply OR create GitHub Issue + Slack alert

### GitHub Actions Workflow

```yaml
on:
  schedule:  '*/5 * * * *' }]  # every 5 min
  workflow_dispatch:                     # manual trigger

steps:
  - terraform plan -detailed-exitcode -out=tfplan
  # exit 0 = no changes, exit 1 = error, exit 2 = changes present
  - if: steps.plan.outputs.exitcode == '2'
    run: terraform apply -auto-approve tfplan
  - if: failure()
    uses: actions/github-script  # create GitHub issue
```

### Drift Detection vs Prevention

| Approach | Tool | When Triggered |
|---|---|---|
| Detection | `terraform plan -detailed-exitcode` | Scheduled (cron) |
| Prevention | SCPs + IAM deny policies | At API call time |
| Remediation | `terraform apply` (auto or manual) | After drift detected |
| Notification | Slack webhook / GitHub Issues | On drift or failure |

### Exit Codes (Critical for Automation)

| Code | Meaning |
|---|---|
| `0` | Plan = no changes (state matches config) |
| `1` | Terraform encountered an error |
| `2` | Plan succeeded, changes present (drift detected) — requires `-detailed-exitcode` flag |

### Force Unlo Stuck State

```bash
terraform force-unlock <LOCK_ID>
```

Get lock ID from error message. Only use if you're certain no other process is running.

> **Interview Tip:** Auto-remediation is powerful but dangerous. Gate it: only auto-apply tag changes, security group additions. For destructive changes (instance replace, subnet delete), require human approval via PR or `workflow_dispatch` + required reviewers.

---

## Day 31: Drift Detection with Terraform Cloud
*Native drift detection, health assessments, notifications*

### Terraform Cloud Drift Detection vs GitHub Actions

| Feature | GitHub Actions (Day 30) | Terraform Cloud |
|---|---|---|
| Setup | Workflow YAML + OIDC | Toggle in workspace UI |
| Drift check freq | Cron you define | Every 24h (auto) or on-demand |
| State access | Backend config needed | Native — no extra config |
| Cost | Free (GH Actions minutes) | Included in Plus/Enterprise |
| Auto-remediation | Manual in workflow | Configurable in workspace |
| Notifications | Slack webhook code Native integrations |

### Terraform Cloud Health Assessments

Health assessments run plan in read-only mode, compare to state, generate drift report in UI. Configure: Workspace → Settings → Health → Enable drift detection.

### Structured Run Tasks

Run tasks integrate third-party tools (Sentinel, Snyk, Infracost) into the plan/apply pipeline. Block apply if policy violated.

### Full Course Summary — What You Can Now Do

| Skill | Topics Covered |
|---|---|
| Write IaC | Variables, types, functioitionals, dynamic blocks |
| Manage state | Remote backend, locking, import, workspaces |
| Build modules | Reusable, parameterised, composable |
| Deploy real infra | VPC, EKS, RDS, ALB, ASG, Lambda, CloudFront |
| Secure infra | IAM, SCP, Config rules, Secrets Manager, OIDC |
| Run CI/CD | GitHub Actions, plan-on-PR, apply-on-merge, OIDC auth |
| GitOps | ArgoCD, Kustomize, self-healing, auto-sync |
| Monitor | CloudTrail, CloudWatch, SNS, drift detection |

### Quick Reference — The Commands That MatteMost

```bash
terraform init / plan / apply / destroy
terraform fmt -recursive && terraform validate
terraform state list / show / mv / rm / import
terraform workspace new/select/list
terraform force-unlock <id>
aws eks update-kubeconfig --region us-east-1 --name cluster
kubectl get pods -A / kubectl describe / kubectl logs -f
argocd app sync / argocd app logs
```

---

> ✅ **You're Ready** — You can now discuss, design, implement, and troubleshoot production Terraform + AWS + Kubernetes infrastructure.d luck in your interviews and on the job!
