# TypeScript

> TypeScript is JavaScript with a type system. It catches entire categories of bugs at compile time, serves as living documentation, and makes large codebases navigable. This file covers the type system fundamentals, generics, advanced types, and the patterns that make TypeScript actually useful — not just a `any`-sprinkled JavaScript.

---

## 1. What & Why

TypeScript is a statically typed superset of JavaScript created by Microsoft. It compiles to plain JavaScript, so it runs anywhere JavaScript runs. TypeScript adds:
- **Static type checking**: catch type errors before runtime
- **IDE intelligence**: autocomplete, go-to-definition, rename refactoring
- **Self-documenting code**: types serve as always-up-to-date documentation
- **Safe refactoring**: the compiler tells you every call site that breaks when you change a function signature

The TypeScript type system is **structural** (not nominal) and operates at **compile time only** — types are erased when compiled to JavaScript. There is zero runtime overhead.

---

## 2. Core Concepts

### Static vs Dynamic Typing

```
JavaScript (dynamic): types checked at runtime — errors surface when code runs
TypeScript (static):  types checked at compile time — errors surface before code runs
```

```typescript
// JavaScript — error only discovered at runtime
function add(a, b) { return a + b; }
add(1, "2");  // returns "12" silently — no error

// TypeScript — error at compile time
function add(a: number, b: number): number { return a + b; }
add(1, "2");  // Argument of type 'string' is not assignable to parameter of type 'number'
```

### Structural Typing (Duck Typing)

TypeScript uses structural typing: two types are compatible if their structures are compatible, regardless of their names.

```typescript
interface Point2D { x: number; y: number; }
interface Coordinate { x: number; y: number; }

function plot(p: Point2D) { /* ... */ }

const coord: Coordinate = { x: 1, y: 2 };
plot(coord);  // OK — structure matches, even though names differ

// Object literals check MORE strictly (excess property checks)
plot({ x: 1, y: 2, z: 3 });  // Error — 'z' not in Point2D
const p = { x: 1, y: 2, z: 3 };
plot(p);  // OK — structural check, extra properties allowed via variable assignment
```

---

### Primitive Types

```typescript
let name: string = "Alice";
let age: number = 30;         // includes integers, floats, NaN, Infinity
let active: boolean = true;
let id: bigint = 9007199254740991n;
let sym: symbol = Symbol('key');
let nothing: null = null;
let notDefined: undefined = undefined;

// TypeScript infers types from initializers — annotations are often optional
let inferred = "Alice";  // type: string (inferred)
```

---

### Union Types

A value of a union type can be any of the member types:
```typescript
type StringOrNumber = string | number;
let value: StringOrNumber = "hello";
value = 42;  // also fine

// Discriminated union — each member has a literal type field
type Result<T> =
  | { success: true;  value: T }
  | { success: false; error: string };

function divide(a: number, b: number): Result<number> {
  if (b === 0) return { success: false, error: "Division by zero" };
  return { success: true, value: a / b };
}

const r = divide(10, 2);
if (r.success) {
  console.log(r.value);  // TypeScript knows value exists here
} else {
  console.error(r.error);  // TypeScript knows error exists here
}
```

---

### Intersection Types

Combines multiple types — the value must satisfy ALL members:
```typescript
type Serializable = { serialize(): string };
type Loggable = { log(): void };

type SerializableLoggable = Serializable & Loggable;

// Used heavily for mixins and extending types
type UserWithPermissions = User & { permissions: string[] };
```

---

### Literal Types

A type that represents a specific value:
```typescript
type Direction = "north" | "south" | "east" | "west";
type StatusCode = 200 | 201 | 400 | 401 | 403 | 404 | 500;
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

function move(dir: Direction) { /* ... */ }
move("north");  // OK
move("diagonal");  // Error — not a valid Direction

// Template literal types
type EventName = `on${Capitalize<string>}`;
type CssUnit = `${number}px` | `${number}rem` | `${number}%`;
type UserId = `user_${string}`;
```

