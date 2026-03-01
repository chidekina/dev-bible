# Princípios SOLID

## 1. O que são e por que importam

SOLID é um acrônimo para cinco princípios de design introduzidos por Robert C. Martin ("Uncle Bob") que guiam o design de software orientado a objetos. Esses princípios ajudam desenvolvedores a construir sistemas fáceis de manter, estender e testar ao longo do tempo.

| Letra | Princípio | Resumo em uma linha |
|-------|-----------|---------------------|
| **S** | Single Responsibility | Uma classe deve ter um, e apenas um, motivo para mudar |
| **O** | Open/Closed | Aberta para extensão, fechada para modificação |
| **L** | Liskov Substitution | Subtipos devem poder substituir seus tipos base |
| **I** | Interface Segregation | Nenhum cliente deve ser forçado a depender de métodos que não usa |
| **D** | Dependency Inversion | Dependa de abstrações, não de implementações concretas |

Por que isso importa na prática? Software que viola esses princípios tende a exibir modos de falha comuns: classes que quebram quando você mexe em funcionalidades não relacionadas, cadeias de `if/else` que crescem a cada sprint, testes que exigem configurar metade da aplicação e hierarquias de herança que silenciosamente violam contratos.

> 💡 Os princípios SOLID são diretrizes, não leis. Aplique-os onde reduzem complexidade — não cegamente em toda situação. Um script de dois arquivos não precisa de SOLID; um módulo de processamento de pagamentos usado por 50 engenheiros, sim.

---

## 2. Conceitos Fundamentais

Cada princípio ataca um eixo específico de mudança:

- **SRP** ataca a *coesão* — manter lógica relacionada junta e lógica não relacionada separada.
- **OCP** ataca a *extensibilidade* — adicionar comportamento sem quebrar o código existente.
- **LSP** ataca a *correção* — garantir que o polimorfismo funciona como esperado.
- **ISP** ataca o *acoplamento* — evitar que consumidores dependam de contratos que não usam.
- **DIP** ataca a *flexibilidade* — desacoplar a política de alto nível da implementação de baixo nível.

Juntos, eles formam um ciclo de feedback: SRP cria classes pequenas, DIP as conecta, OCP permite estendê-las com segurança, LSP garante que a herança está correta e ISP mantém os contratos enxutos.

---

## 3. Como Funcionam

Os princípios operam no nível de classes e interfaces. Eles são avaliados com perguntas como:

1. Se eu mudar o requisito X, quais classes precisam mudar?
2. Posso adicionar comportamento sem tocar no código existente?
3. Posso substituir qualquer subclasse sem alterar o comportamento?
4. Esta interface contém métodos que alguns clientes jamais usam?
5. Esta classe instancia suas próprias dependências?

Cada "sim" para os problemas 1–5 indica uma violação.

---

## 4. Exemplos de Código (TypeScript)

### S — Single Responsibility Principle

**Violação:** `UserService` lida com lógica de negócio *e* envia notificações por e-mail. Dois motivos para mudar: alterações na lógica de usuário e mudanças no template de e-mail.

```typescript
// VIOLAÇÃO — duas responsabilidades em uma só classe
class UserService {
  constructor(private db: Database) {}

  async createUser(data: CreateUserDto): Promise<User> {
    const user = await this.db.users.create(data);

    // Preocupação com e-mail misturada com criação de usuário
    const html = `<h1>Bem-vindo, ${user.name}!</h1>`;
    await sendEmail({
      to: user.email,
      subject: 'Bem-vindo!',
      html,
    });

    return user;
  }

  async updateUser(id: string, data: UpdateUserDto): Promise<User> {
    const user = await this.db.users.update(id, data);

    // Novamente misturando notificação com lógica de domínio
    if (data.email) {
      await sendEmail({
        to: user.email,
        subject: 'E-mail atualizado',
        html: `<p>Seu e-mail foi alterado.</p>`,
      });
    }

    return user;
  }
}
```

