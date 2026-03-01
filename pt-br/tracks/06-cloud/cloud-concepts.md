# Conceitos de Cloud

## Visão Geral

Cloud computing é a entrega sob demanda de recursos computacionais — servidores, armazenamento, bancos de dados, redes, software — via internet com cobrança pelo uso. Em vez de comprar hardware físico e gerenciar data centers, você aluga capacidade de um provedor de cloud e paga apenas pelo que utiliza.

A cloud computing mudou fundamentalmente a forma como software é construído e operado. Startups podem lançar globalmente no primeiro dia. Times podem provisionar um cluster de 100 nós em minutos e desligá-lo quando terminar. Infraestrutura vira código.

Este capítulo constrói a base conceitual para trabalhar com cloud: modelos de serviço, modelos de deploy, responsabilidade compartilhada, princípios centrais de design e a economia que orienta decisões na cloud.

---

## Pré-requisitos

- Entendimento básico de redes (HTTP, DNS, portas)
- Familiaridade com Linux e Docker
- Nenhuma experiência anterior com cloud é necessária

---

## Conceitos Fundamentais

### Modelos de Serviço

Os serviços de cloud são entregues em diferentes níveis de abstração:

```
Espectro de abstração (mais controle ← → mais gerenciado)

IaaS                    PaaS                    SaaS
(Infrastructure)        (Platform)              (Software)
    |                       |                       |
Virtual machines        Managed databases       Gmail
Object storage          App hosting             Slack
Load balancers          Serverless functions    Figma
Networks                Container services      GitHub
    |                       |                       |
Você gerencia:          Provedor gerencia:      Provedor gerencia:
OS, runtime,            OS, runtime,            Tudo
app, dados              scaling, patches
```

**IaaS (Infrastructure as a Service):** Você recebe compute, storage e networking brutos. É responsável por tudo acima do hypervisor: SO, patches de segurança, runtime e a sua aplicação.
- Exemplos: AWS EC2, Google Compute Engine, Azure VMs, DigitalOcean Droplets

**PaaS (Platform as a Service):** O provedor gerencia a infraestrutura subjacente. Você faz deploy de código ou containers.
- Exemplos: AWS Elastic Beanstalk, Google App Engine, Heroku, Railway, Render

**SaaS (Software as a Service):** Aplicações totalmente gerenciadas. Você as usa, não as opera.
- Exemplos: GitHub, Datadog, Stripe, Sendgrid

**FaaS (Function as a Service / Serverless):** Você faz deploy de funções individuais. Sem servidores para gerenciar. Escala até zero.
- Exemplos: AWS Lambda, Google Cloud Functions, Cloudflare Workers

### Modelos de Deploy

| Modelo | Descrição | Caso de uso |
|--------|-----------|-------------|
| **Public cloud** | Recursos compartilhados entre múltiplos clientes (isolados) | Maioria das aplicações |
| **Private cloud** | Infraestrutura dedicada para uma organização | Setores regulados |
| **Hybrid cloud** | Combinação de public + private | Migração gradual, requisitos de compliance |
| **Multi-cloud** | Múltiplos provedores de public cloud | Evitar vendor lock-in, melhor de cada provedor |

### Modelo de Responsabilidade Compartilhada

Um conceito crítico: provedores de cloud protegem **a** cloud; clientes protegem **o que está** na cloud.

```
Responsabilidade do Provedor de Cloud:
  ✓ Segurança física dos data centers
  ✓ Manutenção de hardware
  ✓ Segurança do hypervisor
  ✓ Disponibilidade de serviços gerenciados
  ✓ Certificações de compliance (SOC2, ISO27001, etc.)

Sua Responsabilidade (varia pelo modelo de serviço):
  IaaS: Patches de SO, regras de firewall, segurança da app, criptografia, gestão de acesso
  PaaS: Segurança da app, dados, gestão de acesso, API keys
  SaaS: Gestão de acesso, dados que você insere, permissões de usuários
```

Má configuração é a principal causa de vazamentos de dados na cloud. Um S3 bucket definido como público, uma role IAM com permissões excessivas, ou um security group com `0.0.0.0/0` na porta 5432 — tudo isso é sua responsabilidade.

### Regions e Availability Zones

Os provedores de cloud operam infraestrutura distribuída globalmente:

```
Region: us-east-1 (Northern Virginia)
  ├── Availability Zone: us-east-1a
  │     └── Múltiplos data centers
  ├── Availability Zone: us-east-1b
  │     └── Múltiplos data centers
  └── Availability Zone: us-east-1c
        └── Múltiplos data centers

Region: eu-west-1 (Irlanda)
  ├── Availability Zone: eu-west-1a
  └── Availability Zone: eu-west-1b
```

**Region:** Uma área geográfica com múltiplos data centers. A escolha da region afeta latência (prefira próxima dos seus usuários) e compliance (leis de soberania de dados).

**Availability Zone (AZ):** Um data center isolado e fisicamente separado dentro de uma region. Fazer deploy em múltiplas AZs protege contra falhas em um único data center.

**Edge locations:** Pontos de presença (PoPs) usados por CDNs e serviços como AWS CloudFront. Muito mais numerosos que as regions.

### Economia da Cloud

**CapEx vs OpEx:**
- TI tradicional: grande gasto de capital (CapEx) upfront em hardware
- Cloud: gasto operacional (OpEx) contínuo — pague conforme usa

**Modelos de precificação:**

| Modelo | Descrição | Melhor para |
|--------|-----------|-------------|
| On-demand | Pague por hora/segundo. Sem compromisso. | Cargas variáveis, dev/test |
| Reserved | Compromisso de 1 ou 3 anos. 30-70% de desconto. | Cargas estáveis e previsíveis |
| Spot/Preemptible | Lance em capacidade ociosa. 70-90% de desconto. Pode ser interrompido. | Batch jobs, CI, cargas stateless |
| Savings Plans | Compromisso flexível de gasto. 30-65% de desconto. | Específico da AWS, reserved flexível |

**Principais direcionadores de custo:**
- Compute (horas de vCPU)
- Storage (GB-mês)
- Transferência de dados (egress quase sempre é pago; ingress geralmente é gratuito)
- Requisições (chamadas de API, serviços por requisição)
- Planos de suporte

---

## Exemplos Práticos

### Pensando em Termos Cloud-Native

Deploy tradicional:
```
Servidor → roda processo Node.js → conecta ao Postgres local → serve tráfego
```

Equivalente cloud-native:
```
ECS/EKS task → Node.js containerizado → RDS Postgres (gerenciado) → atrás de ALB
     |                                        |                       |
Escala horizontalmente           Failover automático           Health checks
Auto-substituição em falha       Backups automatizados         SSL termination
Deploy sem downtime              Point-in-time recovery        Integração com WAF
```

### Calculando Custos de Cloud

Exemplo: aplicação web de produção pequena na AWS

```
Componente            | Tamanho        | Custo Mensal (aprox.)
----------------------|----------------|----------------------
EC2 t3.small          | 2 vCPU, 2 GB   | $15/mês (on-demand)
EBS gp3 volume        | 20 GB          | $1,60/mês
RDS db.t3.micro       | 1 vCPU, 1 GB   | $13/mês
Application Load Balancer | 1 ALB      | $16/mês + uso
Data transfer out     | ~50 GB/mês     | $4,50/mês
Route 53              | 1 hosted zone  | $0,50/mês
----------------------|----------------|----------------------
Total                 |                | ~$50/mês
```

Compare com VPS Hetzner (CX21: 2 vCPU, 4 GB): $7/mês — mas sem banco de dados gerenciado, sem load balancer, sem auto-scaling, sem backups gerenciados.

Cloud custa dinheiro, mas tempo de gerenciamento custa mais. A economia favorece cloud para times que valorizam velocidade de desenvolvimento sobre frugalidade com infraestrutura.

### Os 5 Rs da Migração para Cloud

Ao mover aplicações existentes para cloud:

| Estratégia | Descrição | Quando usar |
|------------|-----------|-------------|
| **Rehost** (Lift & Shift) | Mover VMs como estão para VMs na cloud | Migração mais rápida, mínima mudança |
| **Replatform** | Otimizações menores (banco de dados gerenciado, PaaS) | Ganhos rápidos com alguns benefícios da cloud |
| **Repurchase** | Migrar para SaaS | Substituir solução própria por SaaS comercial |
| **Refactor** | Redesenhar como cloud-native | Benefício máximo, maior esforço |
| **Retire** | Desativar sistemas não usados | Redução de custos |

---

## Padrões e Boas Práticas

### Design para Falha

