# JavaScript

> JavaScript is the only language that runs natively in every browser. Understanding it deeply — the event loop, closures, prototypes, `this`, and async patterns — separates developers who debug effectively from those who cargo-cult solutions from Stack Overflow.

---

## 1. What & Why

JavaScript was created in 10 days in 1995 and became the universal language of the web. Today it runs in browsers, servers (Node.js), mobile (React Native), desktop (Electron), IoT devices, and cloud workers. Despite its ubiquity, many developers use it without understanding its fundamental mechanics.

Understanding JavaScript's runtime model lets you:
- Reason about execution order in async code
- Avoid classic bugs like closure captures in loops and `this` in callbacks
- Write correct, performant async code with Promises and async/await
- Debug confusing behaviour in the browser and Node.js
- Use ES6+ features correctly

---

## 2. Core Concepts

### The Event Loop

JavaScript is single-threaded — it can only execute one piece of code at a time. The event loop is the mechanism that allows non-blocking I/O despite this constraint.

**The four components:**

1. **Call Stack**: where function calls live. LIFO (Last In, First Out). Synchronous code runs here.
2. **Web APIs / Node APIs**: browser/runtime-provided APIs (setTimeout, fetch, DOM events, fs.readFile). They handle async operations outside the JS thread.
3. **Macrotask Queue** (callback queue): completed callbacks from Web APIs (setTimeout, setInterval, I/O callbacks, UI events) queue here.
4. **Microtask Queue**: Promise callbacks (`.then`, `.catch`, `.finally`), `queueMicrotask()`, and `MutationObserver` callbacks. **Microtasks always drain completely before the next macrotask runs.**

**Execution order rule:** Call stack empties → drain ALL microtasks → run ONE macrotask → drain ALL microtasks → repeat.

**ASCII walkthrough:**
```javascript
console.log('1');           // sync

setTimeout(() => {
  console.log('4');         // macrotask — queued after 0ms delay
}, 0);

Promise.resolve()
  .then(() => console.log('3'));   // microtask

console.log('2');           // sync
```

```
Step 1: console.log('1')    → Stack: [log('1')]        → Output: 1
Step 2: setTimeout          → Stack: [setTimeout]      → Registers callback in Web API
Step 3: Promise.resolve     → Stack: [Promise.resolve] → .then callback → Microtask Queue
Step 4: console.log('2')    → Stack: [log('2')]        → Output: 2

Call stack now empty.
→ Drain microtasks: console.log('3')           → Output: 3
→ Microtask queue empty.
→ Run one macrotask: console.log('4')          → Output: 4

Final output: 1  2  3  4
```

> 💡 `Promise.resolve().then()` always runs before `setTimeout(fn, 0)` even though setTimeout says 0ms. Microtasks drain first.

---

### Hoisting

Hoisting is the JavaScript mechanism that moves declarations to the top of their scope before execution.

**`var` — declaration hoisted, initialized to `undefined`:**
```javascript
console.log(x);  // undefined (not ReferenceError)
var x = 5;
console.log(x);  // 5

// Equivalent to:
var x;           // hoisted, initialized to undefined
console.log(x);  // undefined
x = 5;
console.log(x);  // 5
```

**Function declarations — fully hoisted (declaration + body):**
```javascript
sayHello();  // works! → "Hello"

function sayHello() {
  console.log("Hello");
}
```

**`let` and `const` — hoisted but NOT initialized (Temporal Dead Zone):**
```javascript
console.log(y);  // ReferenceError: Cannot access 'y' before initialization
let y = 10;

// The declaration IS hoisted (JS knows y exists in scope)
// but it is in the TDZ until the declaration line is reached
```

**Temporal Dead Zone (TDZ):** The period between the start of a block scope and the `let`/`const` declaration line. Accessing the variable in the TDZ throws a `ReferenceError`.

```javascript
{
  // TDZ for z starts here
  typeof z;  // ReferenceError (even typeof does not escape TDZ for let/const)
  let z = 5; // TDZ ends here
}
```

---

### Closures

A closure is a function that **retains access to its outer scope's variables** even after the outer function has returned.

JavaScript uses **lexical scoping**: a function's scope is determined by where it is defined, not where it is called.

