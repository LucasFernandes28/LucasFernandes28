---
layout: home
title: IaC Infra Documentation
---

# Documentação da Infraestrutura como Código (IaC) - iac-infra

## 📋 Visão Geral

O projeto **iac-infra** é uma implementação modular de Infraestrutura como Código (IaC) utilizando **Terraform** com padrões escaláveis e reutilizáveis. A estrutura foi organizada para facilitar a criação e manutenção de recursos AWS, GitHub e outras integrações em diferentes ambientes (dev, qa, prod).

---

## 🏗️ Estrutura do Projeto

```
iac-infra/
├── modules/                    # Módulos reutilizáveis do Terraform
│   ├── alb-module-iac/        # Load Balancer (ALB)
│   ├── dynamo-module-iac/     # DynamoDB Tables
│   ├── ec2-module-iac/        # EC2 Instances
│   ├── gh-repo-module-iac/    # GitHub Repositories
│   ├── network-module-iac/    # VPC, Subnets, IGW, Route Tables
│   ├── r53-module-iac/        # Route 53 DNS
│   ├── s3-module-iac/         # S3 Buckets
│   ├── subnets-module-iac/    # Subnets adicionais
│   └── vpc-module-iac/        # VPC específica
├── stack/                      # Configurações por ambiente
│   ├── dev/                    # Ambiente de desenvolvimento
│   ├── qa/                     # Ambiente de QA
│   └── prod/                   # Ambiente de produção
├── iac-repos/                  # Configurações raiz
├── iac-tfstate-backend/        # Backend para estado Terraform
└── DOCUMENTACAO_IaC.md         # Este arquivo
```

---

## 📦 Módulos Detalhados

### 1️⃣ Network Module (`network-module-iac`)

**Objetivo:** Criar infraestrutura de rede completa incluindo VPC, subnets públicas/privadas, Internet Gateway e routing.

**Recursos Criados:**
- **VPC** - Virtual Private Cloud com suporte a DNS
- **Subnets Públicas** - Múltiplas subnets com auto-assign de IP público
- **Subnets Privadas** - Múltiplas subnets sem acesso público direto
- **Internet Gateway** - Gateway para acesso à internet
- **Route Tables** - Tabelas de roteamento públicas e associações

**Variáveis Principais:**
| Variável | Descrição | Default |
|----------|-----------|---------|
| `project_name` | Nome do projeto para identificação | Obrigatória |
| `vpc_cidr` | CIDR block da VPC | `10.0.0.0/16` |
| `public_subnets` | Lista de CIDR públicas | `["10.0.1.0/24", "10.0.2.0/24"]` |
| `private_subnets` | Lista de CIDR privadas | `["10.0.101.0/24", "10.0.102.0/24"]` |
| `azs` | Zonas de disponibilidade | `["us-east-1a", "us-east-1b"]` |
| `tags` | Tags adicionais | `{}` |

**Outputs:**
- `vpc_id` - ID da VPC criada
- `public_subnet_ids` - IDs das subnets públicas
- `private_subnet_ids` - IDs das subnets privadas
- `igw_id` - ID do Internet Gateway

**Padrões Implementados:**
- ✅ Uso de `merge()` para consolidação de tags
- ✅ `count` para criar múltiplas subnets dinamicamente
- ✅ `element()` para distribuição entre zonas de disponibilidade
- ✅ Convenção de nomenclatura com sufixos numéricos
- ✅ DNS habilitado por padrão (melhor prática)

---

### 2️⃣ DynamoDB Module (`dynamo-module-iac`)

**Objetivo:** Provisionar tabelas DynamoDB com configurações flexíveis de atributos e billing.

**Recursos Criados:**
- **DynamoDB Table** - Tabela com suporte a múltiplos atributos e modos de billing

