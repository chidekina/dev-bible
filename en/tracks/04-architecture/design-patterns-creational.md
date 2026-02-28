# Design Patterns: Creational

## 1. What & Why

Creational design patterns deal with object creation. They abstract the instantiation process, making a system independent of how its objects are created, composed, and represented.

Without creational patterns, object creation logic often leaks into client code, coupling it to concrete classes and making testing and extension difficult.

The five classic creational patterns from the Gang of Four book (Gamma, Helm, Johnson, Vlissides — 1994):

| Pattern | Problem it solves |
|---------|------------------|
| **Singleton** | Ensure only one instance of a class exists |
| **Factory Method** | Let subclasses decide which class to instantiate |
| **Abstract Factory** | Create families of related objects without specifying concrete classes |
| **Builder** | Construct complex objects step by step |
| **Prototype** | Clone existing objects without depending on their concrete classes |

> 💡 Creational patterns are most valuable when object creation involves complex logic, significant resource cost (DB connections, network), or when the exact type of object needs to vary at runtime.

---

## 2. Core Concepts

All creational patterns share a common theme: **encapsulate knowledge about which classes the system uses**. They shift the responsibility of deciding what to create, and how, away from client code.

The key trade-off: flexibility vs. complexity. Adding indirection (a factory, a builder) makes code more flexible but also adds more moving parts.

---

## 3. How It Works

Each pattern addresses a specific variation axis:

- **Singleton:** What if only one instance should exist?
- **Factory Method:** What if the exact class depends on runtime input?
- **Abstract Factory:** What if multiple related classes must match each other (e.g., a UI theme)?
- **Builder:** What if the object requires many optional parameters or a specific construction order?
- **Prototype:** What if the new object should start as a copy of an existing one?

---

## 4. Code Examples (TypeScript)

### Singleton — Database Connection Pool

A connection pool should be created once and reused. Creating multiple pools wastes resources and can exceed connection limits.

```typescript
// src/infrastructure/database/DatabasePool.ts
import { Pool } from 'pg';

export class DatabasePool {
  private static instance: Pool | null = null;

  // Private constructor prevents external instantiation
  private constructor() {}

  static getInstance(): Pool {
    if (!DatabasePool.instance) {
      DatabasePool.instance = new Pool({
        host: process.env.DB_HOST ?? 'localhost',
        port: Number(process.env.DB_PORT ?? 5432),
        database: process.env.DB_NAME,
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        max: 20,                // maximum pool size
        idleTimeoutMillis: 30_000,
        connectionTimeoutMillis: 2_000,
      });
    }
    return DatabasePool.instance;
  }
}

// Usage — always the same pool instance
const pool = DatabasePool.getInstance();
const result = await pool.query('SELECT * FROM users WHERE id = $1', [userId]);
```

> ⚠️ Singleton is the most misused pattern. It introduces global mutable state, makes testing difficult (tests share state), and creates hidden coupling. Use it only for truly shared resources like connection pools, caches, or configuration. Never use Singleton for business logic classes.

A safer alternative for most cases is **dependency injection** — pass the single instance from the composition root without the class enforcing its own uniqueness:

```typescript
// Prefer this in DI-based systems:
const pool = new Pool({ ... });
// Inject 'pool' wherever needed — it's still one instance, but testable
```

---

### Factory Method — Notification Factory

The application sends notifications via Email, SMS, or Push. The exact type depends on user preferences and configuration.

```typescript
// src/domain/notifications/INotifier.ts
export interface INotifier {
  send(to: string, message: string): Promise<void>;
  readonly channel: string;
}
```

```typescript
// Concrete implementations
class EmailNotifier implements INotifier {
  readonly channel = 'email';

  async send(to: string, message: string): Promise<void> {
    console.log(`[EMAIL] to=${to}: ${message}`);
    // Real: call SMTP or SES
  }
}

class SmsNotifier implements INotifier {
  readonly channel = 'sms';

  async send(to: string, message: string): Promise<void> {
    console.log(`[SMS] to=${to}: ${message}`);
    // Real: call Twilio
  }
}

class PushNotifier implements INotifier {
  readonly channel = 'push';

  async send(to: string, message: string): Promise<void> {
    console.log(`[PUSH] to=${to}: ${message}`);
    // Real: call FCM
  }
}
```

