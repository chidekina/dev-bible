# Padrões de Projeto: Criacionais

## 1. O que são e por que importam

Padrões criacionais lidam com a criação de objetos. Eles abstraem o processo de instanciação, tornando um sistema independente de como seus objetos são criados, compostos e representados.

Sem padrões criacionais, a lógica de criação de objetos muitas vezes vaza para o código cliente, acoplando-o a classes concretas e dificultando testes e extensão.

Os cinco padrões criacionais clássicos do livro Gang of Four (Gamma, Helm, Johnson, Vlissides — 1994):

| Padrão | Problema que resolve |
|--------|---------------------|
| **Singleton** | Garantir que existe apenas uma instância de uma classe |
| **Factory Method** | Deixar as subclasses decidirem qual classe instanciar |
| **Abstract Factory** | Criar famílias de objetos relacionados sem especificar classes concretas |
| **Builder** | Construir objetos complexos passo a passo |
| **Prototype** | Clonar objetos existentes sem depender de suas classes concretas |

> 💡 Padrões criacionais são mais valiosos quando a criação de objetos envolve lógica complexa, custo significativo de recursos (conexões de banco, rede) ou quando o tipo exato do objeto precisa variar em tempo de execução.

---

## 2. Conceitos Fundamentais

Todos os padrões criacionais compartilham um tema comum: **encapsular o conhecimento sobre quais classes o sistema usa**. Eles transferem a responsabilidade de decidir o que criar — e como — para fora do código cliente.

O trade-off principal: flexibilidade vs. complexidade. Adicionar indireção (uma factory, um builder) torna o código mais flexível, mas também adiciona mais peças móveis.

---

## 3. Como Funcionam

Cada padrão aborda um eixo específico de variação:

- **Singleton:** E se apenas uma instância deve existir?
- **Factory Method:** E se a classe exata depende de entrada em runtime?
- **Abstract Factory:** E se múltiplas classes relacionadas devem combinar entre si (ex: um tema de UI)?
- **Builder:** E se o objeto requer muitos parâmetros opcionais ou uma ordem específica de construção?
- **Prototype:** E se o novo objeto deve começar como cópia de um existente?

---

## 4. Exemplos de Código (TypeScript)

### Singleton — Pool de Conexão com Banco de Dados

Um pool de conexões deve ser criado uma vez e reutilizado. Criar múltiplos pools desperdiça recursos e pode exceder os limites de conexão.

```typescript
// src/infrastructure/database/DatabasePool.ts
import { Pool } from 'pg';

export class DatabasePool {
  private static instance: Pool | null = null;

  // Construtor privado evita instanciação externa
  private constructor() {}

  static getInstance(): Pool {
    if (!DatabasePool.instance) {
      DatabasePool.instance = new Pool({
        host: process.env.DB_HOST ?? 'localhost',
        port: Number(process.env.DB_PORT ?? 5432),
        database: process.env.DB_NAME,
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        max: 20,                // tamanho máximo do pool
        idleTimeoutMillis: 30_000,
        connectionTimeoutMillis: 2_000,
      });
    }
    return DatabasePool.instance;
  }
}

// Uso — sempre a mesma instância do pool
const pool = DatabasePool.getInstance();
const result = await pool.query('SELECT * FROM users WHERE id = $1', [userId]);
```

> ⚠️ Singleton é o padrão mais mal utilizado. Ele introduz estado global mutável, dificulta testes (os testes compartilham estado) e cria acoplamento oculto. Use-o apenas para recursos verdadeiramente compartilhados como pools de conexão, caches ou configuração. Nunca use Singleton para classes de lógica de negócio.

Uma alternativa mais segura para a maioria dos casos é **injeção de dependência** — passe a instância única a partir da raiz de composição sem que a classe imponha sua própria unicidade:

```typescript
// Prefira isso em sistemas baseados em DI:
const pool = new Pool({ ... });
// Injete 'pool' onde for necessário — ainda é uma instância, mas testável
```

---

### Factory Method — Factory de Notificações

A aplicação envia notificações por E-mail, SMS ou Push. O tipo exato depende das preferências do usuário e configuração.

```typescript
// src/domain/notifications/INotifier.ts
export interface INotifier {
  send(to: string, message: string): Promise<void>;
  readonly channel: string;
}
```

