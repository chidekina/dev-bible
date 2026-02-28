# SOLID Principles

## 1. What & Why

SOLID is an acronym for five design principles introduced by Robert C. Martin ("Uncle Bob") that guide object-oriented software design. These principles help developers build systems that are easy to maintain, extend, and test over time.

| Letter | Principle | One-liner |
|--------|-----------|-----------|
| **S** | Single Responsibility | A class should have one, and only one, reason to change |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subtypes must be substitutable for their base types |
| **I** | Interface Segregation | No client should be forced to depend on methods it does not use |
| **D** | Dependency Inversion | Depend on abstractions, not concretions |

Why does this matter in practice? Software that violates these principles tends to exhibit common failure modes: classes that break when you touch unrelated features, chains of `if/else` that grow every sprint, tests that require setting up half the application, and inheritance hierarchies that silently break contracts.

> 💡 SOLID principles are guidelines, not laws. Apply them where they reduce complexity — not blindly in every situation. A two-file script does not need SOLID; a payment processing module serving 50 engineers does.

---

## 2. Core Concepts

Each principle targets a specific axis of change:

- **SRP** targets *cohesion* — keeping related logic together and unrelated logic apart.
- **OCP** targets *extensibility* — adding behavior without breaking existing code.
- **LSP** targets *correctness* — ensuring polymorphism works as expected.
- **ISP** targets *coupling* — preventing consumers from depending on unused contracts.
- **DIP** targets *flexibility* — decoupling high-level policy from low-level implementation.

Together they form a feedback loop: SRP creates small classes, DIP wires them together, OCP lets you extend them safely, LSP ensures inheritance is correct, and ISP keeps contracts lean.

---

## 3. How It Works

The principles operate at the class/interface boundary level. They are evaluated by asking:

1. If I change requirement X, which classes must change?
2. Can I add behavior without touching existing code?
3. Can I substitute any subclass without changing behavior?
4. Does this interface contain methods that some clients never use?
5. Does this class instantiate its own dependencies?

Each "yes" to problems 1–5 indicates a violation.

---

## 4. Code Examples (TypeScript)

### S — Single Responsibility Principle

**Violation:** `UserService` handles business logic *and* sends email notifications. Two reasons to change: user logic changes, and email template changes.

```typescript
// VIOLATION — two responsibilities in one class
class UserService {
  constructor(private db: Database) {}

  async createUser(data: CreateUserDto): Promise<User> {
    const user = await this.db.users.create(data);

    // Email concern mixed with user creation concern
    const html = `<h1>Welcome, ${user.name}!</h1>`;
    await sendEmail({
      to: user.email,
      subject: 'Welcome!',
      html,
    });

    return user;
  }

  async updateUser(id: string, data: UpdateUserDto): Promise<User> {
    const user = await this.db.users.update(id, data);

    // Again mixing notification with domain logic
    if (data.email) {
      await sendEmail({
        to: user.email,
        subject: 'Email updated',
        html: `<p>Your email was changed.</p>`,
      });
    }

    return user;
  }
}
```

**Fix:** Extract email concern into a dedicated `WelcomeEmailService`. Each class now has exactly one reason to change.

```typescript
// ✅ Single Responsibility — two separate classes, each with one job
interface EmailPayload {
  to: string;
  subject: string;
  html: string;
}

class WelcomeEmailService {
  async sendWelcome(user: User): Promise<void> {
    await sendEmail({
      to: user.email,
      subject: 'Welcome!',
      html: `<h1>Welcome, ${user.name}!</h1>`,
    });
  }

  async sendEmailChanged(user: User): Promise<void> {
    await sendEmail({
      to: user.email,
      subject: 'Email updated',
      html: `<p>Your email was changed.</p>`,
    });
  }
}

class UserService {
  constructor(
    private db: Database,
    private welcomeEmail: WelcomeEmailService,
  ) {}

  async createUser(data: CreateUserDto): Promise<User> {
    const user = await this.db.users.create(data);
    await this.welcomeEmail.sendWelcome(user);
    return user;
  }

  async updateUser(id: string, data: UpdateUserDto): Promise<User> {
    const user = await this.db.users.update(id, data);
    if (data.email) {
      await this.welcomeEmail.sendEmailChanged(user);
    }
    return user;
  }
}
```