```typescript
// src/domain/notifications/NotifierFactory.ts
export type NotificationChannel = 'email' | 'sms' | 'push';

export class NotifierFactory {
  // Factory Method — centralizes the creation decision
  static create(channel: NotificationChannel): INotifier {
    switch (channel) {
      case 'email': return new EmailNotifier();
      case 'sms':   return new SmsNotifier();
      case 'push':  return new PushNotifier();
      default:
        // TypeScript exhaustiveness check
        const _exhaustive: never = channel;
        throw new Error(`Unknown notification channel: ${_exhaustive}`);
    }
  }
}

// Usage — client code doesn't import concrete classes
const notifier = NotifierFactory.create(user.preferredChannel);
await notifier.send(user.contact, 'Your order has been shipped!');
```

> 💡 The exhaustiveness check (`const _exhaustive: never`) ensures TypeScript will produce a compile error if a new channel is added to the union type but not handled in the switch.

---

### Abstract Factory — UI Theme System

A UI library supports DarkTheme and LightTheme. Each theme must produce a matching set of components (Button, Input, Modal). Abstract Factory ensures consistency — you cannot accidentally mix a dark Button with a light Modal.

```typescript
// src/ui/components.ts — Abstract Products
export interface Button {
  render(): string;
  onClick(handler: () => void): void;
}

export interface Input {
  render(): string;
  getValue(): string;
}

export interface Modal {
  render(): string;
  open(): void;
  close(): void;
}

// src/ui/IThemeFactory.ts — Abstract Factory
export interface IThemeFactory {
  createButton(label: string): Button;
  createInput(placeholder: string): Input;
  createModal(title: string, content: string): Modal;
}
```

```typescript
// src/ui/themes/DarkTheme.ts — Concrete Factory
export class DarkThemeFactory implements IThemeFactory {
  createButton(label: string): Button {
    return {
      render: () => `<button class="btn-dark">${label}</button>`,
      onClick: (handler) => { /* register handler */ },
    };
  }

  createInput(placeholder: string): Input {
    return {
      render: () => `<input class="input-dark" placeholder="${placeholder}"/>`,
      getValue: () => '',
    };
  }

  createModal(title: string, content: string): Modal {
    let isOpen = false;
    return {
      render: () => `<div class="modal-dark"><h2>${title}</h2><p>${content}</p></div>`,
      open: () => { isOpen = true; console.log('Dark modal opened'); },
      close: () => { isOpen = false; },
    };
  }
}

export class LightThemeFactory implements IThemeFactory {
  createButton(label: string): Button {
    return {
      render: () => `<button class="btn-light">${label}</button>`,
      onClick: (handler) => { /* register handler */ },
    };
  }

  createInput(placeholder: string): Input {
    return {
      render: () => `<input class="input-light" placeholder="${placeholder}"/>`,
      getValue: () => '',
    };
  }

  createModal(title: string, content: string): Modal {
    return {
      render: () => `<div class="modal-light"><h2>${title}</h2><p>${content}</p></div>`,
      open: () => console.log('Light modal opened'),
      close: () => {},
    };
  }
}
```

```typescript
// Application — depends on IThemeFactory, not concrete classes
function buildCheckoutForm(theme: IThemeFactory): string {
  const emailInput = theme.createInput('Enter your email');
  const submitBtn = theme.createButton('Complete Purchase');
  const confirmModal = theme.createModal('Order Confirmed', 'Thank you!');

  return `
    ${emailInput.render()}
    ${submitBtn.render()}
    ${confirmModal.render()}
  `;
}

// Swap themes at runtime — zero changes to buildCheckoutForm
const userTheme: IThemeFactory = userPrefersDark
  ? new DarkThemeFactory()
  : new LightThemeFactory();

buildCheckoutForm(userTheme);
```

---

### Builder — HTTP Request Builder

The Builder pattern shines when constructing complex objects with many optional parameters. It avoids telescoping constructors and makes the code read like a sentence.

