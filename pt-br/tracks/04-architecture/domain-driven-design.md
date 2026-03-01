# Domain-Driven Design

## 1. O que é e por que importa

Domain-Driven Design (DDD) é uma abordagem de desenvolvimento de software introduzida por Eric Evans em seu livro de 2003 *Domain-Driven Design: Tackling Complexity in the Heart of Software*. Ela centraliza o processo de design no domínio de negócio — as regras, a linguagem e os conceitos que fazem o negócio funcionar — em vez de na tecnologia.

DDD não é um framework nem um conjunto de estruturas de pastas. É uma forma de pensar e se comunicar. Suas ferramentas se dividem em dois grupos:

- **DDD Estratégico:** Como dividir um sistema grande em partes significativas e como os times devem se comunicar.
- **DDD Tático:** Como modelar o domínio de negócio no código — os blocos de construção.

> 💡 A ideia mais poderosa no DDD não são aggregates ou repositórios — é a *linguagem ubíqua*. Um vocabulário compartilhado entre desenvolvedores e especialistas de domínio, usado consistentemente em conversas, código, testes e documentação, elimina categorias inteiras de mal-entendidos.

Quando o DDD compensa?

- Domínios complexos com regras de negócio não triviais
- Múltiplos times trabalhando no mesmo sistema grande
- Sistemas que precisam evoluir ao longo de anos conforme o negócio muda

Quando o DDD adiciona complexidade desnecessária?

- Aplicações CRUD simples sem lógica de negócio real
- Sistemas de relatórios ou analytics (fluxos de dados, não modelos de domínio)
- Startups em fase inicial onde o domínio ainda está sendo descoberto

---

## 2. Conceitos Fundamentais

### DDD Estratégico

#### Linguagem Ubíqua

Um vocabulário rigoroso e compartilhado usado por especialistas de domínio e desenvolvedores igualmente, em conversas, no código e na documentação. Se o especialista de domínio diz "fatura" e o desenvolvedor escreve `Bill`, existe uma lacuna que vai causar bugs.

```typescript
// RUIM: linguagem inventada pelo desenvolvedor
class Bill { items: LineItem[] }
class LineItem { productCode: string; qty: number; }

// BOM: linguagem ubíqua do domínio
class Invoice { lineItems: InvoiceLine[] }
class InvoiceLine { sku: string; quantity: number; unitPrice: Money; }
```

#### Bounded Context

Uma fronteira dentro da qual um modelo de domínio específico se aplica. O mesmo conceito (ex: "Cliente") pode significar coisas diferentes em diferentes bounded contexts:

- Em *Vendas*: um prospect com dados de contato e pontuação de lead
- Em *Faturamento*: uma conta com métodos de pagamento e faturas
- Em *Logística*: um destinatário com endereço de entrega

Cada bounded context tem seu próprio modelo. Tentar criar uma única entidade `Customer` unificada para todos os contextos produz uma bagunça inchada e acoplada.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Sales BC      │     │   Billing BC    │     │  Shipping BC    │
│                 │     │                 │     │                 │
│  Customer       │     │  BillingAccount │     │  Recipient      │
│  (prospect)     │     │  (payments)     │     │  (address)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

#### Context Map

Um diagrama mostrando como os bounded contexts se relacionam e se integram:

- **Shared Kernel:** Dois contextos compartilham um subconjunto pequeno do modelo de domínio
- **Customer/Supplier:** A saída de um contexto alimenta a entrada de outro
- **Anti-Corruption Layer (ACL):** Uma camada de tradução que protege um contexto do modelo de outro
- **Conformist:** Um contexto downstream adota o modelo upstream como está
- **Open Host Service:** Um contexto expõe uma API bem definida para outros

### DDD Tático

| Bloco de Construção | Descrição | Tem identidade? |
|--------------------|-----------|----------------|
| **Entity** | Um objeto definido por identidade, não por atributos | Sim (ID estável) |
| **Value Object** | Um objeto definido por seus atributos; imutável | Não |
| **Aggregate** | Um cluster de entities e value objects com uma única raiz | A raiz tem identidade |
| **Domain Event** | Um registro imutável de que algo aconteceu | N/A |
| **Repository** | Abstração para persistir e recuperar aggregates | N/A |
| **Domain Service** | Lógica sem estado que naturalmente não pertence a uma entity | N/A |
| **Application Service** | Orquestra use cases; camada fina acima do domínio | N/A |