```javascript
function createCounter(start = 0) {
  let count = start;  // this variable is in the closure

  return {
    increment() { count++; },
    decrement() { count--; },
    value() { return count; },
  };
}

const counter = createCounter(10);
counter.increment();
counter.increment();
console.log(counter.value());  // 12
// `count` is private — inaccessible from outside, but the methods can access it
```

**Practical uses:**
```javascript
// 1. Module pattern — private state
function createCache() {
  const store = new Map();
  return {
    get: (key) => store.get(key),
    set: (key, val) => store.set(key, val),
    clear: () => store.clear(),
  };
}

// 2. Factory functions — parameterized behaviour
const multiply = (factor) => (n) => n * factor;
const double = multiply(2);
const triple = multiply(3);
console.log(double(5));  // 10
console.log(triple(5));  // 15

// 3. Memoization — cache results using closure
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
```

**Closure pitfall in loops with `var`:**
```javascript
// Bug — all callbacks share the same `i` (var is function-scoped)
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);  // prints 3, 3, 3
}

// Fix 1: use let (block-scoped, separate binding per iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);  // prints 0, 1, 2
}

// Fix 2: use IIFE (create new scope per iteration)
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 100);
  })(i);
}
```

---

### `this` Binding

`this` refers to the execution context of a function call. It has four binding rules, applied in order of precedence:

**1. Default binding (lowest precedence):**
```javascript
function show() { console.log(this); }
show();  // window (browser) or global (Node) — undefined in strict mode
```

**2. Implicit binding (method call):**
```javascript
const obj = {
  name: 'Alice',
  greet() { console.log(this.name); },
};
obj.greet();  // 'Alice' — this = obj

// Pitfall: detaching the method loses the binding
const fn = obj.greet;
fn();  // undefined — this = global (or undefined in strict)
```

**3. Explicit binding (call/apply/bind):**
```javascript
function greet(greeting) { console.log(`${greeting}, ${this.name}`); }
const user = { name: 'Bob' };

greet.call(user, 'Hello');    // Hello, Bob — this = user, args as list
greet.apply(user, ['Hi']);    // Hi, Bob — this = user, args as array
const boundGreet = greet.bind(user);
boundGreet('Hey');            // Hey, Bob — returns new function with fixed this
```

**4. `new` binding (highest precedence):**
```javascript
function Person(name) {
  this.name = name;  // this = newly created object
}
const alice = new Person('Alice');
console.log(alice.name);  // 'Alice'
```

**Arrow functions — no own `this`:**
Arrow functions inherit `this` from their enclosing lexical scope. They cannot be bound with `call`/`apply`/`bind` (the `this` argument is ignored).

```javascript
const obj = {
  name: 'Alice',
  greetLater() {
    setTimeout(() => {
      console.log(this.name);  // 'Alice' — arrow inherits this from greetLater
    }, 1000);
  },
  greetLaterBroken() {
    setTimeout(function() {
      console.log(this.name);  // undefined — regular function, this = global
    }, 1000);
  },
};
```

---

### Promises

A Promise represents the eventual result of an async operation. It has three states:
- **pending**: initial state, neither fulfilled nor rejected
- **fulfilled**: operation succeeded, has a value
- **rejected**: operation failed, has a reason

```javascript
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve(42), 1000);
});

p.then((value) => console.log(value))   // 42
 .catch((err) => console.error(err))
 .finally(() => console.log('done'));
```

**Promise combinators:**
```javascript
// Promise.all — all must succeed; rejects on first failure
const [user, posts] = await Promise.all([
  fetchUser(id),
  fetchPosts(id),
]);

// Promise.allSettled — waits for all, never rejects
const results = await Promise.allSettled([
  fetchUser(id),
  fetchPosts(id),
]);
results.forEach((r) => {
  if (r.status === 'fulfilled') console.log(r.value);
  else console.error(r.reason);
});

// Promise.race — resolves/rejects with the first to settle
const result = await Promise.race([
  fetch(url),
  new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), 5000)),
]);

// Promise.any — resolves with first success; rejects only if ALL reject
const fastest = await Promise.any([mirror1, mirror2, mirror3]);
```

---

### async/await

`async`/`await` is syntactic sugar over Promises. An `async` function always returns a Promise. `await` pauses execution until the Promise settles.

