# Arquitetura Hexagonal

## 1. O que é e por que importa

A Arquitetura Hexagonal — também chamada de "Ports and Adapters" — foi criada por Alistair Cockburn em 2005. A ideia central é isolar o núcleo da aplicação (sua lógica de negócio) de tudo que está fora: bancos de dados, HTTP, filas de mensagens, ferramentas de CLI, serviços de terceiros.

O nome "hexagonal" é metafórico — os seis lados representam múltiplos ports, não literalmente seis. O que importa é a forma: um núcleo rodeado de adapters plugáveis.

> 💡 A insight fundamental: se você consegue executar toda a lógica de domínio com zero dependências externas (sem banco de dados, sem HTTP), você tem uma arquitetura hexagonal saudável.

Por que Arquitetura Hexagonal?

- **Testabilidade sem infraestrutura:** Substitua bancos reais por fakes em memória; teste a lógica de negócio em milissegundos.
- **Agnositismo tecnológico:** O núcleo não se importa se é acionado por HTTP, CLI ou uma fila de mensagens.
- **Desenvolvimento paralelo:** Adapters de infraestrutura e lógica de negócio podem ser desenvolvidos simultaneamente.
- **Fronteiras explícitas:** Toda interação com o mundo externo passa por um port nomeado.

A Arquitetura Hexagonal e a Clean Architecture resolvem o mesmo problema. A Clean Architecture é mais prescritiva sobre camadas; a Hexagonal é mais focada no vocabulário de port/adapter.

---

## 2. Conceitos Fundamentais

### As Três Zonas

```
┌─────────────────────────────────────────────────────────┐
│  EXTERNO                                                │
│  (HTTP, CLI, DB, Email, Queue, APIs de terceiros)       │
│                                                         │
│         [Primary Adapter]     [Secondary Adapter]       │
│                │                       │                │
│         Primary Port            Secondary Port          │
│                │                       │                │
│  ┌─────────────▼───────────────────────▼─────────────┐ │
│  │              NÚCLEO DA APLICAÇÃO                   │ │
│  │         (Lógica de Domínio + Use Cases)            │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Ports

Um **port** é uma interface — um contrato que define como o mundo externo se comunica com o núcleo (ou o núcleo com o mundo externo).

- **Primary ports (ports de acionamento):** Chamados *por* atores externos para acionar o comportamento da aplicação. Exemplos: `IOrderService`, `ICheckoutService`. Controllers HTTP, comandos de CLI e suites de teste são primary adapters que chamam esses ports.
- **Secondary ports (ports acionados):** Chamados *pelo* núcleo da aplicação para acessar sistemas externos. Exemplos: `IOrderRepository`, `IPaymentGateway`, `IEmailSender`. Adapters de banco de dados e integrações de terceiros implementam esses ports.

### Adapters

Um **adapter** traduz entre uma tecnologia externa e a interface do port.

- **Primary adapters:** Um route handler do Fastify que chama `IOrderService.placeOrder()`. Um comando de CLI que chama o mesmo serviço. Ambos são primary adapters para o mesmo primary port.
- **Secondary adapters:** `PrismaOrderRepository implements IOrderRepository`. `StripePaymentAdapter implements IPaymentGateway`. `NodemailerEmailAdapter implements IEmailSender`.

> ⚠️ Ports são definidos *dentro* do núcleo da aplicação. Adapters vivem *fora* e dependem do núcleo, nunca o contrário. É o mesmo Dependency Inversion Principle do SOLID — aplicado em escala arquitetural.

---

## 3. Como Funciona

Ciclo de vida de uma requisição em uma aplicação Hexagonal:

```
1. HTTP POST /orders
       │
       ▼
2. FastifyOrderController (Primary Adapter)
   - Valida a HTTP request
   - Mapeia para PlaceOrderCommand
   - Chama IOrderService.placeOrder()
       │
       ▼
3. OrderService (Núcleo da Aplicação)
   - Impõe regras de negócio
   - Chama IOrderRepository.findById()
   - Chama IPaymentGateway.charge()
   - Chama INotificationPort.notifyUser()
   - Retorna OrderPlacedResult
       │
       ▼
4. Adapters executam:
   - PrismaOrderRepository → PostgreSQL
   - StripePaymentAdapter → Stripe API
   - SESNotificationAdapter → AWS SES
       │
       ▼
