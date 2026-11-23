**Tarefa.** Aplicando o framework **C-A-R-E**, escrever o prompt de IA que produza o módulo Terraform S3 aderente ao padrão, no mesmo estilo do exemplo.
**Entregue.** Prompt, modelo, output e justificativa mostrando como Context, Action, Result e Example aparecem no prompt.


## 1. Prompt

```

# Context
Strickland, que fecha a ponta de segurança e compliance da empresa, publicou o padrão interno de IaC que todo módulo Terraform novo precisa seguir

# Action
Crie um módulo Terraform reutilizável pra criar buckets S3 aderentes ao padrão IaC da empresa, no mesmo estilo do módulo de VPC de referência. O módulo vai ser consumido por todos os times da empresa, então precisa vir com exemplo de uso

# Result
O padrão exige:

- Tags obrigatórias em todo recurso: `Owner`, `CostCenter`, `Environment`.
- Prefixo `hvt-` nos nomes de recursos.
- Todo bucket S3 com: encryption habilitada (SSE-S3 mínimo), versioning ativo, block public access total, logging configurado.
- Variáveis de entrada em `variables.tf` com `description` e `type` obrigatórios.

# Example
Como referência de estilo, o módulo de VPC que já existe na empresa:

variable "environment" { description = "Nome do ambiente (dev, staging, production)" type = string } locals { common_tags = { Owner = var.owner CostCenter = var.cost_center Environment = var.environment } } resource "aws_vpc" "this" { cidr_block = var.cidr_block tags = merge(local.common_tags, { Name = "hvt-vpc-${var.environment}" }) }

 
```


## 2. Modelo

```
Haiku 5.5
```


## 3. Output
---
Não consigo criar arquivos nesta conversa, então o módulo está completo abaixo, separado por arquivo. Em `main.tf` e `outputs.tf`, o código segue o estilo do módulo de VPC de referência (`locals` com `common_tags`, `merge` e `Name` com prefixo `hvt-`).

```
modules/s3-bucket/
├── versions.tf
├── variables.tf
├── main.tf
├── outputs.tf
└── examples/
    └── basic/
        └── main.tf
```

#### `versions.tf`

hcl

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

#### `variables.tf`

hcl

```hcl
variable "name" {
  description = "Nome lógico do bucket, sem o prefixo hvt- (ex.: relatorios-financeiros). O nome final será hvt-<name>-<environment>."
  type        = string
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environment deve ser dev, staging ou production."
  }
}

variable "owner" {
  description = "Time ou pessoa responsável pelo bucket (tag Owner)"
  type        = string
}

variable "cost_center" {
  description = "Centro de custo para alocação de gastos (tag CostCenter)"
  type        = string
}

variable "logging_target_bucket" {
  description = "Nome do bucket de destino dos access logs. Obrigatório pelo padrão de segurança."
  type        = string
}

variable "logging_target_prefix" {
  description = "Prefixo dos access logs dentro do bucket de destino. Se omitido, usa logs/<nome-do-bucket>/."
  type        = string
  default     = null
}

variable "force_destroy" {
  description = "Permite destruir o bucket mesmo com objetos dentro. Use apenas em dev."
  type        = bool
  default     = false
}

variable "extra_tags" {
  description = "Tags adicionais opcionais. As tags obrigatórias (Owner, CostCenter, Environment) sempre prevalecem sobre estas."
  type        = map(string)
  default     = {}
}
```

#### `main.tf`

hcl

```hcl
locals {
  bucket_name = "hvt-${var.name}-${var.environment}"

  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }

  # As tags obrigatórias vêm por último, então não podem ser sobrescritas por extra_tags
  tags = merge(var.extra_tags, local.common_tags)
}

resource "aws_s3_bucket" "this" {
  bucket        = local.bucket_name
  force_destroy = var.force_destroy

  tags = merge(local.tags, {
    Name = local.bucket_name
  })
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "this" {
  bucket        = aws_s3_bucket.this.id
  target_bucket = var.logging_target_bucket
  target_prefix = coalesce(var.logging_target_prefix, "logs/${local.bucket_name}/")
}
```

#### `outputs.tf`

hcl

```hcl
output "bucket_id" {
  description = "Nome do bucket criado"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN do bucket criado"
  value       = aws_s3_bucket.this.arn
}

output "bucket_regional_domain_name" {
  description = "Domínio regional do bucket"
  value       = aws_s3_bucket.this.bucket_regional_domain_name
}

output "tags" {
  description = "Tags aplicadas ao bucket"
  value       = local.tags
}
```

#### `examples/basic/main.tf`

Este é o exemplo de uso para os times consumidores. Ajuste o `source` para o caminho real do repositório de módulos da empresa.

hcl

```hcl
module "s3_relatorios_financeiros" {
  source = "git::https://github.com/<org>/terraform-modules.git//modules/s3-bucket?ref=v1.0.0"

  name                  = "relatorios-financeiros"
  environment           = "production"
  owner                 = "time-dados"
  cost_center           = "CC-1234"
  logging_target_bucket = "hvt-logs-central-production"

  extra_tags = {
    Projeto = "fechamento-mensal"
  }
}

output "bucket_arn" {
  value = module.s3_relatorios_financeiros.bucket_arn
}
```

Isso cria o bucket `hvt-relatorios-financeiros-production`.

#### Pontos de atenção

- **Tags nos sub-recursos:** encryption, versioning, public access block e logging não aceitam tags na API da AWS. As tags obrigatórias ficam no `aws_s3_bucket`, que é o único recurso taggeable do módulo.
- **Prefixo `hvt-`:** os sub-recursos não têm nome próprio, então o prefixo só se aplica ao bucket.
- **Bucket de logs:** o `logging_target_bucket` precisa existir antes e ter a policy que permite ao serviço de logging do S3 gravar nele. O módulo não cria esse bucket.
- **Limite de nome:** buckets S3 têm no máximo 63 caracteres. O Terraform não valida isso entre variáveis, então vale conferir nomes longos no plan.
- **Criptografia:** SSE-S3 (`AES256`) é o mínimo exigido. Se o padrão evoluir para KMS, o módulo precisará de uma variável `kms_key_arn`.
- **Validação:** não consegui executar o código aqui. Antes de publicar, rode `terraform fmt -check`, `terraform validate` e `tflint`, e passe o módulo pelo scanner de compliance que o time da Strickland usa.

## 4. Justificativa

O ponto forte do framework é o exemplo que vai no prompt. Isso reduz o risco de ambiguidade e aumenta a assertividade da resposta. E reposta foi satisfatória, atendendo a todos os requisitos do padrão interno do modulo IaC, como tags obrigatorias, prefixo dos recurso, criação de arquivo de variáveis com descrição e tipos obrigatorios.