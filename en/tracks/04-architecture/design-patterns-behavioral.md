# Design Patterns: Behavioral

## 1. What & Why

Behavioral design patterns deal with communication and responsibility between objects. They define how objects interact and distribute work, making complex control flow manageable and explicit.

The key problem behavioral patterns solve: as systems grow, the "who does what and when" logic tends to scatter across the codebase, creating tight coupling and untestable conditional chains. Behavioral patterns give that logic a home.

The six most important patterns with full implementations:

| Pattern | Core idea |
|---------|-----------|
| **Strategy** | Select an algorithm at runtime |
| **Observer** | Notify subscribers of state changes |
| **Command** | Encapsulate a request as an object (enables undo/redo) |
| **State** | Change behavior based on internal state |
| **Chain of Responsibility** | Pass a request through a pipeline of handlers |
| **Template Method** | Define algorithm skeleton; subclasses fill in steps |

Brief coverage: Iterator, Mediator, Memento, Visitor.

> 💡 Behavioral patterns are the ones you encounter most often in day-to-day development. You have used them even if you did not know their names: event listeners are Observer, Express middleware is Chain of Responsibility, Redux actions are Command.

---

## 2. Core Concepts

Behavioral patterns fall into two categories:
- **Class-based:** Template Method (uses inheritance)
- **Object-based:** Strategy, Observer, Command, State, Chain of Responsibility, Iterator, Mediator, Memento, Visitor (use composition and delegation)

The dominant trend in modern TypeScript: prefer composition. Strategy, Observer, and Command are all composition-based and integrate naturally with functional programming style.

---

## 3. How It Works

The general mechanism: extract the varying part of an algorithm or control flow into a separate object. The client depends on an abstraction (interface) and the concrete behavior is injected or switched at runtime.

---

## 4. Code Examples (TypeScript)

### Strategy — Payment Methods

The Strategy pattern defines a family of algorithms and makes them interchangeable. The client is decoupled from the algorithm it uses.

**Real-world use:** An e-commerce platform supports CreditCard, PayPal, and Crypto payments.

```typescript
// Strategy interface
interface PaymentStrategy {
  readonly name: string;
  pay(amount: number, currency: string): Promise<PaymentReceipt>;
  canRefund(): boolean;
}

interface PaymentReceipt {
  transactionId: string;
  amount: number;
  method: string;
}

// Concrete strategies
class CreditCardStrategy implements PaymentStrategy {
  readonly name = 'credit_card';

  constructor(
    private readonly cardToken: string,
    private readonly lastFour: string,
  ) {}

  async pay(amount: number, currency: string): Promise<PaymentReceipt> {
    console.log(`[CreditCard] Charging ${currency} ${amount} to card ending ${this.lastFour}`);
    // Call Stripe / Adyen / etc.
    return { transactionId: `cc_${Date.now()}`, amount, method: this.name };
  }

  canRefund(): boolean { return true; }
}

class PayPalStrategy implements PaymentStrategy {
  readonly name = 'paypal';

  constructor(private readonly paypalAccountId: string) {}

  async pay(amount: number, currency: string): Promise<PaymentReceipt> {
    console.log(`[PayPal] Charging ${currency} ${amount} to account ${this.paypalAccountId}`);
    return { transactionId: `pp_${Date.now()}`, amount, method: this.name };
  }

  canRefund(): boolean { return true; }
}

class CryptoStrategy implements PaymentStrategy {
  readonly name = 'crypto';

  constructor(
    private readonly walletAddress: string,
    private readonly coin: 'BTC' | 'ETH' | 'USDC',
  ) {}

  async pay(amount: number, currency: string): Promise<PaymentReceipt> {
    console.log(`[Crypto] Sending ${amount} ${this.coin} to ${this.walletAddress}`);
    return { transactionId: `crypto_${Date.now()}`, amount, method: this.name };
  }

  canRefund(): boolean { return false; } // crypto is non-refundable on-chain
}

// Context — uses a strategy, doesn't know its concrete type
class Checkout {
  constructor(private strategy: PaymentStrategy) {}

  setStrategy(strategy: PaymentStrategy): void {
    this.strategy = strategy;
  }

  async pay(amount: number, currency: string): Promise<PaymentReceipt> {
    return this.strategy.pay(amount, currency);
  }

  get supportsRefund(): boolean { return this.strategy.canRefund(); }
}

// Usage
const checkout = new Checkout(new CreditCardStrategy('tok_visa', '4242'));
const receipt = await checkout.pay(99.99, 'USD');

// Switch strategy at runtime
checkout.setStrategy(new CryptoStrategy('0xABCD...', 'ETH'));
console.log('Refunds supported:', checkout.supportsRefund); // false
```

