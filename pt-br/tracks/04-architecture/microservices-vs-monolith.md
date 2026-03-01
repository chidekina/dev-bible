# Microservices vs Monólito

## 1. O que é e por que importa

Uma das decisões arquiteturais mais consequentes que um time toma é se construir um monólito ou um conjunto de microservices. Ambas são abordagens válidas — a escolha certa depende do tamanho do time, complexidade do domínio, padrões de tráfego e maturidade organizacional.

Errar essa escolha é caro: uma arquitetura de microservices prematura pode paralisar um time com complexidade operacional; um monólito que nunca foi projetado para modularidade se torna uma "big ball of mud" impossível de manter.

> 💡 Essa decisão não é binária. A maioria dos sistemas bem-sucedidos começa como monólitos bem estruturados e migra em direção a serviços conforme o time e o domínio crescem — um caminho conhecido como Strangler Fig Pattern.

---

## 2. Conceitos Fundamentais

### O Monólito

Um monólito é uma única unidade implantável contendo toda a funcionalidade da aplicação.

```
┌─────────────────────────────────────────┐
│            Processo do Monólito         │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │  Orders  │  │ Payments │  │ Users │ │
│  └──────────┘  └──────────┘  └───────┘ │
│                                         │
│  ┌──────────┐  ┌──────────┐            │
│  │Inventory │  │Shipping  │            │
│  └──────────┘  └──────────┘            │
│                                         │
│         Banco de Dados Único            │
└─────────────────────────────────────────┘
```

**Dois tipos muito diferentes de monólito:**

| Tipo | Estrutura | Qualidade de código | Implantabilidade |
|------|-----------|---------------------|-----------------|
| Monólito Modular | Fronteiras claras de módulo, interfaces explícitas | Alta | Fácil |
| Big Ball of Mud | Sem fronteiras, tudo depende de tudo | Baixa | Perigosa |

Um monólito modular não é um modo de falha — é uma escolha deliberada e muitas vezes correta.

#### Monólito Modular: Como o Bom Parece

```typescript
// src/modules/orders/
//   orders.controller.ts
//   orders.service.ts
//   orders.repository.ts
//   orders.types.ts

// src/modules/payments/
//   payments.controller.ts
//   payments.service.ts
//   ...

// Módulos se comunicam por APIs públicas explícitas, não por imports diretos de internos
// orders.service.ts
import { PaymentsService } from '../payments/payments.service'; // apenas a API pública

// NÃO isso:
import { createPaymentRecord } from '../payments/internal/ledger'; // detalhe interno vazou
```

Use regras de boundary do ESLint (ex: `eslint-plugin-boundaries`) para garantir que módulos não podem importar os internos uns dos outros.

#### Big Ball of Mud: Como o Ruim Parece

```typescript
// Sem estrutura de módulos — tudo em um único diretório plano
// Qualquer arquivo pode importar qualquer outro
import { sendEmail } from './emailHelper';
import { deductInventory } from './inventoryUtils';
import { chargeCard } from './paymentUtils';
import { createShipment } from './shippingService';

// Uma função fazendo tudo
async function placeOrder(data: any) {
  // 400 linhas de preocupações misturadas
}
```

---

### Microservices

Microservices decompõem o sistema em serviços independentemente implantáveis, cada um tendo seus próprios dados e se comunicando pela rede.

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Orders     │    │  Payments   │    │   Users     │
│  Service    │    │  Service    │    │  Service    │
│  :3001      │    │  :3002      │    │  :3003      │
│  Postgres   │    │  Postgres   │    │  Postgres   │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                   Message Bus / API Gateway
```

Propriedades principais:
- Cada serviço possui seu próprio banco de dados (sem DB compartilhado)
- Serviços se comunicam via HTTP/gRPC (síncrono) ou eventos/filas (assíncrono)
- Cada serviço pode ser implantado, escalado e reiniciado independentemente
- Cada serviço pode usar a linguagem/framework mais adequada para sua função

---

## 3. Prós e Contras

### Monólito

| Prós | Contras |
|------|---------|
| Simples de desenvolver localmente | Pode ser difícil de manter se não for modular |
| Fácil de testar (processo único) | Não é possível escalar componentes individuais |
| Sem latência de rede entre módulos | Um bug em qualquer módulo pode derrubar todo o serviço |
| Fácil de refatorar entre fronteiras de módulo | Deploy de uma feature exige reimplantação completa |
| Baixo overhead operacional | Lock-in tecnológico (todos os módulos usam o mesmo stack) |

### Microservices

| Prós | Contras |
|------|---------|
| Deploy independente por serviço | Complexidade de sistemas distribuídos (rede, latência, falha parcial) |
| Escalabilidade independente por serviço | Consistência de dados é difícil (sem ACID entre serviços) |
| Flexibilidade tecnológica por serviço | Mais difícil testar end-to-end |
| Isolamento de falhas (um serviço falha, outros continuam) | Maior overhead operacional (logging, tracing, service discovery) |
| Alinha com fronteiras de ownership do time | Decomposição excessiva pode criar um "micro-monólito" |

---

## 4. Fronteiras de Serviço

### Alinhe com Bounded Contexts

As melhores fronteiras para microservices são as mesmas de bounded contexts no DDD. Cada serviço:
- Tem sua própria linguagem ubíqua
- Possui seu próprio modelo de dados
- Expõe uma API bem definida
- É de propriedade de um time (Lei de Conway)

```
Bounded Context → Microservice (quando a complexidade justifica)

