# Design Patterns: Structural

## 1. What & Why

Structural design patterns deal with how objects and classes are composed to form larger structures. They make it easier to build complex structures from simpler parts by identifying simple ways to realize relationships between entities.

The seven structural patterns from the Gang of Four:

| Pattern | Problem it solves |
|---------|------------------|
| **Adapter** | Make an incompatible interface work with another |
| **Bridge** | Separate an abstraction from its implementation so both can vary |
| **Composite** | Treat individual objects and compositions uniformly |
| **Decorator** | Add behavior to objects dynamically without subclassing |
| **Facade** | Provide a simplified interface to a complex subsystem |
| **Flyweight** | Share state to support a large number of fine-grained objects efficiently |
| **Proxy** | Provide a surrogate that controls access to another object |

> 💡 Structural patterns are about composition — they answer the question "how do these pieces fit together?" rather than "how should I create this object?" or "how should these objects communicate?"

---

## 2. Core Concepts

Structural patterns use two fundamental mechanisms:
- **Class-based patterns** (use inheritance): Adapter (class version)
- **Object-based patterns** (use composition): Adapter (object version), Bridge, Composite, Decorator, Facade, Flyweight, Proxy

Composition is generally preferred over inheritance in modern TypeScript design. Every structural pattern here (except the class-form of Adapter) uses composition.

---

## 3. How It Works

The general mechanism: a structural pattern introduces an intermediate object (adapter, decorator, proxy, facade) that sits between the client and the real implementation. This intermediary provides the translation, enhancement, or access control the pattern requires.

---

## 4. Code Examples (TypeScript)

### Adapter — Legacy Payment API

A modern checkout system expects `IPaymentProcessor`. A legacy payment provider has a completely different API. The Adapter wraps the legacy API and presents the expected interface.

```typescript
// Modern interface the system expects
interface IPaymentProcessor {
  charge(amount: number, currency: string, cardToken: string): Promise<PaymentResult>;
  refund(transactionId: string, amount: number): Promise<void>;
}

interface PaymentResult {
  transactionId: string;
  status: 'approved' | 'declined';
  errorCode?: string;
}

// Legacy API (third party — cannot change)
class LegacyPaymentGateway {
  processPayment(params: {
    amount_cents: number;
    currency_code: string;
    card_token: string;
    merchant_id: string;
  }): { tx_id: string; result_code: '00' | '01' | '99'; message: string } {
    // Simulate legacy response
    return { tx_id: 'legacy_txn_001', result_code: '00', message: 'Approved' };
  }

  reverseTransaction(tx_id: string, amount_cents: number): { success: boolean } {
    return { success: true };
  }
}

// Adapter: wraps legacy, implements modern interface
class LegacyPaymentAdapter implements IPaymentProcessor {
  private readonly MERCHANT_ID = process.env.LEGACY_MERCHANT_ID ?? 'MERCHANT_001';

  constructor(private readonly legacy: LegacyPaymentGateway) {}

  async charge(amount: number, currency: string, cardToken: string): Promise<PaymentResult> {
    const response = this.legacy.processPayment({
      amount_cents: Math.round(amount * 100),
      currency_code: currency,
      card_token: cardToken,
      merchant_id: this.MERCHANT_ID,
    });

    return {
      transactionId: response.tx_id,
      status: response.result_code === '00' ? 'approved' : 'declined',
      errorCode: response.result_code !== '00' ? response.result_code : undefined,
    };
  }

  async refund(transactionId: string, amount: number): Promise<void> {
    const result = this.legacy.reverseTransaction(transactionId, Math.round(amount * 100));
    if (!result.success) {
      throw new Error(`Refund failed for transaction ${transactionId}`);
    }
  }
}

// CheckoutService knows nothing about the legacy API
class CheckoutService {
  constructor(private readonly payment: IPaymentProcessor) {}

  async processOrder(order: { total: number; currency: string; cardToken: string }): Promise<string> {
    const result = await this.payment.charge(order.total, order.currency, order.cardToken);
    if (result.status === 'declined') throw new Error(`Payment declined: ${result.errorCode}`);
    return result.transactionId;
  }
}

// Wiring
const legacy = new LegacyPaymentGateway();
const adapter = new LegacyPaymentAdapter(legacy);
const checkout = new CheckoutService(adapter); // modern interface, legacy backend
```

