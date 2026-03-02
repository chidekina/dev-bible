# TypeScript Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## Tipos Primitivos

```ts
let n: number = 42;
let s: string = "hello";
let b: boolean = true;
let u: undefined = undefined;
let nl: null = null;
let sym: symbol = Symbol("id");
let big: bigint = 9007199254740991n;
let any: any = "anything";
let unk: unknown = "mais seguro que any";
let nv: never;          // função que nunca retorna
let vd: void;           // função que não retorna nada útil
```

---

## Type Aliases e Interfaces

```ts
type Point = { x: number; y: number };
type ID = string | number;
type Callback = (err: Error | null, data: string) => void;

interface User {
  id: number;
  name: string;
  email?: string;           // opcional
  readonly createdAt: Date; // imutável
}

// Extendendo
interface Admin extends User {
  role: "admin" | "superadmin";
}

// Mesclagem (apenas interfaces)
interface Window { myLib: boolean; }
interface Window { version: string; }
```

**Type vs Interface:** Use `interface` para formatos de objetos (extensíveis); use `type` para unions, primitivos e tuples.

---

## Union e Intersection

```ts
type StringOrNum = string | number;
type AdminUser = User & { role: string };

// Union discriminada
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; width: number; height: number };

function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;
    case "rect":   return s.width * s.height;
  }
}
```

---

## Generics

```ts
function identity<T>(x: T): T { return x; }

// Restrições
function getLength<T extends { length: number }>(x: T): number {
  return x.length;
}

// Múltiplos parâmetros de tipo
function pair<A, B>(a: A, b: B): [A, B] { return [a, b]; }

// Interface genérica
interface Repository<T, ID = number> {
  findById(id: ID): Promise<T | null>;
  save(entity: T): Promise<T>;
}

// Classe genérica
class Stack<T> {
  private items: T[] = [];
  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
}

// Parâmetro de tipo com valor padrão
type ApiResponse<T = unknown> = { data: T; status: number };
```

---

## Utility Types

| Utility | Descrição | Exemplo |
|---------|-----------|---------|
| `Partial<T>` | Todas as props opcionais | `Partial<User>` |
| `Required<T>` | Todas as props obrigatórias | `Required<User>` |
| `Readonly<T>` | Todas as props readonly | `Readonly<Config>` |
| `Record<K, V>` | Objeto com chaves K e valores V | `Record<string, number>` |
| `Pick<T, K>` | Subconjunto de props | `Pick<User, "id" \| "name">` |
| `Omit<T, K>` | Excluir props | `Omit<User, "password">` |
| `Exclude<T, U>` | Excluir da union | `Exclude<"a"\|"b"\|"c", "a">` |
| `Extract<T, U>` | Extrair da union | `Extract<"a"\|"b", "a">` |
| `NonNullable<T>` | Remover null/undefined | `NonNullable<string \| null>` |
| `ReturnType<T>` | Tipo de retorno da função | `ReturnType<typeof fn>` |
| `Parameters<T>` | Tuple dos parâmetros da função | `Parameters<typeof fn>` |
| `InstanceType<T>` | Tipo da instância do construtor | `InstanceType<typeof MyClass>` |
| `Awaited<T>` | Desembrulhar Promise | `Awaited<Promise<string>>` → `string` |
| `ConstructorParameters<T>` | Tuple dos params do construtor | — |

---

## Mapped Types

```ts
type Optional<T> = { [K in keyof T]?: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };
type ReadonlyDeep<T> = { readonly [K in keyof T]: T[K] };

// Mapped type condicional
type NoFunctions<T> = {
  [K in keyof T as T[K] extends Function ? never : K]: T[K];
};
```

---

## Tipos Condicionais

```ts
type IsString<T> = T extends string ? true : false;
type Flatten<T> = T extends Array<infer Item> ? Item : T;
type UnpackPromise<T> = T extends Promise<infer U> ? U : T;

// Distributivo
type ToArray<T> = T extends any ? T[] : never;
// ToArray<string | number> => string[] | number[]
```

---

## Template Literal Types