> 💡 A practical heuristic: if you need to use "and" when describing what a class does ("it creates users **and** sends emails"), it likely violates SRP.

---

### O — Open/Closed Principle

**Violation:** A discount calculator uses a growing `if/else` chain. Every new discount type requires modifying this class — risking breakage of existing logic.

```typescript
// VIOLATION — must modify this function to add new discount types
function calculateDiscount(order: Order, discountType: string): number {
  if (discountType === 'percentage') {
    return order.total * 0.1;
  } else if (discountType === 'fixed') {
    return 10;
  } else if (discountType === 'bogo') {
    return order.total / 2;
  } else if (discountType === 'loyalty') {
    // Added later — touched existing function
    return order.total * 0.15 * order.loyaltyYears;
  }
  return 0;
}
```

**Fix:** Define a `DiscountStrategy` interface. Adding a new discount type means adding a new class — zero changes to existing code.

```typescript
// ✅ Open/Closed — extend via new classes, never modify existing ones
interface DiscountStrategy {
  calculate(order: Order): number;
}

class PercentageDiscount implements DiscountStrategy {
  constructor(private rate: number) {}

  calculate(order: Order): number {
    return order.total * this.rate;
  }
}

class FixedDiscount implements DiscountStrategy {
  constructor(private amount: number) {}

  calculate(order: Order): number {
    return Math.min(this.amount, order.total);
  }
}

class BuyOneGetOneDiscount implements DiscountStrategy {
  calculate(order: Order): number {
    return order.total / 2;
  }
}

class LoyaltyDiscount implements DiscountStrategy {
  calculate(order: Order): number {
    return order.total * 0.15 * (order.loyaltyYears ?? 0);
  }
}

// The calculator never changes — it's closed for modification
class OrderPricer {
  constructor(private strategy: DiscountStrategy) {}

  finalPrice(order: Order): number {
    const discount = this.strategy.calculate(order);
    return Math.max(0, order.total - discount);
  }
}

// Usage
const pricer = new OrderPricer(new LoyaltyDiscount());
const price = pricer.finalPrice(order);
```

> ⚠️ OCP does not mean "never change a file." It means new behavior should be expressed as new code, not modifications to stable, tested logic.

---

### L — Liskov Substitution Principle

**Violation:** A `Square` extends `Rectangle` and overrides `setWidth`/`setHeight` to enforce equal sides. Code that works with `Rectangle` silently breaks when handed a `Square`.

```typescript
// VIOLATION — Square breaks Rectangle's contract
class Rectangle {
  constructor(protected width: number, protected height: number) {}

  setWidth(w: number): void { this.width = w; }
  setHeight(h: number): void { this.height = h; }
  area(): number { return this.width * this.height; }
}

class Square extends Rectangle {
  // Enforces equal sides — violates Rectangle's contract
  setWidth(w: number): void {
    this.width = w;
    this.height = w; // silent mutation
  }

  setHeight(h: number): void {
    this.width = h;
    this.height = h; // silent mutation
  }
}

// This function expects Rectangle behavior
function testRectangle(rect: Rectangle): void {
  rect.setWidth(5);
  rect.setHeight(4);
  // Expects 20, gets 16 with Square — LSP violation!
  console.assert(rect.area() === 20, 'Area should be 20');
}

testRectangle(new Square(10)); // assertion fails silently
```

**Fix:** Do not model the Square-is-a-Rectangle geometric relationship in code. Use separate, independent classes with a common `Shape` interface if needed.

```typescript
// ✅ Liskov Substitution — separate classes, shared interface
interface Shape {
  area(): number;
  perimeter(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}

  area(): number { return this.width * this.height; }
  perimeter(): number { return 2 * (this.width + this.height); }
}

class Square implements Shape {
  constructor(private side: number) {}

  area(): number { return this.side * this.side; }
  perimeter(): number { return 4 * this.side; }
}

// Any Shape works here — substitution is safe
function printShapeInfo(shape: Shape): void {
  console.log(`Area: ${shape.area()}, Perimeter: ${shape.perimeter()}`);
}

printShapeInfo(new Rectangle(5, 4)); // Area: 20, Perimeter: 18
printShapeInfo(new Square(5));       // Area: 25, Perimeter: 20
```