**Variáveis Principais:**
| Variável | Descrição | Default |
|----------|-----------|---------|
| `name` | Nome da tabela | Obrigatória |
| `hash_key` | Chave primária | `"LockID"` |
| `billing_mode` | Modo de cobrança | `"PAY_PER_REQUEST"` |
| `attributes` | Lista de atributos | `[{name = "LockID", type = "S"}]` |
| `deletion_protection_enabled` | Proteção contra deleção | `true` |
| `tags` | Tags customizadas | `{}` |

**Outputs:**
- `table_name` - Nome da tabela
- `billing_mode` - Modo de cobrança configurado
- `deletion_protection_enabled` - Status da proteção
- `bucket_arn` - ARN da tabela

**Padrões Implementados:**
- ✅ `dynamic` block para atributos flexíveis
- ✅ Proteção contra deleção acidental por padrão
- ✅ Suporte a tipos de dados: String (S), Number (N), Binary (B)
- ✅ Consolidação de tags com `merge()`

**Casos de Uso:**
- Terraform State Locking (com LockID padrão)
- Armazenamento de dados de aplicação
- Cache distribuído

---

### 3️⃣ EC2 Module (`ec2-module-iac`)

**Objetivo:** Provisionar instâncias EC2 com configurações de rede, segurança e naming flexíveis.

**Recursos Criados:**
- **EC2 Instance** - Máquina virtual com customização de tipo, AMI e networking

**Variáveis Principais:**
| Variável | Descrição | Default |
|----------|-----------|---------|
| `instance_name` | Nome base da instância | Obrigatória |
| `name_suffix` | Sufixo para o nome | `""` |
| `ami_id` | AMI a usar | `"ami-08c40ec9ead489470"` (Amazon Linux 2) |
| `instance_type` | Tipo da instância | `"t2.micro"` (Free Tier) |
| `subnet_id` | Subnet de deployment | Obrigatória |
| `security_group_ids` | Lista de SGs | `[]` |
| `associate_public_ip` | Associar IP público | `true` |
| `tags` | Tags customizadas | `{}` |

**Outputs:**
- `instance_id` - ID da instância
- `instance_public_ip` - IP público (se houver)
- `instance_private_ip` - IP privado
- `instance_arn` - ARN da instância

**Padrões Implementados:**
- ✅ `format()` para naming com sufixos dinâmicos
- ✅ AMI padrão Amazon Linux 2 (t2.micro gratuito)
- ✅ Integração com Security Groups e Subnets
- ✅ Tags dinâmicas com `merge()`

**Casos de Uso:**
- Servidores web
- Servidores de aplicação
- Ferramentas de CI/CD
- Bastions para acesso SSH

---

### 4️⃣ S3 Module (`s3-module-iac`)

**Objetivo:** Criar buckets S3 com segurança, versionamento e criptografia habilitados por padrão.

**Recursos Criados:**
- **S3 Bucket** - Armazenamento de objetos
- **S3 Versioning** - Controle de versões
- **S3 Encryption** - Criptografia AES256
- **S3 Public Access Block** - Proteção contra acesso público

**Variáveis Principais:**
| Variável | Descrição | Default |
|----------|-----------|---------|
| `bucket_name` | Nome do bucket | Obrigatória |
| `region` | Região | `"us-east-1"` |
| `enable_versioning` | Ativar versionamento | `false` |
| `block_public_acls` | Bloquear ACLs públicas | `true` |
| `block_public_policy` | Bloquear políticas públicas | `true` |
| `ignore_public_acls` | Ignorar ACLs públicas | `true` |
| `restrict_public_buckets` | Restringir acesso público | `true` |
| `tags` | Tags customizadas | `{}` |

**Outputs:**
- `bucket_name` - Nome do bucket
- `bucket_id` - ID do bucket
- `bucket_arn` - ARN do bucket
- `bucket_region` - Região configurada

**Padrões Implementados:**
- ✅ Segurança por padrão (public access block)
- ✅ Criptografia AES256 automática
- ✅ Versionamento opcional
- ✅ Tags consolidadas com `merge()`

**Casos de Uso:**
- Armazenamento de arquivos estáticos
- Backup de dados
- Armazenamento de logs e traces
- Terraform state (com versionamento e encryption)

---