```typescript
// Implementações concretas
class EmailNotifier implements INotifier {
  readonly channel = 'email';

  async send(to: string, message: string): Promise<void> {
    console.log(`[EMAIL] para=${to}: ${message}`);
    // Real: chama SMTP ou SES
  }
}

class SmsNotifier implements INotifier {
  readonly channel = 'sms';

  async send(to: string, message: string): Promise<void> {
    console.log(`[SMS] para=${to}: ${message}`);
    // Real: chama Twilio
  }
}

class PushNotifier implements INotifier {
  readonly channel = 'push';

  async send(to: string, message: string): Promise<void> {
    console.log(`[PUSH] para=${to}: ${message}`);
    // Real: chama FCM
  }
}
```

```typescript
// src/domain/notifications/NotifierFactory.ts
export type NotificationChannel = 'email' | 'sms' | 'push';

export class NotifierFactory {
  // Factory Method — centraliza a decisão de criação
  static create(channel: NotificationChannel): INotifier {
    switch (channel) {
      case 'email': return new EmailNotifier();
      case 'sms':   return new SmsNotifier();
      case 'push':  return new PushNotifier();
      default:
        // Verificação de exhaustiveness do TypeScript
        const _exhaustive: never = channel;
        throw new Error(`Canal de notificação desconhecido: ${_exhaustive}`);
    }
  }
}

// Uso — o código cliente não importa classes concretas
const notifier = NotifierFactory.create(user.preferredChannel);
await notifier.send(user.contact, 'Seu pedido foi enviado!');
```

> 💡 A verificação de exhaustiveness (`const _exhaustive: never`) garante que o TypeScript produzirá um erro de compilação se um novo canal for adicionado ao tipo union mas não tratado no switch.

---

### Abstract Factory — Sistema de Temas de UI

Uma biblioteca de UI suporta DarkTheme e LightTheme. Cada tema deve produzir um conjunto combinado de componentes (Button, Input, Modal). Abstract Factory garante consistência — você não pode misturar acidentalmente um Button escuro com um Modal claro.

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
      onClick: (handler) => { /* registra handler */ },
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
      open: () => { isOpen = true; console.log('Modal escuro aberto'); },
      close: () => { isOpen = false; },
    };
  }
}

export class LightThemeFactory implements IThemeFactory {
  createButton(label: string): Button {
    return {
      render: () => `<button class="btn-light">${label}</button>`,
      onClick: (handler) => { /* registra handler */ },
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
      open: () => console.log('Modal claro aberto'),
      close: () => {},
    };
  }
}
```

```typescript
// Aplicação — depende de IThemeFactory, não de classes concretas
function buildCheckoutForm(theme: IThemeFactory): string {
  const emailInput = theme.createInput('Digite seu e-mail');
  const submitBtn = theme.createButton('Finalizar Compra');
  const confirmModal = theme.createModal('Pedido Confirmado', 'Obrigado!');

  return `
    ${emailInput.render()}
    ${submitBtn.render()}
    ${confirmModal.render()}
  `;
}

// Troca de tema em runtime — zero mudanças em buildCheckoutForm
const userTheme: IThemeFactory = userPrefersDark
  ? new DarkThemeFactory()
  : new LightThemeFactory();

buildCheckoutForm(userTheme);
```

---

### Builder — Builder de Requisições HTTP

O padrão Builder brilha ao construir objetos complexos com muitos parâmetros opcionais. Evita construtores telescópicos e faz o código parecer uma frase natural.

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
    if (!this.request.url) throw new Error('URL é obrigatória');
    if (!this.request.method) throw new Error('Método é obrigatório');
    return this.request as HttpRequest;
  }
}

// Uso — lê como linguagem natural
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

Outro uso comum: Query Builder de SQL.

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
    if (!this.table) throw new Error('Tabela é obrigatória');

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

// Uso
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

### Prototype — Clone Profundo de Objetos de Configuração

O padrão Prototype clona objetos existentes. Em TypeScript, é usado quando o custo de inicialização é alto (parsing, busca de dados remotos) e você quer cópias que podem ser modificadas independentemente.

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

  // Clone profundo — retorna uma nova instância independente
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
    return this.clone().config; // nunca exponha referência mutável
  }
}

// Configuração base
const baseConfig = new ConfigPrototype({
  env: 'production',
  database: { host: 'db.prod.internal', port: 5432, name: 'app', pool: { min: 2, max: 20 } },
  featureFlags: { newCheckout: false, betaDashboard: false },
  rateLimit: { windowMs: 60_000, max: 100 },
});

// Configuração de teste — clone e sobrescreva, zero mutação da base
const testConfig = baseConfig
  .withDatabase({ host: 'localhost', name: 'app_test', pool: { min: 1, max: 5 } })
  .withFeatureFlag('newCheckout', true)
  .get();
```