Sales BC        → Order Service
Billing BC      → Payment Service
Fulfillment BC  → Shipping Service
Identity BC     → User Service
```

### Lei de Conway

> Organizações que projetam sistemas... são constrangidas a produzir designs que são cópias das estruturas de comunicação dessas organizações. — Melvin Conway (1968)

Se sua organização tem 3 times, seu sistema naturalmente evoluirá para 3 subsistemas. Use a Lei de Conway como ferramenta: projete estruturas de times e fronteiras de serviço juntos. Um serviço de propriedade de dois times se tornará um gargalo de coordenação.

> 💡 Inverse Conway's Maneuver: deliberadamente projete sua estrutura de times para corresponder à arquitetura desejada, em vez de deixar a arquitetura espelhar o organograma existente.

---

## 5. Comunicação Entre Serviços

### Síncrono: REST e gRPC

```typescript
// Order Service chama Payment Service via HTTP
class OrderService {
  constructor(private readonly paymentClient: PaymentServiceClient) {}

  async placeOrder(order: Order): Promise<PlacedOrder> {
    // Chamada síncrona — Order Service aguarda resposta
    const paymentResult = await this.paymentClient.charge({
      customerId: order.customerId,
      amount: order.total,
      currency: order.currency,
    });

    if (paymentResult.status === 'declined') {
      throw new OrderPaymentDeclinedError(paymentResult.errorCode);
    }

    return this.finalizeOrder(order, paymentResult.transactionId);
  }
}
```

**Trade-offs da comunicação síncrona:**
- Simples de raciocinar (request/response)
- Acoplamento: se o Payment Service estiver fora do ar, Order Service também falha
- Latência se acumula entre saltos de serviço
- Bom para: consultas, operações voltadas ao usuário que precisam de confirmação imediata

### Assíncrono: Eventos e Filas de Mensagens

```typescript
// Order Service publica um evento; Payment Service o consome
// Sem dependência direta entre serviços

// No Order Service
class OrderService {
  constructor(private readonly eventBus: IEventBus) {}

  async createOrder(input: CreateOrderInput): Promise<string> {
    const order = Order.create(input);
    await this.orderRepo.save(order);

    // Dispara e continua — sem esperar
    await this.eventBus.publish('order.placed', {
      orderId: order.id,
      customerId: order.customerId,
      total: order.total,
      currency: order.currency,
      items: order.items,
    });

    return order.id;
  }
}

// No Payment Service (processo separado, deploy separado)
eventBus.subscribe('order.placed', async (event) => {
  const result = await paymentGateway.charge(event.customerId, event.total, event.currency);
  if (result.status === 'approved') {
    await eventBus.publish('payment.completed', { orderId: event.orderId, ...result });
  } else {
    await eventBus.publish('payment.failed', { orderId: event.orderId, reason: result.errorCode });
  }
});
```

**Trade-offs da comunicação assíncrona:**
- Desacoplado: serviços não conhecem uns aos outros
- Maior resiliência: produtor continua mesmo se o consumidor estiver fora do ar
- Consistência eventual: sem confirmação imediata
- Mais difícil de depurar: eventos espalhados nos logs
- Bom para: notificações, processamento em background, integração entre bounded contexts

---

## 6. Gerenciamento de Dados: Padrão Saga

Quando uma operação de negócio abrange múltiplos serviços, você não pode usar uma única transação de banco de dados. O padrão Saga coordena workflows multi-etapa que cruzam serviços.

### Saga Baseada em Coreografia

Sem coordenador central — cada serviço reage a eventos e publica os seus próprios.

```
Order Service    →  order.placed    →  Payment Service
Payment Service  →  payment.done    →  Inventory Service
Inventory Service→  stock.reserved  →  Shipping Service
Shipping Service →  shipment.created→  Order Service (marca completo)