**Correção:** Extraia a preocupação com e-mail para um `WelcomeEmailService` dedicado. Cada classe agora tem exatamente um motivo para mudar.

```typescript
// ✅ Single Responsibility — duas classes separadas, cada uma com uma só responsabilidade
interface EmailPayload {
  to: string;
  subject: string;
  html: string;
}

class WelcomeEmailService {
  async sendWelcome(user: User): Promise<void> {
    await sendEmail({
      to: user.email,
      subject: 'Bem-vindo!',
      html: `<h1>Bem-vindo, ${user.name}!</h1>`,
    });
  }

  async sendEmailChanged(user: User): Promise<void> {
    await sendEmail({
      to: user.email,
      subject: 'E-mail atualizado',
      html: `<p>Seu e-mail foi alterado.</p>`,
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

> 💡 Uma heurística prática: se você precisa usar "e" ao descrever o que uma classe faz ("ela cria usuários **e** envia e-mails"), provavelmente viola o SRP.

---

### O — Open/Closed Principle

**Violação:** Uma calculadora de descontos usa uma cadeia crescente de `if/else`. Cada novo tipo de desconto exige modificar esta classe — arriscando quebrar a lógica existente.

```typescript
// VIOLAÇÃO — é preciso modificar esta função para adicionar novos tipos de desconto
function calculateDiscount(order: Order, discountType: string): number {
  if (discountType === 'percentage') {
    return order.total * 0.1;
  } else if (discountType === 'fixed') {
    return 10;
  } else if (discountType === 'bogo') {
    return order.total / 2;
  } else if (discountType === 'loyalty') {
    // Adicionado depois — tocou na função existente
    return order.total * 0.15 * order.loyaltyYears;
  }
  return 0;
}
```

**Correção:** Defina uma interface `DiscountStrategy`. Adicionar um novo tipo de desconto significa adicionar uma nova classe — zero mudanças no código existente.

```typescript
// ✅ Open/Closed — estenda via novas classes, nunca modifique as existentes
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

// A calculadora nunca muda — está fechada para modificação
class OrderPricer {
  constructor(private strategy: DiscountStrategy) {}

  finalPrice(order: Order): number {
    const discount = this.strategy.calculate(order);
    return Math.max(0, order.total - discount);
  }
}

// Uso
const pricer = new OrderPricer(new LoyaltyDiscount());
const price = pricer.finalPrice(order);
```

> ⚠️ OCP não significa "nunca mude um arquivo". Significa que novo comportamento deve ser expresso como novo código, não como modificações em lógica estável e testada.

---

### L — Liskov Substitution Principle

**Violação:** Um `Square` estende `Rectangle` e sobrescreve `setWidth`/`setHeight` para forçar lados iguais. Código que funciona com `Rectangle` quebra silenciosamente quando recebe um `Square`.

```typescript
// VIOLAÇÃO — Square quebra o contrato de Rectangle
class Rectangle {
  constructor(protected width: number, protected height: number) {}

  setWidth(w: number): void { this.width = w; }
  setHeight(h: number): void { this.height = h; }
  area(): number { return this.width * this.height; }
}

class Square extends Rectangle {
  // Força lados iguais — viola o contrato de Rectangle
  setWidth(w: number): void {
    this.width = w;
    this.height = w; // mutação silenciosa
  }

  setHeight(h: number): void {
    this.width = h;
    this.height = h; // mutação silenciosa
  }
}

// Esta função espera o comportamento de Rectangle
function testRectangle(rect: Rectangle): void {
  rect.setWidth(5);
  rect.setHeight(4);
  // Espera 20, recebe 16 com Square — violação de LSP!
  console.assert(rect.area() === 20, 'Área deveria ser 20');
}