---

### Bridge — Notification System

Bridge separates abstraction (what you send) from implementation (how you send it). Both sides can vary independently.

```typescript
// Implementation interface (how to send)
interface IMessageSender {
  send(destination: string, content: string): Promise<void>;
}

// Concrete implementations
class EmailSender implements IMessageSender {
  async send(destination: string, content: string): Promise<void> {
    console.log(`[EMAIL] to=${destination}: ${content}`);
  }
}

class SmsSender implements IMessageSender {
  async send(destination: string, content: string): Promise<void> {
    console.log(`[SMS] to=${destination}: ${content}`);
  }
}

// Abstraction (what you send) — holds a reference to the implementation
abstract class Notification {
  constructor(protected sender: IMessageSender) {}

  abstract send(recipient: { email: string; phone: string }): Promise<void>;
}

// Refined abstractions
class OrderConfirmationNotification extends Notification {
  constructor(sender: IMessageSender, private readonly orderId: string) {
    super(sender);
  }

  async send(recipient: { email: string; phone: string }): Promise<void> {
    const content = `Your order #${this.orderId} has been confirmed.`;
    // Choose destination based on sender type — could also inject target
    await this.sender.send(recipient.email, content);
  }
}

class PasswordResetNotification extends Notification {
  constructor(sender: IMessageSender, private readonly resetLink: string) {
    super(sender);
  }

  async send(recipient: { email: string; phone: string }): Promise<void> {
    const content = `Reset your password: ${this.resetLink}`;
    await this.sender.send(recipient.phone, content);
  }
}

// Mix any notification type with any sender — 2×2 without creating 4 subclasses
const emailOrder = new OrderConfirmationNotification(new EmailSender(), 'ORD-001');
const smsReset = new PasswordResetNotification(new SmsSender(), 'https://app.com/reset/abc');
```

> 💡 The key insight: without Bridge, you would need `EmailOrderConfirmation`, `SmsOrderConfirmation`, `EmailPasswordReset`, `SmsPasswordReset` — four classes for two axes of variation. Bridge reduces it to 2 + 2 = 4 classes instead of 2×2 = 4 (with each additional axis being additive, not multiplicative).

---

### Composite — File System

The Composite pattern lets you treat individual objects and compositions uniformly. A `File` and a `Directory` both implement `FileSystemItem`, so code that walks the tree doesn't need to distinguish between them.

```typescript
// Component interface
interface FileSystemItem {
  name: string;
  size(): number;
  print(indent?: string): void;
}

// Leaf — no children
class File implements FileSystemItem {
  constructor(
    public readonly name: string,
    private readonly _size: number,
  ) {}

  size(): number { return this._size; }

  print(indent: string = ''): void {
    console.log(`${indent}📄 ${this.name} (${this._size} bytes)`);
  }
}

// Composite — has children
class Directory implements FileSystemItem {
  private children: FileSystemItem[] = [];

  constructor(public readonly name: string) {}

  add(item: FileSystemItem): this {
    this.children.push(item);
    return this;
  }

  remove(name: string): void {
    this.children = this.children.filter(c => c.name !== name);
  }

  size(): number {
    return this.children.reduce((total, child) => total + child.size(), 0);
  }

  print(indent: string = ''): void {
    console.log(`${indent}📁 ${this.name}/ (${this.size()} bytes)`);
    for (const child of this.children) {
      child.print(indent + '  ');
    }
  }
}