Compensação (em caso de falha):
Payment Service  →  payment.failed  →  Order Service  (cancela pedido)
```

```typescript
// Cada serviço é autônomo — sem estado compartilhado
// Payment Service
eventBus.subscribe('order.placed', async (event) => {
  try {
    const result = await charge(event);
    await eventBus.publish('payment.completed', { orderId: event.orderId });
  } catch {
    await eventBus.publish('payment.failed', { orderId: event.orderId });
  }
});

// Order Service
eventBus.subscribe('payment.failed', async (event) => {
  await orderService.cancel(event.orderId, 'Pagamento falhou');
});
```

### Saga Baseada em Orquestração

Um orquestrador central (coordenador de Saga) conduz o workflow explicitamente.

```typescript
class PlaceOrderSaga {
  async run(orderId: string): Promise<void> {
    try {
      // Passo 1
      const payment = await paymentService.charge(orderId);

      // Passo 2
      const reservation = await inventoryService.reserve(orderId);

      // Passo 3
      await shippingService.schedule(orderId);

      await orderService.markCompleted(orderId);
    } catch (error) {
      // Compensa em ordem reversa
      await this.compensate(orderId, error);
    }
  }

  private async compensate(orderId: string, error: unknown): Promise<void> {
    await inventoryService.releaseReservation(orderId).catch(() => {});
    await paymentService.refund(orderId).catch(() => {});
    await orderService.markFailed(orderId, String(error));
  }
}
```

| Aspecto | Coreografia | Orquestração |
|---------|------------|-------------|
| Acoplamento | Baixo — serviços não se conhecem | Maior — orquestrador conhece todos os passos |
| Visibilidade | Difícil ver o fluxo completo | Fácil — orquestrador o define |
| Tratamento de falhas | Eventos de compensação distribuídos | Lógica de rollback centralizada |
| Melhor para | Fluxos simples com poucos passos | Workflows complexos e condicionais com múltiplas etapas |

---

## 7. Strangler Fig Pattern

O Strangler Fig Pattern é a forma recomendada de migrar de um monólito para microservices incrementalmente, sem uma reescrita arriscada do zero.

```
Fase 1: Monólito atende todo o tráfego
┌─────────────────────────┐
│   Monólito              │ ← todas as requisições
│   (Orders, Payments,    │
│   Users, Shipping)      │
└─────────────────────────┘

Fase 2: Extrai o primeiro serviço; roteia parte do tráfego
       ┌────────────────┐
       │  API Gateway   │
       └───────┬────────┘
               │
    ┌──────────┴──────────┐
    │                     │
┌───▼────┐          ┌─────▼───────────┐
│ User   │          │   Monólito      │
│Service │          │   (Orders,      │
│(novo)  │          │   Payments,     │
└────────┘          │   Shipping)     │
                    └─────────────────┘

Fase N: Monólito é substituído
┌────────────────────────────────────────────┐
│                 API Gateway                │
└───┬───────────┬───────────┬───────────┬───┘
    │           │           │           │