testRectangle(new Square(10)); // assertion falha silenciosamente
```

**Correção:** Não modele a relação geométrica "quadrado é um retângulo" no código. Use classes separadas e independentes com uma interface `Shape` comum.

```typescript
// ✅ Liskov Substitution — classes separadas, interface compartilhada
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

// Qualquer Shape funciona aqui — a substituição é segura
function printShapeInfo(shape: Shape): void {
  console.log(`Área: ${shape.area()}, Perímetro: ${shape.perimeter()}`);
}

printShapeInfo(new Rectangle(5, 4)); // Área: 20, Perímetro: 18
printShapeInfo(new Square(5));       // Área: 25, Perímetro: 20
```

> 💡 LSP é sobre compatibilidade comportamental, não apenas de tipos. Dois tipos podem compartilhar uma interface e ainda assim violar o LSP se um quebra silenciosamente as suposições que o outro garante.

---

### I — Interface Segregation Principle

**Violação:** Uma interface `IAnimal` "gorda" força todos os animais a implementar métodos que não se aplicam a eles (ex: `fly()` em um `Dog`).

```typescript
// VIOLAÇÃO — todo Animal deve implementar todos os métodos, mesmo os irrelevantes
interface IAnimal {
  eat(): void;
  sleep(): void;
  fly(): void;    // Nem todos os animais voam
  swim(): void;   // Nem todos os animais nadam
  run(): void;    // Nem todos os animais correm
}

class Dog implements IAnimal {
  eat(): void { console.log('comendo'); }
  sleep(): void { console.log('dormindo'); }
  run(): void { console.log('correndo'); }
  fly(): void { throw new Error('Cachorros não voam!'); }  // violação!
  swim(): void { console.log('nadando de cachorrinho'); }
}

class Eagle implements IAnimal {
  eat(): void { console.log('comendo'); }
  sleep(): void { console.log('dormindo'); }
  fly(): void { console.log('planando'); }
  swim(): void { throw new Error('Águias não nadam!'); } // violação!
  run(): void { console.log('correndo'); }
}
```

**Correção:** Divida em interfaces focadas. As classes implementam apenas as capacidades que realmente possuem.

```typescript
// ✅ Interface Segregation — interfaces granulares e combináveis
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

// Dog: pode comer, dormir, correr, nadar — mas não voar
class Dog implements ILivingThing, IRunnable, ISwimmable {
  eat(): void { console.log('comendo'); }
  sleep(): void { console.log('dormindo'); }
  run(): void { console.log('correndo a 48 km/h'); }
  swim(): void { console.log('nadando de cachorrinho'); }
  divingDepth(): number { return 0.5; }
}

// Eagle: pode comer, dormir, correr, voar — mas não nadar
class Eagle implements ILivingThing, IRunnable, IFlyable {
  eat(): void { console.log('comendo'); }
  sleep(): void { console.log('dormindo'); }
  run(): void { console.log('correndo'); }
  fly(): void { console.log('planando em altitude'); }
  landingSpeed(): number { return 35; }
}

// Duck: todas as capacidades
class Duck implements ILivingThing, IRunnable, IFlyable, ISwimmable {
  eat(): void { console.log('comendo'); }
  sleep(): void { console.log('dormindo'); }
  run(): void { console.log('caminhando como pato'); }
  fly(): void { console.log('voando para o sul'); }
  swim(): void { console.log('flutuando'); }
  landingSpeed(): number { return 15; }
  divingDepth(): number { return 1.2; }
}

// Funções dependem apenas da capacidade que precisam
function makeItFly(flyer: IFlyable): void {
  flyer.fly();
  console.log(`Velocidade de pouso: ${flyer.landingSpeed()} km/h`);
}
```

> 💡 ISP é especialmente importante em TypeScript porque as interfaces são estruturais. Interfaces enxutas também tornam o mock em testes trivial — você só precisa implementar o que o teste exercita.

---

### D — Dependency Inversion Principle

**Violação:** `UserService` instancia diretamente `MySQLUserRepository`. Está fortemente acoplado a uma implementação concreta de banco de dados — você não pode trocá-la, e os testes exigem um MySQL real.

```typescript
// VIOLAÇÃO — módulo de alto nível instancia módulo de baixo nível
class MySQLUserRepository {
  async findById(id: string): Promise<User | null> {
    // Query direta no MySQL
    const row = await mysql.query('SELECT * FROM users WHERE id = ?', [id]);
    return row ? mapToUser(row) : null;
  }