```typescript
// src/lib/http/HttpRequestBuilder.ts
interface HttpRequest {
  url: string;
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  headers: Record<string, string>;
  body?: unknown;
  timeout: number;
  retries: number;
}

export class HttpRequestBuilder {
  private request: Partial<HttpRequest> = {
    method: 'GET',
    headers: {},
    timeout: 5000,
    retries: 0,
  };

  url(url: string): this {
    this.request.url = url;
    return this;
  }

  method(method: HttpRequest['method']): this {
    this.request.method = method;
    return this;
  }

  header(key: string, value: string): this {
    this.request.headers = { ...this.request.headers, [key]: value };
    return this;
  }

  bearerToken(token: string): this {
    return this.header('Authorization', `Bearer ${token}`);
  }

  contentType(type: string): this {
    return this.header('Content-Type', type);
  }

  body(data: unknown): this {
    this.request.body = data;
    return this;
  }

  timeout(ms: number): this {
    this.request.timeout = ms;
    return this;
  }

  retries(count: number): this {
    this.request.retries = count;
    return this;
  }

  build(): HttpRequest {
    if (!this.request.url) throw new Error('URL is required');
    if (!this.request.method) throw new Error('Method is required');
    return this.request as HttpRequest;
  }
}

// Usage — reads like natural language
const request = new HttpRequestBuilder()
  .url('https://api.example.com/orders')
  .method('POST')
  .bearerToken(authToken)
  .contentType('application/json')
  .body({ customerId: 'cust_1', items: [] })
  .timeout(10_000)
  .retries(3)
  .build();
```

Another common use: SQL Query Builder.

```typescript
// src/lib/db/QueryBuilder.ts
export class SelectQueryBuilder {
  private table: string = '';
  private conditions: string[] = [];
  private columns: string[] = ['*'];
  private limitValue?: number;
  private orderByClause?: string;

  from(table: string): this {
    this.table = table;
    return this;
  }

  select(...columns: string[]): this {
    this.columns = columns;
    return this;
  }

  where(condition: string): this {
    this.conditions.push(condition);
    return this;
  }

  orderBy(column: string, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.orderByClause = `${column} ${direction}`;
    return this;
  }

  limit(n: number): this {
    this.limitValue = n;
    return this;
  }

  build(): string {
    if (!this.table) throw new Error('Table is required');

    let sql = `SELECT ${this.columns.join(', ')} FROM ${this.table}`;

    if (this.conditions.length > 0) {
      sql += ` WHERE ${this.conditions.join(' AND ')}`;
    }
    if (this.orderByClause) {
      sql += ` ORDER BY ${this.orderByClause}`;
    }
    if (this.limitValue !== undefined) {
      sql += ` LIMIT ${this.limitValue}`;
    }

    return sql;
  }
}

// Usage
const query = new SelectQueryBuilder()
  .from('orders')
  .select('id', 'customer_id', 'total', 'status')
  .where("status = 'placed'")
  .where('total > 100')
  .orderBy('created_at', 'DESC')
  .limit(20)
  .build();
// → SELECT id, customer_id, total, status FROM orders
//   WHERE status = 'placed' AND total > 100
//   ORDER BY created_at DESC LIMIT 20
```

---

### Prototype — Deep Clone of Configuration Objects

The Prototype pattern clones existing objects. In TypeScript, it is used when the initialization cost is high (parsing, fetching remote data) and you want copies that can be modified independently.

```typescript
// src/config/AppConfig.ts
export interface DatabaseConfig {
  host: string;
  port: number;
  name: string;
  pool: { min: number; max: number };
}

export interface AppConfig {
  env: string;
  database: DatabaseConfig;
  featureFlags: Record<string, boolean>;
  rateLimit: { windowMs: number; max: number };
}

export class ConfigPrototype {
  private config: AppConfig;

  constructor(config: AppConfig) {
    this.config = config;
  }

  // Deep clone — returns a new independent instance
  clone(): ConfigPrototype {
    return new ConfigPrototype(JSON.parse(JSON.stringify(this.config)));
  }

  withDatabase(overrides: Partial<DatabaseConfig>): ConfigPrototype {
    const clone = this.clone();
    clone.config.database = { ...clone.config.database, ...overrides };
    return clone;
  }

  withFeatureFlag(flag: string, value: boolean): ConfigPrototype {
    const clone = this.clone();
    clone.config.featureFlags = { ...clone.config.featureFlags, [flag]: value };
    return clone;
  }

  get(): AppConfig {
    return this.clone().config; // never expose mutable reference
  }
}

// Base configuration
const baseConfig = new ConfigPrototype({
  env: 'production',
  database: { host: 'db.prod.internal', port: 5432, name: 'app', pool: { min: 2, max: 20 } },
  featureFlags: { newCheckout: false, betaDashboard: false },
  rateLimit: { windowMs: 60_000, max: 100 },
});

// Test configuration — clone and override, zero mutation of base
const testConfig = baseConfig
  .withDatabase({ host: 'localhost', name: 'app_test', pool: { min: 1, max: 5 } })
  .withFeatureFlag('newCheckout', true)
  .get();
```