---

### Interfaces vs Type Aliases

Both define object shapes. The practical differences:

```typescript
// Interface — can be extended by declaration merging, preferred for public APIs
interface User {
  id: number;
  name: string;
  email: string;
}

// Extending an interface
interface AdminUser extends User {
  permissions: string[];
}

// Declaration merging (augmenting an existing interface)
interface Window {
  myCustomProperty: string;
}

// Type alias — required for unions, intersections, tuples, primitives
type ID = string | number;
type Point = [number, number];  // tuple
type Callback = (err: Error | null, result?: string) => void;
type UserOrAdmin = User | AdminUser;

// Both can define object shapes identically for most purposes
type UserType = {
  id: number;
  name: string;
};
```

> 💡 **Rule of thumb**: Use `interface` for object and class shapes that might be extended or augmented. Use `type` for everything else — unions, intersections, tuples, mapped types, and complex computed types.

---

### Generics

Generics allow writing code that works with multiple types while preserving type information:

```typescript
// Generic function — T is inferred from the argument
function identity<T>(value: T): T {
  return value;
}
const s = identity("hello");  // T = string, return type = string
const n = identity(42);        // T = number, return type = number

// Generic constraint — T must have a .length property
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}
longest("abc", "xy");      // OK — strings have length
longest([1, 2, 3], [1]);   // OK — arrays have length
longest(42, 10);            // Error — numbers don't have length

// Multiple type parameters
function zip<A, B>(a: A[], b: B[]): Array<[A, B]> {
  return a.map((item, i) => [item, b[i]]);
}
const zipped = zip([1, 2, 3], ["a", "b", "c"]);  // [number, string][]

// Generic interface
interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: string): Promise<void>;
}
```

---

### Conditional Types

Types that resolve to one type or another based on a condition:

```typescript
type IsArray<T> = T extends any[] ? true : false;
type A = IsArray<string[]>;  // true
type B = IsArray<string>;    // false

// Distributive conditional types — applied to each member of a union
type ToArray<T> = T extends any ? T[] : never;
type StrOrNumArr = ToArray<string | number>;  // string[] | number[]

// The `infer` keyword — extract a type from a conditional
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type Awaited<T> = T extends Promise<infer U> ? U : T;

type Fn = () => string;
type R = ReturnType<Fn>;  // string

type GetFirst<T> = T extends [infer First, ...any[]] ? First : never;
type First = GetFirst<[string, number, boolean]>;  // string
```

---

### Mapped Types

Transform all properties of a type using a mapping:

```typescript
// Make all properties optional (same as built-in Partial<T>)
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

// Make all properties readonly
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Pick specific keys
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Remap keys with 'as'
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type UserGetters = Getters<{ name: string; age: number }>;
// { getName: () => string; getAge: () => number }
```

---

### Utility Types

TypeScript's built-in generic utility types:

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Partial<T> — all properties optional
type UserUpdate = Partial<User>;

// Required<T> — all properties required (opposite of Partial)
type StrictUser = Required<User>;

// Pick<T, K> — keep only specified keys
type PublicUser = Pick<User, 'id' | 'name' | 'email'>;

// Omit<T, K> — remove specified keys
type SafeUser = Omit<User, 'password'>;

// Record<K, V> — object with keys K and values V
type UserById = Record<string, User>;
const cache: Record<string, User> = {};

// Readonly<T> — all properties readonly
type ImmutableUser = Readonly<User>;

// ReturnType<T> — extract return type of a function
async function fetchUser(id: string): Promise<User> { /* ... */ }
type FetchResult = Awaited<ReturnType<typeof fetchUser>>;  // User

// Parameters<T> — extract parameter types as a tuple
type FetchParams = Parameters<typeof fetchUser>;  // [string]

// NonNullable<T> — remove null and undefined from a type
type MaybeUser = User | null | undefined;
type DefiniteUser = NonNullable<MaybeUser>;  // User

