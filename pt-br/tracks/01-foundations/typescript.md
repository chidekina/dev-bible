# TypeScript

> TypeScript é JavaScript com um sistema de tipos. Ele captura categorias inteiras de bugs em tempo de compilação, serve como documentação viva e torna grandes bases de código navegáveis. Este arquivo cobre os fundamentos do sistema de tipos, generics, tipos avançados e os padrões que tornam o TypeScript realmente útil — não apenas um JavaScript polvilhado de `any`.

---

## 1. O que é e por que importa

TypeScript é um superset estaticamente tipado do JavaScript criado pela Microsoft. Compila para JavaScript puro, então roda em qualquer lugar que JavaScript roda. TypeScript adiciona:
- **Verificação de tipos estática**: captura erros de tipo antes do runtime
- **Inteligência da IDE**: autocomplete, ir para definição, rename refactoring
- **Código autodocumentado**: tipos servem como documentação sempre atualizada
- **Refatoração segura**: o compilador te diz todos os call sites que quebram quando você altera a assinatura de uma função

O sistema de tipos do TypeScript é **estrutural** (não nominal) e opera apenas em **tempo de compilação** — os tipos são apagados quando compilados para JavaScript. Não há overhead em tempo de execução.

---

## 2. Conceitos Fundamentais

### Tipagem Estática vs Dinâmica

```
JavaScript (dinâmico): tipos verificados em runtime — erros aparecem quando o código roda
TypeScript (estático):  tipos verificados em compilação — erros aparecem antes do código rodar
```

```typescript
// JavaScript — erro descoberto apenas em runtime
function add(a, b) { return a + b; }
add(1, "2");  // retorna "12" silenciosamente — nenhum erro

// TypeScript — erro em tempo de compilação
function add(a: number, b: number): number { return a + b; }
add(1, "2");  // Argument of type 'string' is not assignable to parameter of type 'number'
```

### Tipagem Estrutural (Duck Typing)

TypeScript usa tipagem estrutural: dois tipos são compatíveis se suas estruturas são compatíveis, independentemente dos nomes.

```typescript
interface Point2D { x: number; y: number; }
interface Coordinate { x: number; y: number; }

function plot(p: Point2D) { /* ... */ }

const coord: Coordinate = { x: 1, y: 2 };
plot(coord);  // OK — estrutura compatível, mesmo que os nomes difiram

// Literais de objeto verificam de forma MAIS estrita (excess property checks)
plot({ x: 1, y: 2, z: 3 });  // Erro — 'z' não existe em Point2D
const p = { x: 1, y: 2, z: 3 };
plot(p);  // OK — verificação estrutural, propriedades extras permitidas via variável
```

---

### Tipos Primitivos

```typescript
let name: string = "Alice";
let age: number = 30;         // inclui inteiros, floats, NaN, Infinity
let active: boolean = true;
let id: bigint = 9007199254740991n;
let sym: symbol = Symbol('key');
let nothing: null = null;
let notDefined: undefined = undefined;

// TypeScript infere tipos de inicializadores — anotações frequentemente são opcionais
let inferred = "Alice";  // tipo: string (inferido)
```

---

### Union Types

Um valor de tipo union pode ser qualquer um dos tipos membros:
```typescript
type StringOrNumber = string | number;
let value: StringOrNumber = "hello";
value = 42;  // também válido

// Union discriminada — cada membro tem um campo de tipo literal
type Result<T> =
  | { success: true;  value: T }
  | { success: false; error: string };

function divide(a: number, b: number): Result<number> {
  if (b === 0) return { success: false, error: "Divisão por zero" };
  return { success: true, value: a / b };
}

const r = divide(10, 2);
if (r.success) {
  console.log(r.value);  // TypeScript sabe que value existe aqui
} else {
  console.error(r.error);  // TypeScript sabe que error existe aqui
}
```

---

### Intersection Types

Combina múltiplos tipos — o valor deve satisfazer TODOS os membros:
```typescript
type Serializable = { serialize(): string };
type Loggable = { log(): void };

type SerializableLoggable = Serializable & Loggable;

// Usado bastante para mixins e extensão de tipos
type UserWithPermissions = User & { permissions: string[] };
```

---

### Tipos Literais