### 5️⃣ GitHub Repository Module (`gh-repo-module-iac`)

**Objetivo:** Provisionar repositórios GitHub com proteção de branches e labels automáticas.

**Recursos Criados:**
- **GitHub Repository** - Repositório com inicialização automática
- **GitHub Issue Labels** - Labels customizadas para issues
- **GitHub Branch Protection** - Proteção da branch master/main

**Variáveis Principais:**
| Variável | Descrição | Default |
|----------|-----------|---------|
| `repository_name` | Nome do repositório | Obrigatória |
| `description` | Descrição do repo | `""` |
| `visibility` | Visibilidade (public/private) | `"public"` |
| `labels` | Lista de labels | `["dev", "qa", "plan"]` |

**Outputs:**
- `repository_name` - Nome do repositório
- `repository_full_name` - Nome completo (org/repo)
- `repository_html_url` - URL do repositório
- `repository_ssh_clone_url` - URL SSH para clone
- `repository_http_clone_url` - URL HTTPS para clone

**Padrões Implementados:**
- ✅ `for_each` para criação dinâmica de labels
- ✅ Branch protection automática na master
- ✅ Require pull request reviews
- ✅ Auto-init para estrutura inicial do repo

**Casos de Uso:**
- Provisionamento de repos para novos projetos
- Aplicação consistente de políticas de branch
- Automação de governance

---

### 6️⃣ Route 53 Module (`r53-module-iac`)

**Objetivo:** Gerenciar registros DNS na AWS.

**Status:** Módulo em desenvolvimento (README presente, implementação em andamento)

---

### 7️⃣ ALB Module (`alb-module-iac`)

**Objetivo:** Provisionar Application Load Balancers com configuração de listeners e target groups.

**Status:** Estrutura presente, detalhes em implementação

---

### 8️⃣ Subnets Module (`subnets-module-iac`)

**Objetivo:** Criar subnets adicionais com maior granularidade.

**Status:** Estrutura presente, detalhes em implementação

---

### 9️⃣ VPC Module (`vpc-module-iac`)

**Objetivo:** Módulo especializado para VPC com configurações avançadas.

**Status:** Estrutura presente, detalhes em implementação

---

## 🏢 Ambientes (Stack)

A estrutura está organizada em três ambientes:

### **Development (dev/)**
- Ambiente para testes e desenvolvimento
- Configurações com custo otimizado
- Status: Vazio (pronto para implementação)

### **Quality Assurance (qa/)**
- Ambiente para testes de qualidade
- Configurações próximas à produção
- Status: Vazio (pronto para implementação)

### **Production (prod/)**
- Ambiente de produção
- Configurações de alta disponibilidade e disaster recovery
- Status: Vazio (pronto para implementação)

---

## 🔑 Padrões de Design Implementados

### 1. **Modularização**
- Cada recurso AWS tem seu módulo dedicado
- Reutilização através de `source` em Terragrunt/Terraform
- Separação de concerns clara

### 2. **Variabilização**
- Valores `default` sensatos
- Variáveis obrigatórias claramente marcadas
- Documentação inline em `description`

### 3. **Naming Convention**
```terraform
resource "type" "this" {
  # Esta convenção facilita o entendimento e evita conflitos
}
```

### 4. **Tag Consolidation**
```terraform
tags = merge(
  {
    Name = "resource-name"
  },
  var.tags  # Tags customizadas
)
```

### 5. **Dynamic Blocks**
- Uso de `dynamic` para listas de objetos complexos
- `for_each` para iteração sobre mapas
- `count` para múltiplas instâncias

### 6. **Output Strategy**
- Outputs descritivos para cada recurso principal
- ARNs, IDs e URLs sempre exportados
- Facilita integração com outros stacks

---

## 🔒 Segurança por Padrão

### S3 Module
- ✅ Criptografia AES256 habilitada
- ✅ Public Access Block aplicado
- ✅ Versioning opcional

### EC2 Module
- ✅ IP público associável (não forçado)
- ✅ Integração com Security Groups
- ✅ Uso de subnets específicas