5. Resultado flui de volta pelas camadas
6. Controller mapeia para resposta HTTP
```

O núcleo (passo 3) não tem conhecimento de Fastify, Prisma, Stripe ou SES. Ele conhece apenas as interfaces de seus próprios ports.

---

## 4. Exemplos de Código (TypeScript)

### Domínio: Order Aggregate

```typescript
// src/core/domain/Order.ts
export interface OrderItem {
  productId: string;
  quantity: number;
  unitPrice: number;
}

export class Order {
  private _items: OrderItem[] = [];
  private _status: 'pending' | 'confirmed' | 'cancelled' = 'pending';

  constructor(
    public readonly id: string,
    public readonly customerId: string,
    private readonly createdAt: Date = new Date(),
  ) {}

  addItem(item: OrderItem): void {
    if (this._status !== 'pending') {
      throw new Error('Não é possível modificar um pedido confirmado ou cancelado');
    }
    if (item.quantity <= 0) {
      throw new Error('A quantidade deve ser positiva');
    }
    this._items.push(item);
  }

  get total(): number {
    return this._items.reduce((sum, i) => sum + i.unitPrice * i.quantity, 0);
  }

  get items(): ReadonlyArray<OrderItem> { return this._items; }
  get status(): string { return this._status; }

  confirm(): void {
    if (this._items.length === 0) throw new Error('Não é possível confirmar um pedido vazio');
    this._status = 'confirmed';
  }
}
```

### Secondary Ports (definidos no núcleo)

```typescript
// src/core/ports/secondary/IOrderRepository.ts
import { Order } from '../domain/Order';

export interface IOrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}
```

```typescript
// src/core/ports/secondary/IPaymentGateway.ts
export interface PaymentResult {
  transactionId: string;
  status: 'approved' | 'declined';
}

export interface IPaymentGateway {
  charge(customerId: string, amount: number, currency: string): Promise<PaymentResult>;
  refund(transactionId: string, amount: number): Promise<void>;
}
```

```typescript
// src/core/ports/secondary/INotificationPort.ts
export interface INotificationPort {
  notifyOrderConfirmed(customerId: string, orderId: string, total: number): Promise<void>;
}
```

### Primary Port (definido no núcleo)

```typescript
// src/core/ports/primary/IOrderService.ts
import { Order } from '../domain/Order';

export interface PlaceOrderCommand {
  customerId: string;
  items: Array<{ productId: string; quantity: number; unitPrice: number }>;
}

export interface PlaceOrderResult {
  orderId: string;
  total: number;
  transactionId: string;
}

export interface IOrderService {
  placeOrder(command: PlaceOrderCommand): Promise<PlaceOrderResult>;
  getOrder(orderId: string): Promise<Order | null>;
}
```

### Núcleo da Aplicação: OrderService

```typescript
// src/core/OrderService.ts
import { randomUUID } from 'crypto';
import { Order } from './domain/Order';
import { IOrderRepository } from './ports/secondary/IOrderRepository';
import { IPaymentGateway } from './ports/secondary/IPaymentGateway';
import { INotificationPort } from './ports/secondary/INotificationPort';
import { IOrderService, PlaceOrderCommand, PlaceOrderResult } from './ports/primary/IOrderService';

export class OrderService implements IOrderService {
  constructor(
    private readonly orderRepo: IOrderRepository,
    private readonly paymentGateway: IPaymentGateway,
    private readonly notifications: INotificationPort,
  ) {}

  async placeOrder(command: PlaceOrderCommand): Promise<PlaceOrderResult> {
    // 1. Constrói o aggregate
    const order = new Order(randomUUID(), command.customerId);
    for (const item of command.items) {
      order.addItem(item);
    }

    // 2. Regra de negócio: valor mínimo do pedido
    if (order.total < 1) {
      throw new Error('O valor do pedido deve ser de pelo menos R$ 1,00');
    }

    // 3. Cobra via secondary port
    const payment = await this.paymentGateway.charge(
      command.customerId,
      order.total,
      'BRL',
    );

    if (payment.status === 'declined') {
      throw new Error('Pagamento recusado');
    }

    // 4. Confirma o pedido
    order.confirm();

    // 5. Persiste via secondary port
    await this.orderRepo.save(order);

    // 6. Notifica via secondary port
    await this.notifications.notifyOrderConfirmed(
      command.customerId,
      order.id,
      order.total,
    );

    return {
      orderId: order.id,
      total: order.total,
      transactionId: payment.transactionId,
    };
  }