Um tipo que representa um valor específico:
```typescript
type Direction = "north" | "south" | "east" | "west";
type StatusCode = 200 | 201 | 400 | 401 | 403 | 404 | 500;
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

function move(dir: Direction) { /* ... */ }
move("north");     // OK
move("diagonal");  // Erro — não é uma Direction válida

// Template literal types
type EventName = `on${Capitalize<string>}`;
type CssUnit = `${number}px` | `${number}rem` | `${number}%`;
type UserId = `user_${string}`;
```

---

### Interfaces vs Type Aliases

Ambos definem formatos de objetos. As diferenças práticas:

```typescript
// Interface — pode ser estendida via declaration merging, preferida para APIs públicas
interface User {
  id: number;
  name: string;
  email: string;
}

// Estendendo uma interface
interface AdminUser extends User {
  permissions: string[];
}

// Declaration merging (aumentando uma interface existente)
interface Window {
  myCustomProperty: string;
}

// Type alias — necessário para unions, intersections, tuples, primitivos
type ID = string | number;
type Point = [number, number];  // tuple
type Callback = (err: Error | null, result?: string) => void;
type UserOrAdmin = User | AdminUser;

// Ambos podem definir formatos de objeto de forma idêntica para a maioria dos casos
type UserType = {
  id: number;
  name: string;
};
```

> **Regra prática**: Use `interface` para formatos de objetos e classes que possam ser estendidos ou aumentados. Use `type` para tudo mais — unions, intersections, tuples, mapped types e tipos complexos computados.

---

### Generics

Generics permitem escrever código que funciona com múltiplos tipos preservando as informações de tipo:

```typescript
// Função genérica — T é inferido do argumento
function identity<T>(value: T): T {
  return value;
}
const s = identity("hello");  // T = string, tipo de retorno = string
const n = identity(42);        // T = number, tipo de retorno = number

// Restrição genérica — T deve ter uma propriedade .length
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}
longest("abc", "xy");      // OK — strings têm length
longest([1, 2, 3], [1]);   // OK — arrays têm length
longest(42, 10);            // Erro — números não têm length

// Múltiplos parâmetros de tipo
function zip<A, B>(a: A[], b: B[]): Array<[A, B]> {
  return a.map((item, i) => [item, b[i]]);
}
const zipped = zip([1, 2, 3], ["a", "b", "c"]);  // [number, string][]

// Interface genérica
interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: string): Promise<void>;
}
```

---

### Tipos Condicionais

Tipos que se resolvem em um tipo ou outro com base em uma condição:

```typescript
type IsArray<T> = T extends any[] ? true : false;
type A = IsArray<string[]>;  // true
type B = IsArray<string>;    // false

// Tipos condicionais distributivos — aplicados a cada membro de uma union
type ToArray<T> = T extends any ? T[] : never;
type StrOrNumArr = ToArray<string | number>;  // string[] | number[]

// A palavra-chave `infer` — extrai um tipo de um condicional
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type Awaited<T> = T extends Promise<infer U> ? U : T;

type Fn = () => string;
type R = ReturnType<Fn>;  // string

type GetFirst<T> = T extends [infer First, ...any[]] ? First : never;
type First = GetFirst<[string, number, boolean]>;  // string
```

---

### Mapped Types

Transforma todas as propriedades de um tipo usando um mapeamento:

```typescript
// Torna todas as propriedades opcionais (igual ao Partial<T> embutido)
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

// Torna todas as propriedades readonly
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Seleciona chaves específicas
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Remapeia chaves com 'as'
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type UserGetters = Getters<{ name: string; age: number }>;
// { getName: () => string; getAge: () => number }
```

---

### Utility Types

Os utility types genéricos embutidos do TypeScript:

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Partial<T> — todas as propriedades opcionais
type UserUpdate = Partial<User>;

// Required<T> — todas as propriedades obrigatórias (oposto de Partial)
type StrictUser = Required<User>;

// Pick<T, K> — mantém apenas as chaves especificadas
type PublicUser = Pick<User, 'id' | 'name' | 'email'>;

// Omit<T, K> — remove as chaves especificadas
type SafeUser = Omit<User, 'password'>;

// Record<K, V> — objeto com chaves K e valores V
type UserById = Record<string, User>;
const cache: Record<string, User> = {};

// Readonly<T> — todas as propriedades readonly
type ImmutableUser = Readonly<User>;

