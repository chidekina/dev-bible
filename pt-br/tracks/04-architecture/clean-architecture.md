# Clean Architecture

## 1. O que é e por que importa

A Clean Architecture, proposta por Robert C. Martin em seu livro de 2017 de mesmo nome, organiza o código em camadas concêntricas onde as camadas internas não conhecem nada sobre as externas. Ela unifica ideias anteriores — Arquitetura Hexagonal, Onion Architecture e BCE (Boundary-Control-Entity) — sob uma estrutura única e opinativa.

A promessa central: **sua lógica de negócio sobrevive a mudanças tecnológicas.** Troque Fastify por Express, PostgreSQL por MongoDB, REST por gRPC — o código central do domínio não muda. O framework é um mecanismo de entrega, não a fundação.

> 💡 A Regra de Dependência é a ideia mais importante da Clean Architecture: *as dependências do código-fonte só podem apontar para dentro.* Nada em uma camada interna pode conhecer nada de uma camada externa.

Por que isso importa?

- **Testabilidade:** Casos de uso podem ser testados com zero infraestrutura (sem banco de dados, sem servidor HTTP).
- **Independência de frameworks:** A lógica de negócio não está acoplada a Fastify, Prisma ou qualquer outra ferramenta.
- **Substituibilidade:** Trocar um banco de dados ou UI é uma tarefa mecânica, não arquitetural.
- **Longevidade:** O domínio central pode sobreviver a múltiplas gerações de frameworks.

---

## 2. Conceitos Fundamentais

### As Quatro Camadas

```
┌──────────────────────────────────────────────────────┐
│  Frameworks & Drivers (mais externa)                 │
│  ┌────────────────────────────────────────────────┐  │
│  │  Interface Adapters                            │  │
│  │  ┌──────────────────────────────────────────┐ │  │
│  │  │  Use Cases (Regras de Negócio da App)    │ │  │
│  │  │  ┌──────────────────────────────────┐    │ │  │
│  │  │  │  Entities (Regras Empresariais)  │    │ │  │
│  │  │  └──────────────────────────────────┘    │ │  │
│  │  └──────────────────────────────────────────┘ │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

| Camada | O que vive aqui | Exemplos |
|--------|----------------|---------|
| **Entities** | Regras de negócio de toda a empresa, objetos de domínio | `User`, `Order`, `Money`, domain events |
| **Use Cases** | Regras de negócio específicas da aplicação, orquestração | `CreateUserUseCase`, `PlaceOrderUseCase` |
| **Interface Adapters** | Converte dados entre use cases e formatos externos | Controllers, presenters, implementações de repositório |
| **Frameworks & Drivers** | Todas as ferramentas e frameworks externos | Fastify, Prisma, Redis, S3 SDK |

### A Regra de Dependência

As dependências fluem apenas para dentro:

```
Frameworks → Interface Adapters → Use Cases → Entities
```

Nunca o contrário. Uma entidade `User` jamais importa do Fastify. Um `CreateUserUseCase` jamais importa do Prisma. Isso é garantido por interfaces definidas nas camadas internas e implementadas nas camadas externas (o Dependency Inversion Principle em escala arquitetural).

### Cruzando Fronteiras

Dados que cruzam uma fronteira são sempre convertidos em estruturas de dados simples (objetos planos ou DTOs). Nenhuma entidade de ORM, objeto de request de framework ou tipo de biblioteca cruza uma fronteira de camada para dentro.

---

## 3. Como Funciona

O fluxo típico de uma requisição em uma aplicação Clean Architecture:

```
HTTP Request
    │
    ▼
[Fastify Route Handler]   ← Frameworks & Drivers
    │  (parseia a request, cria InputDto)
    ▼
[Controller]              ← Interface Adapters
    │  (chama o use case com InputDto)
    ▼
[CreateUserUseCase]       ← Use Cases
    │  (chama a interface IUserRepository)
    ▼
[IUserRepository]         ← fronteira do Use Case (interface definida aqui)
    │
    ▼
[PrismaUserRepository]    ← Interface Adapters (implementa a interface)
    │
    ▼