```ts
type EventName<T extends string> = `on${Capitalize<T>}`;
type ClickEvent = EventName<"click">; // "onClick"

type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type Endpoint = `/api/${string}`;
type Route = `${HttpMethod} ${Endpoint}`;
```

---

## Type Guards e Narrowing

```ts
// guard com typeof
function pad(x: string | number) {
  if (typeof x === "string") x.toUpperCase(); // string aqui
}

// guard com instanceof
function handle(x: Date | string) {
  if (x instanceof Date) x.getFullYear();
}

// guard com in
function isAdmin(u: User | Admin): u is Admin {
  return "role" in u;
}

// Type guard customizado
function isString(x: unknown): x is string {
  return typeof x === "string";
}

// Função de assertion
function assert(cond: boolean, msg: string): asserts cond {
  if (!cond) throw new Error(msg);
}
```

---

## Enums

```ts
// Numérico (evite — prefira objeto const)
enum Direction { Up, Down, Left, Right }

// String enum
enum Status { Active = "ACTIVE", Inactive = "INACTIVE" }

// Const enum (substituído em tempo de compilação)
const enum HttpMethod { GET = "GET", POST = "POST" }

// Padrão preferido (sem enum)
const ROLES = { Admin: "admin", User: "user" } as const;
type Role = typeof ROLES[keyof typeof ROLES]; // "admin" | "user"
```

---

## Classes

```ts
class Animal {
  // declaração de propriedade abreviada
  constructor(
    public name: string,
    protected age: number,
    private _id: number
  ) {}

  get id(): number { return this._id; }

  speak(): string { return `${this.name} faz um som`; }
}

class Dog extends Animal {
  constructor(name: string, age: number, id: number, public breed: string) {
    super(name, age, id);
  }

  override speak(): string { return `${this.name} late`; }
}

// Classe abstrata
abstract class Shape {
  abstract area(): number;
  toString(): string { return `Área: ${this.area()}`; }
}
```

---

## Decorators (experimental / Stage 3)

```ts
// tsconfig: "experimentalDecorators": true

// Decorator de classe
function Sealed(constructor: Function) {
  Object.seal(constructor);
}

// Decorator de método
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log(`Chamando ${key} com`, args);
    return original.apply(this, args);
  };
  return descriptor;
}

@Sealed
class MyService {
  @Log
  doWork(x: number): number { return x * 2; }
}
```

---

## Padrões Assíncronos

```ts
// Async/await
async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<User>;
}

// Padrão Result (neverthrow)
import { ok, err, Result } from "neverthrow";

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) return err("Divisão por zero");
  return ok(a / b);
}
```

---

## Padrões Comuns

```ts
// Operador satisfies (TS 4.9)
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
} satisfies Record<string, string | number[]>;
palette.red; // number[] (não string | number[])

// as const
const config = { host: "localhost", port: 5432 } as const;
// typeof config.port => 5432 (literal, não number)

// Coalescência nula e encadeamento opcional
const name = user?.profile?.displayName ?? "Anônimo";

// Asserção de não-nulo (use com moderação)
const el = document.getElementById("app")!;

// Index signature
interface StringMap { [key: string]: string }

// Sobrecargas de função
function process(x: string): string;
function process(x: number): number;
function process(x: string | number): string | number {
  return typeof x === "string" ? x.toUpperCase() : x * 2;
}
```

---

## Referência Rápida do tsconfig

| Opção | Valor | Efeito |
|-------|-------|--------|
| `strict` | `true` | Ativa todas as verificações estritas |
| `target` | `"ES2022"` | Versão do JS de saída |
| `module` | `"NodeNext"` | Sistema de módulos |
| `moduleResolution` | `"NodeNext"` | Resolução de imports |
| `paths` | `{ "@/*": ["src/*"] }` | Aliases de caminho |
| `noUncheckedIndexedAccess` | `true` | Acesso mais seguro a arrays/objetos |
| `exactOptionalPropertyTypes` | `true` | Sem `undefined` em props opcionais |
| `noImplicitAny` | `true` | Incluído em `strict` |
| `strictNullChecks` | `true` | Incluído em `strict` |
| `declaration` | `true` | Emitir arquivos `.d.ts` |
| `sourceMap` | `true` | Emitir source maps |