---

### Observer — Event System

The Observer pattern defines a one-to-many relationship so that when one object changes state, all its dependents are notified automatically.

**Real-world use:** A simple typed event emitter, similar to how React state, DOM events, and message buses work.

```typescript
// Typed event system
type EventMap = Record<string, unknown>;

class TypedEventEmitter<T extends EventMap> {
  private listeners = new Map<keyof T, Set<(data: unknown) => void>>();

  on<K extends keyof T>(event: K, listener: (data: T[K]) => void): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(listener as (data: unknown) => void);

    // Return unsubscribe function
    return () => this.off(event, listener);
  }

  off<K extends keyof T>(event: K, listener: (data: T[K]) => void): void {
    this.listeners.get(event)?.delete(listener as (data: unknown) => void);
  }

  emit<K extends keyof T>(event: K, data: T[K]): void {
    this.listeners.get(event)?.forEach(listener => listener(data));
  }
}

// Domain-specific events
interface OrderEvents {
  placed: { orderId: string; customerId: string; total: number };
  cancelled: { orderId: string; reason: string };
  shipped: { orderId: string; trackingCode: string };
}

class OrderEventBus extends TypedEventEmitter<OrderEvents> {}

// Subscribers (observers)
const bus = new OrderEventBus();

const unsubscribeWarehouse = bus.on('placed', ({ orderId, total }) => {
  console.log(`[Warehouse] New order ${orderId} worth $${total} — reserve stock`);
});

bus.on('placed', ({ customerId, orderId }) => {
  console.log(`[Email] Send order confirmation to customer ${customerId} for order ${orderId}`);
});

bus.on('shipped', ({ orderId, trackingCode }) => {
  console.log(`[Email] Send tracking code ${trackingCode} for order ${orderId}`);
});

// Publishing events
bus.emit('placed', { orderId: 'ORD-001', customerId: 'CUST-1', total: 149.99 });
bus.emit('shipped', { orderId: 'ORD-001', trackingCode: 'TRK-ABC123' });

// Unsubscribe when done
unsubscribeWarehouse();
```

---

### Command — Text Editor Undo/Redo

The Command pattern encapsulates a request as an object, enabling parameterization, queuing, logging, and undoable operations.

```typescript
// Command interface
interface Command {
  execute(): void;
  undo(): void;
  description: string;
}

// Receiver — the text editor
class TextEditor {
  private content = '';
  private cursorPos = 0;

  getContent(): string { return this.content; }

  insertText(text: string, position: number): void {
    this.content = this.content.slice(0, position) + text + this.content.slice(position);
    this.cursorPos = position + text.length;
  }

  deleteText(position: number, length: number): void {
    this.content = this.content.slice(0, position) + this.content.slice(position + length);
    this.cursorPos = position;
  }

  getText(position: number, length: number): string {
    return this.content.slice(position, position + length);
  }
}

// Concrete commands
class InsertCommand implements Command {
  readonly description: string;

  constructor(
    private readonly editor: TextEditor,
    private readonly text: string,
    private readonly position: number,
  ) {
    this.description = `Insert "${text}" at position ${position}`;
  }

  execute(): void { this.editor.insertText(this.text, this.position); }
  undo(): void { this.editor.deleteText(this.position, this.text.length); }
}

class DeleteCommand implements Command {
  private deletedText = '';
  readonly description: string;

  constructor(
    private readonly editor: TextEditor,
    private readonly position: number,
    private readonly length: number,
  ) {
    this.description = `Delete ${length} chars at position ${position}`;
  }

  execute(): void {
    this.deletedText = this.editor.getText(this.position, this.length);
    this.editor.deleteText(this.position, this.length);
  }

  undo(): void { this.editor.insertText(this.deletedText, this.position); }
}

// Invoker — manages command history
class CommandHistory {
  private history: Command[] = [];
  private redoStack: Command[] = [];

  execute(command: Command): void {
    command.execute();
    this.history.push(command);
    this.redoStack = []; // clear redo stack on new command
    console.log(`[History] Executed: ${command.description}`);
  }

  undo(): void {
    const command = this.history.pop();
    if (!command) { console.log('[History] Nothing to undo'); return; }
    command.undo();
    this.redoStack.push(command);
    console.log(`[History] Undid: ${command.description}`);
  }

  redo(): void {
    const command = this.redoStack.pop();
    if (!command) { console.log('[History] Nothing to redo'); return; }
    command.execute();
    this.history.push(command);
    console.log(`[History] Redid: ${command.description}`);
  }
}

// Usage
const editor = new TextEditor();
const history = new CommandHistory();

history.execute(new InsertCommand(editor, 'Hello', 0));
history.execute(new InsertCommand(editor, ' World', 5));
console.log(editor.getContent()); // "Hello World"

history.undo();
console.log(editor.getContent()); // "Hello"

history.redo();
console.log(editor.getContent()); // "Hello World"
```