---

## 3. Como Funciona

Um modelo de domínio em DDD é uma colaboração de entities, value objects e aggregates que impõem invariantes de negócio. O aggregate é o bloco de construção fundamental:

- A **raiz do aggregate** é o único ponto de entrada — todas as mudanças passam por ela.
- A raiz do aggregate impõe **invariantes** (regras de negócio que devem sempre ser válidas).
- Aggregates se comunicam entre si apenas por **domain events** ou **IDs de aggregate** — nunca por referências diretas a objetos.
- **Repositórios** carregam e salvam aggregates completos como uma unidade.

O fluxo:

```
Application Service (orquestração fina)
    │
    ├── carrega aggregate via Repository
    │
    ├── chama métodos na raiz do aggregate (lógica de negócio executa aqui)
    │
    ├── aggregate levanta Domain Events
    │
    ├── salva aggregate via Repository
    │
    └── publica Domain Events (para event bus, handlers, outros BCs)
```

---

## 4. Exemplos de Código (TypeScript)

### Value Object: Money

```typescript
// src/domain/value-objects/Money.ts
export class Money {
  private constructor(
    private readonly _amount: number,   // em centavos para evitar ponto flutuante
    private readonly _currency: string,
  ) {
    if (_amount < 0) throw new Error('O valor monetário não pode ser negativo');
    if (!_currency || _currency.length !== 3) throw new Error('Código de moeda inválido');
  }

  static of(amount: number, currency: string): Money {
    return new Money(Math.round(amount * 100), currency.toUpperCase());
  }

  static zero(currency: string): Money {
    return new Money(0, currency.toUpperCase());
  }

  get amount(): number { return this._amount / 100; }
  get currency(): string { return this._currency; }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this._amount + other._amount, this._currency);
  }

  multiply(factor: number): Money {
    if (factor < 0) throw new Error('O fator deve ser não-negativo');
    return new Money(Math.round(this._amount * factor), this._currency);
  }

  equals(other: Money): boolean {
    return this._amount === other._amount && this._currency === other._currency;
  }

  isGreaterThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount > other._amount;
  }

  toString(): string {
    return `${this.currency} ${this.amount.toFixed(2)}`;
  }

  private assertSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new Error(`Moedas incompatíveis: ${this._currency} vs ${other._currency}`);
    }
  }
}
```

### Value Object: OrderItem

```typescript
// src/domain/value-objects/OrderItem.ts
import { Money } from './Money';

export class OrderItem {
  private constructor(
    public readonly productId: string,
    public readonly productName: string,
    public readonly quantity: number,
    public readonly unitPrice: Money,
  ) {
    if (quantity <= 0) throw new Error('A quantidade deve ser positiva');
    if (!productId) throw new Error('ID do produto é obrigatório');
  }

  static create(
    productId: string,
    productName: string,
    quantity: number,
    unitPrice: Money,
  ): OrderItem {
    return new OrderItem(productId, productName, quantity, unitPrice);
  }

  get subtotal(): Money {
    return this.unitPrice.multiply(this.quantity);
  }

  equals(other: OrderItem): boolean {
    return this.productId === other.productId && this.quantity === other.quantity;
  }
}
```

### Domain Event

```typescript
// src/domain/events/OrderPlaced.ts
export class OrderPlaced {
  public readonly occurredAt: Date;

  constructor(
    public readonly orderId: string,
    public readonly customerId: string,
    public readonly total: number,
    public readonly currency: string,
  ) {
    this.occurredAt = new Date();
  }
}
```

```typescript
// src/domain/events/OrderCancelled.ts
export class OrderCancelled {
  public readonly occurredAt: Date;

  constructor(
    public readonly orderId: string,
    public readonly customerId: string,
    public readonly reason: string,
  ) {
    this.occurredAt = new Date();
  }
}
```

### Entity: Order (Raiz do Aggregate)