---

## 5. Common Mistakes & Pitfalls

| Pattern | Mistake | Fix |
|---------|---------|-----|
| Singleton | Using it for business logic or services that have state | Use DI; inject a single instance from the composition root |
| Singleton | Not resetting state between tests | Use dependency injection so tests can pass fresh instances |
| Factory Method | Putting complex initialization inside the factory | Factories should construct, not initialize asynchronously |
| Abstract Factory | Creating a factory per class instead of per family | One factory should create one cohesive family of objects |
| Builder | Returning a mutable builder from `build()` | Freeze or deep-clone the result; builder is for construction only |
| Prototype | Shallow copy when deep copy is needed | Use `structuredClone()` or `JSON.parse(JSON.stringify(...))` for plain objects; write explicit clone methods for complex classes |

> ⚠️ The Prototype pattern using `JSON.parse(JSON.stringify(...))` does not handle: Dates (become strings), functions (dropped), circular references (throws), `undefined` values (dropped). Use `structuredClone()` in modern Node.js 17+ for most cases, or write explicit clone methods.

---

## 6. When to Use / Not Use

| Pattern | Use when | Avoid when |
|---------|----------|-----------|
| Singleton | One shared resource (pool, cache, config) | Business logic; testability is a concern |
| Factory Method | The exact class varies at runtime | Only one concrete class exists |
| Abstract Factory | Families of related objects must match | Only one or two unrelated objects |
| Builder | Many optional parameters; complex construction order | Simple objects with 2-3 fields |
| Prototype | Copying expensive-to-initialize objects | Objects are cheap to create from scratch |

---

## 7. Real-World Scenario

A notification service must deliver via Email, SMS, or Push depending on user preferences and message priority. An Abstract Factory creates matching notifier + logger + retry-handler combinations per channel.

```typescript
interface NotificationComponents {
  notifier: INotifier;
  rateLimiter: IRateLimiter;
  retryPolicy: IRetryPolicy;
}

interface INotificationComponentFactory {
  createComponents(): NotificationComponents;
}

class EmailComponentFactory implements INotificationComponentFactory {
  createComponents(): NotificationComponents {
    return {
      notifier: new EmailNotifier(),
      rateLimiter: new PerMinuteRateLimiter(10),   // 10/min for email
      retryPolicy: new ExponentialBackoffPolicy(3),
    };
  }
}

class SmsComponentFactory implements INotificationComponentFactory {
  createComponents(): NotificationComponents {
    return {
      notifier: new SmsNotifier(),
      rateLimiter: new PerMinuteRateLimiter(3),    // 3/min for SMS (cost)
      retryPolicy: new LinearBackoffPolicy(2),
    };
  }
}
```

---

## 8. Interview Questions

**Q1: What is the difference between Factory Method and Abstract Factory?**

A: Factory Method is a single method (or class) that creates one type of object, letting the decision of which subclass to instantiate be made at runtime. Abstract Factory creates *families* of related objects — all products from one factory are designed to work together. Use Factory Method when you need one flexible creation point; use Abstract Factory when you need a consistent set of related objects.

---

**Q2: Why is Singleton considered an anti-pattern by many developers?**

A: Because it introduces global mutable state, makes it impossible to inject test doubles, creates hidden coupling (any class can call `getInstance()` without declaring its dependency), and makes concurrent testing unreliable. The single-instance behavior is better achieved by creating one instance in the composition root and injecting it.

