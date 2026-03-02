# Terraform Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## Fluxo Principal

```bash
terraform init                        # baixar providers e módulos
terraform init -upgrade               # atualizar providers para o mais recente permitido
terraform init -reconfigure           # reconfigurar backend
terraform fmt                         # formatar todos os arquivos .tf
terraform fmt -check                  # verificar formatação (uso em CI)
terraform validate                    # validar sintaxe e configuração
terraform plan                        # mostrar plano de execução
terraform plan -out=tfplan            # salvar plano em arquivo
terraform plan -target=aws_instance.web  # planejar um recurso específico
terraform apply                       # aplicar com confirmação
terraform apply tfplan                # aplicar plano salvo (sem prompt)
terraform apply -auto-approve         # pular confirmação
terraform apply -target=aws_instance.web
terraform destroy                     # destruir todos os recursos gerenciados
terraform destroy -auto-approve
terraform destroy -target=aws_instance.web
```

---

## Comandos de State

```bash
terraform state list                  # listar todos os recursos gerenciados
terraform state show aws_instance.web # mostrar state de um recurso
terraform state mv aws_instance.old aws_instance.new  # renomear recurso no state
terraform state rm aws_instance.web   # remover recurso do state (não destrói)
terraform state pull                  # baixar e exibir state remoto
terraform state push terraform.tfstate # enviar state local para o remoto
terraform state replace-provider \
  registry.terraform.io/hashicorp/aws \
  registry.terraform.io/custom/aws   # substituir provider no state
terraform refresh                     # sincronizar state com infra real (obsoleto: use apply -refresh-only)
terraform apply -refresh-only         # atualizar state sem aplicar mudanças
```

---

## Comandos de Workspace

```bash
terraform workspace list              # listar workspaces
terraform workspace new staging       # criar workspace
terraform workspace select staging    # trocar workspace
terraform workspace show              # nome do workspace atual
terraform workspace delete staging    # deletar workspace (não pode ser o atual)
```

```hcl
# Usar workspace na configuração
resource "aws_s3_bucket" "dados" {
  bucket = "minha-app-${terraform.workspace}-dados"
}
```

---

## Sintaxe de Resource e Data Source

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

# Data source (lê infra existente, sem gerenciamento de ciclo de vida)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

# Referenciar data source
resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
}

# Meta-argumento lifecycle
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags, user_data]
  }
}

# Dependência explícita
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy.policy]
}
```

---

## Variáveis

```hcl
# Variáveis de entrada (variables.tf)
variable "region" {
  description = "Região AWS"
  type        = string
  default     = "us-east-1"
}

variable "cidr_permitido" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}

variable "tags" {
  type    = map(string)
  default = {}
}

variable "tipo_instancia" {
  type = string
  validation {
    condition     = contains(["t3.micro", "t3.small"], var.tipo_instancia)
    error_message = "Deve ser t3.micro ou t3.small."
  }
}

# Valores locais (locals.tf)
locals {
  tags_comuns = merge(var.tags, {
    Projeto    = "minha-app"
    GeridoPor  = "terraform"
  })
  prefixo_env = "${var.environment}-${var.projeto}"
}

# Outputs (outputs.tf)
output "ip_instancia" {
  value       = aws_instance.web.public_ip
  description = "IP público do servidor web"
  sensitive   = false
}

output "senha_db" {
  value     = random_password.db.result
  sensitive = true    # mascarado na saída, ainda armazenado no state
}
```

```bash
# Passar variáveis
terraform apply -var="region=eu-west-1"
terraform apply -var-file="prod.tfvars"
# Arquivos carregados automaticamente: terraform.tfvars, *.auto.tfvars
# Variáveis de ambiente: TF_VAR_region=eu-west-1
```

---

## Módulos

```hcl
# Chamando um módulo
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "minha-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}

# Módulo local
module "app" {
  source = "./modules/app"

  tipo_instancia = "t3.small"
  vpc_id         = module.vpc.vpc_id
}

# Output de módulo
output "vpc_id" {
  value = module.vpc.vpc_id
}
```

```
# Estrutura de diretório de módulo
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
    tags = local.tags_comuns
  }
}

# Múltiplas instâncias de provider (alias)
provider "aws" {
  alias  = "eu"
  region = "eu-west-1"
}

resource "aws_s3_bucket" "bucket_eu" {
  provider = aws.eu
  bucket   = "meu-bucket-eu"
}
```

---

## Funções Comuns

```hcl
# Conversão de tipo
toset(["a", "b", "a"])        # => {"a", "b"}  (dedup + sem ordem)
tolist(toset(["b", "a"]))     # => ["a", "b"]  (ordenado)
tomap({a = 1, b = 2})