[Prisma / PostgreSQL]     ← Frameworks & Drivers
```

O use case nunca conhece o Prisma. A entidade nunca conhece a estrutura da request. Cada camada se comunica por estruturas de dados de entrada/saída bem definidas.

---

## 4. Exemplos de Código (TypeScript)

### Estrutura do Projeto

```
src/
├── domain/                    # Entities (mais interna)
│   ├── entities/
│   │   └── User.ts
│   └── value-objects/
│       └── Email.ts
│
├── application/               # Use Cases
│   ├── use-cases/
│   │   └── create-user/
│   │       ├── CreateUserUseCase.ts
│   │       ├── CreateUserInput.ts
│   │       └── CreateUserOutput.ts
│   └── repositories/          # Interfaces (definidas aqui, implementadas na infra)
│       └── IUserRepository.ts
│
├── infrastructure/            # Interface Adapters + Frameworks
│   ├── repositories/
│   │   └── PrismaUserRepository.ts
│   ├── http/
│   │   └── controllers/
│   │       └── UserController.ts
│   └── database/
│       └── prisma-client.ts
│
└── main.ts                    # Raiz de composição
```

### Camada 1: Entity

```typescript
// src/domain/value-objects/Email.ts
export class Email {
  private readonly value: string;

  private constructor(value: string) {
    this.value = value;
  }

  static create(raw: string): Email {
    const normalized = raw.trim().toLowerCase();
    if (!normalized.includes('@')) {
      throw new Error(`E-mail inválido: ${raw}`);
    }
    return new Email(normalized);
  }

  toString(): string { return this.value; }

  equals(other: Email): boolean { return this.value === other.value; }
}
```

```typescript
// src/domain/entities/User.ts
import { Email } from '../value-objects/Email';

interface UserProps {
  id: string;
  name: string;
  email: Email;
  createdAt: Date;
  isActive: boolean;
}

export class User {
  private readonly props: UserProps;

  private constructor(props: UserProps) {
    this.props = props;
  }

  static create(props: Omit<UserProps, 'createdAt' | 'isActive'>): User {
    return new User({ ...props, createdAt: new Date(), isActive: true });
  }

  static reconstitute(props: UserProps): User {
    return new User(props);
  }

  get id(): string { return this.props.id; }
  get name(): string { return this.props.name; }
  get email(): Email { return this.props.email; }
  get createdAt(): Date { return this.props.createdAt; }
  get isActive(): boolean { return this.props.isActive; }

  deactivate(): void {
    if (!this.props.isActive) throw new Error('Usuário já está inativo');
    (this.props as UserProps).isActive = false;
  }
}
```

### Camada 2: Interface do Repositório e Use Case

```typescript
// src/application/repositories/IUserRepository.ts
import { User } from '../../domain/entities/User';

export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
}
```

```typescript
// src/application/use-cases/create-user/CreateUserUseCase.ts
import { randomUUID } from 'crypto';
import { User } from '../../../domain/entities/User';
import { Email } from '../../../domain/value-objects/Email';
import { IUserRepository } from '../../repositories/IUserRepository';

export interface CreateUserInput {
  name: string;
  email: string;
}

export interface CreateUserOutput {
  id: string;
  name: string;
  email: string;
  createdAt: string;
}

export class CreateUserUseCase {
  constructor(private readonly userRepo: IUserRepository) {}

  async execute(input: CreateUserInput): Promise<CreateUserOutput> {
    // 1. Valida regras de domínio
    const email = Email.create(input.email);

    // 2. Regra de negócio: e-mail deve ser único
    const existing = await this.userRepo.findByEmail(email.toString());
    if (existing) throw new Error('E-mail já está em uso');

    // 3. Cria a entidade
    const user = User.create({ id: randomUUID(), name: input.name.trim(), email });

    // 4. Persiste via abstração
    await this.userRepo.save(user);

    // 5. Retorna DTO plano — nenhuma entidade vaza para fora
    return {
      id: user.id,
      name: user.name,
      email: user.email.toString(),
      createdAt: user.createdAt.toISOString(),
    };
  }
}
```

> 💡 Observe: o use case não tem nenhum import de Fastify, Prisma ou qualquer biblioteca de infraestrutura. É pura orquestração de lógica de domínio e chamadas ao repositório.

### Camada 3: Implementação do Repositório

```typescript
// src/infrastructure/repositories/PrismaUserRepository.ts
import { PrismaClient } from '@prisma/client';
import { User } from '../../domain/entities/User';
import { Email } from '../../domain/value-objects/Email';
import { IUserRepository } from '../../application/repositories/IUserRepository';