// Usage — client treats files and directories identically
const root = new Directory('project')
  .add(new File('README.md', 1024))
  .add(new File('package.json', 512))
  .add(
    new Directory('src')
      .add(new File('index.ts', 2048))
      .add(new File('config.ts', 768))
      .add(
        new Directory('services')
          .add(new File('UserService.ts', 4096))
          .add(new File('OrderService.ts', 3200))
      )
  );

root.print();
// 📁 project/ (11648 bytes)
//   📄 README.md (1024 bytes)
//   📄 package.json (512 bytes)
//   📁 src/ (10112 bytes)
//     📄 index.ts (2048 bytes)
//     ...
```

---

### Decorator — Caching and Logging Wrappers

The Decorator pattern adds behavior to an object dynamically without changing the class. It is the runtime equivalent of inheritance — composable and reversible.

```typescript
interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

// Base repository
class PostgresUserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    console.log(`[DB] SELECT * FROM users WHERE id = '${id}'`);
    return { id, name: 'Alice', email: 'alice@example.com' } as User;
  }

  async findByEmail(email: string): Promise<User | null> {
    console.log(`[DB] SELECT * FROM users WHERE email = '${email}'`);
    return null;
  }

  async save(user: User): Promise<void> {
    console.log(`[DB] UPSERT user ${user.id}`);
  }
}

// Caching Decorator
class CachedUserRepository implements IUserRepository {
  private cache = new Map<string, User>();

  constructor(private readonly inner: IUserRepository) {}

  async findById(id: string): Promise<User | null> {
    if (this.cache.has(id)) {
      console.log(`[CACHE] HIT for id=${id}`);
      return this.cache.get(id)!;
    }
    const user = await this.inner.findById(id);
    if (user) this.cache.set(id, user);
    return user;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.inner.findByEmail(email); // no cache for email lookups
  }

  async save(user: User): Promise<void> {
    await this.inner.save(user);
    this.cache.set(user.id, user); // update cache on write
  }
}

// Logging Decorator
class LoggedUserRepository implements IUserRepository {
  constructor(
    private readonly inner: IUserRepository,
    private readonly logger: { info(msg: string): void },
  ) {}

  async findById(id: string): Promise<User | null> {
    const start = Date.now();
    const result = await this.inner.findById(id);
    this.logger.info(`findById(${id}) → ${result ? 'found' : 'null'} [${Date.now() - start}ms]`);
    return result;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.inner.findByEmail(email);
  }

  async save(user: User): Promise<void> {
    await this.inner.save(user);
    this.logger.info(`save(${user.id}) completed`);
  }
}

// Compose decorators — order matters: Logged wraps Cached wraps Postgres
const repo: IUserRepository = new LoggedUserRepository(
  new CachedUserRepository(
    new PostgresUserRepository()
  ),
  console,
);
```

> ⚠️ Decorators must implement the same interface as the wrapped class. If the interface changes, all decorators must be updated. Keep interfaces lean (ISP) to reduce this burden.

---

### Facade — Payment Facade

A Facade provides a simplified interface to a complex subsystem. Here, placing an order involves a payment gateway, fraud detection, and invoice generation — the facade hides this complexity.

```typescript
// Complex subsystems
class PaymentGateway {
  async charge(customerId: string, amount: number): Promise<string> {
    console.log(`[Gateway] Charging ${amount} to customer ${customerId}`);
    return 'txn_' + Date.now();
  }
}

class FraudDetectionService {
  async check(customerId: string, amount: number): Promise<boolean> {
    console.log(`[Fraud] Checking customer ${customerId} for amount ${amount}`);
    return amount < 10_000; // simple rule: flag amounts >= $10k
  }
}

class InvoiceService {
  async generate(customerId: string, transactionId: string, amount: number): Promise<string> {
    const invoiceId = `INV-${Date.now()}`;
    console.log(`[Invoice] Generated ${invoiceId} for txn ${transactionId}`);
    return invoiceId;
  }
}

class EmailService {
  async sendReceipt(email: string, invoiceId: string): Promise<void> {
    console.log(`[Email] Receipt for invoice ${invoiceId} sent to ${email}`);
  }
}