```typescript
// src/domain/entities/Order.ts
import { OrderItem } from '../value-objects/OrderItem';
import { Money } from '../value-objects/Money';
import { OrderPlaced } from '../events/OrderPlaced';
import { OrderCancelled } from '../events/OrderCancelled';

export type OrderStatus = 'draft' | 'placed' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';

type DomainEvent = OrderPlaced | OrderCancelled;

export class Order {
  private _items: OrderItem[] = [];
  private _status: OrderStatus = 'draft';
  private _domainEvents: DomainEvent[] = [];

  private constructor(
    private readonly _id: string,
    private readonly _customerId: string,
    private readonly _currency: string,
    private readonly _placedAt?: Date,
  ) {}

  // Factory method para novos pedidos
  static create(id: string, customerId: string, currency: string): Order {
    if (!customerId) throw new Error('ID do cliente é obrigatório');
    return new Order(id, customerId, currency);
  }

  // Factory method para reconstituição a partir da persistência
  static reconstitute(props: {
    id: string;
    customerId: string;
    currency: string;
    status: OrderStatus;
    items: OrderItem[];
    placedAt?: Date;
  }): Order {
    const order = new Order(props.id, props.customerId, props.currency, props.placedAt);
    order._status = props.status;
    order._items = props.items;
    return order;
  }

  // --- Getters ---
  get id(): string { return this._id; }
  get customerId(): string { return this._customerId; }
  get status(): OrderStatus { return this._status; }
  get items(): ReadonlyArray<OrderItem> { return this._items; }
  get placedAt(): Date | undefined { return this._placedAt; }

  get total(): Money {
    return this._items.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.zero(this._currency),
    );
  }

  get domainEvents(): ReadonlyArray<DomainEvent> { return this._domainEvents; }

  clearEvents(): void { this._domainEvents = []; }

  // --- Comportamento de negócio (invariantes impositosaquui) ---
  addItem(item: OrderItem): void {
    if (this._status !== 'draft') {
      throw new Error(`Não é possível adicionar itens a um pedido com status: ${this._status}`);
    }

    const existing = this._items.find(i => i.productId === item.productId);
    if (existing) {
      // Substitui o item existente (aggregate mantém consistência)
      this._items = this._items.filter(i => i.productId !== item.productId);
    }

    this._items.push(item);
  }

  removeItem(productId: string): void {
    if (this._status !== 'draft') {
      throw new Error('Não é possível remover itens de um pedido não-rascunho');
    }
    this._items = this._items.filter(i => i.productId !== productId);
  }

  place(): void {
    if (this._status !== 'draft') {
      throw new Error('O pedido já foi efetuado');
    }
    if (this._items.length === 0) {
      throw new Error('Não é possível efetuar um pedido vazio');
    }
    if (this.total.isGreaterThan(Money.zero(this._currency)) === false) {
      throw new Error('O valor total do pedido deve ser maior que zero');
    }

    this._status = 'placed';

    this._domainEvents.push(
      new OrderPlaced(this._id, this._customerId, this.total.amount, this._currency)
    );
  }

  cancel(reason: string): void {
    const cancellableStatuses: OrderStatus[] = ['draft', 'placed', 'confirmed'];
    if (!cancellableStatuses.includes(this._status)) {
      throw new Error(`Não é possível cancelar pedido com status: ${this._status}`);
    }

    this._status = 'cancelled';

    this._domainEvents.push(
      new OrderCancelled(this._id, this._customerId, reason)
    );
  }

  confirm(): void {
    if (this._status !== 'placed') {
      throw new Error('Apenas pedidos efetuados podem ser confirmados');
    }
    this._status = 'confirmed';
  }

  ship(): void {
    if (this._status !== 'confirmed') {
      throw new Error('Apenas pedidos confirmados podem ser enviados');
    }
    this._status = 'shipped';
  }

  deliver(): void {
    if (this._status !== 'shipped') {
      throw new Error('Apenas pedidos enviados podem ser entregues');
    }
    this._status = 'delivered';
  }
}
```

### Interface do Repositório

```typescript
// src/domain/repositories/IOrderRepository.ts
import { Order } from '../entities/Order';

export interface IOrderRepository {
  findById(id: string): Promise<Order | null>;
  findByCustomerId(customerId: string): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(id: string): Promise<void>;
}
```

### Application Service