> 💡 LSP is about behavioral compatibility, not type compatibility. Two types can share an interface and still violate LSP if one silently breaks assumptions the other upholds.

---

### I — Interface Segregation Principle

**Violation:** A fat `IAnimal` interface forces every animal to implement methods that don't apply (e.g., `fly()` on a `Dog`).

```typescript
// VIOLATION — every Animal must implement all methods, even irrelevant ones
interface IAnimal {
  eat(): void;
  sleep(): void;
  fly(): void;    // Not all animals fly
  swim(): void;   // Not all animals swim
  run(): void;    // Not all animals run
}

class Dog implements IAnimal {
  eat(): void { console.log('eating'); }
  sleep(): void { console.log('sleeping'); }
  run(): void { console.log('running'); }
  fly(): void { throw new Error('Dogs cannot fly!'); }  // violation!
  swim(): void { console.log('paddling'); }
}

class Eagle implements IAnimal {
  eat(): void { console.log('eating'); }
  sleep(): void { console.log('sleeping'); }
  fly(): void { console.log('soaring'); }
  swim(): void { throw new Error('Eagles cannot swim!'); } // violation!
  run(): void { console.log('running'); }
}
```

**Fix:** Split into focused interfaces. Classes implement only the capabilities they actually have.

```typescript
// ✅ Interface Segregation — fine-grained, composable interfaces
interface ILivingThing {
  eat(): void;
  sleep(): void;
}

interface IRunnable {
  run(): void;
}

interface IFlyable {
  fly(): void;
  landingSpeed(): number;
}

interface ISwimmable {
  swim(): void;
  divingDepth(): number;
}

// Dog: can eat, sleep, run, swim — but not fly
class Dog implements ILivingThing, IRunnable, ISwimmable {
  eat(): void { console.log('eating'); }
  sleep(): void { console.log('sleeping'); }
  run(): void { console.log('running at 48 km/h'); }
  swim(): void { console.log('paddling'); }
  divingDepth(): number { return 0.5; }
}

// Eagle: can eat, sleep, run, fly — but not swim
class Eagle implements ILivingThing, IRunnable, IFlyable {
  eat(): void { console.log('eating'); }
  sleep(): void { console.log('sleeping'); }
  run(): void { console.log('running'); }
  fly(): void { console.log('soaring at altitude'); }
  landingSpeed(): number { return 35; }
}

// Duck: all capabilities
class Duck implements ILivingThing, IRunnable, IFlyable, ISwimmable {
  eat(): void { console.log('eating'); }
  sleep(): void { console.log('sleeping'); }
  run(): void { console.log('waddling'); }
  fly(): void { console.log('flying south'); }
  swim(): void { console.log('floating'); }
  landingSpeed(): number { return 15; }
  divingDepth(): number { return 1.2; }
}

// Functions depend only on the capability they need
function makeItFly(flyer: IFlyable): void {
  flyer.fly();
  console.log(`Landing speed: ${flyer.landingSpeed()} km/h`);
}
```

> 💡 ISP is especially important in TypeScript because interfaces are structural. Lean interfaces also make mocking in tests trivial — you only need to implement what the test exercises.

---

### D — Dependency Inversion Principle

**Violation:** `UserService` directly instantiates `MySQLUserRepository`. It is tightly coupled to a concrete database implementation — you cannot swap it out, and testing requires a real MySQL database.

```typescript
// VIOLATION — high-level module instantiates low-level module
class MySQLUserRepository {
  async findById(id: string): Promise<User | null> {
    // Direct MySQL query
    const row = await mysql.query('SELECT * FROM users WHERE id = ?', [id]);
    return row ? mapToUser(row) : null;
  }

  async save(user: User): Promise<void> {
    await mysql.query('INSERT INTO users ...', [...]);
  }
}

class UserService {
  // Creates its own dependency — cannot be tested without MySQL
  private repo = new MySQLUserRepository();

  async getUser(id: string): Promise<User | null> {
    return this.repo.findById(id);
  }
}
```

