# TypeScript Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## Primitive Types

```ts
let n: number = 42;
let s: string = "hello";
let b: boolean = true;
let u: undefined = undefined;
let nl: null = null;
let sym: symbol = Symbol("id");
let big: bigint = 9007199254740991n;
let any: any = "anything";
let unk: unknown = "safer than any";
let nv: never;          // function that never returns
let vd: void;           // function that returns nothing useful
```

---

## Type Aliases & Interfaces

```ts
type Point = { x: number; y: number };
type ID = string | number;
type Callback = (err: Error | null, data: string) => void;

interface User {
  id: number;
  name: string;
  email?: string;           // optional
  readonly createdAt: Date; // immutable
}

// Extending
interface Admin extends User {
  role: "admin" | "superadmin";
}

// Merging (interfaces only)
interface Window { myLib: boolean; }
interface Window { version: string; }
```

**Type vs Interface:** Use `interface` for object shapes (extendable); use `type` for unions, primitives, tuples.

---

## Union & Intersection

```ts
type StringOrNum = string | number;
type AdminUser = User & { role: string };

// Discriminated union
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

// Constraints
function getLength<T extends { length: number }>(x: T): number {
  return x.length;
}

// Multiple type params
function pair<A, B>(a: A, b: B): [A, B] { return [a, b]; }

// Generic interface
interface Repository<T, ID = number> {
  findById(id: ID): Promise<T | null>;
  save(entity: T): Promise<T>;
}

// Generic class
class Stack<T> {
  private items: T[] = [];
  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
}

// Default type parameter
type ApiResponse<T = unknown> = { data: T; status: number };
```

---

## Utility Types

| Utility | Description | Example |
|---------|-------------|---------|
| `Partial<T>` | All props optional | `Partial<User>` |
| `Required<T>` | All props required | `Required<User>` |
| `Readonly<T>` | All props readonly | `Readonly<Config>` |
| `Record<K, V>` | Object with keys K and values V | `Record<string, number>` |
| `Pick<T, K>` | Subset of props | `Pick<User, "id" \| "name">` |
| `Omit<T, K>` | Exclude props | `Omit<User, "password">` |
| `Exclude<T, U>` | Exclude from union | `Exclude<"a"\|"b"\|"c", "a">` |
| `Extract<T, U>` | Extract from union | `Extract<"a"\|"b", "a">` |
| `NonNullable<T>` | Remove null/undefined | `NonNullable<string \| null>` |
| `ReturnType<T>` | Return type of function | `ReturnType<typeof fn>` |
| `Parameters<T>` | Parameters tuple of function | `Parameters<typeof fn>` |
| `InstanceType<T>` | Instance type of constructor | `InstanceType<typeof MyClass>` |
| `Awaited<T>` | Unwrap Promise | `Awaited<Promise<string>>` → `string` |
| `ConstructorParameters<T>` | Constructor params tuple | — |

---

## Mapped Types

```ts
type Optional<T> = { [K in keyof T]?: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };
type ReadonlyDeep<T> = { readonly [K in keyof T]: T[K] };

// Conditional mapped type
type NoFunctions<T> = {
  [K in keyof T as T[K] extends Function ? never : K]: T[K];
};
```

---

## Conditional Types

```ts
type IsString<T> = T extends string ? true : false;
type Flatten<T> = T extends Array<infer Item> ? Item : T;
type UnpackPromise<T> = T extends Promise<infer U> ? U : T;

// Distributive
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

## Type Guards & Narrowing

```ts
// typeof guard
function pad(x: string | number) {
  if (typeof x === "string") x.toUpperCase(); // string here
}

// instanceof guard
function handle(x: Date | string) {
  if (x instanceof Date) x.getFullYear();
}

// in guard
function isAdmin(u: User | Admin): u is Admin {
  return "role" in u;
}

// Custom type guard
function isString(x: unknown): x is string {
  return typeof x === "string";
}

// Assertion function
function assert(cond: boolean, msg: string): asserts cond {
  if (!cond) throw new Error(msg);
}
```

---

## Enums

```ts
// Numeric (avoid — prefer const object)
enum Direction { Up, Down, Left, Right }

// String enum
enum Status { Active = "ACTIVE", Inactive = "INACTIVE" }

// Const enum (inlined at compile time)
const enum HttpMethod { GET = "GET", POST = "POST" }

// Preferred pattern (no enum)
const ROLES = { Admin: "admin", User: "user" } as const;
type Role = typeof ROLES[keyof typeof ROLES]; // "admin" | "user"
```

---

## Classes

```ts
class Animal {
  // shorthand property declaration
  constructor(
    public name: string,
    protected age: number,
    private _id: number
  ) {}

  get id(): number { return this._id; }

  speak(): string { return `${this.name} makes a sound`; }
}

class Dog extends Animal {
  constructor(name: string, age: number, id: number, public breed: string) {
    super(name, age, id);
  }

  override speak(): string { return `${this.name} barks`; }
}

// Abstract class
abstract class Shape {
  abstract area(): number;
  toString(): string { return `Area: ${this.area()}`; }
}
```

---

## Decorators (experimental / Stage 3)

```ts
// tsconfig: "experimentalDecorators": true

// Class decorator
function Sealed(constructor: Function) {
  Object.seal(constructor);
}

// Method decorator
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log(`Calling ${key} with`, args);
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

## Async Patterns

```ts
// Async/await
async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<User>;
}

// Result pattern (neverthrow)
import { ok, err, Result } from "neverthrow";

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) return err("Division by zero");
  return ok(a / b);
}
```

---

## Common Patterns

```ts
// Satisfies operator (TS 4.9)
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
} satisfies Record<string, string | number[]>;
palette.red; // number[] (not string | number[])

// as const
const config = { host: "localhost", port: 5432 } as const;
// typeof config.port => 5432 (literal, not number)

// Nullish coalescing & optional chaining
const name = user?.profile?.displayName ?? "Anonymous";

// Non-null assertion (use sparingly)
const el = document.getElementById("app")!;

// Index signature
interface StringMap { [key: string]: string }

// Function overloads
function process(x: string): string;
function process(x: number): number;
function process(x: string | number): string | number {
  return typeof x === "string" ? x.toUpperCase() : x * 2;
}
```

---

## tsconfig Quick Reference

| Option | Value | Effect |
|--------|-------|--------|
| `strict` | `true` | Enables all strict checks |
| `target` | `"ES2022"` | Output JS version |
| `module` | `"NodeNext"` | Module system |
| `moduleResolution` | `"NodeNext"` | Import resolution |
| `paths` | `{ "@/*": ["src/*"] }` | Path aliases |
| `noUncheckedIndexedAccess` | `true` | Safer array/obj access |
| `exactOptionalPropertyTypes` | `true` | No `undefined` in optional |
| `noImplicitAny` | `true` | Included in `strict` |
| `strictNullChecks` | `true` | Included in `strict` |
| `declaration` | `true` | Emit `.d.ts` files |
| `sourceMap` | `true` | Emit source maps |