// ReturnType<T> — extrai o tipo de retorno de uma função
async function fetchUser(id: string): Promise<User> { /* ... */ }
type FetchResult = Awaited<ReturnType<typeof fetchUser>>;  // User

// Parameters<T> — extrai os tipos de parâmetros como uma tuple
type FetchParams = Parameters<typeof fetchUser>;  // [string]

// NonNullable<T> — remove null e undefined de um tipo
type MaybeUser = User | null | undefined;
type DefiniteUser = NonNullable<MaybeUser>;  // User

// Extract<T, U> — mantém apenas tipos de T que são atribuíveis a U
type StringOrNumber = string | number | boolean;
type OnlyStrings = Extract<StringOrNumber, string>;  // string

// Exclude<T, U> — remove tipos de T que são atribuíveis a U
type NoStrings = Exclude<StringOrNumber, string>;  // number | boolean
```

---

### Narrowing de Tipos

TypeScript estreita o tipo dentro de blocos condicionais com base em verificações:

```typescript
function processInput(input: string | number | null) {
  // narrowing com typeof
  if (typeof input === 'string') {
    console.log(input.toUpperCase());  // input: string aqui
  } else if (typeof input === 'number') {
    console.log(input.toFixed(2));     // input: number aqui
  } else {
    console.log('input nulo');         // input: null aqui
  }
}

// narrowing com instanceof
function formatError(err: unknown) {
  if (err instanceof Error) {
    return err.message;  // err: Error aqui
  }
  return String(err);
}

// narrowing com operador in (unions discriminadas)
type Cat = { kind: 'cat'; meow(): void };
type Dog = { kind: 'dog'; bark(): void };

function makeSound(animal: Cat | Dog) {
  if (animal.kind === 'cat') {
    animal.meow();  // animal: Cat aqui
  } else {
    animal.bark();  // animal: Dog aqui
  }
}

// Type predicate (type guard definido pelo usuário)
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
}

// Assertion function (lança se o tipo não corresponder)
function assertIsString(val: unknown): asserts val is string {
  if (typeof val !== 'string') {
    throw new TypeError(`Esperava string, recebi ${typeof val}`);
  }
}
```

---

### `unknown` vs `any` vs `never`

```typescript
// any — desativa a verificação de tipos (escape hatch inseguro)
// Evite: erros em código `any` se propagam silenciosamente
let x: any = 42;
x.toUpperCase();  // sem erro em compilação — crash em runtime!

// unknown — alternativa segura a any
// Deve ser estreitado antes de usar
let y: unknown = fetchSomething();
y.toUpperCase();  // Erro — deve ser estreitado primeiro
if (typeof y === 'string') {
  y.toUpperCase();  // OK — estreitado para string
}

// never — representa um valor que nunca pode existir
// Usado para: verificações exaustivas, código inalcançável, tipo base
function assertNever(x: never): never {
  throw new Error(`Valor inesperado: ${JSON.stringify(x)}`);
}

type Shape = 'circle' | 'square' | 'triangle';
function area(shape: Shape): number {
  switch (shape) {
    case 'circle': return Math.PI;
    case 'square': return 1;
    case 'triangle': return 0.5;
    default: return assertNever(shape);  // Erro de compilação se um case estiver faltando
  }
}
```

> **Quando usar `unknown`**: Sempre que você tem um valor cujo tipo não conhece (respostas de API, resultados de `JSON.parse`, erros na cláusula `catch`). Força você a verificar antes de usar, tornando-o seguro. Use `any` apenas como último recurso ao migrar do JavaScript.

---

## 3. Como Funciona

### Opções Chave do tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",            // versão JS de saída
    "module": "ESNext",            // sistema de módulos (CommonJS, ESNext, etc.)
    "moduleResolution": "bundler", // como os módulos são resolvidos (Node16, Bundler)
    "strict": true,                // habilita todas as verificações estritas (altamente recomendado)
    "noUncheckedIndexedAccess": true, // arr[0] é T | undefined, não T
    "exactOptionalPropertyTypes": true, // { x?: string } — x pode ser string ou ausente, não undefined
    "noImplicitReturns": true,     // função deve retornar em todos os caminhos de código
    "noImplicitOverride": true,    // palavra-chave override obrigatória ao sobrescrever membros de classe
    "paths": {                     // aliases de caminho (use com bundler)
      "@/*": ["./src/*"]
    },
    "baseUrl": ".",
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,           // gera arquivos .d.ts
    "sourceMap": true
  }
}
```