---

## 5. Erros Comuns e Armadilhas

| Padrão | Erro | Correção |
|--------|------|----------|
| Singleton | Usar para lógica de negócio ou serviços com estado | Use DI; injete uma instância única da raiz de composição |
| Singleton | Não resetar estado entre testes | Use injeção de dependência para que testes recebam instâncias frescas |
| Factory Method | Colocar inicialização complexa dentro da factory | Factories devem construir, não inicializar de forma assíncrona |
| Abstract Factory | Criar uma factory por classe em vez de por família | Uma factory deve criar uma família coesa de objetos |
| Builder | Retornar um builder mutável de `build()` | Congele ou faça clone profundo do resultado; o builder é para construção apenas |
| Prototype | Clone raso quando clone profundo é necessário | Use `structuredClone()` ou `JSON.parse(JSON.stringify(...))` para objetos planos; escreva métodos explícitos de clone para classes complexas |

> ⚠️ O padrão Prototype com `JSON.parse(JSON.stringify(...))` não lida com: Dates (viram strings), funções (descartadas), referências circulares (lança), valores `undefined` (descartados). Use `structuredClone()` no Node.js 17+ para a maioria dos casos, ou escreva métodos explícitos de clone.

---

## 6. Quando Usar / Não Usar

| Padrão | Use quando | Evite quando |
|--------|-----------|-------------|
| Singleton | Um único recurso compartilhado (pool, cache, config) | Lógica de negócio; testabilidade é uma preocupação |
| Factory Method | A classe exata varia em runtime | Existe apenas uma classe concreta |
| Abstract Factory | Famílias de objetos relacionados devem combinar | Apenas um ou dois objetos não relacionados |
| Builder | Muitos parâmetros opcionais; ordem complexa de construção | Objetos simples com 2–3 campos |
| Prototype | Copiar objetos caros de inicializar | Objetos são baratos de criar do zero |

---

## 7. Cenário Real

Um serviço de notificações deve entregar por E-mail, SMS ou Push dependendo das preferências do usuário e prioridade da mensagem. Uma Abstract Factory cria combinações combinadas de notificador + logger + handler de retry por canal.

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
      rateLimiter: new PerMinuteRateLimiter(10),   // 10/min para e-mail
      retryPolicy: new ExponentialBackoffPolicy(3),
    };
  }
}