# Coleções
merge({a = 1}, {b = 2, a = 3})           # => {a = 3, b = 2}
concat(["a", "b"], ["c"])                 # => ["a", "b", "c"]
flatten([["a", "b"], ["c"]])              # => ["a", "b", "c"]
length(var.lista)
keys(var.mapa)
values(var.mapa)
lookup(var.mapa, "chave", "padrão")       # acesso seguro com valor padrão
contains(var.lista, "valor")

# String
format("Olá, %s!", var.nome)
formatlist("Olá, %s!", var.nomes)         # aplicar a uma lista
join(", ", ["a", "b", "c"])               # => "a, b, c"
split(",", "a,b,c")                       # => ["a", "b", "c"]
trimspace("  olá  ")
replace("hello world", "world", "there")
regex("(\\d+)", "abc123")

# Condicionais
coalesce("", null, "fallback")            # => "fallback" (primeiro não-vazio/não-nulo)
coalesce(var.nome_custom, "padrão")
try(jsondecode(var.json), {})             # => {} se decodificação falhar
can(regex("^[a-z]+$", var.nome))          # => bool (true se não der erro)

# Codificação
base64encode("olá")
base64decode("b2zDoQ==")
jsonencode({chave = "valor"})
jsondecode("{\"chave\":\"valor\"}")
```

---

## Loops

```hcl
# count (baseado em índice, simples)
resource "aws_iam_user" "devs" {
  count = length(var.nomes_dev)
  name  = var.nomes_dev[count.index]
}
# Referência: aws_iam_user.devs[0].arn

# for_each (baseado em chave, preferível para mapas/sets)
resource "aws_iam_user" "devs" {
  for_each = toset(var.nomes_dev)
  name     = each.value
}
# Referência: aws_iam_user.devs["alice"].arn

resource "aws_s3_bucket" "ambientes" {
  for_each = {
    dev  = "us-east-1"
    prod = "eu-west-1"
  }
  bucket = "minha-app-${each.key}"
  tags = {
    Regiao = each.value
  }
}

# Expressão for (transformar coleções)
locals {
  nomes_maiusc = [for nome in var.nomes : upper(nome)]
  mapa_nomes   = {for nome in var.nomes : nome => upper(nome)}
  nomes_longos = [for nome in var.nomes : nome if length(nome) > 4]
}

# dynamic blocks (loop em blocos aninhados)
resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.regras_ingress
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

## Blocos Import e Move

```hcl
# Importar recurso existente (Terraform 1.5+)
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}

# Mover recurso (renomear sem destruir/recriar)
moved {
  from = aws_instance.nome_antigo
  to   = aws_instance.nome_novo
}

# Mover para dentro de módulo
moved {
  from = aws_instance.web
  to   = module.app.aws_instance.web
}
```

```bash
# Import via CLI (anterior ao 1.5)
terraform import aws_instance.web i-0abc123def456
terraform import 'aws_iam_user.devs["alice"]' alice   # recurso com for_each
terraform import 'aws_iam_user.devs[0]' alice          # recurso com count
```

---

## Configuração de Backend

```hcl
# Backend S3 (state remoto + locking com DynamoDB)
terraform {
  backend "s3" {
    bucket         = "meu-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# Backend local (padrão)
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}
```

---

## Flags e Variáveis de Ambiente Úteis

```bash
# Flags de CLI
-var="chave=valor"              # definir variável
-var-file="arquivo.tfvars"      # carregar arquivo de variáveis
-target=recurso.nome            # limitar a um recurso
-replace=recurso.nome           # forçar substituição (substitui o taint)
-refresh=false                  # pular refresh do state (plan mais rápido)
-compact-warnings               # reduzir verbosidade de warnings
-json                           # saída legível por máquina
-no-color                       # desativar cores ANSI (CI)
-lock=false                     # pular locking do state (perigoso)
-parallelism=10                 # operações simultâneas (padrão: 10)

# Variáveis de ambiente
TF_VAR_nome=valor               # sobrescrever variável de entrada
TF_LOG=DEBUG                    # nível de log: TRACE|DEBUG|INFO|WARN|ERROR
TF_LOG_PATH=./terraform.log     # salvar log em arquivo
TF_CLI_ARGS_plan="-no-color"    # flags padrão para subcomando
TF_DATA_DIR="./.terraform"      # diretório de cache de plugins
TF_WORKSPACE=staging            # selecionar workspace
AWS_PROFILE=meuperfil           # usar perfil nomeado do AWS CLI
```