// Extract<T, U> — keep only types from T that are assignable to U
type StringOrNumber = string | number | boolean;
type OnlyStrings = Extract<StringOrNumber, string>;  // string

// Exclude<T, U> — remove types from T that are assignable to U
type NoStrings = Exclude<StringOrNumber, string>;  // number | boolean
```

---

### Type Narrowing

TypeScript narrows the type within conditional blocks based on checks:

```typescript
function processInput(input: string | number | null) {
  // typeof narrowing
  if (typeof input === 'string') {
    console.log(input.toUpperCase());  // input: string here
  } else if (typeof input === 'number') {
    console.log(input.toFixed(2));     // input: number here
  } else {
    console.log('null input');         // input: null here
  }
}

// instanceof narrowing
function formatError(err: unknown) {
  if (err instanceof Error) {
    return err.message;  // err: Error here
  }
  return String(err);
}

// in operator narrowing (discriminated unions)
type Cat = { kind: 'cat'; meow(): void };
type Dog = { kind: 'dog'; bark(): void };

function makeSound(animal: Cat | Dog) {
  if (animal.kind === 'cat') {
    animal.meow();  // animal: Cat here
  } else {
    animal.bark();  // animal: Dog here
  }
}

// Type predicate (user-defined type guard)
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
}

// Assertion function (throws if type does not match)
function assertIsString(val: unknown): asserts val is string {
  if (typeof val !== 'string') {
    throw new TypeError(`Expected string, got ${typeof val}`);
  }
}
```

---

### `unknown` vs `any` vs `never`

```typescript
// any — opts OUT of type checking (unsafe escape hatch)
// Avoid: type errors in `any` code propagate silently
let x: any = 42;
x.toUpperCase();  // no error at compile time — crashes at runtime!

// unknown — type-safe alternative to any
// Must narrow before using
let y: unknown = fetchSomething();
y.toUpperCase();  // Error — must narrow first
if (typeof y === 'string') {
  y.toUpperCase();  // OK — narrowed to string
}

// never — represents a value that can never exist
// Used for: exhaustive checks, unreachable code, bottom type
function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(x)}`);
}

type Shape = 'circle' | 'square' | 'triangle';
function area(shape: Shape): number {
  switch (shape) {
    case 'circle': return Math.PI;
    case 'square': return 1;
    case 'triangle': return 0.5;
    default: return assertNever(shape);  // Compile error if a case is missing
  }
}
```

> 💡 **When to use `unknown`**: Whenever you have a value whose type you do not know (API responses, `JSON.parse` results, `catch` clause errors). It forces you to check before using, making it safe. Use `any` only as a last resort when migrating from JavaScript.

---

## 3. How It Works

### tsconfig.json Key Options

```json
{
  "compilerOptions": {
    "target": "ES2022",            // output JS version
    "module": "ESNext",            // module system (CommonJS, ESNext, etc.)
    "moduleResolution": "bundler", // how modules are resolved (Node16, Bundler)
    "strict": true,                // enables all strict checks (highly recommended)
    "noUncheckedIndexedAccess": true, // arr[0] is T | undefined, not T
    "exactOptionalPropertyTypes": true, // { x?: string } — x can be string or absent, not undefined
    "noImplicitReturns": true,     // function must return in all code paths
    "noImplicitOverride": true,    // override keyword required when overriding class members
    "paths": {                     // path aliases (use with bundler)
      "@/*": ["./src/*"]
    },
    "baseUrl": ".",
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,           // generate .d.ts files
    "sourceMap": true
  }
}
```

> ⚠️ **Always enable `strict: true`**. It enables: `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictPropertyInitialization`, and others. Disabling strict mode eliminates most of TypeScript's value.

---

## 4. Code Examples