> **Sempre habilite `strict: true`**. Habilita: `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictPropertyInitialization`, e outros. Desabilitar o modo strict elimina a maior parte do valor do TypeScript.

---

## 4. Exemplos de Código

### Wrapper fetch genérico com tipo de retorno inferido
```typescript
async function fetchJson<T>(url: string, options?: RequestInit): Promise<T> {
  const res = await fetch(url, options);

  if (!res.ok) {
    throw new Error(`HTTP ${res.status}: ${res.statusText}`);
  }

  return res.json() as Promise<T>;
}

// Uso — T é fornecido explicitamente
interface Post { id: number; title: string; body: string; }
const post = await fetchJson<Post>('/api/posts/1');
console.log(post.title);  // TypeScript sabe que post tem title: string
```

### Event emitter com tipos seguros
```typescript
type EventMap = Record<string, unknown>;

class TypedEventEmitter<Events extends EventMap> {
  private listeners: Partial<{
    [K in keyof Events]: Array<(data: Events[K]) => void>;
  }> = {};

  on<K extends keyof Events>(event: K, listener: (data: Events[K]) => void): this {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(listener);
    return this;
  }

  off<K extends keyof Events>(event: K, listener: (data: Events[K]) => void): this {
    const list = this.listeners[event];
    if (list) {
      this.listeners[event] = list.filter((l) => l !== listener) as typeof list;
    }
    return this;
  }

  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    this.listeners[event]?.forEach((listener) => listener(data));
  }
}

// Uso
interface AppEvents {
  userLogin: { userId: string; timestamp: Date };
  userLogout: { userId: string };
  error: { message: string; code: number };
}

const emitter = new TypedEventEmitter<AppEvents>();
emitter.on('userLogin', ({ userId, timestamp }) => {
  console.log(`Usuário ${userId} logou às ${timestamp}`);
});
emitter.emit('userLogin', { userId: '123', timestamp: new Date() });
emitter.emit('error', { message: 'oops', code: 500 });
// emitter.emit('unknown', {});  // Erro — 'unknown' não está em AppEvents
```

### Utility type DeepReadonly
```typescript
type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

interface Config {
  server: {
    host: string;
    port: number;
    tls: {
      enabled: boolean;
      certPath: string;
    };
  };
  features: string[];
}

type ImmutableConfig = DeepReadonly<Config>;

const config: ImmutableConfig = {
  server: { host: 'localhost', port: 3000, tls: { enabled: false, certPath: '' } },
  features: ['auth', 'logging'],
};

// config.server.host = 'prod';     // Erro — profundamente readonly
// config.features.push('newFeat'); // Erro — ReadonlyArray
```

### Reducer estilo Redux com union discriminada
```typescript
interface TodoItem {
  id: string;
  text: string;
  done: boolean;
}

interface TodoState {
  items: TodoItem[];
  filter: 'all' | 'active' | 'done';
  loading: boolean;
}

type TodoAction =
  | { type: 'ADD_TODO';    payload: { text: string } }
  | { type: 'TOGGLE_TODO'; payload: { id: string } }
  | { type: 'DELETE_TODO'; payload: { id: string } }
  | { type: 'SET_FILTER';  payload: { filter: TodoState['filter'] } }
  | { type: 'SET_LOADING'; payload: { loading: boolean } };

function todoReducer(state: TodoState, action: TodoAction): TodoState {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        items: [
          ...state.items,
          { id: crypto.randomUUID(), text: action.payload.text, done: false },
        ],
      };
    case 'TOGGLE_TODO':
      return {
        ...state,
        items: state.items.map((item) =>
          item.id === action.payload.id ? { ...item, done: !item.done } : item
        ),
      };
    case 'DELETE_TODO':
      return {
        ...state,
        items: state.items.filter((item) => item.id !== action.payload.id),
      };
    case 'SET_FILTER':
      return { ...state, filter: action.payload.filter };
    case 'SET_LOADING':
      return { ...state, loading: action.payload.loading };
    default:
      // Verificação exaustiva — TypeScript acusa erro se um case estiver faltando
      const _exhaustive: never = action;
      return state;
  }
}
```

---

## 5. Erros Comuns e Armadilhas

