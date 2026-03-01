# Padrões de Projeto: Comportamentais

## 1. O que são e por que importam

Padrões comportamentais lidam com comunicação e responsabilidade entre objetos. Eles definem como objetos interagem e distribuem trabalho, tornando fluxos de controle complexos gerenciáveis e explícitos.

O problema central que os padrões comportamentais resolvem: conforme os sistemas crescem, a lógica de "quem faz o quê e quando" tende a se espalhar pelo codebase, criando acoplamento forte e cadeias condicionais impossíveis de testar. Padrões comportamentais dão um lar para essa lógica.

Os seis padrões mais importantes com implementações completas:

| Padrão | Ideia central |
|--------|--------------|
| **Strategy** | Selecionar um algoritmo em runtime |
| **Observer** | Notificar assinantes de mudanças de estado |
| **Command** | Encapsular uma requisição como objeto (habilita undo/redo) |
| **State** | Mudar comportamento com base no estado interno |
| **Chain of Responsibility** | Passar uma requisição por um pipeline de handlers |
| **Template Method** | Definir esqueleto do algoritmo; subclasses preenchem as etapas |

Cobertura resumida: Iterator, Mediator, Memento, Visitor.

> 💡 Padrões comportamentais são os que você encontra com mais frequência no desenvolvimento do dia a dia. Você já os usou mesmo sem saber os nomes: event listeners são Observer, middleware do Express é Chain of Responsibility, actions do Redux são Command.

---

## 2. Conceitos Fundamentais

Padrões comportamentais se dividem em duas categorias:
- **Baseados em classes:** Template Method (usa herança)
- **Baseados em objetos:** Strategy, Observer, Command, State, Chain of Responsibility, Iterator, Mediator, Memento, Visitor (usam composição e delegação)

A tendência dominante no TypeScript moderno: prefira composição. Strategy, Observer e Command são todos baseados em composição e se integram naturalmente com estilo de programação funcional.

---

## 3. Como Funcionam

O mecanismo geral: extraia a parte variável de um algoritmo ou fluxo de controle para um objeto separado. O cliente depende de uma abstração (interface) e o comportamento concreto é injetado ou trocado em runtime.

---

## 4. Exemplos de Código (TypeScript)

### Strategy — Métodos de Pagamento

O padrão Strategy define uma família de algoritmos e os torna intercambiáveis. O cliente é desacoplado do algoritmo que usa.

**Caso real:** Uma plataforma de e-commerce suporta pagamentos por Cartão de Crédito, PayPal e Crypto.

```typescript
// Interface de strategy
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

// Strategies concretas
class CreditCardStrategy implements PaymentStrategy {
  readonly name = 'credit_card';

  constructor(
    private readonly cardToken: string,
    private readonly lastFour: string,
  ) {}

  async pay(amount: number, currency: string): Promise<PaymentReceipt> {
    console.log(`[Cartão] Cobrando ${currency} ${amount} no cartão final ${this.lastFour}`);
    // Chama Stripe / Adyen / etc.
    return { transactionId: `cc_${Date.now()}`, amount, method: this.name };
  }

  canRefund(): boolean { return true; }
}

class PayPalStrategy implements PaymentStrategy {
  readonly name = 'paypal';

  constructor(private readonly paypalAccountId: string) {}

  async pay(amount: number, currency: string): Promise<PaymentReceipt> {
    console.log(`[PayPal] Cobrando ${currency} ${amount} na conta ${this.paypalAccountId}`);
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
    console.log(`[Crypto] Enviando ${amount} ${this.coin} para ${this.walletAddress}`);
    return { transactionId: `crypto_${Date.now()}`, amount, method: this.name };
  }

  canRefund(): boolean { return false; } // crypto não é reembolsável on-chain
}

// Contexto — usa uma strategy, não conhece seu tipo concreto
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

// Uso
const checkout = new Checkout(new CreditCardStrategy('tok_visa', '4242'));
const receipt = await checkout.pay(99.99, 'BRL');

// Troca de strategy em runtime
checkout.setStrategy(new CryptoStrategy('0xABCD...', 'ETH'));
console.log('Suporta reembolso:', checkout.supportsRefund); // false
```

---

### Observer — Sistema de Eventos

O padrão Observer define uma relação um-para-muitos para que quando um objeto muda de estado, todos os seus dependentes sejam notificados automaticamente.

**Caso real:** Um event emitter tipado simples, similar ao funcionamento do estado React, eventos DOM e message buses.

```typescript
// Sistema de eventos tipado
type EventMap = Record<string, unknown>;

class TypedEventEmitter<T extends EventMap> {
  private listeners = new Map<keyof T, Set<(data: unknown) => void>>();

  on<K extends keyof T>(event: K, listener: (data: T[K]) => void): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(listener as (data: unknown) => void);

    // Retorna função de unsubscribe
    return () => this.off(event, listener);
  }

  off<K extends keyof T>(event: K, listener: (data: T[K]) => void): void {
    this.listeners.get(event)?.delete(listener as (data: unknown) => void);
  }

  emit<K extends keyof T>(event: K, data: T[K]): void {
    this.listeners.get(event)?.forEach(listener => listener(data));
  }
}

// Eventos específicos do domínio
interface OrderEvents {
  placed: { orderId: string; customerId: string; total: number };
  cancelled: { orderId: string; reason: string };
  shipped: { orderId: string; trackingCode: string };
}

class OrderEventBus extends TypedEventEmitter<OrderEvents> {}

// Assinantes (observers)
const bus = new OrderEventBus();

const unsubscribeWarehouse = bus.on('placed', ({ orderId, total }) => {
  console.log(`[Estoque] Novo pedido ${orderId} de R$ ${total} — reservar estoque`);
});

bus.on('placed', ({ customerId, orderId }) => {
  console.log(`[Email] Enviar confirmação ao cliente ${customerId} para pedido ${orderId}`);
});

bus.on('shipped', ({ orderId, trackingCode }) => {
  console.log(`[Email] Enviar código de rastreio ${trackingCode} para pedido ${orderId}`);
});

// Publicando eventos
bus.emit('placed', { orderId: 'ORD-001', customerId: 'CUST-1', total: 149.99 });
bus.emit('shipped', { orderId: 'ORD-001', trackingCode: 'TRK-ABC123' });

// Unsubscribe quando terminar
unsubscribeWarehouse();
```

---

### Command — Undo/Redo em Editor de Texto

O padrão Command encapsula uma requisição como um objeto, habilitando parametrização, enfileiramento, logging e operações desfazíveis.

```typescript
// Interface Command
interface Command {
  execute(): void;
  undo(): void;
  description: string;
}

// Receiver — o editor de texto
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

// Commands concretos
class InsertCommand implements Command {
  readonly description: string;

  constructor(
    private readonly editor: TextEditor,
    private readonly text: string,
    private readonly position: number,
  ) {
    this.description = `Inserir "${text}" na posição ${position}`;
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
    this.description = `Deletar ${length} chars na posição ${position}`;
  }

  execute(): void {
    this.deletedText = this.editor.getText(this.position, this.length);
    this.editor.deleteText(this.position, this.length);
  }

  undo(): void { this.editor.insertText(this.deletedText, this.position); }
}

// Invoker — gerencia o histórico de commands
class CommandHistory {
  private history: Command[] = [];
  private redoStack: Command[] = [];

  execute(command: Command): void {
    command.execute();
    this.history.push(command);
    this.redoStack = []; // limpa redo stack em novo command
    console.log(`[Histórico] Executado: ${command.description}`);
  }

  undo(): void {
    const command = this.history.pop();
    if (!command) { console.log('[Histórico] Nada para desfazer'); return; }
    command.undo();
    this.redoStack.push(command);
    console.log(`[Histórico] Desfeito: ${command.description}`);
  }

  redo(): void {
    const command = this.redoStack.pop();
    if (!command) { console.log('[Histórico] Nada para refazer'); return; }
    command.execute();
    this.history.push(command);
    console.log(`[Histórico] Refeito: ${command.description}`);
  }
}

// Uso
const editor = new TextEditor();
const history = new CommandHistory();

history.execute(new InsertCommand(editor, 'Olá', 0));
history.execute(new InsertCommand(editor, ' Mundo', 3));
console.log(editor.getContent()); // "Olá Mundo"

history.undo();
console.log(editor.getContent()); // "Olá"

history.redo();
console.log(editor.getContent()); // "Olá Mundo"
```

---

### State — Máquina de Estado de Pedido

O padrão State permite que um objeto altere seu comportamento quando seu estado interno muda. O objeto parecerá mudar de classe.

```typescript
// Interface State
interface OrderState {
  readonly name: string;
  confirm(order: OrderContext): void;
  ship(order: OrderContext): void;
  deliver(order: OrderContext): void;
  cancel(order: OrderContext, reason: string): void;
}

// Contexto
class OrderContext {
  private _state: OrderState;
  private _history: Array<{ from: string; to: string; at: Date }> = [];

  constructor(private readonly id: string) {
    this._state = new PendingState();
  }

  setState(newState: OrderState): void {
    this._history.push({ from: this._state.name, to: newState.name, at: new Date() });
    console.log(`[Pedido ${this.id}] ${this._state.name} → ${newState.name}`);
    this._state = newState;
  }

  confirm(): void { this._state.confirm(this); }
  ship(): void { this._state.ship(this); }
  deliver(): void { this._state.deliver(this); }
  cancel(reason: string): void { this._state.cancel(this, reason); }

  get status(): string { return this._state.name; }
  get history(): ReadonlyArray<{ from: string; to: string; at: Date }> { return this._history; }
}

// States concretos
class PendingState implements OrderState {
  readonly name = 'pending';

  confirm(order: OrderContext): void { order.setState(new ConfirmedState()); }
  ship(_order: OrderContext): void { throw new Error('Não é possível enviar um pedido pendente'); }
  deliver(_order: OrderContext): void { throw new Error('Não é possível entregar um pedido pendente'); }
  cancel(order: OrderContext, reason: string): void {
    console.log(`[Cancelamento] Motivo: ${reason}`);
    order.setState(new CancelledState());
  }
}

class ConfirmedState implements OrderState {
  readonly name = 'confirmed';

  confirm(_order: OrderContext): void { throw new Error('Pedido já está confirmado'); }
  ship(order: OrderContext): void { order.setState(new ShippedState()); }
  deliver(_order: OrderContext): void { throw new Error('Não é possível entregar antes do envio'); }
  cancel(order: OrderContext, reason: string): void {
    console.log(`[Cancelamento] Motivo: ${reason}`);
    order.setState(new CancelledState());
  }
}

class ShippedState implements OrderState {
  readonly name = 'shipped';

  confirm(_order: OrderContext): void { throw new Error('Pedido já foi enviado'); }
  ship(_order: OrderContext): void { throw new Error('Pedido já foi enviado'); }
  deliver(order: OrderContext): void { order.setState(new DeliveredState()); }
  cancel(_order: OrderContext, _reason: string): void {
    throw new Error('Não é possível cancelar um pedido enviado');
  }
}

class DeliveredState implements OrderState {
  readonly name = 'delivered';

  confirm(): void { throw new Error('Pedido já foi entregue'); }
  ship(): void { throw new Error('Pedido já foi entregue'); }
  deliver(): void { throw new Error('Pedido já foi entregue'); }
  cancel(): void { throw new Error('Não é possível cancelar um pedido entregue'); }
}

class CancelledState implements OrderState {
  readonly name = 'cancelled';

  confirm(): void { throw new Error('Não é possível modificar um pedido cancelado'); }
  ship(): void { throw new Error('Não é possível modificar um pedido cancelado'); }
  deliver(): void { throw new Error('Não é possível modificar um pedido cancelado'); }
  cancel(): void { throw new Error('Pedido já está cancelado'); }
}

// Uso
const order = new OrderContext('ORD-001');
order.confirm();                   // pending → confirmed
order.ship();                      // confirmed → shipped
order.deliver();                   // shipped → delivered
console.log(order.status);         // 'delivered'
console.log(order.history.length); // 3 transições
```

> 💡 O padrão State elimina grandes blocos `if/else` ou `switch` baseados em status. Cada classe de state trata apenas as transições que permite — transições inválidas lançam imediatamente em vez de silenciosamente não fazer nada.

---

### Chain of Responsibility — Pipeline de Middleware HTTP

Cada handler na cadeia decide tratar a requisição ou passá-la ao próximo handler.

```typescript
// Interface Handler
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

// Handler base abstrato
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

// Handlers concretos
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
      res.body = { error: 'Não autorizado' };
      return; // para a cadeia
    }
    // Simula verificação de token
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
      res.body = { error: 'Muitas requisições' };
      return; // para a cadeia
    }

    await this.callNext(req, res);
  }
}

class RouteHandler extends BaseMiddleware {
  async handle(req: Request, res: Response): Promise<void> {
    res.status = 200;
    res.body = { message: `Olá de ${req.path}`, user: req.user };
  }
}

// Constrói o pipeline
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

### Template Method — Pipeline de Exportação de Dados

O Template Method define o esqueleto de um algoritmo em uma classe base. Subclasses sobrescrevem etapas específicas sem mudar a estrutura geral.

```typescript
// Classe abstrata com o esqueleto do algoritmo
abstract class DataExporter {
  // Template method — define o algoritmo invariante
  async export(query: string): Promise<void> {
    console.log('[Exporter] Iniciando exportação...');
    const rawData = await this.readData(query);
    const processedData = this.processData(rawData);
    await this.writeData(processedData);
    this.notifyCompletion();
    console.log('[Exporter] Exportação concluída.');
  }

  // Etapas a serem implementadas pelas subclasses
  protected abstract readData(query: string): Promise<Record<string, unknown>[]>;
  protected abstract processData(data: Record<string, unknown>[]): string;
  protected abstract writeData(data: string): Promise<void>;

  // Hook — subclasses podem sobrescrever, mas não são obrigadas
  protected notifyCompletion(): void {
    console.log('[Exporter] Exportação finalizada (notificação padrão).');
  }
}

// Exporter CSV
class CsvExporter extends DataExporter {
  protected async readData(query: string): Promise<Record<string, unknown>[]> {
    console.log(`[CSV] Buscando dados: ${query}`);
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
    console.log(`[CSV] Escrevendo no arquivo:\n${data}`);
    // Real: fs.writeFile(...)
  }
}

// Exporter JSON — reutiliza readData, sobrescreve processData e writeData
class JsonExporter extends DataExporter {
  protected async readData(query: string): Promise<Record<string, unknown>[]> {
    console.log(`[JSON] Buscando dados: ${query}`);
    return [
      { id: 1, name: 'Alice', email: 'alice@example.com' },
    ];
  }

  protected processData(data: Record<string, unknown>[]): string {
    return JSON.stringify(data, null, 2);
  }

  protected async writeData(data: string): Promise<void> {
    console.log(`[JSON] Escrevendo:\n${data}`);
  }

  protected notifyCompletion(): void {
    console.log('[JSON] Webhook de notificação enviado ao concluir.');
  }
}

// Uso — o cliente chama o template method, não as etapas
const csvExporter = new CsvExporter();
await csvExporter.export('SELECT * FROM users');

const jsonExporter = new JsonExporter();
await jsonExporter.export('SELECT * FROM users');
```

> ⚠️ Template Method usa herança, o que cria acoplamento forte entre a classe base e a subclasse. Se a estrutura do algoritmo na classe base muda, todas as subclasses podem ser afetadas. Prefira Strategy quando as etapas do algoritmo variam significativamente entre os casos — Strategy usa composição em vez de herança.

---

## 5. Cobertura Resumida: Outros Padrões Comportamentais

### Iterator

Fornece uma forma de acessar sequencialmente elementos de uma coleção sem expor sua representação subjacente.

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

Em TypeScript, use iterables nativos (`for...of`, generators) em vez de implementar o Iterator do GoF manualmente.

### Mediator

Define um objeto que encapsula como um conjunto de objetos interage, promovendo acoplamento fraco evitando referências diretas entre eles.

```typescript
// Sala de chat como mediator
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

Captura e externaliza o estado interno de um objeto para que possa ser restaurado posteriormente — sem violar o encapsulamento.

```typescript
// Memento — snapshot opaco
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

// Uso
const editor = new TextEditorWithMemento();
editor.type('Olá');
const snapshot = editor.save();
editor.type(' Mundo — isso é indesejado');
editor.restore(snapshot);
console.log(editor.getContent()); // 'Olá'
```

### Visitor

Permite adicionar operações a objetos sem modificar suas classes. Útil para percorrer estruturas complexas com muitos tipos de elementos.

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
shapes.forEach(s => console.log(`Área: ${s.accept(calculator).toFixed(2)}`));
```

---

## 6. Erros Comuns e Armadilhas

| Padrão | Erro | Correção |
|--------|------|----------|
| Strategy | Selecionar strategy dentro do contexto | A seleção pertence à factory ou ao chamador; o contexto apenas usa |
| Observer | Esquecer de desinscrever | Sempre retorne e chame a função de unsubscribe; use `AbortController` no browser |
| Command | Armazenar referências mutáveis em command objects | Commands devem capturar o estado no momento da criação (ou ser imutáveis) |
| State | Colocar lógica de transição no contexto | Cada classe de state decide suas próprias transições válidas |
| Chain of Responsibility | Handlers assumindo que a cadeia continuará | Sempre chame `next` explicitamente ou pare; não confie em fluxo implícito |
| Template Method | Classe base com muitos métodos abstratos | Extraia para Strategy se a maioria das etapas variar independentemente |

> ⚠️ Memory leaks de Observer são um bug muito comum. Toda chamada `on()` que não é pareada com `off()` ou uma função de cleanup vaza um listener. Em frameworks: cleanup do `useEffect` do React, `removeListener` do EventEmitter do Node.js.

---

## 7. Quando Usar / Não Usar

| Padrão | Use quando | Evite quando |
|--------|-----------|-------------|
| Strategy | Múltiplos algoritmos intercambiáveis | Existe apenas um algoritmo |
| Observer | Notificações desacopladas um-para-muitos | Dois componentes fortemente acoplados onde chamadas diretas são mais claras |
| Command | Operações desfazíveis, enfileiramento de requisições, logging | Operações simples e não reversíveis onde o overhead não se justifica |
| State | Objetos com muitos comportamentos dependentes de status | Apenas 2 estados com transições simples — um flag booleano pode ser suficiente |
| Chain of Responsibility | Pipelines de processamento dinâmicos e ordenados | Processamento fixo com handlers conhecidos — uma sequência simples de chamadas é mais clara |
| Template Method | Algoritmo com estrutura invariante e etapas variáveis | As etapas variam tanto entre as subclasses que a herança se torna estranha — use Strategy |

---

## 8. Perguntas de Entrevista

**Q1: Qual é a diferença entre Strategy e State?**

R: Ambos os padrões são estruturalmente idênticos (um contexto mantém referência a uma interface, classes concretas a implementam). A diferença é de intenção. Strategy seleciona um algoritmo *de fora* — o contexto não sabe qual será usado, e a troca é acionada pelo cliente. State faz transições *de dentro* do objeto — o contexto muda seu próprio estado com base em suas próprias regras, frequentemente acionado por chamadas de métodos no próprio contexto.

---

**Q2: Que problema o padrão Command resolve que uma simples chamada de função não resolve?**

R: Uma chamada de função é fire-and-forget. O padrão Command torna a chamada um objeto — ele pode ser armazenado em uma fila, logado, serializado, adiado, reexecutado e, mais importante, revertido. Isso habilita undo/redo, logs de transação e filas de jobs. O Command carrega tudo necessário para executar e desfazer a operação.

---

**Q3: Qual é o risco de cadeias de Observer profundamente aninhadas?**

R: Eventos em cascata — o handler A emite o evento B, o handler B emite o evento C — podem criar call stacks difíceis de rastrear e loops infinitos. Um alto número de listeners em um único evento também pode degradar a performance. Mitigação: mantenha hierarquias de eventos rasas, use dispatch assíncrono para quebrar cadeias síncronas e defina contagem máxima de listeners.

---

**Q4: Como Chain of Responsibility difere de um Decorator?**

R: Ambos passam controle de um handler/wrapper para o próximo. A diferença principal: Decorator sempre delega ao próximo wrapper (ele aprimora o comportamento). Em Chain of Responsibility, qualquer handler pode parar a cadeia — ele trata a requisição *ou* a passa, não os dois. COR é sobre rotear uma requisição para o primeiro handler capaz; Decorator é sobre camadas de comportamento.

---

**Q5: Por que Template Method usa herança e isso é um problema?**

R: Template Method é explicitamente baseado em herança por definição. Isso cria acoplamento forte: a estrutura do algoritmo na classe base é fixa, e as subclasses estão acopladas a ela. Se a estrutura do algoritmo mudar, todas as subclasses podem quebrar. Por essa razão, muitos desenvolvedores modernos preferem Strategy — ele usa composição para alcançar o mesmo objetivo sem acoplamento de herança. Template Method é apropriado quando a classe base realmente possui o esqueleto do algoritmo e a variação é genuinamente mínima.

---

**Q6: Como implementar undo/redo com histórico de tamanho limitado?**

R: Use um buffer circular (ou um array simples com tamanho máximo). Quando o histórico atingir o limite, remova o command mais antigo antes de adicionar o novo. O redo stack é sempre limpo quando um novo command é executado.

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

**Q7: Quando usar Observer vs. uma chamada direta de método?**

R: Use Observer quando: (1) o publisher não deve saber quem são os assinantes (desacoplado), (2) o número de assinantes varia em runtime, (3) múltiplos componentes independentes precisam reagir ao mesmo evento. Use chamada direta quando: o relacionamento é um-para-um, a dependência é intencional e estável, ou a indireção de um evento obscureceria o fluxo sem benefício.

---

## 9. Exercícios

**Exercício 1: Strategy — calculadora de desconto**

Implemente um `PricingEngine` que aplica uma `PricingStrategy`. Strategies: `NoDiscount`, `PercentageDiscount(rate)`, `FixedDiscount(amount)`, `BuyXGetYFreeDiscount(x, y)`. Teste cada strategy com a mesma instância de `PricingEngine`.

*Dica: O engine apenas chama `strategy.apply(basePrice): number`. As strategies encapsulam toda a lógica de cálculo.*

---

**Exercício 2: Observer — monitor de preço de ações**

Construa um `StockTracker` que emite atualizações de preço para símbolos de ticker. Implemente dois observers: `AlertObserver` (dispara quando o preço cai abaixo de um limite) e `LogObserver` (loga toda atualização). Teste que desinscrever `AlertObserver` para de receber atualizações.

---

**Exercício 3: Command — carrinho de compras**

Implemente um `AddToCart` e `RemoveFromCart` command para um carrinho de compras. Use `CommandHistory` para suportar undo. Após undo de `RemoveFromCart`, o item deve estar de volta no carrinho.

---

**Exercício 4: State — semáforo**

Modele um semáforo como uma máquina de estado com estados: `Red`, `Green`, `Yellow`. Cada state tem um método `next()` que faz a transição para o próximo estado correto. Verifique que as transições ocorrem apenas na ordem correta: Vermelho → Verde → Amarelo → Vermelho.

---

## 10. Leitura Complementar

- **Design Patterns: Elements of Reusable Object-Oriented Software** — Gamma, Helm, Johnson, Vlissides
- **[Refactoring Guru — Behavioral Patterns](https://refactoring.guru/design-patterns/behavioral-patterns)** — exemplos ilustrados em TypeScript
- **Head First Design Patterns** — Freeman & Robson (capítulos de Strategy e Observer são excelentes)
- **[RxJS](https://rxjs.dev/)** — implementação poderosa de Observer com composição funcional
- **[XState](https://xstate.js.org/)** — biblioteca de State Machine de nível de produção para TypeScript