Instances de cloud falham. Redes partem. Serviços ficam temporariamente indisponíveis. Projete para isso:

```typescript
// Retry com backoff exponencial
async function callExternalService<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxRetries) throw err;
      const delay = Math.min(1000 * 2 ** attempt, 30000) + Math.random() * 1000;
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
  throw new Error('Max retries exceeded');
}
```

### Princípio do Mínimo Privilégio

Cada recurso de cloud deve ter apenas as permissões de que precisa:

```json
// RUIM — permissões com wildcard
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// BOM — permissões específicas em recursos específicos
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-app-uploads/*"
}
```

### Infrastructure as Code (IaC)

Nunca crie infraestrutura de produção clicando em consoles de cloud. Defina como código:

```typescript
// Exemplo com AWS CDK
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';

export class AppStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'AppVpc', { maxAzs: 2 });

    const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

    new ecs.FargateService(this, 'ApiService', {
      cluster,
      taskDefinition: new ecs.FargateTaskDefinition(this, 'Task'),
      desiredCount: 2,
    });
  }
}
```

Benefícios do IaC:
- Ambientes reproduzíveis (staging = produção)
- Mudanças de infraestrutura versionadas no git
- Code review para modificações de infraestrutura
- Recuperação de desastres (recriar todo o stack a partir do código)

### Infraestrutura Imutável

Não aplique patches em servidores em execução — substitua-os:

```
Tradicional:   faz push do código → SSH → git pull → reinicia app
Cloud-native:  build da imagem → push → deploy de novos containers → mata containers antigos
```

Containers antigos nunca são modificados. Se algo quebrar, faça rollback para a imagem anterior.

### O App 12-Factor

Metodologia para construir aplicações cloud-native:

1. **Codebase:** Um repositório por app, muitos deploys
2. **Dependências:** Declare explicitamente (package.json), não dependa de pacotes do sistema
3. **Config:** Armazene em variáveis de ambiente, não no código
4. **Backing services:** Trate bancos de dados, filas como recursos anexados (configurados via URL)
5. **Build/release/run:** Estágios estritamente separados
6. **Processos:** Processos stateless — sem estado local
7. **Port binding:** Exporte serviços via porta
8. **Concorrência:** Escale via modelo de processos
9. **Descartabilidade:** Inicialização rápida, desligamento gracioso
10. **Paridade dev/prod:** Mantenha os ambientes o mais similares possível
11. **Logs:** Trate logs como streams de eventos (apenas stdout)
12. **Processos admin:** Execute tarefas pontuais como processos únicos

---

## Anti-Padrões a Evitar

### Snowflake Servers

Servidores configurados manualmente que não podem ser reproduzidos. Se o servidor morrer, você não consegue recriá-lo. Use IaC e trate servidores como gado, não como pets.

### Armazenar Estado em Servidores de Aplicação

```
RUIM:  Usuário faz upload de arquivo → salvo em /app/uploads na instance EC2
       Quando o EC2 é substituído, os uploads somem

BOM:   Usuário faz upload de arquivo → salvo em bucket S3
       Servidores de aplicação são stateless e substituíveis
```

### Abrir Todas as Portas "Temporariamente"

```
Security Group Inbound: 0.0.0.0/0 ALL TRAFFIC
"Só para debug" → nunca removido → vazamento de dados
```

### Ignorar Custos de Egress

Transferência de dados para fora da cloud é cara. Arquiteturas que movem grandes volumes de dados para fora da AWS (para usuários, para outras clouds) acumulam custos significativos. CDNs reduzem custos de egress consideravelmente.

### Deploy em AZ Única para Produção

```
RUIM:  Todos os recursos somente em us-east-1a
       Falha do data center → indisponibilidade total

BOM:   Recursos distribuídos entre us-east-1a, us-east-1b, us-east-1c
       Falha do data center → failover transparente
```

---

## Debugging e Solução de Problemas

### Estratégia Geral de Debug na Cloud

```
1. Verifique o status do serviço:
   → AWS Status: status.aws.amazon.com
   → GCP Status: status.cloud.google.com
   → Azure Status: status.azure.com

2. Verifique a saúde do seu recurso:
   → AWS: CloudWatch, Health Dashboard
   → Anomalias de custo: console de billing

3. Verifique os logs da sua aplicação:
   → CloudWatch Logs, Cloud Logging, Azure Monitor

4. Verifique conectividade de rede:
   → Security groups / regras de firewall
   → Tabelas de roteamento de VPC
   → Resolução de DNS (dig, nslookup)

5. Verifique permissões IAM:
   → IAM Policy Simulator (AWS)
   → Erros "Access denied" geralmente indicam permissões ausentes
```