> **Uso excessivo de `any`**: Uma vez que você converte algo para `any`, TypeScript para de verificá-lo — e qualquer acesso a propriedade ou chamada de método fica sem checagem. Erros introduzidos via `any` se propagam silenciosamente. Use `unknown` e estreite antes de usar.

> **Casting com `as` para silenciar erros**: `value as TargetType` é uma asserção de tipo. Sobrescreve a inferência do TypeScript. Não muda o valor em runtime. Usar para silenciar erros ("o compilador está errado") geralmente mascara um bug real.
> ```typescript
> const user = JSON.parse(data) as User;  // perigoso — sem garantia em runtime
> // Melhor: use um validador em runtime (zod, valibot) e então tipifique o resultado
> const user = UserSchema.parse(JSON.parse(data));  // validado em runtime
> ```

> **Não habilitar o modo `strict`**: Sem `strictNullChecks`, `null` e `undefined` são atribuíveis a todo tipo, o que anula boa parte da segurança do TypeScript. Sem `noImplicitAny`, TypeScript infere silenciosamente `any` para parâmetros sem tipo.

> **Ignorar tipos de acesso indexado com `noUncheckedIndexedAccess`**: Sem essa opção, `arr[0]` é tipado como `T`, não `T | undefined`. Se o array estiver vazio, você recebe um erro em runtime. Habilite `noUncheckedIndexedAccess` para que acesso indexado retorne `T | undefined`, forçando você a tratar o caso undefined.

> **Confusão entre interface e type**: Escolher errado não é um bug em runtime, mas causa fricção. A diferença chave: interfaces podem ser aumentadas via declaration merging (importante para tipos de biblioteca), type aliases não podem. Use interfaces para coisas que código externo vai estender.

---

## 6. Quando Usar / Não Usar

**Use TypeScript quando:**
- Construindo qualquer coisa não trivial em tamanho ou equipe
- APIs públicas ou bibliotecas (arquivos `.d.ts` fornecem tipos aos consumidores)
- Projetos que precisam de refatoração confiante
- Código que será mantido além de 3 meses

**Use `unknown` quando:**
- Recebendo dados de fora (respostas de API, JSON.parse, cláusulas catch no TypeScript 4+)

**Use `never` quando:**
- Construindo verificações exaustivas em unions discriminadas
- Sinalizando caminhos de código que nunca devem ser alcançados
- O tipo base em computações de utility types

**Use `type` alias quando:**
- Definindo unions, intersections, tuples ou mapped types
- O tipo nunca vai precisar de declaration merging

**Use `interface` quando:**
- Definindo o formato de objetos ou classes
- Construindo tipos de biblioteca que consumidores podem estender
- Trabalhando com declarações `implements` em classes

---

## 7. Cenário Real

### Camada de API com validação Zod e tipos seguros

```typescript
import { z } from 'zod';

// Define schema — Zod valida em runtime E infere o tipo TypeScript
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'moderator']),
  createdAt: z.coerce.date(),
});

const PaginatedUsersSchema = z.object({
  users: z.array(UserSchema),
  total: z.number().int().nonnegative(),
  page: z.number().int().positive(),
  pageSize: z.number().int().positive(),
});

// Tipo inferido do schema — sem duplicação
type User = z.infer<typeof UserSchema>;
type PaginatedUsers = z.infer<typeof PaginatedUsersSchema>;

// Fetcher genérico validado
async function fetchValidated<T>(
  url: string,
  schema: z.ZodType<T>,
  options?: RequestInit
): Promise<T> {
  const res = await fetch(url, options);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);

  const raw = await res.json();
  return schema.parse(raw);  // lança ZodError se o formato não corresponder
}

// Uso — totalmente tipado, validado em runtime
async function getUsers(page: number): Promise<PaginatedUsers> {
  return fetchValidated(
    `/api/users?page=${page}`,
    PaginatedUsersSchema
  );
}

const result = await getUsers(1);
result.users.forEach((user) => {
  // user é tipado como User — autocomplete funciona
  console.log(user.email);    // string
  console.log(user.role);     // 'admin' | 'user' | 'moderator'
  console.log(user.createdAt); // Date (convertido de string ISO)
});
```

---

## 8. Perguntas de Entrevista

**Q1: Qual a diferença entre interface e type alias?**

