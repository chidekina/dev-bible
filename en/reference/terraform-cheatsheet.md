# Terraform Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## Core Workflow

```bash
terraform init                        # download providers + modules
terraform init -upgrade               # upgrade providers to latest allowed
terraform init -reconfigure           # reconfigure backend
terraform fmt                         # format all .tf files
terraform fmt -check                  # check formatting (CI use)
terraform validate                    # validate syntax + config
terraform plan                        # show execution plan
terraform plan -out=tfplan            # save plan to file
terraform plan -target=aws_instance.web  # plan single resource
terraform apply                       # apply with confirmation
terraform apply tfplan                # apply saved plan (no prompt)
terraform apply -auto-approve         # skip confirmation
terraform apply -target=aws_instance.web
terraform destroy                     # destroy all managed resources
terraform destroy -auto-approve
terraform destroy -target=aws_instance.web
```

---

## State Commands

```bash
terraform state list                  # list all managed resources
terraform state show aws_instance.web # show state for resource
terraform state mv aws_instance.old aws_instance.new  # rename resource in state
terraform state rm aws_instance.web   # remove resource from state (won't destroy)
terraform state pull                  # download + display remote state
terraform state push terraform.tfstate # upload local state to remote
terraform state replace-provider \
  registry.terraform.io/hashicorp/aws \
  registry.terraform.io/custom/aws   # replace provider in state
terraform refresh                     # sync state with real infra (deprecated: use apply -refresh-only)
terraform apply -refresh-only         # update state without changes
```

---

## Workspace Commands

```bash
terraform workspace list              # list workspaces
terraform workspace new staging       # create workspace
terraform workspace select staging    # switch workspace
terraform workspace show              # current workspace name
terraform workspace delete staging    # delete workspace (must not be current)
```

```hcl
# Use workspace in config
resource "aws_s3_bucket" "data" {
  bucket = "my-app-${terraform.workspace}-data"
}
```

---

## Resource & Data Source Syntax

```hcl
# Resource
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name        = "web-server"
    Environment = var.environment
  }
}

# Data source (read existing infra, no lifecycle management)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

# Reference data source
resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
}

# Lifecycle meta-argument
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags, user_data]
  }
}

# Depends on
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy.policy]
}
```

---

## Variables

```hcl
# Input variables (variables.tf)
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "allowed_cidr" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}

variable "tags" {
  type    = map(string)
  default = {}
}

variable "instance_type" {
  type = string
  validation {
    condition     = contains(["t3.micro", "t3.small"], var.instance_type)
    error_message = "Must be t3.micro or t3.small."
  }
}

# Local values (locals.tf)
locals {
  common_tags = merge(var.tags, {
    Project     = "my-app"
    ManagedBy   = "terraform"
  })
  env_prefix = "${var.environment}-${var.project}"
}

# Outputs (outputs.tf)
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of web server"
  sensitive   = false
}

output "db_password" {
  value     = random_password.db.result
  sensitive = true    # masked in CLI output, still stored in state
}
```

```bash
# Pass variables
terraform apply -var="region=eu-west-1"
terraform apply -var-file="prod.tfvars"
# Auto-loaded files: terraform.tfvars, *.auto.tfvars
# Environment variables: TF_VAR_region=eu-west-1
```

---

## Modules

```hcl
# Calling a module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}

# Local module
module "app" {
  source = "./modules/app"

  instance_type = "t3.small"
  vpc_id        = module.vpc.vpc_id
}

# Output from module
output "vpc_id" {
  value = module.vpc.vpc_id
}
```

```
# Module directory layout
modules/
  app/
    main.tf
    variables.tf
    outputs.tf
    versions.tf
```

---

## Providers

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.0"
    }
  }
}

provider "aws" {
  region = var.region

  default_tags {
    tags = local.common_tags
  }
}

# Multiple provider instances (aliasing)
provider "aws" {
  alias  = "eu"
  region = "eu-west-1"
}