```javascript
async function fetchUserData(id) {
  try {
    const response = await fetch(`/api/users/${id}`);

    if (!response.ok) {
      throw new Error(`HTTP error ${response.status}`);
    }

    const user = await response.json();
    return user;
  } catch (err) {
    console.error('Failed to fetch user:', err);
    throw err;  // re-throw so callers can handle it
  }
}

// Sequential (each waits for the previous)
async function sequential() {
  const user = await fetchUser(1);   // waits ~200ms
  const posts = await fetchPosts(1); // waits ~200ms after user
  // Total: ~400ms
}

// Parallel (start both simultaneously)
async function parallel() {
  const [user, posts] = await Promise.all([
    fetchUser(1),
    fetchPosts(1),
  ]);
  // Total: ~200ms (both run concurrently)
}
```

> ⚠️ **Forgetting `await` inside loops**: `forEach` does not await Promises. Use `for...of` instead.
> ```javascript
> // Wrong — all fire simultaneously, errors may be lost
> items.forEach(async (item) => { await process(item); });
>
> // Correct — sequential
> for (const item of items) { await process(item); }
>
> // Correct — parallel
> await Promise.all(items.map((item) => process(item)));
> ```

---

## 3. How It Works

### Scope Chain

When JavaScript resolves a variable name, it walks up the scope chain from the current scope outward:
```
local scope → closure scope(s) → module scope → global scope
```

```javascript
const global = 'global';

function outer() {
  const outerVar = 'outer';

  function inner() {
    const innerVar = 'inner';
    console.log(innerVar);  // found in local scope
    console.log(outerVar);  // found in outer (closure) scope
    console.log(global);    // found in global scope
    console.log(unknown);   // ReferenceError — not found in any scope
  }

  inner();
}
```

### Prototypal Inheritance

Every JavaScript object has an internal `[[Prototype]]` slot pointing to another object (or null). When you access a property, JS walks the prototype chain until it finds it or hits null.

```javascript
const animal = {
  breathe() { return 'breathing'; },
};

const dog = Object.create(animal);  // dog's [[Prototype]] = animal
dog.bark = function() { return 'woof'; };

console.log(dog.bark());     // found on dog directly
console.log(dog.breathe());  // not on dog → found on animal (prototype)
console.log(dog.toString()); // not on dog/animal → found on Object.prototype

// Class syntax is syntactic sugar over prototypal inheritance
class Animal {
  breathe() { return 'breathing'; }
}
class Dog extends Animal {
  bark() { return 'woof'; }
}
const rex = new Dog();
// rex.__proto__ === Dog.prototype
// Dog.prototype.__proto__ === Animal.prototype
// Animal.prototype.__proto__ === Object.prototype
// Object.prototype.__proto__ === null
```

---

## 4. Code Examples

### Event delegation
```javascript
// Instead of attaching listeners to 1000 list items,
// attach one listener to the parent and check the target
document.querySelector('#item-list').addEventListener('click', (event) => {
  const item = event.target.closest('[data-item-id]');
  if (!item) return;

  const id = item.dataset.itemId;
  handleItemClick(id);
});
```

### Debounce implementation
```javascript
function debounce(fn, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

const onSearch = debounce((query) => {
  fetchResults(query);
}, 300);

searchInput.addEventListener('input', (e) => onSearch(e.target.value));
```

### Throttle implementation
```javascript
function throttle(fn, interval) {
  let lastCall = 0;
  return function(...args) {
    const now = Date.now();
    if (now - lastCall >= interval) {
      lastCall = now;
      return fn.apply(this, args);
    }
  };
}

const onScroll = throttle(() => { /* expensive calculation */ }, 100);
window.addEventListener('scroll', onScroll);
```

### Memoize with WeakMap for object keys
```javascript
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const expensive = memoize((n) => {
  let result = 0;
  for (let i = 0; i <= n; i++) result += i;
  return result;
});

console.log(expensive(1000000));  // computed
console.log(expensive(1000000));  // from cache
```

### Promise.all from scratch
```javascript
function myPromiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) return resolve([]);

    const results = new Array(promises.length);
    let remaining = promises.length;

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then((value) => {
          results[index] = value;
          remaining--;
          if (remaining === 0) resolve(results);
        })
        .catch(reject);  // first rejection rejects all
    });
  });
}
```