class SmsComponentFactory implements INotificationComponentFactory {
  createComponents(): NotificationComponents {
    return {
      notifier: new SmsNotifier(),
      rateLimiter: new PerMinuteRateLimiter(3),    // 3/min para SMS (custo)
      retryPolicy: new LinearBackoffPolicy(2),
    };
  }
}
```

---

## 8. Perguntas de Entrevista

**Q1: Qual é a diferença entre Factory Method e Abstract Factory?**

R: Factory Method é um único método (ou classe) que cria um tipo de objeto, deixando a decisão de qual subclasse instanciar para o runtime. Abstract Factory cria *famílias* de objetos relacionados — todos os produtos de uma factory são projetados para trabalhar juntos. Use Factory Method quando precisar de um único ponto de criação flexível; use Abstract Factory quando precisar de um conjunto consistente de objetos relacionados.

---

**Q2: Por que Singleton é considerado um anti-padrão por muitos desenvolvedores?**

R: Porque introduz estado global mutável, torna impossível injetar test doubles, cria acoplamento oculto (qualquer classe pode chamar `getInstance()` sem declarar sua dependência) e torna os testes concorrentes não confiáveis. O comportamento de instância única é melhor alcançado criando uma instância na raiz de composição e injetando-a.

---

**Q3: Quando você escolheria Builder em vez de um construtor com muitos parâmetros?**

R: Quando uma classe tem mais de 3–4 parâmetros (especialmente opcionais), um construtor telescópico se torna ilegível (`new User(name, email, null, null, true, false, 'admin')`). O Builder torna parâmetros opcionais explícitos e lê como linguagem natural. Builder também é apropriado quando a construção deve seguir uma ordem específica ou validar estado parcial antes de concluir.

---

**Q4: Qual é a diferença entre Prototype e um copy constructor?**

R: Um copy constructor é um construtor que recebe uma instância da mesma classe e a copia. Prototype é um padrão onde o objeto sabe como clonar a si mesmo via método `clone()`. A principal vantagem do Prototype é que o chamador não precisa conhecer a classe concreta — ele simplesmente chama `clone()` em qualquer objeto que tiver, alcançando clonagem polimórfica.

---

**Q5: Como o padrão Factory Method suporta o Open/Closed Principle?**

R: O factory method centraliza a decisão de criação. Quando uma nova classe é adicionada, você adiciona um novo ramo na factory (ou uma nova subclasse de factory) sem tocar no código cliente que usa os objetos criados. O cliente está fechado para modificação; a factory é o ponto de extensão.

---

**Q6: Como implementar um Singleton thread-safe em TypeScript?**

R: TypeScript roda em um event loop single-threaded (Node.js), então preocupações tradicionais de threading não se aplicam. Porém, inicialização assíncrona ainda pode causar condições de corrida. Use uma constante no nível do módulo (o sistema de módulos faz cache do resultado) ou lazy initialization com uma promise:

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

**Q7: Qual é o papel do `director` no padrão Builder e quando você precisa dele?**

R: Um Director é uma classe opcional que conhece uma sequência específica de chamadas do builder para produzir uma configuração comum. Por exemplo, um `ReportDirector` pode chamar `builder.setHeader().setBody().setFooter().setPageNumbers()` na ordem correta. O Director encapsula receitas comuns de construção. É útil quando você tem múltiplas configurações comuns de produto que são construídas da mesma forma toda vez.

---

## 9. Exercícios

**Exercício 1: Singleton — logger**

Implemente um `Logger` singleton com métodos `info()`, `warn()` e `error()` que escrevem em um array `logs: string[]`. Então refatore para aceitar o array de logs via injeção no construtor, para que testes possam passar um array fresco.

*Dica: Compare como a testabilidade muda ao trocar de `Logger.getInstance()` para `Logger` injetado.*

---

**Exercício 2: Factory Method — processador de pagamento**

Implemente um `PaymentProcessorFactory` que cria `StripeProcessor`, `PayPalProcessor` ou `PixProcessor` baseado em uma string. Cada processador tem um método `process(amount: number): Promise<string>`. Adicione verificação de exhaustiveness do TypeScript.

*Dica: Use um union type para o parâmetro de canal e um guard `never` no caso default.*

---

**Exercício 3: Builder — perfil de usuário**

Construa um `UserProfileBuilder` para um objeto com: `id`, `name`, `email`, `role` (opcional, default `'user'`), `permissions` (opcional, default `[]`), `avatar` (opcional), `bio` (opcional). O método `build()` deve validar que `id`, `name` e `email` estão presentes.

*Dica: Use method chaining (retorne `this`). O builder deve ser impossível de construir sem os campos obrigatórios.*

---

**Exercício 4: Abstract Factory — geradores de relatório**

Crie um sistema de geração de relatórios com duas factories: `CsvReportFactory` e `JsonReportFactory`. Cada factory cria um `IRowFormatter` e `IHeaderFormatter` combinados. O código cliente que gera o relatório deve depender apenas da interface da factory.

*Dica: A função de renderização do relatório recebe `IReportFactory` como parâmetro e chama ambos os formatters.*

---

## 10. Leitura Complementar

- **Design Patterns: Elements of Reusable Object-Oriented Software** — Gamma, Helm, Johnson, Vlissides (GoF, 1994)
- **Head First Design Patterns** — Freeman & Robson (mais acessível, exemplos em Java facilmente traduzíveis)
- **[Refactoring Guru — Creational Patterns](https://refactoring.guru/design-patterns/creational-patterns)** — excelentes diagramas e exemplos
- **[TypeScript Design Patterns](https://www.typescriptlang.org/docs/handbook/2/types-from-types.html)** — tipos avançados do TypeScript usados em padrões
- **Dive Into Design Patterns** — Alexander Shvets (amostra gratuita em refactoring.guru)