**Fix:** Define an `IUserRepository` interface. `UserService` depends on the abstraction. The concrete `MySQLUserRepository` or `InMemoryUserRepository` is injected from outside.

```typescript
// ✅ Dependency Inversion — depend on abstractions
interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}

// Low-level module implements the abstraction
class MySQLUserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    const row = await mysql.query('SELECT * FROM users WHERE id = ?', [id]);
    return row ? mapToUser(row) : null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const row = await mysql.query('SELECT * FROM users WHERE email = ?', [email]);
    return row ? mapToUser(row) : null;
  }

  async save(user: User): Promise<void> {
    await mysql.query('INSERT INTO users ...', [...]);
  }

  async delete(id: string): Promise<void> {
    await mysql.query('DELETE FROM users WHERE id = ?', [id]);
  }
}

// High-level module depends only on the interface
class UserService {
  constructor(private repo: IUserRepository) {}

  async getUser(id: string): Promise<User | null> {
    return this.repo.findById(id);
  }

  async deactivateUser(id: string): Promise<void> {
    const user = await this.repo.findById(id);
    if (!user) throw new Error('User not found');
    user.deactivate();
    await this.repo.save(user);
  }
}

// In production — wire real implementation
const service = new UserService(new MySQLUserRepository());

// In tests — wire in-memory fake, no MySQL needed
class InMemoryUserRepository implements IUserRepository {
  private store = new Map<string, User>();

  async findById(id: string): Promise<User | null> {
    return this.store.get(id) ?? null;
  }

  async findByEmail(email: string): Promise<User | null> {
    return [...this.store.values()].find(u => u.email === email) ?? null;
  }

  async save(user: User): Promise<void> {
    this.store.set(user.id, user);
  }

  async delete(id: string): Promise<void> {
    this.store.delete(id);
  }
}

// Test — zero infrastructure
const repo = new InMemoryUserRepository();
const testService = new UserService(repo);
```

---

## 5. Common Mistakes & Pitfalls

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Creating a `BaseService` that all services extend | Forces coupling to a shared parent, violates SRP and LSP | Use composition and interfaces |
| One interface per class with identical shape | Pointless abstraction — not DIP | Only extract interfaces when you have multiple implementations or need testability |
| Applying OCP to configuration values | Config changes are not new behavior — they are just data | Use config files, not strategy classes |
| Giant `IRepository` with 30 methods | ISP violation — most consumers use 3-4 methods | Split or use partial types in consumers |
| LSP: throwing `NotImplemented` in overrides | Breaks behavioral contract | Don't inherit if you can't fulfill the contract |
| Injecting the DI container itself | The container becomes a service locator, hiding dependencies | Inject actual dependencies, not the container |

> ⚠️ Over-engineering in the name of SOLID is a real risk. Applying all five principles to a `HealthCheckController` with one method is wasteful. Use them where they provide measurable value.

---

## 6. When to Use / Not Use

**Apply SOLID aggressively when:**
- Building modules consumed by multiple teams
- The domain is stable but implementations change (databases, third-party APIs, notification channels)
- You need fast, isolated unit tests
- The team size exceeds ~5 engineers on the same codebase

**Relax SOLID when:**
- Prototyping or building an MVP — speed matters more than extensibility
- The class is a thin, trivial data mapper with no logic
- The abstraction cost (new files, new interfaces) clearly outweighs the benefit
- You are writing glue code (CLI scripts, one-off migrations)

---

## 7. Real-World Scenario: Adding a New Payment Provider

An e-commerce platform starts with Stripe. After growth, they need to support PayPal and Pix (Brazilian instant payment). A SOLID design makes this a 15-minute task.