### Generic fetch wrapper with inferred return type
```typescript
async function fetchJson<T>(url: string, options?: RequestInit): Promise<T> {
  const res = await fetch(url, options);

  if (!res.ok) {
    throw new Error(`HTTP ${res.status}: ${res.statusText}`);
  }

  return res.json() as Promise<T>;
}

// Usage — T is explicitly provided
interface Post { id: number; title: string; body: string; }
const post = await fetchJson<Post>('/api/posts/1');
console.log(post.title);  // TypeScript knows post has title: string
```

### Type-safe event emitter
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

// Usage
interface AppEvents {
  userLogin: { userId: string; timestamp: Date };
  userLogout: { userId: string };
  error: { message: string; code: number };
}

const emitter = new TypedEventEmitter<AppEvents>();
emitter.on('userLogin', ({ userId, timestamp }) => {
  console.log(`User ${userId} logged in at ${timestamp}`);
});
emitter.emit('userLogin', { userId: '123', timestamp: new Date() });
emitter.emit('error', { message: 'oops', code: 500 });
// emitter.emit('unknown', {});  // Error — 'unknown' not in AppEvents
```

### DeepReadonly utility type
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

// config.server.host = 'prod';     // Error — deeply readonly
// config.features.push('newFeat'); // Error — ReadonlyArray
```

### Redux-style reducer with discriminated union
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
      // Exhaustive check — TypeScript errors if a case is missing
      const _exhaustive: never = action;
      return state;
  }
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Overusing `any`**: Once you cast something to `any`, TypeScript stops checking it — and any property access or method call on it is unchecked. Errors introduced via `any` propagate silently. Use `unknown` instead and narrow before use.

> ⚠️ **Casting with `as` to silence errors**: `value as TargetType` is a type assertion. It overrides TypeScript's inference. It does NOT change the runtime value. Using it to silence errors ("the compiler is wrong") is often masking a real bug.
> ```typescript
> const user = JSON.parse(data) as User;  // dangerous — no runtime guarantee
> // Better: use a runtime validator (zod, valibot) and then type the result
> const user = UserSchema.parse(JSON.parse(data));  // validated at runtime
> ```

> ⚠️ **Not enabling `strict` mode**: Without `strictNullChecks`, `null` and `undefined` are assignable to every type, which defeats a huge part of TypeScript's safety. Without `noImplicitAny`, TypeScript silently infers `any` for untyped parameters.

> ⚠️ **Ignoring indexed access types with `noUncheckedIndexedAccess`**: Without this option, `arr[0]` is typed as `T`, not `T | undefined`. If the array is empty, you get a runtime error. Enable `noUncheckedIndexedAccess` to make indexed access return `T | undefined`, forcing you to handle the undefined case.

> ⚠️ **Interface vs type confusion**: Choosing incorrectly is not a runtime bug, but it causes friction. The key difference: interfaces can be augmented via declaration merging (important for library types), type aliases cannot. Use interfaces for things external code will extend.

---

## 6. When to Use / Not Use

**Use TypeScript when:**
- Building anything non-trivial in size or team
- Public APIs or libraries (`.d.ts` files provide consumers with types)
- Projects that need confident refactoring
- Code that will be maintained beyond 3 months

**Use `unknown` when:**
- Receiving data from outside (API responses, JSON.parse, catch clauses in TypeScript 4+)

**Use `never` when:**
- Building exhaustive switch/if checks over discriminated unions
- Signalling code paths that should never be reached
- The bottom type in utility type computations

**Use `type` alias when:**
- Defining unions, intersections, tuples, or mapped types
- The type will never need declaration merging

**Use `interface` when:**
- Defining the shape of objects or classes
- Building library types that consumers might extend
- Working with class implements declarations

---

## 7. Real-World Scenario

### Type-safe API layer with Zod validation