---

**Q3: When would you choose Builder over a constructor with many parameters?**

A: When a class has more than 3-4 parameters (especially optional ones), a telescoping constructor becomes unreadable (`new User(name, email, null, null, true, false, 'admin')`). The Builder makes optional parameters explicit and reads like natural language. Builder is also appropriate when the construction must follow a specific order or validate partial state before completing.

---

**Q4: What is the difference between Prototype and a copy constructor?**

A: A copy constructor is a constructor that takes an instance of the same class and copies it. Prototype is a pattern where the object knows how to clone itself via a `clone()` method. The key advantage of Prototype is that the caller doesn't need to know the concrete class — it just calls `clone()` on whatever object it has, achieving polymorphic cloning.

---

**Q5: How does the Factory Method pattern support the Open/Closed Principle?**

A: The factory method centralizes the creation decision. When a new class is added, you add a new branch to the factory (or a new factory subclass) without touching the client code that uses the created objects. The client is closed for modification; the factory is the extension point.

---

**Q6: How would you implement a thread-safe Singleton in TypeScript?**

A: TypeScript runs in a single-threaded event loop (Node.js), so traditional threading concerns don't apply. However, async initialization can still cause races. Use a module-level constant (module singleton — the module system caches the result) or lazy initialization with a promise:

```typescript
let instancePromise: Promise<ExpensiveResource> | null = null;

export function getResource(): Promise<ExpensiveResource> {
  if (!instancePromise) {
    instancePromise = ExpensiveResource.initialize();
  }
  return instancePromise;
}
```

---

**Q7: What is the Builder pattern's `director` role, and when do you need it?**

A: A Director is an optional class that knows a specific sequence of builder calls to produce a common configuration. For example, a `ReportDirector` might call `builder.setHeader().setBody().setFooter().setPageNumbers()` in the correct order. The Director encapsulates common construction recipes. It is useful when you have multiple common product configurations that are built the same way every time.

---

## 9. Exercises

**Exercise 1: Singleton — logger**

Implement a `Logger` singleton with `info()`, `warn()`, and `error()` methods that write to a `logs: string[]` array. Then refactor it to accept the logs array via constructor injection so tests can pass a fresh array.

*Hint: Compare how testability changes when you switch from `Logger.getInstance()` to injected `Logger`.*

---

**Exercise 2: Factory Method — payment processor**

Implement a `PaymentProcessorFactory` that creates `StripeProcessor`, `PayPalProcessor`, or `PixProcessor` based on a string input. Each processor has a `process(amount: number): Promise<string>` method. Add TypeScript exhaustiveness checking.

*Hint: Use a union type for the channel parameter and a `never` guard in the default case.*

---

**Exercise 3: Builder — user profile**

Build a `UserProfileBuilder` for an object with: `id`, `name`, `email`, `role` (optional, default `'user'`), `permissions` (optional, default `[]`), `avatar` (optional), `bio` (optional). The `build()` method should validate that `id`, `name`, and `email` are present.

*Hint: Use method chaining (return `this`). The builder should be impossible to build without required fields.*

---

**Exercise 4: Abstract Factory — report generators**

Create a report generation system with two factories: `CsvReportFactory` and `JsonReportFactory`. Each factory creates a matching `IRowFormatter` and `IHeaderFormatter`. The client code that generates the report should depend only on the factory interface.

*Hint: The report rendering function takes `IReportFactory` as a parameter and calls both formatters.*

---

## 10. Further Reading

- **Design Patterns: Elements of Reusable Object-Oriented Software** — Gamma, Helm, Johnson, Vlissides (GoF, 1994)
- **Head First Design Patterns** — Freeman & Robson (more accessible, Java examples easily translated)
- **[Refactoring Guru — Creational Patterns](https://refactoring.guru/design-patterns/creational-patterns)** — excellent diagrams and examples
- **[TypeScript Design Patterns](https://www.typescriptlang.org/docs/handbook/2/types-from-types.html)** — TypeScript advanced types used in patterns
- **Dive Into Design Patterns** — Alexander Shvets (free sample at refactoring.guru)