  async save(user: User): Promise<void> {
    await mysql.query('INSERT INTO users ...', [...]);
  }
}

class UserService {
  // Cria sua própria dependência — não pode ser testado sem MySQL
  private repo = new MySQLUserRepository();

  async getUser(id: string): Promise<User | null> {
    return this.repo.findById(id);
  }
}
```

**Correção:** Defina uma interface `IUserRepository`. `UserService` depende da abstração. A implementação concreta é injetada de fora.

```typescript
// ✅ Dependency Inversion — dependa de abstrações
interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}

// Módulo de baixo nível implementa a abstração
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

// Módulo de alto nível depende apenas da interface
class UserService {
  constructor(private repo: IUserRepository) {}

  async getUser(id: string): Promise<User | null> {
    return this.repo.findById(id);
  }

  async deactivateUser(id: string): Promise<void> {
    const user = await this.repo.findById(id);
    if (!user) throw new Error('Usuário não encontrado');
    user.deactivate();
    await this.repo.save(user);
  }
}

// Em produção — conecta a implementação real
const service = new UserService(new MySQLUserRepository());

// Em testes — conecta o fake em memória, sem precisar de MySQL
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

// Teste — zero infraestrutura
const repo = new InMemoryUserRepository();
const testService = new UserService(repo);
```

---

## 5. Erros Comuns e Armadilhas

| Erro | Por que está errado | Correção |
|------|---------------------|----------|
| Criar um `BaseService` que todos os serviços herdam | Força acoplamento a um pai compartilhado, viola SRP e LSP | Use composição e interfaces |
| Uma interface por classe com formato idêntico | Abstração sem propósito — não é DIP | Extraia interfaces só quando houver múltiplas implementações ou necessidade de testabilidade |
| Aplicar OCP a valores de configuração | Mudanças de config não são novo comportamento — são dados | Use arquivos de config, não classes strategy |
| `IRepository` gigante com 30 métodos | Violação de ISP — a maioria dos consumidores usa 3–4 | Divida ou use tipos parciais nos consumidores |
| LSP: lançar `NotImplemented` em overrides | Quebra o contrato comportamental | Não herde se não puder cumprir o contrato |
| Injetar o próprio container de DI | O container vira service locator, escondendo dependências | Injete as dependências reais, não o container |

> ⚠️ Engenharia excessiva em nome do SOLID é um risco real. Aplicar todos os cinco princípios a um `HealthCheckController` com um único método é desperdício. Use-os onde proporcionam valor mensurável.

---

## 6. Quando Usar / Não Usar

**Aplique SOLID com rigor quando:**
- Estiver construindo módulos consumidos por múltiplos times
- O domínio é estável, mas as implementações mudam (bancos de dados, APIs de terceiros, canais de notificação)
- Você precisa de testes unitários rápidos e isolados
- O time tem mais de ~5 engenheiros no mesmo codebase

**Relaxe o SOLID quando:**
- Estiver prototipando ou construindo um MVP — velocidade importa mais que extensibilidade
- A classe é um mapeador de dados trivial e sem lógica
- O custo da abstração (novos arquivos, novas interfaces) claramente supera o benefício
- Você está escrevendo código de "cola" (scripts de CLI, migrações pontuais)

---

## 7. Cenário Real: Adicionando um Novo Provedor de Pagamento

Uma plataforma de e-commerce começa com Stripe. Com o crescimento, precisa suportar PayPal e Pix. Um design SOLID torna isso uma tarefa de 15 minutos.

```typescript
// Abstração definida uma única vez
interface IPaymentGateway {
  charge(amount: number, currency: string, customerId: string): Promise<PaymentResult>;
  refund(transactionId: string, amount: number): Promise<RefundResult>;
  getStatus(transactionId: string): Promise<PaymentStatus>;
}