```typescript
// Abstraction defined once
interface IPaymentGateway {
  charge(amount: number, currency: string, customerId: string): Promise<PaymentResult>;
  refund(transactionId: string, amount: number): Promise<RefundResult>;
  getStatus(transactionId: string): Promise<PaymentStatus>;
}

// Each provider is isolated — no shared state, no cross-contamination
class StripeGateway implements IPaymentGateway {
  constructor(private stripe: Stripe) {}

  async charge(amount: number, currency: string, customerId: string): Promise<PaymentResult> {
    const intent = await this.stripe.paymentIntents.create({ amount, currency, customer: customerId });
    return { transactionId: intent.id, status: 'pending' };
  }

  async refund(transactionId: string, amount: number): Promise<RefundResult> {
    const refund = await this.stripe.refunds.create({ payment_intent: transactionId, amount });
    return { refundId: refund.id, status: 'processed' };
  }

  async getStatus(transactionId: string): Promise<PaymentStatus> {
    const intent = await this.stripe.paymentIntents.retrieve(transactionId);
    return intent.status as PaymentStatus;
  }
}

class PayPalGateway implements IPaymentGateway {
  // New provider — zero changes to existing code
  async charge(amount: number, currency: string, customerId: string): Promise<PaymentResult> {
    // PayPal-specific logic
    return { transactionId: 'pp_xyz', status: 'pending' };
  }

  async refund(transactionId: string, amount: number): Promise<RefundResult> {
    return { refundId: 'ref_pp_xyz', status: 'processed' };
  }

  async getStatus(transactionId: string): Promise<PaymentStatus> {
    return 'completed';
  }
}

class PixGateway implements IPaymentGateway {
  // Brazilian instant payments — added without touching Stripe or PayPal
  async charge(amount: number, currency: string, customerId: string): Promise<PaymentResult> {
    return { transactionId: 'pix_xyz', status: 'pending' };
  }

  async refund(transactionId: string, amount: number): Promise<RefundResult> {
    return { refundId: 'ref_pix_xyz', status: 'processed' };
  }

  async getStatus(transactionId: string): Promise<PaymentStatus> {
    return 'completed';
  }
}

// High-level service — never changes regardless of how many gateways are added
class CheckoutService {
  constructor(private gateway: IPaymentGateway) {}

  async processOrder(order: Order): Promise<PaymentResult> {
    return this.gateway.charge(order.total, order.currency, order.customerId);
  }
}

// Wired at the composition root (e.g., DI container or main.ts)
const gateway = resolveGateway(userPreference); // 'stripe' | 'paypal' | 'pix'
const checkout = new CheckoutService(gateway);
```

Adding Pix required: 1 new file (`PixGateway`), 1 line in the factory. Zero changes to `CheckoutService`, `StripeGateway`, or `PayPalGateway`.

---

## 8. Interview Questions

**Q1: What does "a class should have one reason to change" mean in practice?**

A: It means a class should be responsible for one actor — one stakeholder or one part of the system that drives change. If both the marketing team (email templates) and the engineering team (user persistence logic) can cause changes to the same class, it has two responsibilities. Split them.

---

**Q2: How do SRP and OCP relate to each other?**

A: They reinforce each other. SRP ensures classes are small and focused. OCP ensures you add behavior by adding new classes rather than modifying existing ones. A class that violates SRP is much harder to keep closed for modification because its multiple responsibilities become tangled.

---

**Q3: Give an example of an LSP violation that is not obvious.**

A: A `ReadOnlyList` that extends `List` and throws `UnsupportedOperationException` from `add()`. Code that accepts a `List` and calls `add()` will silently fail at runtime. The subtype doesn't honor the behavioral contract of the parent. The fix is to not inherit from `List` — instead implement a separate `IReadableCollection` interface.

---

**Q4: Is it always wrong to have an interface with many methods?**

A: Not always. If every consumer of the interface uses all methods, there is no segregation problem. ISP is violated when clients are forced to depend on methods they don't use. The size of the interface is less important than whether all its consumers actually need all its members.

---

**Q5: What is the difference between Dependency Injection and Dependency Inversion?**

A: They are related but distinct. Dependency Inversion is a principle: high-level modules should depend on abstractions. Dependency Injection is a technique: the dependencies (which implement those abstractions) are provided from outside rather than instantiated within the class. DI is one way to achieve DIP, but DIP doesn't mandate DI specifically.

---