```typescript
// src/application/PlaceOrderService.ts
import { randomUUID } from 'crypto';
import { Order } from '../domain/entities/Order';
import { OrderItem } from '../domain/value-objects/OrderItem';
import { Money } from '../domain/value-objects/Money';
import { IOrderRepository } from '../domain/repositories/IOrderRepository';

export interface PlaceOrderInput {
  customerId: string;
  currency: string;
  items: Array<{
    productId: string;
    productName: string;
    quantity: number;
    unitPrice: number;
  }>;
}

export interface PlaceOrderOutput {
  orderId: string;
  total: string;
  status: string;
}

// Application services são finos — orquestram, não decidem
export class PlaceOrderService {
  constructor(
    private readonly orderRepo: IOrderRepository,
    private readonly eventBus: { publish(events: unknown[]): Promise<void> },
  ) {}

  async execute(input: PlaceOrderInput): Promise<PlaceOrderOutput> {
    const order = Order.create(randomUUID(), input.customerId, input.currency);

    for (const i of input.items) {
      const item = OrderItem.create(
        i.productId,
        i.productName,
        i.quantity,
        Money.of(i.unitPrice, input.currency),
      );
      order.addItem(item);
    }

    order.place(); // lógica de domínio + evento levantado

    await this.orderRepo.save(order);
    await this.eventBus.publish(order.domainEvents);
    order.clearEvents();

    return {
      orderId: order.id,
      total: order.total.toString(),
      status: order.status,
    };
  }
}
```

### Domain Service: TaxCalculator

Quando uma operação de negócio não pertence naturalmente a uma única entity, extraia um Domain Service:

```typescript
// src/domain/services/TaxCalculator.ts
import { Money } from '../value-objects/Money';
import { Order } from '../entities/Order';

// Domain Service: lógica stateless e ciente do domínio
export class TaxCalculator {
  // Regras de imposto são conhecimento de domínio, não infraestrutura
  calculate(order: Order, taxRate: number): Money {
    if (taxRate < 0 || taxRate > 1) {
      throw new Error('A alíquota de imposto deve estar entre 0 e 1');
    }
    return order.total.multiply(taxRate);
  }
}
```

### Anti-Corruption Layer

Ao integrar com um sistema legado ou serviço externo, uma ACL traduz o modelo deles para a linguagem do seu domínio:

```typescript
// src/infrastructure/acl/LegacyOrderACL.ts
// Sistema legado usa 'order_no', 'cust_id', 'prod_code', 'amt'
interface LegacyOrderRecord {
  order_no: string;
  cust_id: string;
  prod_code: string;
  qty: number;
  amt: string; // "123.45"
}

import { Order } from '../../domain/entities/Order';
import { OrderItem } from '../../domain/value-objects/OrderItem';
import { Money } from '../../domain/value-objects/Money';

export class LegacyOrderACL {
  toDomain(legacy: LegacyOrderRecord): Order {
    const order = Order.create(legacy.order_no, legacy.cust_id, 'BRL');
    const item = OrderItem.create(
      legacy.prod_code,
      legacy.prod_code, // sistema legado não tem nome do produto
      legacy.qty,
      Money.of(parseFloat(legacy.amt), 'BRL'),
    );
    order.addItem(item);
    return order;
  }

  fromDomain(order: Order): LegacyOrderRecord {
    const firstItem = order.items[0];
    return {
      order_no: order.id,
      cust_id: order.customerId,
      prod_code: firstItem?.productId ?? '',
      qty: firstItem?.quantity ?? 0,
      amt: order.total.amount.toFixed(2),
    };
  }
}
```

---

## 5. Erros Comuns e Armadilhas

| Erro | Consequência | Correção |
|------|-------------|----------|
| Modelo de domínio anêmico | Lógica de negócio espalhada em serviços; sem imposição de invariantes | Coloque comportamento nas entities e value objects |
| Compartilhar entities entre bounded contexts | Acoplamento forte; a entity cresce para satisfazer todos os contextos | Defina modelos separados por contexto |
| Aggregates muito grandes | Baixa concorrência; todas as escritas são serializadas por uma única raiz | Identifique a fronteira real de consistência; divida se possível |
| Aggregates muito pequenos | Problemas de consistência entre aggregates | Pergunte: "o que deve ser sempre consistente junto?" |
| Chamar repositórios dentro de entities de domínio | Entities dependem de infraestrutura | Mantenha entities puras; orquestre em application services |
| Tratar DDD como estrutura de pastas | Perdendo a mudança conceitual | Foque em linguagem ubíqua e bounded contexts primeiro |