┌───▼───┐ ┌────▼───┐ ┌─────▼──┐ ┌──────▼──┐
│Orders │ │Payments│ │ Users  │ │Shipping │
└───────┘ └────────┘ └────────┘ └─────────┘
```

Princípios principais do Strangler Fig:
1. Nunca reescreva do zero — migre incrementalmente
2. Use um API Gateway ou reverse proxy para rotear tráfego
3. Extraia um serviço por vez, começando pelo mais valioso ou mais independente
4. Execute antigo e novo em paralelo até ter confiança; então remova o código antigo
5. Migração de dados é a parte mais difícil — considere dual-writes e backfills

---

## 8. Quando Escolher Cada Abordagem

### Escolha um Monólito quando:
- O time é pequeno (menos de ~8 engenheiros no mesmo produto)
- O domínio ainda não é totalmente compreendido (evite decomposição prematura)
- Tráfego baixo — escalabilidade independente não é necessária
- Velocidade de desenvolvimento é crítica (MVP, startup)
- Operações são mínimas — você não pode arcar com Kubernetes, service meshes, tracing distribuído

### Escolha Microservices quando:
- Múltiplos times grandes possuem partes distintas do sistema
- Os serviços têm requisitos de escalabilidade muito diferentes (ex: o serviço de transcodificação de vídeo precisa de 10x mais CPU que o de autenticação)
- Isolamento forte de falhas é necessário (pagamentos não devem afetar catálogo)
- Diferentes serviços precisam de stacks tecnológicos diferentes
- Separação regulatória é necessária (PCI-DSS, LGPD — isole o processamento de dados sensíveis)

### Erros Comuns com Microservices

1. **Serviços muito granulares:** Um microservice "User Preferences" com um endpoint servindo um time é overhead sem benefício. Serviços devem ser grandes o suficiente para justificar o custo operacional.

2. **Banco de dados compartilhado:** Se dois serviços compartilham um banco de dados, não são verdadeiramente independentes. Mudanças de schema no banco compartilhado exigem coordenação entre times — o principal benefício dos microservices (deploy independente) se perde.

3. **Sem distributed tracing:** Sem trace IDs propagados entre serviços, depurar uma requisição falha que tocou 5 serviços é uma sessão de depuração de várias horas. Instrumente com OpenTelemetry desde o primeiro dia.

4. **Chamadas síncronas para tudo:** Cadeias de chamadas HTTP síncronas criam falhas em cascata. Se o Serviço A chama B que chama C e C é lento, os três estão bloqueados.

5. **Sem contract testing:** Sem contract tests, mudanças de API no Serviço A quebram o Serviço B em produção, descobertas no momento do deploy. Use Pact ou schema registries.

> ⚠️ O monólito distribuído é o pior resultado: toda a complexidade operacional dos microservices (chamadas de rede, deploys independentes) sem nenhuma independência (dados fortemente acoplados, banco compartilhado, devem ser implantados juntos). Acontece quando os times dividem serviços por camadas técnicas (serviço frontend, serviço backend, serviço de banco) em vez de capacidades de negócio.

---

## 9. Cenário Real

Uma startup de e-commerce (3 desenvolvedores, 1 produto) lança como um monólito Fastify com Postgres. É um monólito modular desde o primeiro dia:

```
src/
  modules/
    orders/
    payments/
    users/
    inventory/