// Facade — single entry point for the checkout flow
export class PaymentFacade {
  private readonly gateway = new PaymentGateway();
  private readonly fraud = new FraudDetectionService();
  private readonly invoicing = new InvoiceService();
  private readonly email = new EmailService();

  async processPayment(params: {
    customerId: string;
    customerEmail: string;
    amount: number;
  }): Promise<{ transactionId: string; invoiceId: string }> {
    const { customerId, customerEmail, amount } = params;

    // 1. Fraud check
    const isSafe = await this.fraud.check(customerId, amount);
    if (!isSafe) throw new Error('Payment flagged as potentially fraudulent');

    // 2. Charge
    const transactionId = await this.gateway.charge(customerId, amount);

    // 3. Invoice
    const invoiceId = await this.invoicing.generate(customerId, transactionId, amount);

    // 4. Email receipt
    await this.email.sendReceipt(customerEmail, invoiceId);

    return { transactionId, invoiceId };
  }
}

// Client code — one method call, zero knowledge of subsystems
const facade = new PaymentFacade();
const result = await facade.processPayment({
  customerId: 'cust_1',
  customerEmail: 'alice@example.com',
  amount: 299.99,
});
```

---

### Flyweight — Character Rendering

The Flyweight pattern reduces memory usage by sharing common state (intrinsic) across many objects, while keeping unique state (extrinsic) outside.

```typescript
// Intrinsic state — shared across many characters (font, size, style)
interface CharacterStyle {
  font: string;
  size: number;
  bold: boolean;
  italic: boolean;
  color: string;
}

class CharacterFlyweight {
  constructor(public readonly style: CharacterStyle) {}

  render(char: string, x: number, y: number): void {
    // Rendering with shared style + unique position
    console.log(
      `Render '${char}' at (${x},${y}) — ${this.style.font} ${this.style.size}pt ${this.style.color}`
    );
  }
}

// Flyweight Factory — ensures shared instances
class CharacterFlyweightFactory {
  private cache = new Map<string, CharacterFlyweight>();

  getOrCreate(style: CharacterStyle): CharacterFlyweight {
    const key = `${style.font}-${style.size}-${style.bold}-${style.italic}-${style.color}`;
    if (!this.cache.has(key)) {
      this.cache.set(key, new CharacterFlyweight(style));
      console.log(`[Flyweight] Created new style: ${key}`);
    }
    return this.cache.get(key)!;
  }

  get cachedCount(): number { return this.cache.size; }
}

// Extrinsic state — unique per character instance (char value, position)
interface CharacterInstance {
  char: string;
  x: number;
  y: number;
  flyweight: CharacterFlyweight;
}

// Document with 10,000 characters — but only a handful of unique styles
class TextDocument {
  private characters: CharacterInstance[] = [];
  private factory = new CharacterFlyweightFactory();

  addCharacter(char: string, x: number, y: number, style: CharacterStyle): void {
    const flyweight = this.factory.getOrCreate(style);
    this.characters.push({ char, x, y, flyweight });
  }

  render(): void {
    for (const instance of this.characters) {
      instance.flyweight.render(instance.char, instance.x, instance.y);
    }
  }

  get uniqueStyleCount(): number { return this.factory.cachedCount; }
}
```

---

### Proxy — Lazy Loading, Caching, and Access Control

The Proxy controls access to the real object. Three common use cases:

```typescript
// 1. Lazy-loading proxy — only initialize the real object when needed
interface IReportGenerator {
  generate(params: Record<string, unknown>): Promise<string>;
}

class HeavyReportGenerator implements IReportGenerator {
  constructor() {
    // Expensive initialization (ML model, large DB load, etc.)
    console.log('[ReportGenerator] Initialized (expensive)');
  }

  async generate(params: Record<string, unknown>): Promise<string> {
    return `Report generated with params: ${JSON.stringify(params)}`;
  }
}