R: Ambos podem definir formatos de objetos e são amplamente intercambiáveis para esse propósito. Diferenças chave: (1) Type aliases podem representar qualquer tipo — unions, intersections, tuples, primitivos, mapped types. Interfaces só podem representar formatos de objetos e contratos de classes. (2) Interfaces suportam declaration merging — você pode adicionar propriedades a uma interface existente de um arquivo diferente (útil para aumentar tipos de bibliotecas). Type aliases não podem ser reabertos. (3) Mensagens de erro de interfaces tendem a mostrar o nome da interface; erros de type alias frequentemente expandem a definição. Regra prática: use interface para formatos de objetos/classes, use type para tudo mais.

---

**Q2: O que é tipagem estrutural?**

R: O sistema de tipos do TypeScript é estrutural (duck typing): dois tipos são compatíveis se suas formas são compatíveis, independentemente dos nomes ou como foram definidos. Um valor do tipo `Cat { meow(): void }` satisfaz `type Animal { meow(): void }` mesmo sem `implements` explícito. Isso contrasta com tipagem nominal (Java, C#) onde tipos só são compatíveis se estão explicitamente na mesma hierarquia de classes. Consequência prática: você pode passar qualquer objeto para uma função desde que tenha as propriedades necessárias — sem precisar implementar a interface explicitamente.

---

**Q3: Explique generics. Quando você os usaria?**

R: Generics permitem escrever funções, classes e interfaces que funcionam com múltiplos tipos preservando as relações de tipo. Um parâmetro de tipo genérico (`<T>`) é um placeholder resolvido quando o generic é usado. Use generics quando: (1) você tem uma função que retorna o mesmo tipo do seu input (`identity<T>(x: T): T`); (2) você tem um tipo container cujo tipo de elemento varia (`Array<T>`, `Promise<T>`, `Repository<T>`); (3) você tem uma função onde os tipos de input e output são relacionados mas não fixos. Sem generics, você usaria `any` e perderia a segurança de tipos, ou escreveria funções separadas para cada tipo.

---

**Q4: Para que serve `never`?**

R: `never` representa valores que nunca podem existir — o tipo base. Usos: (1) Verificações exaustivas em unions discriminadas — se você adicionar um novo membro da union mas esquecer de tratá-lo em um switch, a chamada `assertNever(x: never)` produz um erro de compilação. (2) Funções que nunca retornam (`throw` ou loop infinito): `function fail(msg: string): never { throw new Error(msg); }`. (3) Manipulação de tipos onde certos ramos devem ser eliminados: `Exclude<T, U>` usa `never` para remover tipos. (4) Em tipos condicionais: `type NonNullable<T> = T extends null | undefined ? never : T`.

---

**Q5: Como funciona o narrowing de tipos?**

R: TypeScript analisa o fluxo de controle para estreitar tipos union dentro de ramos condicionais. Você estreita usando: `typeof` (estreita para tipos primitivos), `instanceof` (estreita para instâncias de classe), operador `in` (estreita para objetos com uma propriedade), verificações de union discriminada (verificar um campo de tipo literal), verificações de veracidade (estreita null/undefined), e type predicates (tipo de retorno `x is T` definido pelo usuário). Uma vez estreitado, TypeScript permite apenas operações válidas para o tipo estreitado. Após o bloco condicional, o tipo alarga de volta para a union.

---

**Q6: O que são utility types? Cite cinco.**

R: Utility types são helpers genéricos embutidos no TypeScript para transformações comuns de tipos. Cinco importantes: (1) `Partial<T>` — torna todas as propriedades opcionais; (2) `Required<T>` — torna todas as propriedades obrigatórias; (3) `Pick<T, K>` — cria um tipo com apenas as chaves especificadas; (4) `Omit<T, K>` — cria um tipo sem as chaves especificadas; (5) `Record<K, V>` — cria um tipo de objeto com chaves K e valores V. Outros que vale conhecer: `Readonly<T>`, `ReturnType<T>`, `Parameters<T>`, `Awaited<T>`, `NonNullable<T>`.

---

**Q7: Qual a diferença entre `unknown` e `any`?**

R: Ambos representam "um valor de qualquer tipo". A diferença está em quão seguramente você pode usá-los. Com `any`, você pode fazer tudo — chamar métodos, acessar propriedades, atribuir em qualquer lugar — TypeScript para de verificar. É uma saída completa do sistema de tipos. Com `unknown`, você não pode fazer nada com o valor até estreitar seu tipo com uma verificação (`typeof`, `instanceof`, um type predicate). Isso te força a tratar o tipo desconhecido de forma segura. Use `unknown` para valores cujo tipo você genuinamente não conhece em tempo de declaração (respostas de API, `JSON.parse`, variáveis da cláusula catch). Prefira `unknown` ao `any` em toda situação onde você teria usado `any`.

---

**Q8: O que é uma union discriminada?**

R: Uma union discriminada (também chamada tagged union) é um tipo union onde cada membro tem uma propriedade comum com um tipo literal único — o discriminante. TypeScript usa essa propriedade literal para estreitar o tipo em ramos condicionais. Exemplo: `type Action = { type: 'ADD'; item: string } | { type: 'REMOVE'; id: number }`. O campo `type` é o discriminante. Em um switch em `action.type`, TypeScript sabe exatamente em qual membro da union você está e te dá acesso às propriedades corretas. Esse padrão é a base dos reducers Redux e máquinas de estado em TypeScript.

---

## 9. Exercícios

**Exercício 1 — Wrapper fetch genérico:**

Escreva uma função `fetchJson<T>(url: string, options?: RequestInit): Promise<T>` que:
- Busca uma URL e faz parse do JSON
- Retorna `Promise<T>` onde `T` é fornecido pelo chamador
- Lança uma classe `HttpError` tipada com `status: number` e `body: string` em respostas não-2xx
- Tem uma segunda sobrecarga `fetchJson<T, E>(url, schema: ZodSchema<T>): Promise<T>` que valida a resposta

---

**Exercício 2 — Event emitter com tipos seguros:**

Construa uma classe `TypedEventEmitter<Events extends Record<string, unknown>>` que:
- `on<K extends keyof Events>(event: K, listener: (data: Events[K]) => void): this`
- `off<K extends keyof Events>(event: K, listener: ...): this`
- `emit<K extends keyof Events>(event: K, data: Events[K]): void`
- TypeScript deve acusar erro em `emit('eventoDesconhecido', ...)` e `on('eventoDesconhecido', ...)`

---

**Exercício 3 — DeepReadonly:**

Implemente `DeepReadonly<T>` que recursivamente torna todas as propriedades (e propriedades aninhadas) readonly:
```typescript
type DeepReadonly<T> = /* sua implementação */

// Deve satisfazer:
type Config = { server: { host: string; port: number }; features: string[] };
type ImmutableConfig = DeepReadonly<Config>;
// ImmutableConfig.server.host deve ser readonly
// ImmutableConfig.features deve ser ReadonlyArray<string>
```

---

**Exercício 4 — Tipifique um reducer estilo Redux:**

Dado um TodoState e um conjunto de ações (ADD_TODO, TOGGLE_TODO, REMOVE_TODO, SET_FILTER), escreva:
- Um tipo union discriminada `TodoAction` para todas as ações
- Um `todoReducer(state: TodoState, action: TodoAction): TodoState` que trata todos os casos
- Uma verificação exaustiva para que TypeScript acuse erro se você adicionar um novo tipo de ação mas esquecer de tratá-lo

---

## 10. Leitura Adicional

- **Manual oficial do TypeScript**: https://www.typescriptlang.org/docs/handbook/intro.html
- **TypeScript playground**: https://www.typescriptlang.org/play — experimente tipos interativamente no navegador
- **"Effective TypeScript" de Dan Vanderkam** — 62 formas específicas de melhorar seu TypeScript
- **Total TypeScript** de Matt Pocock: https://www.totaltypescript.com — o melhor curso estruturado sobre TypeScript avançado
- **Type Challenges**: https://github.com/type-challenges/type-challenges — pratique implementar utility types
- **Zod**: https://zod.dev — validação em runtime que infere tipos TypeScript (excelente combinação com TypeScript)
- **"Programming TypeScript" de Boris Cherny** (O'Reilly) — mergulho profundo no sistema de tipos
- **TypeScript Compiler API**: https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API — para construir ferramentas que trabalham com ASTs TypeScript
- **DefinitelyTyped**: https://github.com/DefinitelyTyped/DefinitelyTyped — definições de tipos da comunidade para bibliotecas JavaScript