### ES6+ features
```javascript
// Destructuring with rename and defaults
const { name: userName = 'Anonymous', age = 0 } = user;
const [first, second = 'default', ...rest] = array;

// Optional chaining — short-circuits on null/undefined
const city = user?.address?.city;
const firstTag = article?.tags?.[0];
const result = user?.getPreferences?.();

// Nullish coalescing — only falls back on null/undefined (not 0, '', false)
const port = config.port ?? 3000;
const theme = userPreference ?? systemPreference ?? 'light';

// Logical assignment
user.name ??= 'Anonymous';    // assign only if null/undefined
config.debug ||= false;       // assign only if falsy
cache.data &&= transform(cache.data);  // assign only if truthy

// Spread in various contexts
const merged = { ...defaults, ...overrides };
const combined = [...arr1, ...arr2];
function log(...args) { console.log(...args); }
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **`var` in loops**: `var` is function-scoped, not block-scoped. All loop iterations share the same binding. Use `let` or `const` in loops — they create a new binding per iteration.

> ⚠️ **Forgetting to `return` in `.then()`**: If you forget to return a value from a `.then()` callback, the next `.then()` in the chain receives `undefined`.
> ```javascript
> fetch('/api/data')
>   .then((res) => res.json())  // must return the Promise
>   .then((data) => console.log(data));  // receives parsed JSON
>
> fetch('/api/data')
>   .then((res) => { res.json(); })  // no return — undefined passed on!
>   .then((data) => console.log(data));  // data is undefined
> ```

> ⚠️ **Unhandled Promise rejections**: Every Promise chain must have a `.catch()` or be inside a `try/catch` if using `await`. Node.js will crash on unhandled rejections in future versions.

> ⚠️ **`this` in callbacks**: Passing a method as a callback loses its `this` binding. Fix: use an arrow function or `.bind(this)`.
> ```javascript
> class Timer {
>   constructor() { this.count = 0; }
>   start() {
>     setInterval(this.tick, 1000);  // wrong — this.tick loses `this`
>     setInterval(() => this.tick(), 1000);  // correct — arrow preserves this
>     setInterval(this.tick.bind(this), 1000);  // also correct
>   }
>   tick() { this.count++; }
> }
> ```

> ⚠️ **`async` function always returns a Promise**: Even if you `return 42` from an async function, the caller gets `Promise<42>`. You must `await` the result or chain `.then()`.

> ⚠️ **`==` vs `===`**: `==` performs type coercion (`0 == false` is `true`, `null == undefined` is `true`). `===` never coerces types. Always use `===` unless you explicitly need type coercion (rare).

> ⚠️ **JSON.stringify with circular references**: Throws `TypeError: Converting circular structure to JSON`. Use a library or provide a replacer function.

---

## 6. When to Use / Not Use

**Use `const` by default, `let` when reassignment is needed, never `var`.**

**Use `async/await` over raw `.then()` chains** for readability, except when you explicitly need the Promise object for combinators.

**Use `Promise.all`** when you have multiple independent async operations — do not `await` them sequentially if they can run in parallel.

**Use `Promise.allSettled`** when you need all results regardless of individual failures (e.g., batch operations where partial failure is acceptable).

**Use event delegation** when attaching identical listeners to many elements — attach one listener to the parent instead.

**Use closures** for encapsulating private state, factory functions, and memoization.

**Do not use `arguments` object** in new code — use rest parameters `...args` instead. `arguments` does not exist in arrow functions.

---

## 7. Real-World Scenario

### Implementing a resilient API client with retry logic

```javascript
class ApiClient {
  #baseUrl;
  #maxRetries;

  constructor(baseUrl, options = {}) {
    this.#baseUrl = baseUrl;
    this.#maxRetries = options.maxRetries ?? 3;
  }

  async #fetchWithRetry(url, options, attempt = 1) {
    try {
      const response = await fetch(url, {
        ...options,
        signal: AbortSignal.timeout(10000),  // 10s timeout
      });

      if (response.status === 429) {
        const retryAfter = Number(response.headers.get('Retry-After') ?? 1);
        await this.#sleep(retryAfter * 1000);
        return this.#fetchWithRetry(url, options, attempt + 1);
      }

      if (!response.ok) {
        throw new ApiError(response.status, await response.text());
      }

      return response;
    } catch (err) {
      if (attempt >= this.#maxRetries) throw err;

      // Retry on network errors (not 4xx client errors)
      if (err instanceof ApiError && err.status < 500) throw err;

      const delay = Math.min(1000 * 2 ** attempt, 30000);  // exponential backoff
      console.warn(`Attempt ${attempt} failed, retrying in ${delay}ms`);
      await this.#sleep(delay);
      return this.#fetchWithRetry(url, options, attempt + 1);
    }
  }