  async getOrder(orderId: string): Promise<Order | null> {
    return this.orderRepo.findById(orderId);
  }
}
```

### Secondary Adapters (infraestrutura)

```typescript
// src/adapters/secondary/StripePaymentAdapter.ts
import Stripe from 'stripe';
import { IPaymentGateway, PaymentResult } from '../../core/ports/secondary/IPaymentGateway';

export class StripePaymentAdapter implements IPaymentGateway {
  constructor(private readonly stripe: Stripe) {}

  async charge(customerId: string, amount: number, currency: string): Promise<PaymentResult> {
    const intent = await this.stripe.paymentIntents.create({
      amount: Math.round(amount * 100), // Stripe usa centavos
      currency,
      customer: customerId,
      confirm: true,
    });
    return {
      transactionId: intent.id,
      status: intent.status === 'succeeded' ? 'approved' : 'declined',
    };
  }

  async refund(transactionId: string, amount: number): Promise<void> {
    await this.stripe.refunds.create({
      payment_intent: transactionId,
      amount: Math.round(amount * 100),
    });
  }
}
```

```typescript
// src/adapters/secondary/PrismaOrderRepository.ts
import { PrismaClient } from '@prisma/client';
import { Order } from '../../core/domain/Order';
import { IOrderRepository } from '../../core/ports/secondary/IOrderRepository';

export class PrismaOrderRepository implements IOrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string): Promise<Order | null> {
    const row = await this.prisma.order.findUnique({
      where: { id },
      include: { items: true },
    });
    if (!row) return null;

    const order = new Order(row.id, row.customerId, row.createdAt);
    for (const item of row.items) {
      order.addItem({
        productId: item.productId,
        quantity: item.quantity,
        unitPrice: Number(item.unitPrice),
      });
    }
    return order;
  }

  async save(order: Order): Promise<void> {
    await this.prisma.order.upsert({
      where: { id: order.id },
      create: {
        id: order.id,
        customerId: order.customerId,
        status: order.status,
        items: {
          create: order.items.map(i => ({
            productId: i.productId,
            quantity: i.quantity,
            unitPrice: i.unitPrice,
          })),
        },
      },
      update: { status: order.status },
    });
  }
}
```

### Primary Adapter: Fastify Controller

```typescript
// src/adapters/primary/http/OrderController.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { IOrderService } from '../../../core/ports/primary/IOrderService';

export class OrderController {
  constructor(private readonly orderService: IOrderService) {}

  async placeOrder(req: FastifyRequest, reply: FastifyReply): Promise<void> {
    const body = req.body as { customerId: string; items: unknown[] };
    try {
      const result = await this.orderService.placeOrder({
        customerId: body.customerId,
        items: body.items as any,
      });
      reply.status(201).send(result);
    } catch (err) {
      if (err instanceof Error) {
        reply.status(400).send({ error: err.message });
      }
    }
  }
}
```

### Testes: Substitua Adapters por Fakes — Sem Framework de Mock

```typescript
// tests/OrderService.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { Order } from '../src/core/domain/Order';
import { OrderService } from '../src/core/OrderService';
import { IOrderRepository } from '../src/core/ports/secondary/IOrderRepository';
import { IPaymentGateway, PaymentResult } from '../src/core/ports/secondary/IPaymentGateway';
import { INotificationPort } from '../src/core/ports/secondary/INotificationPort';

// Fakes em memória — sem framework de mock
class FakeOrderRepository implements IOrderRepository {
  private store = new Map<string, Order>();
  async findById(id: string) { return this.store.get(id) ?? null; }
  async save(order: Order) { this.store.set(order.id, order); }
}

class FakePaymentGateway implements IPaymentGateway {
  shouldDecline = false;

  async charge(): Promise<PaymentResult> {
    if (this.shouldDecline) return { transactionId: '', status: 'declined' };
    return { transactionId: 'txn_test_123', status: 'approved' };
  }

  async refund(): Promise<void> {}
}

class FakeNotifications implements INotificationPort {
  sent: Array<{ customerId: string; orderId: string; total: number }> = [];

  async notifyOrderConfirmed(customerId: string, orderId: string, total: number): Promise<void> {
    this.sent.push({ customerId, orderId, total });
  }
}