export class PrismaUserRepository implements IUserRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    const row = await this.prisma.user.findUnique({ where: { id } });
    if (!row) return null;
    return User.reconstitute({
      id: row.id,
      name: row.name,
      email: Email.create(row.email),
      createdAt: row.createdAt,
      isActive: row.isActive,
    });
  }

  async findByEmail(email: string): Promise<User | null> {
    const row = await this.prisma.user.findUnique({ where: { email } });
    if (!row) return null;
    return User.reconstitute({
      id: row.id,
      name: row.name,
      email: Email.create(row.email),
      createdAt: row.createdAt,
      isActive: row.isActive,
    });
  }

  async save(user: User): Promise<void> {
    await this.prisma.user.upsert({
      where: { id: user.id },
      create: {
        id: user.id,
        name: user.name,
        email: user.email.toString(),
        createdAt: user.createdAt,
        isActive: user.isActive,
      },
      update: {
        name: user.name,
        email: user.email.toString(),
        isActive: user.isActive,
      },
    });
  }
}
```

### Camada 3: Controller

```typescript
// src/infrastructure/http/controllers/UserController.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { CreateUserUseCase } from '../../../application/use-cases/create-user/CreateUserUseCase';

interface CreateUserBody { name: string; email: string; }

export class UserController {
  constructor(private readonly createUser: CreateUserUseCase) {}

  async create(req: FastifyRequest<{ Body: CreateUserBody }>, reply: FastifyReply): Promise<void> {
    try {
      const output = await this.createUser.execute(req.body);
      reply.status(201).send(output);
    } catch (err) {
      if (err instanceof Error && err.message === 'E-mail já está em uso') {
        reply.status(409).send({ error: err.message });
        return;
      }
      reply.status(500).send({ error: 'Erro interno do servidor' });
    }
  }
}
```

### Raiz de Composição

```typescript
// src/main.ts
import Fastify from 'fastify';
import { PrismaClient } from '@prisma/client';
import { PrismaUserRepository } from './infrastructure/repositories/PrismaUserRepository';
import { CreateUserUseCase } from './application/use-cases/create-user/CreateUserUseCase';
import { UserController } from './infrastructure/http/controllers/UserController';

const prisma = new PrismaClient();
const userRepo = new PrismaUserRepository(prisma);
const createUserUseCase = new CreateUserUseCase(userRepo);
const userController = new UserController(createUserUseCase);

const app = Fastify();
app.post('/users', (req, reply) => userController.create(req, reply));
app.listen({ port: 3000 });
```

### Testes Sem Infraestrutura

```typescript
// tests/create-user.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { CreateUserUseCase } from '../src/application/use-cases/create-user/CreateUserUseCase';
import { IUserRepository } from '../src/application/repositories/IUserRepository';
import { User } from '../src/domain/entities/User';

class InMemoryUserRepository implements IUserRepository {
  private store = new Map<string, User>();

  async findById(id: string): Promise<User | null> { return this.store.get(id) ?? null; }

  async findByEmail(email: string): Promise<User | null> {
    return [...this.store.values()].find(u => u.email.toString() === email) ?? null;
  }

  async save(user: User): Promise<void> { this.store.set(user.id, user); }
}