**Q6: Can you violate LSP in TypeScript even when the types compile correctly?**

A: Yes. TypeScript checks structural type compatibility but not behavioral contracts. A subclass can satisfy TypeScript's type checker while violating LSP by: throwing exceptions the base class never throws, returning empty collections instead of throwing (changing the null-object contract), or silently ignoring method calls. LSP is a semantic guarantee, not a syntactic one.

---

**Q7: When is it acceptable to skip Dependency Inversion?**

A: When the concrete implementation will never change and testing it in isolation provides no value — for example, a `Logger` class that wraps `console.log` in a non-critical script. Also acceptable: value objects and pure functions, which have no side effects and no need for substitution.

---

**Q8: How would you convince a skeptical team member that SOLID is worth the extra files and interfaces?**

A: Point to a specific, recent pain point: "Remember when adding the SMS notification required touching 4 classes and broke the email tests? With ISP and DIP, each notification type would be its own class implementing a single `INotifier` interface. The next addition would be one new file." Frame it in terms of change cost, not abstract principles.

---

## 9. Exercises

**Exercise 1: SRP — Split the monolith class**

Take this class and split it into properly responsible classes:

```typescript
class ReportGenerator {
  async generate(userId: string): Promise<void> {
    const data = await fetch(`/api/users/${userId}/orders`).then(r => r.json());
    const csv = data.map((o: Order) => `${o.id},${o.total},${o.date}`).join('\n');
    fs.writeFileSync(`report_${userId}.csv`, csv);
    await sendEmail({ to: 'admin@company.com', subject: 'Report', attachment: csv });
  }
}
```

*Hint: Identify each actor — who triggers a change in the fetch URL? The CSV format? The file path? The email recipient?*

---

**Exercise 2: OCP — Shipping cost calculator**

Refactor this to be open for extension:

```typescript
function getShippingCost(method: string, weight: number): number {
  if (method === 'standard') return weight * 1.5;
  if (method === 'express') return weight * 3.0 + 5;
  if (method === 'overnight') return weight * 5.0 + 20;
  return 0;
}
```

*Hint: Define a `ShippingStrategy` interface with a `calculate(weight: number): number` method.*

---

**Exercise 3: LSP — Find the violation**

Analyze this code and explain the LSP violation:

```typescript
class Bird {
  fly(): void { console.log('flying'); }
}
class Penguin extends Bird {
  fly(): void { throw new Error('Penguins cannot fly'); }
}
function makeBirdFly(bird: Bird): void {
  bird.fly();
}
```

*Hint: What assumption does `makeBirdFly` make? Can that assumption always hold?*

---

**Exercise 4: DIP — Decouple the service**

Introduce an interface and make `OrderService` testable without a real database:

```typescript
class OrderService {
  private db = new PostgresDatabase();
  async getOrder(id: string): Promise<Order | null> {
    return this.db.query('SELECT * FROM orders WHERE id = $1', [id]);
  }
}
```

*Hint: Define `IOrderRepository` with `findById`. Write an `InMemoryOrderRepository` for tests.*

---

**Exercise 5: ISP — Audit the interface**

Split this interface based on real consumer needs:

```typescript
interface IVehicle {
  startEngine(): void;
  stopEngine(): void;
  accelerate(speed: number): void;
  brake(): void;
  openSunroof(): void;
  deployAirbags(): void;
  playMusic(track: string): void;
}
```

*Hint: Which methods belong to driving? Safety? Comfort? Create one interface per concern.*

---

## 10. Further Reading

- **Clean Code** — Robert C. Martin (Chapter 10: Classes)
- **Agile Software Development, Principles, Patterns, and Practices** — Robert C. Martin (full SOLID treatment)
- **[SOLID Principles Every Developer Should Know](https://blog.bitsrc.io/solid-principles-every-developer-should-know-b3bfa96bb688)** — Bits & Pieces
- **[The Principles of OOD](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)** — Uncle Bob's original articles
- **Dependency Injection Principles, Practices, and Patterns** — Mark Seemann & Steven van Deursen
- TypeScript Handbook — [Interfaces](https://www.typescriptlang.org/docs/handbook/2/objects.html)