---

### State — Order State Machine

The State pattern allows an object to alter its behavior when its internal state changes. The object will appear to change its class.

```typescript
// State interface
interface OrderState {
  readonly name: string;
  confirm(order: OrderContext): void;
  ship(order: OrderContext): void;
  deliver(order: OrderContext): void;
  cancel(order: OrderContext, reason: string): void;
}

// Context
class OrderContext {
  private _state: OrderState;
  private _history: Array<{ from: string; to: string; at: Date }> = [];

  constructor(private readonly id: string) {
    this._state = new PendingState();
  }

  setState(newState: OrderState): void {
    this._history.push({ from: this._state.name, to: newState.name, at: new Date() });
    console.log(`[Order ${this.id}] ${this._state.name} → ${newState.name}`);
    this._state = newState;
  }

  confirm(): void { this._state.confirm(this); }
  ship(): void { this._state.ship(this); }
  deliver(): void { this._state.deliver(this); }
  cancel(reason: string): void { this._state.cancel(this, reason); }

  get status(): string { return this._state.name; }
  get history(): ReadonlyArray<{ from: string; to: string; at: Date }> { return this._history; }
}

// Concrete States
class PendingState implements OrderState {
  readonly name = 'pending';

  confirm(order: OrderContext): void { order.setState(new ConfirmedState()); }
  ship(_order: OrderContext): void { throw new Error('Cannot ship a pending order'); }
  deliver(_order: OrderContext): void { throw new Error('Cannot deliver a pending order'); }
  cancel(order: OrderContext, reason: string): void {
    console.log(`[Cancel] Reason: ${reason}`);
    order.setState(new CancelledState());
  }
}

class ConfirmedState implements OrderState {
  readonly name = 'confirmed';

  confirm(_order: OrderContext): void { throw new Error('Order is already confirmed'); }
  ship(order: OrderContext): void { order.setState(new ShippedState()); }
  deliver(_order: OrderContext): void { throw new Error('Cannot deliver before shipping'); }
  cancel(order: OrderContext, reason: string): void {
    console.log(`[Cancel] Reason: ${reason}`);
    order.setState(new CancelledState());
  }
}

class ShippedState implements OrderState {
  readonly name = 'shipped';

  confirm(_order: OrderContext): void { throw new Error('Order is already shipped'); }
  ship(_order: OrderContext): void { throw new Error('Order is already shipped'); }
  deliver(order: OrderContext): void { order.setState(new DeliveredState()); }
  cancel(_order: OrderContext, _reason: string): void {
    throw new Error('Cannot cancel a shipped order');
  }
}

class DeliveredState implements OrderState {
  readonly name = 'delivered';

  confirm(): void { throw new Error('Order is already delivered'); }
  ship(): void { throw new Error('Order is already delivered'); }
  deliver(): void { throw new Error('Order is already delivered'); }
  cancel(): void { throw new Error('Cannot cancel a delivered order'); }
}

class CancelledState implements OrderState {
  readonly name = 'cancelled';

  confirm(): void { throw new Error('Cannot modify a cancelled order'); }
  ship(): void { throw new Error('Cannot modify a cancelled order'); }
  deliver(): void { throw new Error('Cannot modify a cancelled order'); }
  cancel(): void { throw new Error('Order is already cancelled'); }
}

// Usage
const order = new OrderContext('ORD-001');
order.confirm();                   // pending → confirmed
order.ship();                      // confirmed → shipped
order.deliver();                   // shipped → delivered
console.log(order.status);         // 'delivered'
console.log(order.history.length); // 3 transitions
```

> 💡 The State pattern eliminates large `if/else` or `switch` blocks based on status. Each state class handles only the transitions it allows — invalid transitions throw immediately instead of silently doing nothing.

---

### Chain of Responsibility — HTTP Middleware Pipeline

Each handler in the chain decides to handle the request or pass it to the next handler.

```typescript
// Handler interface
interface Middleware {
  setNext(middleware: Middleware): Middleware;
  handle(req: Request, res: Response): Promise<void>;
}

interface Request {
  path: string;
  method: string;
  headers: Record<string, string>;
  body: unknown;
  user?: { id: string; roles: string[] };
}

interface Response {
  status: number;
  body: unknown;
}

// Abstract base handler
abstract class BaseMiddleware implements Middleware {
  private next: Middleware | null = null;

  setNext(middleware: Middleware): Middleware {
    this.next = middleware;
    return middleware;
  }

  protected async callNext(req: Request, res: Response): Promise<void> {
    if (this.next) await this.next.handle(req, res);
  }

  abstract handle(req: Request, res: Response): Promise<void>;
}

// Concrete handlers
class LoggingMiddleware extends BaseMiddleware {
  async handle(req: Request, res: Response): Promise<void> {
    const start = Date.now();
    console.log(`[LOG] ${req.method} ${req.path}`);
    await this.callNext(req, res);
    console.log(`[LOG] ${req.method} ${req.path} → ${res.status} (${Date.now() - start}ms)`);
  }
}

class AuthMiddleware extends BaseMiddleware {
  async handle(req: Request, res: Response): Promise<void> {
    const token = req.headers['authorization']?.replace('Bearer ', '');
    if (!token) {
      res.status = 401;
      res.body = { error: 'Unauthorized' };
      return; // stop the chain
    }
    // Simulate token verification
    req.user = { id: 'user_1', roles: ['user'] };
    await this.callNext(req, res);
  }
}

class RateLimitMiddleware extends BaseMiddleware {
  private requests = new Map<string, number>();

  async handle(req: Request, res: Response): Promise<void> {
    const userId = req.user?.id ?? req.headers['x-forwarded-for'] ?? 'anonymous';
    const count = (this.requests.get(userId) ?? 0) + 1;
    this.requests.set(userId, count);

    if (count > 100) {
      res.status = 429;
      res.body = { error: 'Too Many Requests' };
      return; // stop the chain
    }

    await this.callNext(req, res);
  }
}

class RouteHandler extends BaseMiddleware {
  async handle(req: Request, res: Response): Promise<void> {
    res.status = 200;
    res.body = { message: `Hello from ${req.path}`, user: req.user };
  }
}

// Build the pipeline
function buildPipeline(): Middleware {
  const logging = new LoggingMiddleware();
  const auth = new AuthMiddleware();
  const rateLimit = new RateLimitMiddleware();
  const route = new RouteHandler();

  logging.setNext(auth).setNext(rateLimit).setNext(route);
  return logging;
}

const pipeline = buildPipeline();
const req: Request = {
  path: '/api/orders',
  method: 'GET',
  headers: { authorization: 'Bearer valid_token_here' },
  body: null,
};
const res: Response = { status: 0, body: null };

await pipeline.handle(req, res);
```

---

### Template Method — Data Export Pipeline

The Template Method defines the skeleton of an algorithm in a base class. Subclasses override specific steps without changing the overall structure.

```typescript
// Abstract class with the algorithm skeleton
abstract class DataExporter {
  // Template method — defines the invariant algorithm
  async export(query: string): Promise<void> {
    console.log('[Exporter] Starting export...');
    const rawData = await this.readData(query);
    const processedData = this.processData(rawData);
    await this.writeData(processedData);
    this.notifyCompletion();
    console.log('[Exporter] Export complete.');
  }

  // Steps to be implemented by subclasses
  protected abstract readData(query: string): Promise<Record<string, unknown>[]>;
  protected abstract processData(data: Record<string, unknown>[]): string;
  protected abstract writeData(data: string): Promise<void>;

  // Hook — subclasses may override but are not required to
  protected notifyCompletion(): void {
    console.log('[Exporter] Export finished (default notification).');
  }
}

// CSV exporter
class CsvExporter extends DataExporter {
  protected async readData(query: string): Promise<Record<string, unknown>[]> {
    console.log(`[CSV] Fetching data: ${query}`);
    return [
      { id: 1, name: 'Alice', email: 'alice@example.com' },
      { id: 2, name: 'Bob', email: 'bob@example.com' },
    ];
  }

  protected processData(data: Record<string, unknown>[]): string {
    const headers = Object.keys(data[0] ?? {}).join(',');
    const rows = data.map(row => Object.values(row).join(','));
    return [headers, ...rows].join('\n');
  }

  protected async writeData(data: string): Promise<void> {
    console.log(`[CSV] Writing to file:\n${data}`);
    // Real: fs.writeFile(...)
  }
}

// JSON exporter — reuses readData, overrides process and write
class JsonExporter extends DataExporter {
  protected async readData(query: string): Promise<Record<string, unknown>[]> {
    console.log(`[JSON] Fetching data: ${query}`);
    return [
      { id: 1, name: 'Alice', email: 'alice@example.com' },
    ];
  }

  protected processData(data: Record<string, unknown>[]): string {
    return JSON.stringify(data, null, 2);
  }

  protected async writeData(data: string): Promise<void> {
    console.log(`[JSON] Writing:\n${data}`);
  }

  protected notifyCompletion(): void {
    console.log('[JSON] Sent webhook notification on completion.');
  }
}

// Usage — client calls the template method, not the steps
const csvExporter = new CsvExporter();
await csvExporter.export('SELECT * FROM users');

const jsonExporter = new JsonExporter();
await jsonExporter.export('SELECT * FROM users');
```

> ⚠️ Template Method uses inheritance, which creates a tight coupling between base and subclass. If the algorithm in the base class changes, all subclasses may be affected. Prefer Strategy when the algorithm steps vary significantly across cases — Strategy uses composition instead.

---

## 5. Brief Coverage: Other Behavioral Patterns

### Iterator

Provides a way to sequentially access elements of a collection without exposing its underlying representation.

```typescript
class Range implements Iterable<number> {
  constructor(private readonly start: number, private readonly end: number) {}

  [Symbol.iterator](): Iterator<number> {
    let current = this.start;
    const end = this.end;
    return {
      next(): IteratorResult<number> {
        if (current <= end) return { value: current++, done: false };
        return { value: undefined as never, done: true };
      },
    };
  }
}

for (const n of new Range(1, 5)) {
  console.log(n); // 1 2 3 4 5
}
```

In TypeScript, use built-in iterables (`for...of`, generators) rather than implementing the GoF Iterator manually.

### Mediator

Defines an object that encapsulates how a set of objects interact, promoting loose coupling by preventing direct references between them.

```typescript
// Chat room as mediator
interface IChatMediator {
  send(message: string, sender: ChatUser): void;
  register(user: ChatUser): void;
}

class ChatRoom implements IChatMediator {
  private users: ChatUser[] = [];

  register(user: ChatUser): void { this.users.push(user); }

  send(message: string, sender: ChatUser): void {
    this.users
      .filter(u => u !== sender)
      .forEach(u => u.receive(message, sender.name));
  }
}

class ChatUser {
  constructor(private readonly mediator: IChatMediator, public readonly name: string) {
    mediator.register(this);
  }

  send(message: string): void { this.mediator.send(message, this); }
  receive(message: string, from: string): void { console.log(`[${this.name}] ${from}: ${message}`); }
}
```

### Memento

Captures and externalizes an object's internal state so it can be restored later — without violating encapsulation.

```typescript
// Memento — opaque snapshot
class EditorMemento {
  constructor(private readonly state: string) {}
  getState(): string { return this.state; }
}

class TextEditorWithMemento {
  private content = '';

  type(text: string): void { this.content += text; }
  save(): EditorMemento { return new EditorMemento(this.content); }
  restore(memento: EditorMemento): void { this.content = memento.getState(); }
  getContent(): string { return this.content; }
}

// Usage
const editor = new TextEditorWithMemento();
editor.type('Hello');
const snapshot = editor.save();
editor.type(' World — this is unwanted');
editor.restore(snapshot);
console.log(editor.getContent()); // 'Hello'
```

### Visitor

Lets you add operations to objects without modifying their classes. Useful for traversing complex structures with many element types.

```typescript
interface ShapeVisitor {
  visitCircle(circle: Circle): number;
  visitRectangle(rect: Rectangle): number;
}

interface Shape {
  accept(visitor: ShapeVisitor): number;
}

class Circle implements Shape {
  constructor(public readonly radius: number) {}
  accept(visitor: ShapeVisitor): number { return visitor.visitCircle(this); }
}

class Rectangle implements Shape {
  constructor(public readonly width: number, public readonly height: number) {}
  accept(visitor: ShapeVisitor): number { return visitor.visitRectangle(this); }
}

class AreaCalculator implements ShapeVisitor {
  visitCircle(c: Circle): number { return Math.PI * c.radius ** 2; }
  visitRectangle(r: Rectangle): number { return r.width * r.height; }
}

const shapes: Shape[] = [new Circle(5), new Rectangle(4, 6)];
const calculator = new AreaCalculator();
shapes.forEach(s => console.log(`Area: ${s.accept(calculator).toFixed(2)}`));
```

---

## 6. Common Mistakes & Pitfalls

| Pattern | Mistake | Fix |
|---------|---------|-----|
| Strategy | Selecting strategy inside the context | Selection belongs in the factory or caller; context just uses it |
| Observer | Forgetting to unsubscribe | Always return and call the unsubscribe function; use `AbortController` in browser |
| Command | Storing mutable references in command objects | Commands should capture the state at creation time (or be immutable) |
| State | Putting transition logic in the context | Each state class decides its own valid transitions |
| Chain of Responsibility | Handlers assuming chain will continue | Always explicitly call `next` or stop; don't rely on implicit flow |
| Template Method | Base class having too many abstract methods | Extract to Strategy if most steps vary independently |

> ⚠️ Observer memory leaks are a very common bug. Every `on()` call that is not paired with an `off()` or a cleanup function leaks a listener. In frameworks: React `useEffect` cleanup, Node.js EventEmitter `removeListener`.

---

## 7. When to Use / Not Use

| Pattern | Use when | Avoid when |
|---------|----------|-----------|
| Strategy | Multiple interchangeable algorithms | Only one algorithm exists |
| Observer | Decoupled, one-to-many notifications | Two tightly coupled components where direct calls are clearer |
| Command | Undoable operations, request queuing, logging | Simple, non-reversible operations where the overhead is not justified |
| State | Objects with many status-dependent behaviors | Only 2 states with simple transitions — a boolean flag may suffice |
| Chain of Responsibility | Dynamic, ordered processing pipelines | Fixed processing with known handlers — a simple sequence of calls is clearer |
| Template Method | Algorithm with invariant structure and variable steps | Steps vary so much across subclasses that inheritance becomes awkward — use Strategy |

---

## 8. Interview Questions

**Q1: What is the difference between Strategy and State?**

A: Both patterns look structurally identical (a context holds a reference to an interface, concrete classes implement it). The difference is intent. Strategy selects an algorithm *from outside* — the context doesn't know which one will be used, and switching is driven by the client. State transitions *from within* the object — the context changes its own state based on its own rules, often driven by method calls on the context itself.

---

**Q2: What problem does the Command pattern solve that a simple function call does not?**

A: A function call is fire-and-forget. The Command pattern makes the call an object — it can be stored in a queue, logged, serialized, delayed, retried, and most importantly, reversed. This enables undo/redo, transaction logs, and job queues. The Command carries everything needed to execute and undo the operation.

---

**Q3: What is the risk of deeply nested Observer chains?**

A: Cascading events — handler A emits event B, handler B emits event C — can create difficult-to-trace call stacks and infinite loops. A high count of listeners on a single event can also degrade performance. Mitigation: keep event hierarchies shallow, use async dispatch to break synchronous chains, and set maximum listener counts.

---

**Q4: How does Chain of Responsibility differ from a Decorator?**

A: Both pass control from one handler/wrapper to the next. The key difference: Decorator always delegates to the next wrapper (it enhances behavior). In Chain of Responsibility, any handler can stop the chain — it handles the request *or* passes it, not both. COR is about routing a request to the first capable handler; Decorator is about layering behavior.

---

**Q5: Why does Template Method use inheritance and is that a problem?**

A: Template Method is explicitly inheritance-based by definition. This creates tight coupling: the base class's algorithm structure is fixed, and subclasses are coupled to it. If the algorithm structure changes, all subclasses may break. For this reason, many modern developers prefer Strategy — it uses composition to achieve the same goal without inheritance coupling. Template Method is appropriate when the base class truly owns the algorithm skeleton and variation is genuinely minimal.

---

**Q6: How would you implement undo/redo with a size-limited history?**

A: Use a circular buffer (or a simple array with a maximum size). When the history reaches the limit, remove the oldest command before adding the new one. The redo stack is always cleared when a new command is executed.

```typescript
class BoundedCommandHistory {
  private history: Command[] = [];
  private redoStack: Command[] = [];
  constructor(private readonly maxSize: number) {}

  execute(cmd: Command): void {
    if (this.history.length >= this.maxSize) this.history.shift();
    cmd.execute();
    this.history.push(cmd);
    this.redoStack = [];
  }

  undo(): void {
    const cmd = this.history.pop();
    if (cmd) { cmd.undo(); this.redoStack.push(cmd); }
  }

  redo(): void {
    const cmd = this.redoStack.pop();
    if (cmd) { cmd.execute(); this.history.push(cmd); }
  }
}
```

---

**Q7: When should you use Observer vs. a direct method call?**

A: Use Observer when: (1) the publisher should not know who the subscribers are (decoupled), (2) the number of subscribers varies at runtime, (3) multiple independent components need to react to the same event. Use a direct call when: the relationship is one-to-one, the dependency is intentional and stable, or the indirection of an event would obscure the flow for no benefit.

---

## 9. Exercises

**Exercise 1: Strategy — discount calculator**

Implement a `PricingEngine` that applies a `PricingStrategy`. Strategies: `NoDiscount`, `PercentageDiscount(rate)`, `FixedDiscount(amount)`, `BuyXGetYFreeDiscount(x, y)`. Test each strategy with the same `PricingEngine` instance.

*Hint: The engine just calls `strategy.apply(basePrice): number`. The strategies encapsulate all calculation logic.*

---

**Exercise 2: Observer — stock price monitor**

Build a `StockTracker` that emits price updates for ticker symbols. Implement two observers: `AlertObserver` (fires when price drops below a threshold) and `LogObserver` (logs every update). Test that unsubscribing `AlertObserver` stops it from receiving updates.

---

**Exercise 3: Command — shopping cart**

Implement an `AddToCart` and `RemoveFromCart` command for a shopping cart. Use `CommandHistory` to support undo. After undo of `RemoveFromCart`, the item should be back in the cart.

---

**Exercise 4: State — traffic light**

Model a traffic light as a State machine with states: `Red`, `Green`, `Yellow`. Each state has a `next()` method that transitions to the correct next state. Verify that transitions only happen in the correct order: Red → Green → Yellow → Red.

---

## 10. Further Reading

- **Design Patterns: Elements of Reusable Object-Oriented Software** — Gamma, Helm, Johnson, Vlissides
- **[Refactoring Guru — Behavioral Patterns](https://refactoring.guru/design-patterns/behavioral-patterns)** — illustrated TypeScript examples
- **Head First Design Patterns** — Freeman & Robson (Strategy and Observer chapters are excellent)
- **[RxJS](https://rxjs.dev/)** — a powerful implementation of Observer with functional composition
- **[XState](https://xstate.js.org/)** — production-grade State Machine library for TypeScript