### Erros Comuns na Cloud

**Access Denied / 403:** Permissões IAM ausentes ou política de recurso incorreta. Verifique a ação e o recurso no erro — consulte qual permissão é necessária.

**Connection timeout para banco de dados:** Security group sem a regra de entrada para a porta 5432/3306 a partir do security group da sua aplicação.

**Out of capacity na AZ:** A AWS não consegue provisionar o tipo de instance solicitado nessa AZ. Tente uma AZ ou tipo de instance diferente.

**Service limit exceeded:** A AWS tem limites padrão (ex.: 5 VPCs por region). Solicite aumento de limite em Service Quotas.

---

## Cenários do Mundo Real

### Cenário 1: Escolhendo Entre Cloud e VPS

```
VPS é melhor quando:
- Orçamento mensal < $50
- Aplicação única, desenvolvedor único
- Não precisa de serviços gerenciados
- Controle total sobre a infraestrutura
- Aprendizado / experimentação

Cloud é melhor quando:
- Time com 2+ desenvolvedores
- Precisa escalar de forma imprevisível
- Requer bancos de dados gerenciados, CDN, edge global
- Requisitos de compliance (SOC2, HIPAA precisam de certificações de cloud)
- Construindo produtos que vendem serviços de cloud
```

### Cenário 2: Estimando Custos de Cloud Antes de Comprometer

```
1. Use as calculadoras dos provedores:
   - AWS Pricing Calculator: calculator.aws
   - GCP Pricing Calculator: cloud.google.com/products/calculator
   - Azure Pricing Calculator: azure.microsoft.com/pricing/calculator

2. Comece com recursos mínimos (você pode escalar depois)

3. Configure alertas de billing imediatamente:
   AWS → Billing → Budgets → Create budget → alerta em $X

4. Reserve instances após 2-3 meses de uso estável
   (geralmente economiza 30-50% vs on-demand)
```

### Cenário 3: Multi-Region para Baixa Latência

```
Usuários no Brasil:  roteie para sa-east-1 (São Paulo)
Usuários nos EUA:    roteie para us-east-1 (Virgínia)
Usuários na Europa:  roteie para eu-west-1 (Irlanda)

Usando: Route 53 com roteamento baseado em latência
        CloudFront com múltiplas origens
        Global Accelerator
```

---

## Leitura Adicional

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Google Cloud Architecture Center](https://cloud.google.com/architecture)
- [The 12-Factor App](https://12factor.net/)
- [Cloud Native Computing Foundation](https://www.cncf.io/)
- [AWS Pricing Calculator](https://calculator.aws/)
- [Infrastructure as Code — HashiCorp Learn](https://developer.hashicorp.com/terraform/tutorials)

---

## Resumo

| Conceito | Conclusão Principal |
|----------|---------------------|
| IaaS/PaaS/FaaS | Espectro de abstração — mais gerenciado = menos controle, menos trabalho de ops |
| Responsabilidade compartilhada | Provedor protege a cloud; você protege o que está nela |
| Regions/AZs | Faça deploy em múltiplas AZs para resiliência; escolha regions por latência + compliance |
| On-demand vs Reserved | On-demand para flexibilidade; Reserved para 30-70% de economia em cargas estáveis |
| Mínimo privilégio | Cada recurso recebe apenas as permissões que precisa |
| IaC | Infraestrutura definida como código — reproduzível, revisável, versionada |
| Infra imutável | Substitua, não faça patch — permite rollbacks confiáveis |
| 12-Factor | Blueprint para design de aplicações cloud-native |
| Design para falha | Assuma que qualquer coisa pode falhar; construa retry, circuit breakers, multi-AZ |
| Custos de egress | Dados saindo da cloud custam dinheiro — considere na arquitetura |

A cloud não é apenas "o computador de outra pessoa". É uma plataforma de infraestrutura programável com APIs, serviços gerenciados e alcance global. Entender a economia, o modelo de responsabilidade e os princípios de design separa os praticantes de cloud dos meros usuários de cloud.