resource "aws_s3_bucket" "eu_bucket" {
  provider = aws.eu
  bucket   = "my-eu-bucket"
}
```

---

## Common Functions

```hcl
# Type conversion
toset(["a", "b", "a"])        # => {"a", "b"}  (dedup + unordered)
tolist(toset(["b", "a"]))     # => ["a", "b"]  (sorted)
tomap({a = 1, b = 2})

# Collections
merge({a = 1}, {b = 2, a = 3})           # => {a = 3, b = 2}
concat(["a", "b"], ["c"])                 # => ["a", "b", "c"]
flatten([["a", "b"], ["c"]])              # => ["a", "b", "c"]
length(var.list)
keys(var.map)
values(var.map)
lookup(var.map, "key", "default")         # safe map access with default
contains(var.list, "value")

# String
format("Hello, %s!", var.name)
formatlist("Hello, %s!", var.names)       # apply to list
join(", ", ["a", "b", "c"])               # => "a, b, c"
split(",", "a,b,c")                       # => ["a", "b", "c"]
trimspace("  hello  ")
replace("hello world", "world", "there")
regex("(\\d+)", "abc123")

# Conditionals
coalesce("", null, "fallback")            # => "fallback" (first non-empty/non-null)
coalesce(var.custom_name, "default")
try(jsondecode(var.json), {})             # => {} if decode fails
can(regex("^[a-z]+$", var.name))          # => bool (true if no error)

# Encoding
base64encode("hello")
base64decode("aGVsbG8=")
jsonencode({key = "value"})
jsondecode("{\"key\":\"value\"}")
```

---

## Loops

```hcl
# count (index-based, simple)
resource "aws_iam_user" "devs" {
  count = length(var.developer_names)
  name  = var.developer_names[count.index]
}
# Reference: aws_iam_user.devs[0].arn

# for_each (key-based, preferred for maps/sets)
resource "aws_iam_user" "devs" {
  for_each = toset(var.developer_names)
  name     = each.value
}
# Reference: aws_iam_user.devs["alice"].arn

resource "aws_s3_bucket" "envs" {
  for_each = {
    dev  = "us-east-1"
    prod = "eu-west-1"
  }
  bucket = "my-app-${each.key}"
  tags = {
    Region = each.value
  }
}

# for expression (transform collections)
locals {
  upper_names = [for name in var.names : upper(name)]
  name_map    = {for name in var.names : name => upper(name)}
  long_names  = [for name in var.names : name if length(name) > 4]
}

# dynamic blocks (loop over nested blocks)
resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

---

## Import & Move Blocks

```hcl
# Import existing resource (Terraform 1.5+)
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}

# Move resource (rename without destroy/recreate)
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}

# Move into module
moved {
  from = aws_instance.web
  to   = module.app.aws_instance.web
}
```

```bash
# Pre-1.5 import (CLI)
terraform import aws_instance.web i-0abc123def456
terraform import 'aws_iam_user.devs["alice"]' alice   # for_each resource
terraform import 'aws_iam_user.devs[0]' alice          # count resource
```

---

## Backend Config

```hcl
# S3 backend (remote state + locking with DynamoDB)
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# Local backend (default)
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}
```

---

## Useful CLI Flags & Environment Variables

```bash
# CLI flags
-var="key=value"              # set variable
-var-file="file.tfvars"       # load var file
-target=resource.name         # limit to resource
-replace=resource.name        # force replacement (taint replacement)
-refresh=false                # skip state refresh (faster plan)
-compact-warnings             # reduce warning verbosity
-json                         # machine-readable output
-no-color                     # disable ANSI colors (CI)
-lock=false                   # skip state locking (dangerous)
-parallelism=10               # concurrent operations (default: 10)

# Environment variables
TF_VAR_name=value             # input variable override
TF_LOG=DEBUG                  # log level: TRACE|DEBUG|INFO|WARN|ERROR
TF_LOG_PATH=./terraform.log   # log to file
TF_CLI_ARGS_plan="-no-color"  # default flags for subcommand
TF_DATA_DIR="./.terraform"    # plugin cache dir
TF_WORKSPACE=staging          # select workspace
AWS_PROFILE=myprofile         # use AWS CLI named profile
```