```

Após 18 meses: 50.000 usuários diários, 2 times (produto + checkout). O time de checkout se move em ritmo diferente do time de produto. A Black Friday exige escalar o processamento de pedidos independentemente.

**Plano de migração (Strangler Fig):**
1. Adiciona API Gateway (Nginx/Kong) na frente do monólito
2. Extrai o módulo `payments` — tem maior pressão regulatória e maior valor
3. Migra banco de dados de pagamentos (dual-write por 4 semanas, então corte)
4. Roteia `/api/payments/*` para o novo serviço
5. Em 6 meses, extrai `orders` e `inventory`

O módulo `users` permanece no monólito — baixo tráfego, alta taxa de mudança, sem necessidade de escalabilidade.

---

## 10. Perguntas de Entrevista

**Q1: Quando um monólito é a escolha certa, mesmo para uma empresa grande?**

R: Quando um bounded context é pequeno, estável e de propriedade de um time sem necessidades de escalabilidade independente. Um monólito modular para uma capacidade de negócio bem definida é quase sempre mais simples de desenvolver e operar do que um microservice. O tamanho da empresa não dita a arquitetura — o tamanho do time e a variabilidade do domínio de negócio ditam.

---

**Q2: O que é a Lei de Conway e como ela afeta decisões arquiteturais?**

R: A Lei de Conway diz que a arquitetura de sistema de uma organização tende a espelhar sua estrutura de comunicação. Uma empresa com 3 times isolados produzirá 3 subsistemas fortemente acoplados. A implicação prática: projete sua estrutura de times e fronteiras de serviço juntos. Serviços de propriedade de dois times se tornam gargalos de coordenação. Serviços de propriedade de um time com mandato claro podem se mover rápido.

---

**Q3: O que é o Strangler Fig Pattern e por que é preferido a uma reescrita completa?**

R: O Strangler Fig Pattern é uma estratégia de migração incremental: extraia partes do monólito em novos serviços um por vez, roteando tráfego gradualmente do monólito para o novo serviço. É preferido porque elimina o risco de uma reescrita "big bang" — você nunca tem um ponto onde o sistema está não funcional. Você pode validar cada serviço extraído antes de remover o código antigo.

---

**Q4: Explique a diferença entre coreografia e orquestração no padrão Saga.**

R: Na coreografia, cada serviço publica eventos e reage a eventos de outros serviços — sem coordenador central. É fracamente acoplado, mas difícil de entender como um fluxo completo. Na orquestração, um coordenador central de saga chama cada serviço em sequência e lida com transações compensatórias em caso de falha. Orquestração é mais fácil de entender e depurar, mas introduz um ponto de acoplamento de coordenação.

---

**Q5: Por que um banco de dados compartilhado entre microservices é considerado um anti-padrão?**

R: Derrota o principal benefício dos microservices — implantabilidade independente. Se dois serviços compartilham um banco, uma mudança de schema (adicionar coluna, renomear tabela) deve ser coordenada entre os dois serviços e implantada simultaneamente. Isso cria acoplamento de deploy, o que significa que você perde a capacidade de implantar o Serviço A sem coordenar com o Serviço B. Ownership de dados é uma preocupação arquitetural de primeira classe.

---

**Q6: O que é um monólito distribuído e como você o reconhece?**

R: Um monólito distribuído tem a complexidade operacional dos microservices (processos separados, chamadas de rede, deploys independentes), mas nenhuma das independências (banco compartilhado, APIs fortemente acopladas que devem ser implantadas juntas, sem isolamento de falhas). Sinais: você deve implantar todos os serviços simultaneamente para qualquer mudança, os serviços compartilham um banco, uma falha em um serviço derruba todos os outros.

---

**Q7: Como funciona o distributed tracing e por que é essencial em microservices?**

R: O distributed tracing propaga um ID de trace único entre todos os serviços que tratam uma única requisição. Cada serviço adiciona um span (uma operação cronometrada) ao trace. Ferramentas como Jaeger, Zipkin ou OpenTelemetry agregam esses spans em uma visualização de trace. Sem ele, depurar uma requisição lenta ou falha que tocou 5 serviços requer correlacionar manualmente logs de 5 sistemas — um processo que pode levar horas. Com tracing, você vê todo o ciclo de vida da requisição em uma linha do tempo.

---

**Q8: O que significa "ter seus próprios dados" para microservices e como você lida com consultas entre serviços?**

R: Cada serviço deve ser a fonte única de verdade para seus próprios dados de domínio e nunca deve permitir que outro serviço consulte seu banco diretamente. Consultas entre serviços são tratadas por: (1) chamadas de API (síncrono, tempo real), (2) replicação de dados event-driven (cada serviço mantém um read model local dos dados que precisa de outros), ou (3) serviços dedicados de leitura otimizada (lado de leitura do CQRS). O trade-off é consistência eventual vs. consistência imediata.

---

## 11. Exercícios

**Exercício 1: Design de Monólito Modular**

Pegue uma aplicação de blog simples (usuários, posts, comentários, tags) e projete uma estrutura de monólito modular. Defina as fronteiras de módulo, como é a API pública de cada módulo e quais módulos têm permissão de importar de quais outros.

*Dica: Pense em termos de bounded contexts. O módulo Tags pode existir independentemente? Comments precisa conhecer os internos de Posts, ou apenas o ID do post?*

---

**Exercício 2: Saga — criação de pedido**

Projete uma saga baseada em coreografia para o seguinte fluxo: Criar Pedido → Reservar Estoque → Cobrar Pagamento → Criar Remessa. Escreva os eventos e eventos de compensação para cada cenário de falha.

*Dica: O que acontece se o estoque for reservado mas o pagamento falhar? E se o pagamento for bem-sucedido mas a criação da remessa falhar?*

---

**Exercício 3: Strangler Fig — migrar autenticação**

Dado um monólito com um módulo de autenticação, projete o plano de migração usando o Strangler Fig Pattern. Quais são as etapas? Como você lida com dados de sessão que atualmente vivem no banco do monólito? Como você roteia o tráfego?

*Dica: Pense nas regras de roteamento do API Gateway, no período de dual-write para dados de sessão e no plano de rollback.*

---

## 12. Leitura Complementar

- **Building Microservices** — Sam Newman (2ª edição, 2021, o livro definitivo)
- **Monolith to Microservices** — Sam Newman (focado em padrões de migração)
- **[Microservices](https://martinfowler.com/articles/microservices.html)** — artigo original de Martin Fowler (2014)
- **[Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)** — Martin Fowler
- **[The Pattern: Saga](https://microservices.io/patterns/data/saga.html)** — catálogo de padrões microservices.io
- **[Conway's Law](https://en.wikipedia.org/wiki/Conway%27s_law)** — paper original e interpretação moderna
- **OpenTelemetry** — [opentelemetry.io](https://opentelemetry.io/) — padrão para distributed tracing