class LazyReportProxy implements IReportGenerator {
  private real: HeavyReportGenerator | null = null;

  async generate(params: Record<string, unknown>): Promise<string> {
    if (!this.real) {
      this.real = new HeavyReportGenerator(); // initialized only on first use
    }
    return this.real.generate(params);
  }
}

// 2. Caching proxy
class CachingReportProxy implements IReportGenerator {
  private cache = new Map<string, string>();

  constructor(private readonly inner: IReportGenerator) {}

  async generate(params: Record<string, unknown>): Promise<string> {
    const key = JSON.stringify(params);
    if (this.cache.has(key)) {
      console.log('[Cache] HIT');
      return this.cache.get(key)!;
    }
    const result = await this.inner.generate(params);
    this.cache.set(key, result);
    return result;
  }
}

// 3. Access control proxy
class AuthorizedReportProxy implements IReportGenerator {
  constructor(
    private readonly inner: IReportGenerator,
    private readonly requiredRole: string,
  ) {}

  async generate(params: Record<string, unknown> & { userRole?: string }): Promise<string> {
    if (params.userRole !== this.requiredRole) {
      throw new Error(`Access denied: requires role '${this.requiredRole}'`);
    }
    return this.inner.generate(params);
  }
}

// Compose proxies
const generator: IReportGenerator = new AuthorizedReportProxy(
  new CachingReportProxy(
    new LazyReportProxy()
  ),
  'admin',
);
```

---

## 5. Common Mistakes & Pitfalls

| Pattern | Mistake | Fix |
|---------|---------|-----|
| Adapter | Adapting business logic, not just the interface | Adapter should only translate, not add logic |
| Decorator | Forgetting to delegate all interface methods | Implement every method; missing delegations cause silent bugs |
| Facade | Making the facade a god object with business logic | Facade only coordinates — business logic lives in subsystems |
| Proxy | Using Proxy where Decorator is more appropriate | Proxy controls access; Decorator adds behavior |
| Composite | Allowing `add()`/`remove()` in the leaf interface | Leaf nodes should not expose child management methods |
| Flyweight | Sharing mutable state | Only share immutable (intrinsic) state; extrinsic state stays outside |

> ⚠️ Decorator and Proxy look similar (both wrap an object implementing the same interface) but serve different purposes. Decorator adds behavior; Proxy controls access. In practice, the distinction can blur — focus on intent when naming them.

---

## 6. When to Use / Not Use

| Pattern | Use when | Avoid when |
|---------|----------|-----------|
| Adapter | Integrating legacy or third-party APIs | The interface can be changed directly |
| Bridge | Two dimensions of variation need to evolve independently | Only one dimension varies |
| Composite | Tree structures (menus, file systems, UI layouts) | Flat collections with uniform elements |
| Decorator | Adding optional, composable behaviors at runtime | A fixed set of behaviors — simpler as subclasses |
| Facade | Hiding a complex subsystem behind a single entry point | The subsystem is simple or already well-encapsulated |
| Flyweight | Thousands of similar objects with shared data | Object count is small; memory is not a concern |
| Proxy | Lazy initialization, caching, access control, logging | Direct access is simpler and the overhead is unnecessary |

---

## 7. Real-World Scenario

A service that fetches user profiles applies three structural patterns simultaneously:

```typescript
// Base: PostgresUserRepository
// Decorated: + caching
// Decorated: + logging
// Protected by: access control proxy

const repo = new LoggedUserRepository(
  new CachedUserRepository(
    new PostgresUserRepository(prisma)
  ),
  logger,
);