describe('CreateUserUseCase', () => {
  let useCase: CreateUserUseCase;

  beforeEach(() => {
    useCase = new CreateUserUseCase(new InMemoryUserRepository());
  });

  it('cria um usuário e retorna o DTO de saída', async () => {
    const output = await useCase.execute({ name: 'Alice', email: 'alice@example.com' });
    expect(output.name).toBe('Alice');
    expect(output.email).toBe('alice@example.com');
    expect(output.id).toBeDefined();
  });

  it('rejeita e-mails duplicados', async () => {
    await useCase.execute({ name: 'Alice', email: 'alice@example.com' });
    await expect(
      useCase.execute({ name: 'Alice Clone', email: 'alice@example.com' })
    ).rejects.toThrow('E-mail já está em uso');
  });
});
```

---

## 5. Erros Comuns e Armadilhas

| Erro | Consequência | Correção |
|------|-------------|----------|
| Importar tipos do Prisma nas entities | Entities ficam acopladas ao ORM | Use value objects simples; mapeie no repositório |
| Colocar lógica de negócio nos controllers | Lógica não testável sem setup de HTTP | Mova para o use case |
| Compartilhar DTOs entre camadas | Mudança em qualquer camada se propaga por toda parte | Defina tipos de entrada/saída separados por use case |
| Entities anêmicas (só contêineres de dados) | Lógica de negócio vaza para os use cases | Entities devem impor seus próprios invariantes |
| Chamar use cases de outros use cases | Acoplamento oculto e dependências de ordem | Use domain events ou application services para coordenar |
| Forçar Clean Architecture em todo projeto | Over-engineering em CRUD simples | Aplique onde a complexidade do domínio justifica o custo |

> ⚠️ O maior erro é mapear as camadas da Clean Architecture para nomes de pastas de framework. Uma pasta `controllers/` em uma app Express não é automaticamente "Interface Adapters". O conceito é sobre direção de dependência, não nomenclatura de arquivos.

---

## 6. Quando Usar / Não Usar

**Use Clean Architecture quando:**
- O domínio é complexo com regras de negócio reais (faturamento, estoque, conformidade)
- Existem ou estão planejados múltiplos mecanismos de entrega (REST, CLI, gRPC, background jobs)
- O time precisa testar a lógica de negócio em unidade sem rodar infraestrutura
- O sistema deve durar 5+ anos com mudanças tecnológicas pelo caminho

**Simplifique ou pule quando:**
- O projeto é uma API CRUD simples sem lógica de negócio real
- É um protótipo ou MVP onde velocidade importa mais que estrutura
- O time é pequeno (1–2 devs) e o overhead do mapeamento de camadas supera o benefício
- O "domínio" é apenas ler e escrever dados sem nenhuma decisão

> 💡 Uma heurística útil: se você pode substituir cada use case por uma única query no banco e nenhuma lógica, a Clean Architecture adiciona estrutura sem benefício. O ponto ideal é quando os use cases contêm decisões reais.

---

## 7. Cenário Real

Uma plataforma de e-commerce começa como um monólito Fastify + Prisma. Após 18 meses, o time precisa:

1. Adicionar uma ferramenta CLI que processa reembolsos a partir de um arquivo CSV
2. Expor a mesma lógica de negócio via gRPC para um serviço interno de liquidação

Com Clean Architecture, as duas adições são mecânicas:

```typescript
// O use case é reutilizado exatamente — zero mudanças
const processRefund = new ProcessRefundUseCase(orderRepo, paymentGateway);

// Adapter de CLI
const csvRows = parseCsv(file);
for (const row of csvRows) {
  await processRefund.execute({ orderId: row.order_id, amount: row.amount });
}