// Cada provedor é isolado — sem estado compartilhado, sem contaminação cruzada
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
  // Novo provedor — zero mudanças no código existente
  async charge(amount: number, currency: string, customerId: string): Promise<PaymentResult> {
    // Lógica específica do PayPal
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
  // Pagamentos instantâneos brasileiros — adicionado sem tocar em Stripe ou PayPal
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

// Serviço de alto nível — nunca muda, independente de quantos gateways forem adicionados
class CheckoutService {
  constructor(private gateway: IPaymentGateway) {}

  async processOrder(order: Order): Promise<PaymentResult> {
    return this.gateway.charge(order.total, order.currency, order.customerId);
  }
}

// Conectado na raiz de composição (ex: container de DI ou main.ts)
const gateway = resolveGateway(userPreference); // 'stripe' | 'paypal' | 'pix'
const checkout = new CheckoutService(gateway);
```

Adicionar Pix exigiu: 1 novo arquivo (`PixGateway`), 1 linha na factory. Zero mudanças em `CheckoutService`, `StripeGateway` ou `PayPalGateway`.

---

## 8. Perguntas de Entrevista

**Q1: O que significa "uma classe deve ter um motivo para mudar" na prática?**

R: Significa que uma classe deve ser responsável por um único ator — um stakeholder ou parte do sistema que impulsiona mudanças. Se tanto o time de marketing (templates de e-mail) quanto o time de engenharia (lógica de persistência) podem causar mudanças na mesma classe, ela tem duas responsabilidades. Separe-as.

---

**Q2: Como SRP e OCP se relacionam?**

R: Eles se reforçam mutuamente. SRP garante que as classes são pequenas e focadas. OCP garante que você adiciona comportamento criando novas classes, em vez de modificar as existentes. Uma classe que viola o SRP é muito mais difícil de manter fechada para modificação, porque suas múltiplas responsabilidades ficam entrelaçadas.

---

**Q3: Dê um exemplo de violação de LSP que não é óbvia.**

R: Uma `ReadOnlyList` que estende `List` e lança `UnsupportedOperationException` no `add()`. Código que aceita uma `List` e chama `add()` vai falhar silenciosamente em runtime. O subtipo não honra o contrato comportamental do pai. A correção é não herdar de `List` — implemente uma interface separada `IReadableCollection`.

---

**Q4: É sempre errado ter uma interface com muitos métodos?**

R: Não. Se todos os consumidores da interface usam todos os métodos, não há problema de segregação. ISP é violado quando clientes são forçados a depender de métodos que não usam. O tamanho da interface importa menos do que se todos os seus consumidores realmente precisam de todos os seus membros.

---

**Q5: Qual é a diferença entre Dependency Injection e Dependency Inversion?**

R: São conceitos relacionados, mas distintos. Dependency Inversion é um princípio: módulos de alto nível devem depender de abstrações. Dependency Injection é uma técnica: as dependências são fornecidas de fora, em vez de instanciadas dentro da classe. DI é uma forma de alcançar DIP, mas DIP não exige DI especificamente.

---

**Q6: É possível violar LSP em TypeScript mesmo quando os tipos compilam corretamente?**

R: Sim. TypeScript verifica compatibilidade estrutural de tipos, mas não contratos comportamentais. Uma subclasse pode satisfazer o type checker enquanto viola o LSP ao: lançar exceções que a classe base nunca lança, retornar coleções vazias em vez de lançar, ou ignorar silenciosamente chamadas de métodos. LSP é uma garantia semântica, não sintática.

---

**Q7: Quando é aceitável dispensar o Dependency Inversion?**

R: Quando a implementação concreta nunca vai mudar e testá-la em isolamento não agrega valor — por exemplo, uma classe `Logger` que envolve `console.log` em um script não crítico. Também aceitável: value objects e funções puras, sem efeitos colaterais e sem necessidade de substituição.

---

**Q8: Como convencer um colega cético de que SOLID vale os arquivos e interfaces extras?**

R: Aponte para uma dor recente e específica: "Lembra quando adicionar a notificação por SMS exigiu tocar em 4 classes e quebrou os testes de e-mail? Com ISP e DIP, cada tipo de notificação seria sua própria classe implementando uma única interface `INotifier`. A próxima adição seria um único arquivo novo." Enquadre em termos de custo de mudança, não de princípios abstratos.

---

## 9. Exercícios

**Exercício 1: SRP — Divida a classe monólito**

Pegue esta classe e divida-a em classes com responsabilidades adequadas:

```typescript
class ReportGenerator {
  async generate(userId: string): Promise<void> {
    const data = await fetch(`/api/users/${userId}/orders`).then(r => r.json());
    const csv = data.map((o: Order) => `${o.id},${o.total},${o.date}`).join('\n');
    fs.writeFileSync(`report_${userId}.csv`, csv);
    await sendEmail({ to: 'admin@company.com', subject: 'Relatório', attachment: csv });
  }
}
```

*Dica: Identifique cada ator — quem motiva uma mudança na URL do fetch? No formato CSV? No caminho do arquivo? No destinatário do e-mail?*

---

**Exercício 2: OCP — Calculadora de frete**

Refatore para ser aberto para extensão:

```typescript
function getShippingCost(method: string, weight: number): number {
  if (method === 'standard') return weight * 1.5;
  if (method === 'express') return weight * 3.0 + 5;
  if (method === 'overnight') return weight * 5.0 + 20;
  return 0;
}
```

*Dica: Defina uma interface `ShippingStrategy` com um método `calculate(weight: number): number`.*

---

**Exercício 3: LSP — Encontre a violação**

Analise este código e explique a violação de LSP:

```typescript
class Bird {
  fly(): void { console.log('voando'); }
}
class Penguin extends Bird {
  fly(): void { throw new Error('Pinguins não voam'); }
}
function makeBirdFly(bird: Bird): void {
  bird.fly();
}
```

*Dica: Que suposição `makeBirdFly` faz? Essa suposição sempre pode ser mantida?*

---

**Exercício 4: DIP — Desacople o serviço**

Introduza uma interface e torne `OrderService` testável sem um banco de dados real:

```typescript
class OrderService {
  private db = new PostgresDatabase();
  async getOrder(id: string): Promise<Order | null> {
    return this.db.query('SELECT * FROM orders WHERE id = $1', [id]);
  }
}
```

*Dica: Defina `IOrderRepository` com `findById`. Escreva um `InMemoryOrderRepository` para testes.*

---

**Exercício 5: ISP — Audite a interface**

Divida esta interface com base nas necessidades reais dos consumidores:

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

*Dica: Quais métodos pertencem à condução? À segurança? Ao conforto? Crie uma interface por preocupação.*

---

## 10. Leitura Complementar

- **Clean Code** — Robert C. Martin (Capítulo 10: Classes)
- **Agile Software Development, Principles, Patterns, and Practices** — Robert C. Martin (tratamento completo do SOLID)
- **[SOLID Principles Every Developer Should Know](https://blog.bitsrc.io/solid-principles-every-developer-should-know-b3bfa96bb688)** — Bits & Pieces
- **[The Principles of OOD](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)** — artigos originais de Uncle Bob
- **Dependency Injection Principles, Practices, and Patterns** — Mark Seemann & Steven van Deursen
- TypeScript Handbook — [Interfaces](https://www.typescriptlang.org/docs/handbook/2/objects.html)