### DynamoDB Module
- ✅ Deletion Protection ativada por padrão
- ✅ Encriptação no nível de tabela

### GitHub Module
- ✅ Branch protection obrigatória
- ✅ Require pull request reviews
- ✅ Enforce admins

---

## 📊 Diagrama de Relacionamentos

```
┌─────────────────────────────────────────┐
│      Infrastructure Root (iac-infra)    │
└────────────┬────────────────────────────┘
             │
    ┌────────┴──────────┐
    │                   │
┌───▼────────┐    ┌────▼─────────┐
│   modules/ │    │     stack/    │
├────────────┤    ├───────────────┤
│ network    │    │ dev/          │
│ dynamodb   │    │ qa/           │
│ ec2        │    │ prod/         │
│ s3         │◄───┤               │
│ github     │    │ (each env     │
│ alb        │    │  uses modules)│
│ route53    │    └───────────────┘
│ subnets    │
│ vpc        │
└────────────┘
```

---

## 🚀 Como Utilizar

### Estrutura Recomendada para Stack

```hcl
# stack/dev/terragrunt.hcl

terraform {
  source = "../../modules/network-module-iac"
}

inputs = {
  project_name    = "meu-projeto"
  vpc_cidr         = "10.0.0.0/16"
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.101.0/24", "10.0.102.0/24"]
  
  tags = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

### Composição de Múltiplos Módulos

Para criar uma infraestrutura completa:

1. **Network** - VPC, Subnets, Routing
2. **Security** - Security Groups (não documentado aqui)
3. **Compute** - EC2 Instances
4. **Storage** - S3, DynamoDB
5. **Load Balancing** - ALB
6. **DNS** - Route 53

---

## 📝 Checklist de Implementação

### Phase 1: Infraestrutura Base
- [ ] Implementar Network Stack (vpc, subnets, routing)
- [ ] Configurar Backend Terraform (iac-tfstate-backend)
- [ ] Testar outputs da Network

### Phase 2: Recursos de Compute e Storage
- [ ] Implementar EC2 Module
- [ ] Implementar S3 Module
- [ ] Implementar DynamoDB Module
- [ ] Criar security groups

### Phase 3: Integração e DevOps
- [ ] Implementar ALB Module
- [ ] Implementar Route 53 Module
- [ ] Configurar GitHub Repositories
- [ ] Integrar com CI/CD

### Phase 4: Ambientes
- [ ] Implementar stack/dev
- [ ] Implementar stack/qa
- [ ] Implementar stack/prod
- [ ] Documentar variações por ambiente

---

## 🔧 Variáveis Críticas para Cada Ambiente

### Development
```hcl
instance_type = "t2.micro"      # Cost optimization
enable_versioning = false
deletion_protection = false     # Easier cleanup
```

### QA
```hcl
instance_type = "t2.small"      # Closer to prod
enable_versioning = true
deletion_protection = true      # Prevent accidents
```

### Production
```hcl
instance_type = "t2.medium"     # Or larger
enable_versioning = true        # Essential
deletion_protection = true
tags = {
  Environment = "prod"
  BackupPolicy = "daily"
  CostCenter = "XXX"
}
```

---

## 📚 Referências Adicionais

### Arquivos Importantes
- [Network Module Outputs](./modules/network-module-iac/outputs.tf)
- [EC2 Module README](./modules/ec2-module-iac/README.md)
- [Network Module README](./modules/network-module-iac/README.md)

### Tecnologias Utilizadas
- **Terraform** - Infrastructure as Code
- **Terragrunt** - Wrapper para Terraform (recomendado para stacks)
- **AWS Providers** - Para recursos AWS
- **GitHub Provider** - Para repositórios GitHub

---

## 📧 Contato e Suporte

Para dúvidas ou melhorias na documentação e implementação, consulte o repositório ou a equipe de DevOps.

---

**Última Atualização:** Janeiro 2026  
**Status:** Documentação Completa | Implementação Parcial  
**Versão:** 1.0