> ⚠️ O maior erro no DDD é pular a linguagem ubíqua e ir direto para aggregates e repositórios. Sem linguagem compartilhada, mesmo padrões táticos perfeitos produzem código que os especialistas de domínio não conseguem verificar.

---

## 6. Quando DDD Vale a Pena vs. Quando é Excessivo

**DDD vale a pena quando:**
- O domínio tem regras complexas e não óbvias que mudam independentemente da tecnologia
- Múltiplos times trabalham no mesmo sistema grande e precisam de fronteiras explícitas
- O negócio pode participar das sessões de design (acesso a especialistas de domínio)
- O sistema será mantido por muitos anos com requisitos em evolução

**DDD é excessivo quando:**
- A aplicação é um CRUD simples sobre um banco de dados (sem regras de negócio reais)
- O domínio é trivial e bem compreendido (ex: um blog, uma lista de tarefas)
- O time não tem acesso a especialistas de domínio e nenhuma linguagem compartilhada pode ser construída
- O projeto é descartável ou uma prova de conceito de curto prazo

> 💡 Um teste útil: se você pode descrever toda funcionalidade como "o usuário digita dados, a gente armazena, a gente exibe de volta", DDD adiciona complexidade sem valor. Se você se pega dizendo "mas espera, um pedido só pode ser cancelado se não foi enviado, a menos que seja pré-venda e o armazém confirmou..." — DDD é para você.

---

## 7. Cenário Real

Uma plataforma de e-commerce descobre que o domínio de `Order` está ficando complexo: diferentes regras de cancelamento por status, cálculos de imposto que variam por região, e necessidade de publicar eventos para um sistema de armazém.

Com DDD:

- O aggregate `Order` impõe todas as transições de estado — nenhum status inválido pode ser definido.
- O value object `Money` evita incompatibilidades de moeda e bugs de ponto flutuante.
- O domain event `OrderPlaced` desacopla a notificação do armazém do serviço de pedidos.
- Uma Anti-Corruption Layer traduz entre o modelo moderno de pedido e a API legada do armazém.

O analista de negócio pode ler o código de domínio e verificar as regras sem precisar entender schemas de banco de dados ou handlers HTTP.

---

## 8. Perguntas de Entrevista

**Q1: Qual é a diferença entre Entity e Value Object?**

R: Uma Entity é definida por identidade — duas entities com os mesmos atributos mas IDs diferentes são objetos diferentes. Um Value Object é definido por seus atributos — dois value objects com atributos idênticos são iguais, e são imutáveis. Use value objects para conceitos como Money, Address e DateRange.

---

**Q2: O que é um Aggregate e por que a raiz do aggregate é importante?**

R: Um Aggregate é um cluster de objetos de domínio (entities + value objects) tratados como uma unidade para mudanças de dados. A raiz do aggregate é o único ponto de entrada — todo código externo interage apenas com a raiz, que impõe todos os invariantes. Isso garante que o aggregate está sempre em um estado válido.

---

**Q3: O que é Linguagem Ubíqua e por que é o conceito mais importante do DDD?**

R: Linguagem Ubíqua é um vocabulário compartilhado desenvolvido por desenvolvedores e especialistas de domínio juntos, usado consistentemente no código, testes, documentação e conversas. É o conceito mais importante porque elimina erros de tradução entre o que o negócio precisa e o que o código faz. Quando o código usa os mesmos termos que o negócio usa, mal-entendidos se tornam visíveis imediatamente.

---

**Q4: Como Bounded Contexts ajudam a gerenciar complexidade em sistemas grandes?**

R: Bounded Contexts definem explicitamente onde um modelo de domínio específico se aplica. O mesmo conceito (ex: "Customer") pode ter modelos diferentes em diferentes contextos. Isso evita o anti-padrão de big-ball-of-mud onde uma única entidade `Customer` cresce para servir 10 contextos diferentes e se torna impossível de manter.

---

**Q5: O que é uma Anti-Corruption Layer e quando você precisa de uma?**

R: Uma ACL é uma camada de tradução entre seu modelo de domínio e um modelo externo (sistema legado, API de terceiro ou outro bounded context). Você precisa de uma quando o modelo externo usa conceitos, linguagem ou estrutura diferentes do seu domínio — a ACL evita que o modelo externo "corrompa" seu modelo limpo de domínio.

---

**Q6: Qual é a diferença entre Domain Service e Application Service?**

R: Um Domain Service contém lógica de negócio que naturalmente não pertence a uma única entity (ex: cálculos cross-aggregate, regras de imposto complexas). É stateless e usa vocabulário de domínio. Um Application Service é uma camada fina de orquestração — carrega aggregates via repositórios, chama lógica de domínio, salva e publica eventos. Application Services não têm lógica de negócio própria.

---

**Q7: Como Domain Events habilitam acoplamento fraco entre bounded contexts?**

R: Um Domain Event é um registro imutável de que algo aconteceu no domínio (ex: `OrderPlaced`). Bounded contexts assinam eventos de outros contextos sem depender diretamente de seus modelos. O contexto de `Shipping` assina `OrderPlaced` do contexto de `Sales` e cria um `Shipment` em seu próprio modelo — sem dependência direta.

---

**Q8: Quando você deve dividir um Aggregate em dois?**

R: Divida quando: (1) dois grupos de objetos não precisam ser consistentes na mesma transação, (2) uma raiz tem tantos filhos que escritas concorrentes são frequentemente bloqueadas, ou (3) você carrega um grafo enorme de objetos só para mudar um atributo. A regra de ouro: aggregates devem ser o menor possível enquanto ainda impõem todos os seus invariantes.

---

## 9. Exercícios

**Exercício 1: Complete o aggregate Order**

O aggregate `Order` acima está sem os domain events para `confirm()`, `ship()` e `deliver()`. Adicione `OrderConfirmed`, `OrderShipped` e `OrderDelivered` e levante-os nos métodos correspondentes.

*Dica: Siga o mesmo padrão de `OrderPlaced` — o evento carrega apenas os dados que o consumidor precisa.*

---

**Exercício 2: Implemente um Value Object de desconto**

Crie um value object `Discount` que representa um desconto percentual ou fixo. Adicione um método `applyTo(price: Money): Money`. Garanta que valide: percentual deve ser 0–100, fixo deve ser positivo.

*Dica: Use uma classe imutável com construtor privado e factory method estático.*

---

**Exercício 3: Modele um carrinho de compras como aggregate separado**

Projete um aggregate `Cart` que permite adicionar e remover itens antes de efetuar um pedido. Um `Cart` pode ser convertido em `Order`. Decida: `Cart` e `Order` devem compartilhar o mesmo value object `OrderItem`, ou ter um cada?

*Dica: Provavelmente compartilham os mesmos conceitos de produto e precificação, mas podem ter regras de validação diferentes.*

---

**Exercício 4: Identifique bounded contexts**

Para um sistema de reservas de hotel, identifique pelo menos 3 bounded contexts e o que "Cliente" significa em cada um. Desenhe um context map simples mostrando como eles se relacionam.

*Dica: Pense nos contextos de Reservas, Faturamento, Governança e Fidelidade.*

---

## 10. Leitura Complementar

- **Domain-Driven Design: Tackling Complexity in the Heart of Software** — Eric Evans (2003, o original)
- **Implementing Domain-Driven Design** — Vaughn Vernon (2013, mais prático e acessível)
- **Domain-Driven Design Distilled** — Vaughn Vernon (2016, introdução concisa)
- **[DDD Reference](https://www.domainlanguage.com/ddd/reference/)** — resumo oficial de Eric Evans (PDF gratuito)
- **[Série DDD de Khalil Stemmler](https://khalilstemmler.com/articles/domain-driven-design-intro/)** — excelentes artigos com foco em TypeScript
- **[Artigos de DDD de Martin Fowler](https://martinfowler.com/tags/domain%20driven%20design.html)** — padrões explicados com clareza