describe('OrderService', () => {
  let service: OrderService;
  let payment: FakePaymentGateway;
  let notifications: FakeNotifications;

  beforeEach(() => {
    payment = new FakePaymentGateway();
    notifications = new FakeNotifications();
    service = new OrderService(new FakeOrderRepository(), payment, notifications);
  });

  it('cria o pedido e envia notificação', async () => {
    const result = await service.placeOrder({
      customerId: 'cust_1',
      items: [{ productId: 'p1', quantity: 2, unitPrice: 25 }],
    });

    expect(result.total).toBe(50);
    expect(result.transactionId).toBe('txn_test_123');
    expect(notifications.sent).toHaveLength(1);
    expect(notifications.sent[0].total).toBe(50);
  });

  it('lança quando o pagamento é recusado', async () => {
    payment.shouldDecline = true;

    await expect(
      service.placeOrder({
        customerId: 'cust_1',
        items: [{ productId: 'p1', quantity: 1, unitPrice: 10 }],
      })
    ).rejects.toThrow('Pagamento recusado');

    expect(notifications.sent).toHaveLength(0);
  });
});
```

> 💡 Esses testes rodam em menos de 50ms sem nenhuma chamada de rede. Os fakes são extremamente simples de escrever porque os ports têm interfaces enxutas (ISP aplicado em escala arquitetural).

---

## 5. Erros Comuns e Armadilhas

| Erro | Consequência | Correção |
|------|-------------|----------|
| Definir ports na camada de infraestrutura | Inverte a dependência — núcleo depende da infra | Sempre defina ports dentro do núcleo |
| Ports muito amplos (todos os métodos CRUD) | Fakes precisam implementar métodos irrelevantes | Uma interface por cluster de capacidade |
| Pular primary ports e chamar o serviço diretamente | O serviço fica acoplado à tecnologia de chamada | Sempre defina uma interface de primary port |
| Colocar validação nos adapters | Validação de negócio é duplicada ou omitida | Valide no núcleo; adapters parsam apenas preocupações HTTP/protocolo |
| Usar o mesmo DTO dentro e fora do núcleo | A fronteira fica vazando | Defina tipos de entrada/saída separados por port |

> ⚠️ O erro mais comum: desenvolvedores colocam as interfaces de port no mesmo pacote dos adapters. Isso faz o núcleo depender do diretório de infraestrutura, anulando todo o propósito.

---

## 6. Quando Usar / Não Usar

**Use Arquitetura Hexagonal quando:**
- A aplicação tem múltiplos entry points (HTTP + CLI + event-driven)
- A infraestrutura tende a mudar (migrar de MongoDB para PostgreSQL, trocar provedores de pagamento)
- Testes unitários rápidos e isolados da lógica de negócio são prioridade do time
- O domínio de negócio é suficientemente complexo para justificar um design explícito de fronteiras

**Simplifique ou pule quando:**
- API CRUD simples com um único entry point e infraestrutura estável
- Protótipo ou MVP inicial onde o domínio ainda está sendo descoberto
- O time não está familiarizado com o padrão — o overhead do vocabulário pode atrasar a entrega inicial

> 💡 Você pode aplicar a Arquitetura Hexagonal incrementalmente. Comece com um secondary port apenas para a dependência mais volátil (ex: o gateway de pagamento) e expanda conforme a necessidade de testabilidade e flexibilidade crescer.

---

## 7. Relação com a Clean Architecture

| Aspecto | Hexagonal | Clean Architecture |
|---------|-----------|-------------------|
| Conceito central | Ports e Adapters | Camadas com a Regra de Dependência |
| Terminologia | Ports primários/secundários, adapters primários/secundários | Entities, use cases, interface adapters, frameworks |
| Prescritividade | Menos prescritiva sobre camadas internas | Mais prescritiva (nomes explícitos de camadas) |
| Origem | Alistair Cockburn, 2005 | Robert C. Martin, 2012 |
| Foco | Design de fronteiras externas | Hierarquia completa de camadas incluindo regras de entity |

Ambas dizem: *camadas internas não devem depender de camadas externas.* São complementares, não concorrentes. Muitos times usam o vocabulário de camadas da Clean Architecture com a linguagem de port/adapter da Hexagonal simultaneamente.

---

## 8. Perguntas de Entrevista

**Q1: Qual é a diferença entre um port e um adapter?**

R: Um port é uma interface — um contrato nomeado no núcleo da aplicação que define como a interação externa acontece. Um adapter é uma classe concreta que implementa um port usando uma tecnologia específica (Prisma, Stripe, SES). Ports vivem dentro do núcleo; adapters vivem fora.

---

**Q2: Qual é a diferença entre um primary port e um secondary port?**

R: Primary ports (de acionamento) são chamados por atores externos para acionar o comportamento da aplicação — representam a API da aplicação. Secondary ports (acionados) são chamados pelo núcleo para acessar sistemas externos — representam as dependências da aplicação. Primary adapters (HTTP, CLI) chamam primary ports; secondary adapters (DB, e-mail) implementam secondary ports.

---

**Q3: Por que fakes em memória são melhores que frameworks de mock em arquiteturas hexagonais?**

R: Fakes implementam a interface do port diretamente e podem manter estado, tornando os testes legíveis e realistas sem a fragilidade das expectativas de mock. Mocks verificam que métodos específicos foram chamados de formas específicas — fakes verificam que o resultado está correto. Fakes também são mais fáceis de manter porque rastreiam mudanças de estado reais.

---

**Q4: Como a Arquitetura Hexagonal facilita a adição de um novo mecanismo de entrega?**

R: Como o núcleo da aplicação conhece apenas as interfaces de seus ports, um novo primary adapter (ex: uma CLI ou um servidor gRPC) simplesmente implementa o primary port e chama o serviço. O adapter HTTP existente, todos os testes e toda a lógica de negócio permanecem intocados.

---

**Q5: Você pode ter múltiplos primary adapters para o mesmo primary port?**

R: Sim — esse é um dos principais benefícios. Um primary port `IOrderService` pode ser chamado por um controller HTTP, um comando de CLI, um job agendado e uma suite de testes simultaneamente. Cada um é um adapter diferente para o mesmo port, e não compartilham código.

---

**Q6: Como lidar com configuração e setup de infraestrutura sem poluir o núcleo?**

R: O bootstrap de infraestrutura (Prisma client, Stripe SDK, conexão SMTP) vive inteiramente nos adapters e na raiz de composição (`main.ts`). O núcleo recebe apenas instâncias de suas interfaces de port — ele nunca lê variáveis de ambiente nem cria conexões de banco de dados.

---

## 9. Exercícios

**Exercício 1: Adicione um secondary adapter de notificação por e-mail**

Implemente `NodemailerNotificationAdapter implements INotificationPort` que envia um e-mail real. O `OrderService` do núcleo não deve mudar em nada.

*Dica: O adapter lê credenciais SMTP de variáveis de ambiente. Teste com `FakeNotifications` nos testes unitários e com o adapter real nos testes de integração.*

---

**Exercício 2: Adicione um primary adapter de CLI**

Escreva um script CLI (`cli/place-order.ts`) que lê dados do pedido de `stdin` como JSON, chama `OrderService.placeOrder()` e imprime o resultado. Use o mesmo serviço que o adapter HTTP usa.

*Dica: Conecte os mesmos adapters de repositório e gateway que em `main.ts`. O código do serviço é idêntico.*

---

**Exercício 3: Teste cenários de falha de pagamento**

Usando `FakePaymentGateway`, expanda a suite de testes para cobrir:
- Saldo insuficiente (pagamento recusado)
- Pedido vazio (sem itens)
- Pedido com um único item a preço zero

*Dica: Adicione um campo `declineReason` ao `FakePaymentGateway` para simular diferentes modos de falha.*

---

**Exercício 4: Troque o gateway de pagamento**

Implemente um `PayPalPaymentAdapter implements IPaymentGateway` (implementação stub está ok). Conecte-o na raiz de composição mudando uma única linha. Verifique que todos os testes existentes ainda passam sem modificação.

*Dica: Os testes usam `FakePaymentGateway`, não o real, então não são afetados pela troca.*

---

## 10. Leitura Complementar

- **[Ports and Adapters (Hexagonal Architecture)](https://alistair.cockburn.us/hexagonal-architecture/)** — artigo original de Alistair Cockburn (2005)
- **[DDD, Hexagonal, Onion, Clean, CQRS — How I put it all together](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/)** — herbertograca (comparação abrangente)
- **Growing Object-Oriented Software, Guided by Tests** — Steve Freeman & Nat Pryce (filosofia de fakes sobre mocks)
- **[Hexagonal Architecture in Node.js](https://www.freecodecamp.org/news/implementing-a-hexagonal-architecture/)** — freeCodeCamp
- **Clean Architecture** — Robert C. Martin (Capítulo 22: The Clean Architecture)