// Access control at the service level via Proxy
const protectedRepo = new AccessControlledUserRepository(repo, currentUser);
```

Each layer has a single concern. The PostgreSQL implementation knows nothing about caching. The cache knows nothing about logging. The proxy knows nothing about either. They are composed at the wiring point.

---

## 8. Interview Questions

**Q1: What is the difference between Adapter and Facade?**

A: Adapter makes one interface work with another — it is about compatibility between two existing interfaces. Facade creates a new, simplified interface over a complex subsystem — it is about hiding complexity. Adapter changes nothing; Facade simplifies everything. You can also use both: adapt external APIs first, then facade the result.

---

**Q2: How does Decorator differ from inheritance?**

A: Inheritance is static (decided at compile time) and applies to all instances of the subclass. Decorator is dynamic (composed at runtime) and applies only to the specific object you wrap. You can stack multiple decorators, and their order matters. You cannot achieve this flexibility with inheritance without creating a new subclass for every combination.

---

**Q3: When would you use Bridge instead of just using an interface?**

A: Both decouple abstraction from implementation, but Bridge makes both sides independently extensible. With a plain interface, you can add new implementations (new senders), but adding a new abstraction (new notification type) forces all existing implementations to potentially change. Bridge uses composition so both hierarchies can grow without affecting each other.

---

**Q4: What is the risk of deeply nested decorators?**

A: Deep decorator chains are hard to debug — a stack trace may show 5 levels of wrappers before reaching real logic. Also, if one decorator doesn't delegate all methods correctly, behavior silently breaks. Keep chains short (2-3 levels), name each decorator descriptively, and consider a structural review if you have more than 3 layers.

---

**Q5: How does Composite support the Open/Closed Principle?**

A: New leaf types (new file types, new UI components) can be added without changing the Composite or any client code that uses `FileSystemItem`. The tree-walking logic works with the interface, so it is closed for modification and open for extension.

---

**Q6: Proxy vs Decorator — how do you decide?**

A: Ask about intent. If you are adding new behavior (logging, caching) to enhance the object, use Decorator. If you are controlling access, deferring initialization, or adding cross-cutting infrastructure concerns (auth, rate limiting, circuit breaking) without the client's knowledge, use Proxy. The structural difference is minimal; the semantic difference is clear.

---

**Q7: In what scenario is Flyweight essential, not just nice to have?**

A: In rendering engines (text editors, game engines, map renderers) where tens of thousands of similar objects exist simultaneously. A text editor with a 100,000-character document needs 100,000 character objects — if each stores font, size, color, and style, that is massive memory waste. Flyweight shares the style object, reducing memory usage by orders of magnitude.

---

## 9. Exercises

**Exercise 1: Adapter — currency API**

A legacy `CurrencyConverter` class has a method `convertAmount(from: string, to: string, amt: number): number`. Your system expects `ICurrencyService` with `convert(amount: Money, targetCurrency: string): Promise<Money>`. Write the adapter.

---

**Exercise 2: Decorator — rate limiting**

Add a `RateLimitedUserRepository` decorator that allows at most 100 `findById` calls per minute. After the limit, throw `RateLimitExceededError`. The underlying repository is unaware of rate limiting.

---

**Exercise 3: Composite — UI menu system**

Model a navigation menu with `MenuItem` (leaf) and `MenuGroup` (composite). Both implement `IMenuComponent` with `render(): string` and `isVisible(): boolean`. A `MenuGroup` renders only its visible children.

---

**Exercise 4: Proxy — circuit breaker**

Implement a `CircuitBreakerProxy` wrapping `IPaymentProcessor`. After 3 consecutive failures, it enters "open" state and fails fast for 30 seconds before allowing a retry. Track state (`closed`, `open`, `half-open`).

---

## 10. Further Reading

- **Design Patterns: Elements of Reusable Object-Oriented Software** — Gamma, Helm, Johnson, Vlissides
- **[Refactoring Guru — Structural Patterns](https://refactoring.guru/design-patterns/structural-patterns)** — illustrated examples
- **[TypeScript Decorator Pattern](https://refactoring.guru/design-patterns/decorator/typescript/example)** — Refactoring Guru TypeScript examples
- **Patterns of Enterprise Application Architecture** — Martin Fowler (Proxy and Facade in enterprise context)