  #sleep(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  async get(path) {
    const res = await this.#fetchWithRetry(`${this.#baseUrl}${path}`, {
      method: 'GET',
      headers: { Accept: 'application/json' },
    });
    return res.json();
  }

  async post(path, body) {
    const res = await this.#fetchWithRetry(`${this.#baseUrl}${path}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Accept: 'application/json',
      },
      body: JSON.stringify(body),
    });
    return res.json();
  }
}

class ApiError extends Error {
  constructor(status, message) {
    super(message);
    this.status = status;
    this.name = 'ApiError';
  }
}

// Usage
const api = new ApiClient('https://api.example.com', { maxRetries: 3 });

async function loadDashboard(userId) {
  // Parallel fetching for independent data
  const [user, stats, notifications] = await Promise.all([
    api.get(`/users/${userId}`),
    api.get(`/users/${userId}/stats`),
    api.get(`/users/${userId}/notifications`),
  ]);

  return { user, stats, notifications };
}
```

---

## 8. Interview Questions

**Q1: What is the JavaScript event loop?**

A: JavaScript is single-threaded. The event loop is the mechanism enabling non-blocking async code. It has a call stack (where synchronous execution happens), Web APIs (browser provides setTimeout, fetch, DOM events — they operate outside JS thread), a microtask queue (Promise callbacks, queueMicrotask), and a macrotask queue (setTimeout, setInterval, I/O). The loop rule: run synchronous code until stack is empty → drain ALL microtasks → run ONE macrotask → drain ALL microtasks → repeat. This is why `Promise.resolve().then()` always fires before `setTimeout(fn, 0)`.

---

**Q2: What is the difference between `==` and `===`?**

A: `===` is strict equality — no type coercion. Both value and type must match. `==` is loose equality — performs type coercion before comparison. Quirks: `0 == false` is `true`, `'' == false` is `true`, `null == undefined` is `true`, but `null == 0` is `false`. Always use `===` to avoid unexpected coercions. The only intentional use of `==` is `x == null` to check for both `null` and `undefined` at once.

---

**Q3: What is a closure?**

A: A closure is a function that retains access to variables in its outer lexical scope even after the outer function has returned. This works because JavaScript uses lexical scoping — a function's scope is determined by where it is defined, not where it is called. Practical uses: encapsulating private state (module pattern), factory functions that parameterize behaviour, memoization, and event handlers that need access to data from when they were created.

---

**Q4: How does `this` work in arrow functions?**

A: Arrow functions do not have their own `this` binding. They capture `this` from the surrounding lexical context at the point of definition. This makes them ideal for callbacks and methods that need to preserve the `this` of their containing object. `call`, `apply`, and `bind` cannot change the `this` of an arrow function — the `this` argument is silently ignored. Arrow functions cannot be used as constructors with `new`.

---

**Q5: What is the difference between `Promise.all` and `Promise.allSettled`?**

A: `Promise.all` rejects immediately when any of the input Promises rejects (fail-fast). If one fails, you get the first rejection reason; fulfilled values are lost. Use it when all operations are required for the result to be useful. `Promise.allSettled` waits for ALL Promises to settle (either fulfilled or rejected) and returns an array of result objects with `{ status: 'fulfilled', value }` or `{ status: 'rejected', reason }`. Use it when you need all results and want to handle individual failures — e.g., batch operations where partial success is acceptable.

---

**Q6: What is the Temporal Dead Zone (TDZ)?**

A: The TDZ is the period from the start of a block scope to the point where a `let` or `const` variable is declared. During this period, the variable is in scope (JS knows about it) but accessing it throws a `ReferenceError`. This is different from `var`, which is initialized to `undefined` when hoisted. The TDZ helps catch programmer errors like using a variable before it is set up, which `var`'s silent `undefined` would mask.

---

**Q7: How does prototypal inheritance differ from classical inheritance?**

A: Classical inheritance (Java, C++) is class-based: classes are blueprints, instances are copies. JavaScript uses prototypal inheritance: objects directly inherit from other objects via the prototype chain. When you access a property, JS walks up `__proto__` links until it finds the property or reaches null. The `class` syntax in ES6 is syntactic sugar — under the hood, it still uses prototype chains. Key difference: in prototypal inheritance, you can modify prototypes at runtime (though this is generally a bad idea); in classical inheritance, class structure is fixed at compile time.

---

**Q8: What does an `async` function return?**

A: An `async` function always returns a Promise, regardless of what you explicitly return. If you `return 42`, the caller gets `Promise.resolve(42)`. If the async function throws, the caller gets a rejected Promise. This means you must always `await` the result of an async function or chain `.then()` to get the actual value. Forgetting this — and treating the async function as if it returns synchronously — is a common source of bugs.

---

**Q9: Explain the difference between microtask and macrotask queues.**

A: Macrotasks (also called tasks) come from: `setTimeout`, `setInterval`, `setImmediate` (Node), I/O callbacks, UI rendering, and user input events. Microtasks come from: Promise `.then`/`.catch`/`.finally`, `queueMicrotask()`, and `MutationObserver`. The critical difference: after each macrotask, the engine drains the ENTIRE microtask queue before running the next macrotask. This means a long chain of Promise.then callbacks can delay the next timeout and even block UI rendering. Use `queueMicrotask()` for code that must run asap after current sync code but before any timers.

---

**Q10: What is event delegation?**

A: Event delegation is attaching a single event listener to a parent element instead of multiple listeners on each child. When a child is clicked, the event bubbles up to the parent, where you check `event.target` to determine which child triggered it. Benefits: works for dynamically added children (they do not need their own listeners), uses less memory, requires less setup/teardown code. Use `event.target.closest(selector)` to handle clicks on elements inside the target (e.g., clicking an icon inside a button).

---

## 9. Exercises

**Exercise 1 — Implement debounce:**

Write a `debounce(fn, delay)` function that:
- Returns a new function that delays calling `fn` by `delay` milliseconds
- Resets the delay if called again before it fires
- Preserves `this` context and arguments
- Has a `cancel()` method to cancel the pending call
- Has a `flush()` method to immediately invoke the pending call

Test it by attaching it to an input field's `input` event.

```javascript
// Hint
function debounce(fn, delay) {
  let timer;
  function debounced(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => {
      timer = null;
      fn.apply(this, args);
    }, delay);
  }
  debounced.cancel = () => clearTimeout(timer);
  debounced.flush = () => { /* exercise for the reader */ };
  return debounced;
}
```

---

**Exercise 2 — Implement memoize:**

Write `memoize(fn)` that:
- Caches results by serialized arguments
- Works for functions with primitive arguments
- Handles edge cases: `undefined` return values, functions called with no args

Test with a fibonacci function and verify the call count drops.

---

**Exercise 3 — Implement Promise.all from scratch:**

Write `myPromiseAll(promises)` that:
- Returns a Promise that resolves with an array of values in the original order
- Rejects immediately on the first rejection
- Handles an empty array (resolves with `[]`)
- Handles non-Promise values in the array (wraps them with `Promise.resolve`)

---

**Exercise 4 — Fix the classic loop bug:**

Given this code:
```javascript
const fns = [];
for (var i = 0; i < 5; i++) {
  fns.push(function() { return i; });
}
console.log(fns.map((f) => f()));  // [5, 5, 5, 5, 5] — bug!
```

Fix it three ways:
1. Using `let` instead of `var`
2. Using an IIFE (immediately invoked function expression)
3. Using `Array.from` with an index

---

## 10. Further Reading

- **MDN — JavaScript reference**: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference
- **"You Don't Know JS" by Kyle Simpson** (free on GitHub): https://github.com/getify/You-Dont-Know-JS — the deepest exploration of JavaScript's mechanics
- **"JavaScript: The Good Parts" by Douglas Crockford** — short but important for understanding what to avoid
- **Event loop visualization**: https://www.jsv9000.app/ — interactive event loop visualizer
- **"What the heck is the event loop anyway?"** (Philip Roberts, JSConf EU): https://www.youtube.com/watch?v=8aGhZQkoFbQ
- **Jake Archibald on tasks, microtasks, queues**: https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/
- **MDN — Closures**: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures
- **Exploring ES2025+**: https://exploringjs.com/ — Dr. Axel Rauschmayer's thorough guides to each ECMAScript edition
- **Node.js event loop documentation**: https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick
