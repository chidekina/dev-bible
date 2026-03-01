# Padrões de Projeto: Estruturais

## 1. O que são e por que importam

Padrões estruturais lidam com como objetos e classes são compostos para formar estruturas maiores. Eles facilitam a construção de estruturas complexas a partir de partes simples, identificando formas simples de realizar relacionamentos entre entidades.

Os sete padrões estruturais do Gang of Four:

| Padrão | Problema que resolve |
|--------|---------------------|
| **Adapter** | Fazer uma interface incompatível funcionar com outra |
| **Bridge** | Separar uma abstração de sua implementação para que ambas possam variar |
| **Composite** | Tratar objetos individuais e composições de forma uniforme |
| **Decorator** | Adicionar comportamento a objetos dinamicamente sem subclasses |
| **Facade** | Prover uma interface simplificada para um subsistema complexo |
| **Flyweight** | Compartilhar estado para suportar eficientemente grande quantidade de objetos de baixa granularidade |
| **Proxy** | Prover um substituto que controla o acesso a outro objeto |

> 💡 Padrões estruturais são sobre composição — respondem à pergunta "como essas peças se encaixam?" em vez de "como devo criar este objeto?" ou "como esses objetos devem se comunicar?"

---

## 2. Conceitos Fundamentais

Padrões estruturais usam dois mecanismos fundamentais:
- **Padrões baseados em classes** (usam herança): Adapter (versão de classe)
- **Padrões baseados em objetos** (usam composição): Adapter (versão de objeto), Bridge, Composite, Decorator, Facade, Flyweight, Proxy

Composição é geralmente preferida a herança no design TypeScript moderno. Todo padrão estrutural aqui (exceto a forma de classe do Adapter) usa composição.

---

## 3. Como Funcionam

O mecanismo geral: um padrão estrutural introduz um objeto intermediário (adapter, decorator, proxy, facade) que fica entre o cliente e a implementação real. Esse intermediário fornece a tradução, o aprimoramento ou o controle de acesso que o padrão requer.

---

## 4. Exemplos de Código (TypeScript)

### Adapter — API de Pagamento Legada

Um sistema de checkout moderno espera `IPaymentProcessor`. Um provedor de pagamento legado tem uma API completamente diferente. O Adapter envolve a API legada e apresenta a interface esperada.

```typescript
// Interface moderna que o sistema espera
interface IPaymentProcessor {
  charge(amount: number, currency: string, cardToken: string): Promise<PaymentResult>;
  refund(transactionId: string, amount: number): Promise<void>;
}

interface PaymentResult {
  transactionId: string;
  status: 'approved' | 'declined';
  errorCode?: string;
}

// API legada (terceiro — não pode ser alterada)
class LegacyPaymentGateway {
  processPayment(params: {
    amount_cents: number;
    currency_code: string;
    card_token: string;
    merchant_id: string;
  }): { tx_id: string; result_code: '00' | '01' | '99'; message: string } {
    // Simula resposta legada
    return { tx_id: 'legacy_txn_001', result_code: '00', message: 'Aprovado' };
  }

  reverseTransaction(tx_id: string, amount_cents: number): { success: boolean } {
    return { success: true };
  }
}

// Adapter: envolve o legado, implementa a interface moderna
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
      throw new Error(`Reembolso falhou para a transação ${transactionId}`);
    }
  }
}

// CheckoutService não conhece nada sobre a API legada
class CheckoutService {
  constructor(private readonly payment: IPaymentProcessor) {}

  async processOrder(order: { total: number; currency: string; cardToken: string }): Promise<string> {
    const result = await this.payment.charge(order.total, order.currency, order.cardToken);
    if (result.status === 'declined') throw new Error(`Pagamento recusado: ${result.errorCode}`);
    return result.transactionId;
  }
}

// Wiring
const legacy = new LegacyPaymentGateway();
const adapter = new LegacyPaymentAdapter(legacy);
const checkout = new CheckoutService(adapter); // interface moderna, backend legado
```

---

### Bridge — Sistema de Notificações

Bridge separa a abstração (o que você envia) da implementação (como você envia). Ambos os lados podem variar independentemente.

```typescript
// Interface de implementação (como enviar)
interface IMessageSender {
  send(destination: string, content: string): Promise<void>;
}

// Implementações concretas
class EmailSender implements IMessageSender {
  async send(destination: string, content: string): Promise<void> {
    console.log(`[EMAIL] para=${destination}: ${content}`);
  }
}

class SmsSender implements IMessageSender {
  async send(destination: string, content: string): Promise<void> {
    console.log(`[SMS] para=${destination}: ${content}`);
  }
}

// Abstração (o que você envia) — mantém referência à implementação
abstract class Notification {
  constructor(protected sender: IMessageSender) {}

  abstract send(recipient: { email: string; phone: string }): Promise<void>;
}

// Abstrações refinadas
class OrderConfirmationNotification extends Notification {
  constructor(sender: IMessageSender, private readonly orderId: string) {
    super(sender);
  }

  async send(recipient: { email: string; phone: string }): Promise<void> {
    const content = `Seu pedido #${this.orderId} foi confirmado.`;
    await this.sender.send(recipient.email, content);
  }
}

class PasswordResetNotification extends Notification {
  constructor(sender: IMessageSender, private readonly resetLink: string) {
    super(sender);
  }

  async send(recipient: { email: string; phone: string }): Promise<void> {
    const content = `Redefina sua senha: ${this.resetLink}`;
    await this.sender.send(recipient.phone, content);
  }
}

// Misture qualquer tipo de notificação com qualquer sender — 2×2 sem criar 4 subclasses
const emailOrder = new OrderConfirmationNotification(new EmailSender(), 'ORD-001');
const smsReset = new PasswordResetNotification(new SmsSender(), 'https://app.com/reset/abc');
```

> 💡 A insight central: sem Bridge, você precisaria de `EmailOrderConfirmation`, `SmsOrderConfirmation`, `EmailPasswordReset`, `SmsPasswordReset` — quatro classes para dois eixos de variação. Bridge reduz para 2 + 2 = 4 classes em vez de 2×2 = 4 (com cada eixo adicional sendo aditivo, não multiplicativo).

---

### Composite — Sistema de Arquivos

O padrão Composite permite tratar objetos individuais e composições de forma uniforme. Um `File` e um `Directory` ambos implementam `FileSystemItem`, então o código que percorre a árvore não precisa distinguir entre eles.

```typescript
// Interface do componente
interface FileSystemItem {
  name: string;
  size(): number;
  print(indent?: string): void;
}

// Leaf — sem filhos
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

// Composite — tem filhos
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

// Uso — o cliente trata arquivos e diretórios de forma idêntica
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

### Decorator — Wrappers de Cache e Logging

O padrão Decorator adiciona comportamento a um objeto dinamicamente sem mudar a classe. É o equivalente em runtime da herança — combinável e reversível.

```typescript
interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

// Repositório base
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

// Decorator de Cache
class CachedUserRepository implements IUserRepository {
  private cache = new Map<string, User>();

  constructor(private readonly inner: IUserRepository) {}

  async findById(id: string): Promise<User | null> {
    if (this.cache.has(id)) {
      console.log(`[CACHE] HIT para id=${id}`);
      return this.cache.get(id)!;
    }
    const user = await this.inner.findById(id);
    if (user) this.cache.set(id, user);
    return user;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.inner.findByEmail(email); // sem cache para buscas por e-mail
  }

  async save(user: User): Promise<void> {
    await this.inner.save(user);
    this.cache.set(user.id, user); // atualiza cache na escrita
  }
}

// Decorator de Logging
class LoggedUserRepository implements IUserRepository {
  constructor(
    private readonly inner: IUserRepository,
    private readonly logger: { info(msg: string): void },
  ) {}

  async findById(id: string): Promise<User | null> {
    const start = Date.now();
    const result = await this.inner.findById(id);
    this.logger.info(`findById(${id}) → ${result ? 'encontrado' : 'null'} [${Date.now() - start}ms]`);
    return result;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.inner.findByEmail(email);
  }

  async save(user: User): Promise<void> {
    await this.inner.save(user);
    this.logger.info(`save(${user.id}) concluído`);
  }
}

// Componha decorators — a ordem importa: Logged envolve Cached envolve Postgres
const repo: IUserRepository = new LoggedUserRepository(
  new CachedUserRepository(
    new PostgresUserRepository()
  ),
  console,
);
```

> ⚠️ Decorators devem implementar a mesma interface que a classe envolvida. Se a interface mudar, todos os decorators precisam ser atualizados. Mantenha interfaces enxutas (ISP) para reduzir esse ônus.

---

### Facade — Facade de Pagamento

Uma Facade provê uma interface simplificada para um subsistema complexo. Aqui, realizar um pedido envolve um gateway de pagamento, detecção de fraude e geração de fatura — a facade esconde essa complexidade.

```typescript
// Subsistemas complexos
class PaymentGateway {
  async charge(customerId: string, amount: number): Promise<string> {
    console.log(`[Gateway] Cobrando ${amount} do cliente ${customerId}`);
    return 'txn_' + Date.now();
  }
}

class FraudDetectionService {
  async check(customerId: string, amount: number): Promise<boolean> {
    console.log(`[Fraude] Verificando cliente ${customerId} para valor ${amount}`);
    return amount < 10_000; // regra simples: sinalizar valores >= R$ 10k
  }
}

class InvoiceService {
  async generate(customerId: string, transactionId: string, amount: number): Promise<string> {
    const invoiceId = `INV-${Date.now()}`;
    console.log(`[Fatura] Gerada ${invoiceId} para txn ${transactionId}`);
    return invoiceId;
  }
}

class EmailService {
  async sendReceipt(email: string, invoiceId: string): Promise<void> {
    console.log(`[Email] Recibo da fatura ${invoiceId} enviado para ${email}`);
  }
}

// Facade — único ponto de entrada para o fluxo de checkout
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

    // 1. Verificação de fraude
    const isSafe = await this.fraud.check(customerId, amount);
    if (!isSafe) throw new Error('Pagamento sinalizado como potencialmente fraudulento');

    // 2. Cobrança
    const transactionId = await this.gateway.charge(customerId, amount);

    // 3. Fatura
    const invoiceId = await this.invoicing.generate(customerId, transactionId, amount);

    // 4. E-mail de recibo
    await this.email.sendReceipt(customerEmail, invoiceId);

    return { transactionId, invoiceId };
  }
}

// Código cliente — uma chamada de método, zero conhecimento dos subsistemas
const facade = new PaymentFacade();
const result = await facade.processPayment({
  customerId: 'cust_1',
  customerEmail: 'alice@example.com',
  amount: 299.99,
});
```

---

### Flyweight — Renderização de Caracteres

O padrão Flyweight reduz o uso de memória compartilhando estado comum (intrínseco) entre muitos objetos, mantendo o estado único (extrínseco) fora.

```typescript
// Estado intrínseco — compartilhado entre muitos caracteres (fonte, tamanho, estilo)
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
    // Renderização com estilo compartilhado + posição única
    console.log(
      `Renderiza '${char}' em (${x},${y}) — ${this.style.font} ${this.style.size}pt ${this.style.color}`
    );
  }
}

// Flyweight Factory — garante instâncias compartilhadas
class CharacterFlyweightFactory {
  private cache = new Map<string, CharacterFlyweight>();

  getOrCreate(style: CharacterStyle): CharacterFlyweight {
    const key = `${style.font}-${style.size}-${style.bold}-${style.italic}-${style.color}`;
    if (!this.cache.has(key)) {
      this.cache.set(key, new CharacterFlyweight(style));
      console.log(`[Flyweight] Novo estilo criado: ${key}`);
    }
    return this.cache.get(key)!;
  }

  get cachedCount(): number { return this.cache.size; }
}

// Estado extrínseco — único por instância de caractere (valor, posição)
interface CharacterInstance {
  char: string;
  x: number;
  y: number;
  flyweight: CharacterFlyweight;
}

// Documento com 10.000 caracteres — mas apenas poucos estilos únicos
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

### Proxy — Lazy Loading, Cache e Controle de Acesso

O Proxy controla o acesso ao objeto real. Três casos de uso comuns:

```typescript
// 1. Proxy de lazy loading — inicializa o objeto real apenas quando necessário
interface IReportGenerator {
  generate(params: Record<string, unknown>): Promise<string>;
}

class HeavyReportGenerator implements IReportGenerator {
  constructor() {
    // Inicialização cara (modelo de ML, grande carga de banco, etc.)
    console.log('[ReportGenerator] Inicializado (caro)');
  }

  async generate(params: Record<string, unknown>): Promise<string> {
    return `Relatório gerado com params: ${JSON.stringify(params)}`;
  }
}

class LazyReportProxy implements IReportGenerator {
  private real: HeavyReportGenerator | null = null;

  async generate(params: Record<string, unknown>): Promise<string> {
    if (!this.real) {
      this.real = new HeavyReportGenerator(); // inicializado apenas no primeiro uso
    }
    return this.real.generate(params);
  }
}

// 2. Proxy de cache
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

// 3. Proxy de controle de acesso
class AuthorizedReportProxy implements IReportGenerator {
  constructor(
    private readonly inner: IReportGenerator,
    private readonly requiredRole: string,
  ) {}

  async generate(params: Record<string, unknown> & { userRole?: string }): Promise<string> {
    if (params.userRole !== this.requiredRole) {
      throw new Error(`Acesso negado: requer role '${this.requiredRole}'`);
    }
    return this.inner.generate(params);
  }
}

// Componha proxies
const generator: IReportGenerator = new AuthorizedReportProxy(
  new CachingReportProxy(
    new LazyReportProxy()
  ),
  'admin',
);
```

---

## 5. Erros Comuns e Armadilhas

| Padrão | Erro | Correção |
|--------|------|----------|
| Adapter | Adaptar lógica de negócio, não apenas a interface | Adapter deve apenas traduzir, não adicionar lógica |
| Decorator | Esquecer de delegar todos os métodos da interface | Implemente todos os métodos; delegações ausentes causam bugs silenciosos |
| Facade | Tornar a facade um god object com lógica de negócio | Facade apenas coordena — lógica de negócio vive nos subsistemas |
| Proxy | Usar Proxy onde Decorator seria mais apropriado | Proxy controla acesso; Decorator adiciona comportamento |
| Composite | Permitir `add()`/`remove()` na interface do leaf | Nós leaf não devem expor métodos de gerenciamento de filhos |
| Flyweight | Compartilhar estado mutável | Compartilhe apenas estado imutável (intrínseco); estado extrínseco fica fora |

> ⚠️ Decorator e Proxy parecem similares (ambos envolvem um objeto implementando a mesma interface), mas servem propósitos diferentes. Decorator adiciona comportamento; Proxy controla acesso. Na prática, a distinção pode se tornar tênue — foque na intenção ao nomeá-los.

---

## 6. Quando Usar / Não Usar

| Padrão | Use quando | Evite quando |
|--------|-----------|-------------|
| Adapter | Integrando APIs legadas ou de terceiros | A interface pode ser alterada diretamente |
| Bridge | Duas dimensões de variação precisam evoluir independentemente | Apenas uma dimensão varia |
| Composite | Estruturas em árvore (menus, sistemas de arquivos, layouts de UI) | Coleções planas com elementos uniformes |
| Decorator | Adicionando comportamentos opcionais e combináveis em runtime | Um conjunto fixo de comportamentos — mais simples como subclasses |
| Facade | Escondendo um subsistema complexo atrás de um único entry point | O subsistema é simples ou já bem encapsulado |
| Flyweight | Milhares de objetos similares com dados compartilhados | Quantidade de objetos é pequena; memória não é uma preocupação |
| Proxy | Lazy initialization, cache, controle de acesso, logging | Acesso direto é mais simples e o overhead é desnecessário |

---

## 7. Cenário Real

Um serviço que busca perfis de usuários aplica três padrões estruturais simultaneamente:

```typescript
// Base: PostgresUserRepository
// Decorado: + cache
// Decorado: + logging
// Protegido por: proxy de controle de acesso

const repo = new LoggedUserRepository(
  new CachedUserRepository(
    new PostgresUserRepository(prisma)
  ),
  logger,
);

// Controle de acesso em nível de serviço via Proxy
const protectedRepo = new AccessControlledUserRepository(repo, currentUser);
```

Cada camada tem uma única preocupação. A implementação PostgreSQL não sabe nada sobre cache. O cache não sabe nada sobre logging. O proxy não sabe nada sobre nenhum dos dois. Eles são compostos no ponto de wiring.

---

## 8. Perguntas de Entrevista

**Q1: Qual é a diferença entre Adapter e Facade?**

R: Adapter faz uma interface funcionar com outra — é sobre compatibilidade entre duas interfaces existentes. Facade cria uma nova interface simplificada sobre um subsistema complexo — é sobre esconder complexidade. Adapter não muda nada; Facade simplifica tudo. Você também pode usar ambos: adaptar APIs externas primeiro, depois criar uma facade sobre o resultado.

---

**Q2: Como Decorator difere de herança?**

R: Herança é estática (decidida em tempo de compilação) e se aplica a todas as instâncias da subclasse. Decorator é dinâmico (composto em runtime) e se aplica apenas ao objeto específico que você envolve. Você pode empilhar múltiplos decorators, e a ordem importa. Você não consegue essa flexibilidade com herança sem criar uma nova subclasse para cada combinação.

---

**Q3: Quando você usaria Bridge em vez de simplesmente usar uma interface?**

R: Ambos desacoplam abstração de implementação, mas Bridge torna ambos os lados independentemente extensíveis. Com uma interface simples, você pode adicionar novas implementações (novos senders), mas adicionar uma nova abstração (novo tipo de notificação) pode forçar mudanças em todas as implementações existentes. Bridge usa composição para que ambas as hierarquias possam crescer sem afetar uma a outra.

---

**Q4: Qual é o risco de decorators profundamente aninhados?**

R: Cadeias profundas de decorator são difíceis de depurar — um stack trace pode mostrar 5 níveis de wrappers antes de chegar à lógica real. Também, se um decorator não delega todos os métodos corretamente, o comportamento quebra silenciosamente. Mantenha cadeias curtas (2–3 níveis), nomeie cada decorator descritivamente, e considere uma revisão estrutural se tiver mais de 3 camadas.

---

**Q5: Como Composite suporta o Open/Closed Principle?**

R: Novos tipos de leaf (novos tipos de arquivo, novos componentes de UI) podem ser adicionados sem mudar o Composite ou qualquer código cliente que use `FileSystemItem`. A lógica de percurso da árvore funciona com a interface, então está fechada para modificação e aberta para extensão.

---

**Q6: Proxy vs Decorator — como decidir?**

R: Pergunte sobre a intenção. Se você está adicionando novo comportamento (logging, cache) para aprimorar o objeto, use Decorator. Se você está controlando acesso, adiando inicialização ou adicionando preocupações transversais de infraestrutura (auth, rate limiting, circuit breaking) sem o conhecimento do cliente, use Proxy. A diferença estrutural é mínima; a diferença semântica é clara.

---

**Q7: Em que cenário Flyweight é essencial, não apenas agradável de ter?**

R: Em engines de renderização (editores de texto, game engines, renderers de mapas) onde dezenas de milhares de objetos similares existem simultaneamente. Um editor de texto com um documento de 100.000 caracteres precisa de 100.000 objetos de caractere — se cada um armazena fonte, tamanho, cor e estilo, isso é enorme desperdício de memória. Flyweight compartilha o objeto de estilo, reduzindo o uso de memória em ordens de magnitude.

---

## 9. Exercícios

**Exercício 1: Adapter — API de câmbio**

Uma classe legada `CurrencyConverter` tem um método `convertAmount(from: string, to: string, amt: number): number`. Seu sistema espera `ICurrencyService` com `convert(amount: Money, targetCurrency: string): Promise<Money>`. Escreva o adapter.

---

**Exercício 2: Decorator — rate limiting**

Adicione um decorator `RateLimitedUserRepository` que permite no máximo 100 chamadas `findById` por minuto. Após o limite, lance `RateLimitExceededError`. O repositório subjacente não deve saber sobre rate limiting.

---

**Exercício 3: Composite — sistema de menu de UI**

Modele um menu de navegação com `MenuItem` (leaf) e `MenuGroup` (composite). Ambos implementam `IMenuComponent` com `render(): string` e `isVisible(): boolean`. Um `MenuGroup` renderiza apenas seus filhos visíveis.

---

**Exercício 4: Proxy — circuit breaker**

Implemente um `CircuitBreakerProxy` envolvendo `IPaymentProcessor`. Após 3 falhas consecutivas, entra no estado "open" e falha rapidamente por 30 segundos antes de permitir uma tentativa. Rastreie o estado (`closed`, `open`, `half-open`).

---

## 10. Leitura Complementar

- **Design Patterns: Elements of Reusable Object-Oriented Software** — Gamma, Helm, Johnson, Vlissides
- **[Refactoring Guru — Structural Patterns](https://refactoring.guru/design-patterns/structural-patterns)** — exemplos ilustrados
- **[TypeScript Decorator Pattern](https://refactoring.guru/design-patterns/decorator/typescript/example)** — exemplos TypeScript do Refactoring Guru
- **Patterns of Enterprise Application Architecture** — Martin Fowler (Proxy e Facade em contexto empresarial)
