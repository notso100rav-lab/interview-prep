# Terraform Fundamentals

> Java Backend Engineer Interview Prep — Chapter 4.7

---

## Table of Contents
1. [What is Terraform?](#what-is-terraform)
2. [HCL Basics](#hcl-basics)
3. [Providers](#providers)
4. [Resources](#resources)
5. [Variables & Outputs](#variables--outputs)
6. [State Management](#state-management)
7. [Modules](#modules)
8. [Plan / Apply Lifecycle](#plan--apply-lifecycle)
9. [Q&A](#qa)

---

## What is Terraform?

Terraform is an open-source **Infrastructure as Code (IaC)** tool by HashiCorp. It lets you define cloud infrastructure declaratively in **HCL (HashiCorp Configuration Language)** and manage it through a consistent CLI workflow.

**Key concepts:**
- **Declarative**: you describe *what* you want, Terraform figures out *how* to get there
- **Idempotent**: running `terraform apply` twice does not create duplicate resources
- **Provider-agnostic**: works with AWS, GCP, Azure, Kubernetes, GitHub, and hundreds more
- **Drift detection**: compares real infrastructure to your code and shows differences

---

## HCL Basics

```hcl
# main.tf — a minimal Terraform file

terraform {
  required_version = ">= 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "app_assets" {
  bucket = "my-app-assets-${var.environment}"

  tags = {
    Environment = var.environment
    Project     = "interview-prep"
  }
}
```

**HCL syntax rules:**
- Blocks: `resource`, `provider`, `variable`, `output`, `module`, `data`, `locals`
- Strings: double-quoted; interpolation with `${...}`
- Comments: `#`, `//`, or `/* */`
- References: `<type>.<name>.<attribute>` e.g. `aws_s3_bucket.app_assets.id`

---

## Providers

A **provider** is a plugin that talks to an external API (AWS, GCP, Docker, etc.).

```hcl
# Multiple providers / multiple regions
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

resource "aws_vpc" "primary" {
  provider   = aws.us_east
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc" "dr" {
  provider   = aws.eu_west
  cidr_block = "10.1.0.0/16"
}
```

Providers are downloaded from the **Terraform Registry** when you run `terraform init`.

---

## Resources

Resources are the core building block — each one maps to an API object.

```hcl
# EC2 instance for a Spring Boot application
resource "aws_instance" "app_server" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.app.id]
  iam_instance_profile   = aws_iam_instance_profile.app.name
  key_name               = var.key_pair_name

  user_data = base64encode(<<-EOF
    #!/bin/bash
    yum update -y
    yum install -y java-17-amazon-corretto
    aws s3 cp s3://${var.artifact_bucket}/app.jar /opt/app.jar
    java -jar /opt/app.jar --server.port=8080 &
  EOF
  )

  tags = {
    Name        = "spring-boot-app"
    Environment = var.environment
  }

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [ami]  # don't replace on AMI updates
  }
}

# Security group
resource "aws_security_group" "app" {
  name   = "app-sg-${var.environment}"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 8080
    to_port     = 8080
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
```

**Resource meta-arguments:**
| Argument | Purpose |
|----------|---------|
| `depends_on` | Explicit ordering dependency |
| `count` | Create N copies of a resource |
| `for_each` | Create one resource per map/set element |
| `provider` | Override which provider alias to use |
| `lifecycle` | Control create/destroy/update behavior |

---

## Variables & Outputs

```hcl
# variables.tf
variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "db_password" {
  type      = string
  sensitive = true  # masked in plan output and state
}

# locals.tf — computed values
locals {
  name_prefix = "${var.environment}-myapp"
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# outputs.tf
output "app_server_public_ip" {
  value       = aws_instance.app_server.public_ip
  description = "Public IP of the application server"
}

output "rds_endpoint" {
  value     = aws_db_instance.postgres.endpoint
  sensitive = true
}
```

**Supplying variable values:**
```bash
# Command line
terraform apply -var="environment=prod" -var="instance_type=t3.small"

# .tfvars file (committed)
terraform apply -var-file="prod.tfvars"

# Environment variable
export TF_VAR_db_password="s3cr3t"
terraform apply
```

---

## State Management

Terraform stores the mapping between your configuration and real resources in a **state file** (`terraform.tfstate`).

### Remote State (team/production use)

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "apps/myapp/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true

    # State locking via DynamoDB
    dynamodb_table = "terraform-locks"
  }
}
```

### State Commands

```bash
# List all resources in state
terraform state list

# Show details of one resource
terraform state show aws_instance.app_server

# Move a resource (rename in code without destroying)
terraform state mv aws_instance.app_server aws_instance.web

# Import existing infrastructure into state
terraform import aws_s3_bucket.legacy my-existing-bucket

# Remove from state (don't destroy real resource)
terraform state rm aws_instance.old_server
```

**State locking:** Remote backends (S3 + DynamoDB, Terraform Cloud, GCS) prevent concurrent runs from corrupting state.

---

## Modules

Modules package reusable Terraform code.

```hcl
# modules/spring-boot-app/main.tf
variable "name"          {}
variable "environment"   {}
variable "instance_type" { default = "t3.micro" }
variable "vpc_id"        {}
variable "subnet_ids"    { type = list(string) }

resource "aws_launch_template" "app" { ... }
resource "aws_autoscaling_group" "app" { ... }
resource "aws_lb_target_group" "app" { ... }

output "target_group_arn" {
  value = aws_lb_target_group.app.arn
}

# root/main.tf — consuming the module
module "user_service" {
  source = "./modules/spring-boot-app"

  name         = "user-service"
  environment  = var.environment
  instance_type = "t3.small"
  vpc_id       = aws_vpc.main.id
  subnet_ids   = aws_subnet.private[*].id
}

module "order_service" {
  source = "./modules/spring-boot-app"

  name        = "order-service"
  environment = var.environment
  vpc_id      = aws_vpc.main.id
  subnet_ids  = aws_subnet.private[*].id
}

# Reference module output
resource "aws_lb_listener_rule" "user_service" {
  listener_arn = aws_lb_listener.https.arn

  action {
    type             = "forward"
    target_group_arn = module.user_service.target_group_arn
  }

  condition {
    path_pattern { values = ["/api/users/*"] }
  }
}
```

**Module sources:**
```hcl
source = "./local/path"
source = "hashicorp/consul/aws"          # Terraform Registry
source = "git::https://github.com/..."   # Git
source = "github.com/org/repo//subdir"   # GitHub shorthand
```

---

## Plan / Apply Lifecycle

```bash
# 1. Initialize — download providers & modules
terraform init

# 2. Format code (CI enforcement)
terraform fmt -check -recursive

# 3. Validate syntax and configuration
terraform validate

# 4. Preview changes (dry run)
terraform plan -out=tfplan

# 5. Apply the saved plan
terraform apply tfplan

# 6. Destroy infrastructure
terraform destroy -target=aws_instance.app_server  # selective
terraform destroy                                    # everything
```

### Typical CI/CD Pipeline

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths: ["infra/**"]
  push:
    branches: [main]
    paths: ["infra/**"]

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.x

      - name: Terraform Init
        run: terraform init
        working-directory: infra/

      - name: Terraform Fmt Check
        run: terraform fmt -check -recursive
        working-directory: infra/

      - name: Terraform Validate
        run: terraform validate
        working-directory: infra/

      - name: Terraform Plan
        run: terraform plan -out=tfplan
        working-directory: infra/
        env:
          TF_VAR_environment: prod
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

  apply:
    needs: plan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production   # requires manual approval
    steps:
      - name: Terraform Apply
        run: terraform apply tfplan
        working-directory: infra/
```

---

## Q&A

---

### Q1 — 🟢 What is the difference between `terraform plan` and `terraform apply`?

<details><summary>Click to reveal answer</summary>

`terraform plan` **previews** the changes Terraform will make to reach the desired state — it reads state and calls provider APIs (read-only) but makes no changes. It shows a diff of resources to add (`+`), change (`~`), or destroy (`-`).

`terraform apply` **executes** the changes. When run with a saved plan file (`-out=tfplan` → `apply tfplan`), it applies exactly those changes with no re-evaluation. Without a plan file, it runs plan internally first and prompts for confirmation.

**Best practice:** always use `plan -out` + `apply <file>` in CI to guarantee what was reviewed is what gets applied.

</details>

---

### Q2 — 🟡 What is Terraform state, and why is it important?

<details><summary>Click to reveal answer</summary>

State is a JSON file (`terraform.tfstate`) that records the mapping between your configuration identifiers (e.g. `aws_instance.app_server`) and the real resource IDs in the cloud provider (e.g. `i-0abc123def456`).

**Why it matters:**
1. **Drift detection** — Terraform compares real infrastructure to state to decide what to change.
2. **Dependency graph** — state records attribute values (like IDs) that other resources reference.
3. **Performance** — for large infrastructures, reading state is faster than querying every resource from the API.

**Problems without remote state:**
- Two engineers running `apply` simultaneously can corrupt state (race condition).
- Local state is lost if the machine is destroyed.

Remote backends (S3 + DynamoDB) solve both with encryption, versioning, and locking.

</details>

---

### Q3 — 🟡 Explain the difference between `count` and `for_each`.

<details><summary>Click to reveal answer</summary>

Both create multiple instances of a resource, but they handle identity differently:

```hcl
# count — indexed by integer
resource "aws_subnet" "public" {
  count             = 3
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
# Referenced as: aws_subnet.public[0], [1], [2]

# for_each — indexed by string key
resource "aws_subnet" "private" {
  for_each          = toset(["us-east-1a", "us-east-1b", "us-east-1c"])
  cidr_block        = ...
  availability_zone = each.key
}
# Referenced as: aws_subnet.private["us-east-1a"], etc.
```

**Key difference:** If you remove an element from a `count` list in the middle (e.g. remove index 1 of 3), Terraform will destroy/recreate resources `[1]` and `[2]`. With `for_each`, removing a key only affects that specific resource — safer for production.

**Rule of thumb:** use `for_each` with a map/set when resources have meaningful identifiers; use `count` only for truly identical copies.

</details>

---

### Q4 — 🟡 How do you handle sensitive values like database passwords in Terraform?

<details><summary>Click to reveal answer</summary>

**Option 1 — `sensitive = true` variable + environment variable:**
```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
# Supply via: export TF_VAR_db_password="$(aws secretsmanager get-secret-value ...)"
```

**Option 2 — Read from AWS Secrets Manager at plan time:**
```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/myapp/db_password"
}

resource "aws_db_instance" "postgres" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

**Option 3 — Use a dedicated secrets provider (Vault):**
```hcl
provider "vault" { address = "https://vault.internal" }

data "vault_generic_secret" "db" {
  path = "secret/myapp/db"
}
```

**Important:** Even with `sensitive = true`, values are still stored in plaintext in `terraform.tfstate`. Always encrypt state at rest (S3 SSE) and restrict access to the state bucket.

</details>

---

### Q5 — 🔴 What is a `lifecycle` block and when would you use `create_before_destroy`?

<details><summary>Click to reveal answer</summary>

The `lifecycle` block controls how Terraform handles resource replacement:

```hcl
resource "aws_instance" "app" {
  # ...

  lifecycle {
    create_before_destroy = true   # new instance before old is destroyed
    prevent_destroy       = true   # block terraform destroy (prod safeguard)
    ignore_changes        = [tags] # don't plan changes to these attributes
    replace_triggered_by  = [aws_launch_template.app.latest_version]
  }
}
```

**`create_before_destroy`** is critical for zero-downtime replacements. Normally Terraform destroys first then creates (which causes downtime). With this flag it creates the new resource, updates any references (e.g. ALB target groups), then destroys the old one.

**Common use cases:**
- Any stateless compute resource (EC2, ECS tasks, Lambda)
- TLS certificates (must exist before being attached to ALB)
- DNS records (new CNAME live before old one removed)

**Caveat:** resources with unique name constraints may need a `name_prefix` + auto-suffix instead of a fixed name, so two instances can coexist during transition.

</details>

---

### Q6 — 🔴 How does Terraform handle dependencies between resources?

<details><summary>Click to reveal answer</summary>

Terraform builds a **dependency graph** (DAG — Directed Acyclic Graph) to determine the correct order of operations.

**Implicit dependencies** — automatic when one resource references another's attributes:
```hcl
resource "aws_security_group" "app" { ... }

resource "aws_instance" "app" {
  # Terraform detects the reference and creates the SG first
  vpc_security_group_ids = [aws_security_group.app.id]
}
```

**Explicit dependencies** — use `depends_on` when there's a dependency that can't be expressed through attribute references:
```hcl
resource "aws_s3_bucket_policy" "app" {
  depends_on = [aws_s3_bucket_public_access_block.app]
  # The policy can only be applied after public access block is in place
}
```

**Parallelism:** Terraform applies independent resources in parallel (default: 10 concurrent operations, controlled with `-parallelism=N`). Resources with dependencies are serialized.

**Cycle detection:** Terraform will error if it detects a circular dependency (A depends on B, B depends on A). Resolution usually requires refactoring to break the cycle with an intermediate resource or data source.

</details>