// Adapter gRPC
server.addService(RefundService, {
  processRefund: async (call, callback) => {
    const output = await processRefund.execute(call.request);
    callback(null, output);
  },
});
```

O use case foi escrito uma vez. CLI e gRPC são apenas novos adapters. Nenhuma lógica de negócio foi duplicada.

---

## 8. Perguntas de Entrevista

**Q1: O que é a Regra de Dependência e por que ela é o núcleo da Clean Architecture?**

R: A Regra de Dependência diz que todas as dependências do código-fonte devem apontar para dentro — em direção à política de mais alto nível. Camadas externas (frameworks, UI, DB) dependem das internas (use cases, entities), nunca o contrário. Isso garante que a lógica de negócio seja isolada dos detalhes tecnológicos, tornando-a testável e substituível de forma independente.

---

**Q2: Onde se definem as interfaces de repositório na Clean Architecture?**

R: Na camada de Use Case (aplicação), não na camada de infraestrutura. O use case define o que precisa (a interface), e a infraestrutura fornece a implementação. Isso mantém a dependência apontando para dentro — o use case não conhece o Prisma nem nenhum banco específico.

---

**Q3: O que é um modelo de domínio anêmico e por que é um problema?**

R: Um modelo de domínio anêmico é quando as entities são apenas contêineres de dados sem comportamento. A lógica de negócio vive em use cases ou serviços. O problema é que as entities não impõem seus próprios invariantes — estado inválido pode ser criado de qualquer lugar. Entities devem conter regras de negócio da empresa e impô-las.

---

**Q4: Como lidar com preocupações transversais como logging sem poluir os use cases?**

R: Use o padrão Decorator ou middleware aplicado na camada de Interface Adapters. Um `LoggedCreateUserUseCase` envolve `CreateUserUseCase` e adiciona logging sem modificar o use case. Fronteiras de transação são gerenciadas na implementação do repositório ou via padrão Unit of Work na camada de infraestrutura.

---

**Q5: Qual formato de dados deve cruzar uma fronteira de camada?**

R: Estruturas de dados simples e serializáveis — DTOs — não entities de domínio, modelos de ORM ou tipos específicos de framework. Isso impede que camadas internas conheçam tipos das externas e mantém as fronteiras limpas.

---

**Q6: Clean Architecture é o mesmo que Arquitetura Hexagonal?**

R: Compartilham a mesma ideia central (isolar lógica de negócio da infraestrutura via abstrações), mas diferem na terminologia. A Arquitetura Hexagonal foca em Ports e Adapters com distinção primário/secundário. A Clean Architecture adiciona um modelo de quatro camadas mais explícito e é mais prescritiva sobre o que vai onde.

---

**Q7: Como testar o sistema completo e ainda aproveitar a testabilidade da Clean Architecture?**

R: Use a pirâmide de testes. Teste use cases com fakes em memória — rápidos e numerosos. Teste de integração as implementações de repositório contra um banco real mas efêmero. Teste end-to-end a camada HTTP contra uma instância real do servidor. Falhas em testes unitários apontam para lógica de negócio; falhas em integração apontam para infraestrutura; falhas em e2e apontam para wiring ou contratos de API.

---

## 9. Exercícios

**Exercício 1: Adicione um GetUserByIdUseCase**

Seguindo a estrutura do projeto apresentada acima, implemente um `GetUserByIdUseCase` que aceita `{ id: string }`, retorna o DTO do usuário ou lança se não encontrado, e tem um teste usando `InMemoryUserRepository`.

*Dica: A interface do repositório já tem `findById`. O use case é simples — valide a existência, mapeie para o DTO de saída.*

---

**Exercício 2: Adicione um mecanismo de entrega via CLI**

Escreva um script CLI simples (`cli/create-user.ts`) que lê `name` e `email` de `process.argv`, conecta `PrismaUserRepository` e `CreateUserUseCase`, chama o use case e imprime o resultado.

*Dica: O código do use case não muda nada — apenas o wiring no entry point da CLI difere do `main.ts`.*

---

**Exercício 3: Refatore um controller anêmico**

Dado este controller que mistura preocupações:

```typescript
app.post('/users', async (req, reply) => {
  const { name, email } = req.body;
  const existing = await prisma.user.findUnique({ where: { email } });
  if (existing) return reply.status(409).send({ error: 'E-mail em uso' });
  const user = await prisma.user.create({ data: { name, email } });
  reply.status(201).send(user);
});
```

Extraia um `CreateUserUseCase` adequado com um `IUserRepository`. O controller deve apenas parsear a request e chamar o use case.

*Dica: O controller não deve saber o que "e-mail em uso" significa — isso é conhecimento de domínio.*

---

## 10. Leitura Complementar

- **Clean Architecture: A Craftsman's Guide to Software Structure and Design** — Robert C. Martin (2017)
- **[The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)** — post original de Uncle Bob
- **Implementing Domain-Driven Design** — Vaughn Vernon (complementar na camada de entities)
- **[Clean Architecture with TypeScript](https://khalilstemmler.com/articles/software-design-architecture/organizing-app-logic/)** — série de Khalil Stemmler
- **Dependency Injection Principles, Practices, and Patterns** — Mark Seemann & Steven van Deursen