```typescript
import { z } from 'zod';

// Define schema — Zod validates at runtime AND infers the TypeScript type
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

// Type is inferred from schema — no duplication
type User = z.infer<typeof UserSchema>;
type PaginatedUsers = z.infer<typeof PaginatedUsersSchema>;

// Generic validated fetcher
async function fetchValidated<T>(
  url: string,
  schema: z.ZodType<T>,
  options?: RequestInit
): Promise<T> {
  const res = await fetch(url, options);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);

  const raw = await res.json();
  return schema.parse(raw);  // throws ZodError if shape doesn't match
}

// Usage — fully typed, validated at runtime
async function getUsers(page: number): Promise<PaginatedUsers> {
  return fetchValidated(
    `/api/users?page=${page}`,
    PaginatedUsersSchema
  );
}

const result = await getUsers(1);
result.users.forEach((user) => {
  // user is typed as User — autocompletion works
  console.log(user.email);    // string
  console.log(user.role);     // 'admin' | 'user' | 'moderator'
  console.log(user.createdAt); // Date (coerced from ISO string)
});
```

---

## 8. Interview Questions

**Q1: What is the difference between an interface and a type alias?**

A: Both can define object shapes and are largely interchangeable for that purpose. Key differences: (1) Type aliases can represent any type — unions, intersections, tuples, primitives, mapped types. Interfaces can only represent object shapes and class contracts. (2) Interfaces support declaration merging — you can add properties to an existing interface from a different file (useful for augmenting library types). Type aliases cannot be re-opened. (3) Error messages from interfaces tend to show the interface name; type alias errors often expand the definition. Rule of thumb: use interface for object/class shapes, use type for everything else.

---

**Q2: What is structural typing?**

A: TypeScript's type system is structural (duck typing): two types are compatible if their shapes are compatible, regardless of their names or how they were defined. A value of type `Cat { meow(): void }` satisfies `type Animal { meow(): void }` even without explicit `implements`. This contrasts with nominal typing (Java, C#) where types are only compatible if they are explicitly in the same class hierarchy. Practical consequence: you can pass any object to a function as long as it has the required properties — no explicit interface implementation needed.

---

**Q3: Explain generics. When would you use them?**

A: Generics allow writing functions, classes, and interfaces that work with multiple types while preserving type relationships. A generic type parameter (`<T>`) is a placeholder resolved when the generic is used. Use generics when: (1) you have a function that returns the same type as its input (`identity<T>(x: T): T`); (2) you have a container type whose element type varies (`Array<T>`, `Promise<T>`, `Repository<T>`); (3) you have a function where input and output types are related but not fixed. Without generics, you would use `any` and lose type safety, or write separate functions for each type.

---

**Q4: What is `never` used for?**

A: `never` represents values that can never exist — the bottom type. Uses: (1) Exhaustive checks in discriminated unions — if you add a new union member but forget to handle it in a switch, the `assertNever(x: never)` call produces a compile error. (2) Functions that never return (`throw` or infinite loop): `function fail(msg: string): never { throw new Error(msg); }`. (3) Type manipulation where certain branches should be eliminated: `Exclude<T, U>` uses `never` to remove types. (4) In conditional types: `type NonNullable<T> = T extends null | undefined ? never : T`.

---

**Q5: How does type narrowing work?**

A: TypeScript analyzes control flow to narrow union types within conditional branches. You narrow using: `typeof` (narrows to primitive types), `instanceof` (narrows to class instances), `in` operator (narrows to objects with a property), discriminated union checks (checking a literal-type field), truthiness checks (narrows out null/undefined), and type predicates (user-defined `x is T` return type). Once narrowed, TypeScript allows operations only valid for the narrowed type. After the conditional block, the type widens back to the union.

---

**Q6: What are utility types? Name five.**

A: Utility types are generic helpers built into TypeScript for common type transformations. Five important ones: (1) `Partial<T>` — makes all properties optional; (2) `Required<T>` — makes all properties required; (3) `Pick<T, K>` — creates a type with only the specified keys; (4) `Omit<T, K>` — creates a type without the specified keys; (5) `Record<K, V>` — creates an object type with keys K and values V. Others worth knowing: `Readonly<T>`, `ReturnType<T>`, `Parameters<T>`, `Awaited<T>`, `NonNullable<T>`.

---

**Q7: What is the difference between `unknown` and `any`?**

A: Both represent "a value of any type". The difference is how safely you can use them. With `any`, you can do anything — call methods, access properties, assign anywhere — TypeScript stops checking. It is a complete type system escape hatch. With `unknown`, you cannot do anything with the value until you narrow its type with a check (`typeof`, `instanceof`, a type predicate). This forces you to handle the unknown type safely. Use `unknown` for values whose type you genuinely do not know at declaration time (API responses, `JSON.parse`, catch clause variables). Prefer `unknown` over `any` everywhere you would have reached for `any`.

---

**Q8: What is a discriminated union?**

A: A discriminated union (also called tagged union) is a union type where each member has a common property with a unique literal type — the discriminant. TypeScript uses this literal property to narrow the type in conditional branches. Example: `type Action = { type: 'ADD'; item: string } | { type: 'REMOVE'; id: number }`. The `type` field is the discriminant. In a switch on `action.type`, TypeScript knows exactly which member of the union you are in and gives you access to the correct properties. This pattern is the foundation of Redux reducers and state machines in TypeScript.

---

## 9. Exercises

**Exercise 1 — Generic fetch wrapper:**

Write a `fetchJson<T>(url: string, options?: RequestInit): Promise<T>` function that:
- Fetches a URL and parses JSON
- Returns `Promise<T>` where `T` is provided by the caller
- Throws a typed `HttpError` class with `status: number` and `body: string` on non-2xx responses
- Has a second overload `fetchJson<T, E>(url, schema: ZodSchema<T>): Promise<T>` that validates the response

---

**Exercise 2 — Type-safe event emitter:**

Build a `TypedEventEmitter<Events extends Record<string, unknown>>` class that:
- `on<K extends keyof Events>(event: K, listener: (data: Events[K]) => void): this`
- `off<K extends keyof Events>(event: K, listener: ...): this`
- `emit<K extends keyof Events>(event: K, data: Events[K]): void`
- TypeScript must error on `emit('unknownEvent', ...)` and `on('unknownEvent', ...)`

---

**Exercise 3 — DeepReadonly:**

Implement `DeepReadonly<T>` that recursively makes all properties (and nested properties) readonly:
```typescript
type DeepReadonly<T> = /* your implementation */

// Must satisfy:
type Config = { server: { host: string; port: number }; features: string[] };
type ImmutableConfig = DeepReadonly<Config>;
// ImmutableConfig.server.host must be readonly
// ImmutableConfig.features must be ReadonlyArray<string>
```

---

**Exercise 4 — Type a Redux-style reducer:**

Given a TodoState and a set of actions (ADD_TODO, TOGGLE_TODO, REMOVE_TODO, SET_FILTER), write:
- A discriminated union type `TodoAction` for all actions
- A `todoReducer(state: TodoState, action: TodoAction): TodoState` that handles all cases
- An exhaustive check so TypeScript errors if you add a new action type but forget to handle it

---

## 10. Further Reading

- **TypeScript official handbook**: https://www.typescriptlang.org/docs/handbook/intro.html
- **TypeScript playground**: https://www.typescriptlang.org/play — try types interactively in the browser
- **"Effective TypeScript" by Dan Vanderkam** — 62 specific ways to improve your TypeScript
- **Total TypeScript** by Matt Pocock: https://www.totaltypescript.com — the best structured course on advanced TypeScript
- **Type Challenges**: https://github.com/type-challenges/type-challenges — practice implementing utility types
- **Zod**: https://zod.dev — runtime validation that infers TypeScript types (excellent pairing with TypeScript)
- **"Programming TypeScript" by Boris Cherny** (O'Reilly) — deep dive into the type system
- **TypeScript Compiler API**: https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API — for building tools that work with TypeScript ASTs
- **DefinitelyTyped**: https://github.com/DefinitelyTyped/DefinitelyTyped — community type definitions for JavaScript libraries
