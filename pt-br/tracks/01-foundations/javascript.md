# JavaScript

> JavaScript é a única linguagem que roda nativamente em todos os navegadores. Entendê-la de verdade — o event loop, closures, protótipos, `this` e padrões assíncronos — é o que separa desenvolvedores que debugam com eficiência daqueles que copiam soluções do Stack Overflow sem entender o que estão fazendo.

---

## 1. O que é e por que importa

JavaScript foi criado em 10 dias em 1995 e se tornou a linguagem universal da web. Hoje roda em navegadores, servidores (Node.js), mobile (React Native), desktop (Electron), dispositivos IoT e workers em nuvem. Apesar da ubiquidade, muitos desenvolvedores usam a linguagem sem entender sua mecânica fundamental.

Entender o modelo de execução do JavaScript permite:
- Raciocinar sobre a ordem de execução em código assíncrono
- Evitar bugs clássicos como captura de closures em loops e `this` em callbacks
- Escrever código assíncrono correto e performático com Promises e async/await
- Debugar comportamentos confusos no navegador e no Node.js
- Usar recursos do ES6+ com consciência

---

## 2. Conceitos Fundamentais

### O Event Loop

JavaScript é single-threaded — só executa um trecho de código por vez. O event loop é o mecanismo que permite I/O não-bloqueante mesmo com essa restrição.

**Os quatro componentes:**

1. **Call Stack**: onde as chamadas de função vivem. LIFO (Last In, First Out). Código síncrono roda aqui.
2. **Web APIs / Node APIs**: APIs fornecidas pelo navegador/runtime (setTimeout, fetch, eventos do DOM, fs.readFile). Elas lidam com operações assíncronas fora da thread JS.
3. **Macrotask Queue** (fila de callbacks): callbacks concluídos das Web APIs (setTimeout, setInterval, callbacks de I/O, eventos de UI) ficam na fila aqui.
4. **Microtask Queue**: callbacks de Promise (`.then`, `.catch`, `.finally`), `queueMicrotask()` e callbacks de `MutationObserver`. **As microtasks são todas drenadas antes da próxima macrotask rodar.**

**Regra de execução:** Call stack esvazia → drena TODAS as microtasks → roda UMA macrotask → drena TODAS as microtasks → repete.

**Walkthrough visual:**
```javascript
console.log('1');           // síncrono

setTimeout(() => {
  console.log('4');         // macrotask — enfileirado após 0ms
}, 0);

Promise.resolve()
  .then(() => console.log('3'));   // microtask

console.log('2');           // síncrono
```

```
Passo 1: console.log('1')    → Stack: [log('1')]        → Saída: 1
Passo 2: setTimeout          → Stack: [setTimeout]      → Registra callback na Web API
Passo 3: Promise.resolve     → Stack: [Promise.resolve] → callback .then → Microtask Queue
Passo 4: console.log('2')    → Stack: [log('2')]        → Saída: 2

Call stack vazia agora.
→ Drena microtasks: console.log('3')           → Saída: 3
→ Microtask queue vazia.
→ Roda uma macrotask: console.log('4')         → Saída: 4

Saída final: 1  2  3  4
```

> `Promise.resolve().then()` sempre roda antes de `setTimeout(fn, 0)` mesmo que o timeout seja 0ms. Microtasks são drenadas primeiro.

---

### Hoisting

Hoisting é o mecanismo do JavaScript que move declarações para o topo do escopo antes da execução.

**`var` — declaração elevada, inicializada com `undefined`:**
```javascript
console.log(x);  // undefined (não é ReferenceError)
var x = 5;
console.log(x);  // 5

// Equivalente a:
var x;           // hoisted, inicializado com undefined
console.log(x);  // undefined
x = 5;
console.log(x);  // 5
```

**Declarações de função — completamente elevadas (declaração + corpo):**
```javascript
sayHello();  // funciona! → "Hello"

function sayHello() {
  console.log("Hello");
}
```

**`let` e `const` — elevadas mas NÃO inicializadas (Temporal Dead Zone):**
```javascript
console.log(y);  // ReferenceError: Cannot access 'y' before initialization
let y = 10;

// A declaração É elevada (JS sabe que y existe no escopo)
// mas fica na TDZ até a linha de declaração ser alcançada
```

**Temporal Dead Zone (TDZ):** O período entre o início de um escopo de bloco e a linha de declaração do `let`/`const`. Acessar a variável na TDZ lança `ReferenceError`.

```javascript
{
  // TDZ de z começa aqui
  typeof z;  // ReferenceError (mesmo typeof não escapa da TDZ para let/const)
  let z = 5; // TDZ termina aqui
}
```

---

### Closures

Uma closure é uma função que **mantém acesso às variáveis do escopo externo** mesmo depois que a função externa já retornou.

JavaScript usa **escopo léxico**: o escopo de uma função é determinado por onde ela foi definida, não por onde é chamada.

```javascript
function createCounter(start = 0) {
  let count = start;  // essa variável fica na closure

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
// `count` é privado — inacessível de fora, mas os métodos podem acessá-lo
```

**Usos práticos:**
```javascript
// 1. Padrão módulo — estado privado
function createCache() {
  const store = new Map();
  return {
    get: (key) => store.get(key),
    set: (key, val) => store.set(key, val),
    clear: () => store.clear(),
  };
}

// 2. Factory functions — comportamento parametrizado
const multiply = (factor) => (n) => n * factor;
const double = multiply(2);
const triple = multiply(3);
console.log(double(5));  // 10
console.log(triple(5));  // 15

// 3. Memoização — cacheia resultados usando closure
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

**Armadilha de closure em loops com `var`:**
```javascript
// Bug — todos os callbacks compartilham o mesmo `i` (var tem escopo de função)
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);  // imprime 3, 3, 3
}

// Correção 1: usar let (escopo de bloco, binding separado por iteração)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);  // imprime 0, 1, 2
}

// Correção 2: usar IIFE (cria novo escopo por iteração)
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 100);
  })(i);
}
```

---

### Binding de `this`

`this` se refere ao contexto de execução de uma chamada de função. Há quatro regras de binding, aplicadas em ordem de precedência:

**1. Binding padrão (menor precedência):**
```javascript
function show() { console.log(this); }
show();  // window (browser) ou global (Node) — undefined em strict mode
```

**2. Binding implícito (chamada de método):**
```javascript
const obj = {
  name: 'Alice',
  greet() { console.log(this.name); },
};
obj.greet();  // 'Alice' — this = obj

// Armadilha: desacoplar o método perde o binding
const fn = obj.greet;
fn();  // undefined — this = global (ou undefined em strict)
```

**3. Binding explícito (call/apply/bind):**
```javascript
function greet(greeting) { console.log(`${greeting}, ${this.name}`); }
const user = { name: 'Bob' };

greet.call(user, 'Hello');    // Hello, Bob — this = user, args como lista
greet.apply(user, ['Hi']);    // Hi, Bob — this = user, args como array
const boundGreet = greet.bind(user);
boundGreet('Hey');            // Hey, Bob — retorna nova função com this fixo
```

**4. Binding com `new` (maior precedência):**
```javascript
function Person(name) {
  this.name = name;  // this = objeto recém-criado
}
const alice = new Person('Alice');
console.log(alice.name);  // 'Alice'
```

**Arrow functions — sem `this` próprio:**
Arrow functions herdam `this` do escopo léxico que as envolve. Não podem ser vinculadas com `call`/`apply`/`bind` (o argumento `this` é ignorado silenciosamente).

```javascript
const obj = {
  name: 'Alice',
  greetLater() {
    setTimeout(() => {
      console.log(this.name);  // 'Alice' — arrow herda this de greetLater
    }, 1000);
  },
  greetLaterBroken() {
    setTimeout(function() {
      console.log(this.name);  // undefined — função regular, this = global
    }, 1000);
  },
};
```

---

### Promises

Uma Promise representa o resultado eventual de uma operação assíncrona. Tem três estados:
- **pending**: estado inicial, nem resolvida nem rejeitada
- **fulfilled**: operação concluída com sucesso, tem um valor
- **rejected**: operação falhou, tem um motivo

```javascript
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve(42), 1000);
});

p.then((value) => console.log(value))   // 42
 .catch((err) => console.error(err))
 .finally(() => console.log('done'));
```

**Combinadores de Promise:**
```javascript
// Promise.all — todas devem ter sucesso; rejeita na primeira falha
const [user, posts] = await Promise.all([
  fetchUser(id),
  fetchPosts(id),
]);

// Promise.allSettled — espera por todas, nunca rejeita
const results = await Promise.allSettled([
  fetchUser(id),
  fetchPosts(id),
]);
results.forEach((r) => {
  if (r.status === 'fulfilled') console.log(r.value);
  else console.error(r.reason);
});

// Promise.race — resolve/rejeita com a primeira a se resolver
const result = await Promise.race([
  fetch(url),
  new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), 5000)),
]);

// Promise.any — resolve com o primeiro sucesso; rejeita só se TODAS rejeitarem
const fastest = await Promise.any([mirror1, mirror2, mirror3]);
```

---

### async/await

`async`/`await` é açúcar sintático sobre Promises. Uma função `async` sempre retorna uma Promise. `await` pausa a execução até a Promise se resolver.

```javascript
async function fetchUserData(id) {
  try {
    const response = await fetch(`/api/users/${id}`);

    if (!response.ok) {
      throw new Error(`Erro HTTP ${response.status}`);
    }

    const user = await response.json();
    return user;
  } catch (err) {
    console.error('Falha ao buscar usuário:', err);
    throw err;  // re-lança para que os chamadores possam tratar
  }
}

// Sequencial (cada um espera o anterior)
async function sequential() {
  const user = await fetchUser(1);   // espera ~200ms
  const posts = await fetchPosts(1); // espera ~200ms após user
  // Total: ~400ms
}

// Paralelo (inicia ambos simultaneamente)
async function parallel() {
  const [user, posts] = await Promise.all([
    fetchUser(1),
    fetchPosts(1),
  ]);
  // Total: ~200ms (ambos rodam concorrentemente)
}
```

> **Esquecendo `await` dentro de loops**: `forEach` não aguarda Promises. Use `for...of` em vez disso.
> ```javascript
> // Errado — todos disparam simultaneamente, erros podem se perder
> items.forEach(async (item) => { await process(item); });
>
> // Correto — sequencial
> for (const item of items) { await process(item); }
>
> // Correto — paralelo
> await Promise.all(items.map((item) => process(item)));
> ```

---

## 3. Como Funciona

### Cadeia de Escopo

Quando JavaScript resolve um nome de variável, percorre a cadeia de escopos do escopo atual para fora:
```
escopo local → escopo(s) de closure → escopo do módulo → escopo global
```

```javascript
const global = 'global';

function outer() {
  const outerVar = 'outer';

  function inner() {
    const innerVar = 'inner';
    console.log(innerVar);  // encontrado no escopo local
    console.log(outerVar);  // encontrado no escopo externo (closure)
    console.log(global);    // encontrado no escopo global
    console.log(unknown);   // ReferenceError — não encontrado em nenhum escopo
  }

  inner();
}
```

### Herança Prototípica

Todo objeto JavaScript tem um slot interno `[[Prototype]]` apontando para outro objeto (ou null). Quando você acessa uma propriedade, o JS percorre a cadeia de protótipos até encontrá-la ou chegar ao null.

```javascript
const animal = {
  breathe() { return 'respirando'; },
};

const dog = Object.create(animal);  // [[Prototype]] de dog = animal
dog.bark = function() { return 'au'; };

console.log(dog.bark());     // encontrado diretamente em dog
console.log(dog.breathe());  // não está em dog → encontrado em animal (protótipo)
console.log(dog.toString()); // não está em dog/animal → encontrado em Object.prototype

// A sintaxe class é açúcar sintático sobre herança prototípica
class Animal {
  breathe() { return 'respirando'; }
}
class Dog extends Animal {
  bark() { return 'au'; }
}
const rex = new Dog();
// rex.__proto__ === Dog.prototype
// Dog.prototype.__proto__ === Animal.prototype
// Animal.prototype.__proto__ === Object.prototype
// Object.prototype.__proto__ === null
```

---

## 4. Exemplos de Código

### Delegação de eventos
```javascript
// Em vez de adicionar listeners a 1000 itens de lista,
// adiciona um listener no pai e verifica o target
document.querySelector('#item-list').addEventListener('click', (event) => {
  const item = event.target.closest('[data-item-id]');
  if (!item) return;

  const id = item.dataset.itemId;
  handleItemClick(id);
});
```

### Implementação de debounce
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

### Implementação de throttle
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

const onScroll = throttle(() => { /* cálculo custoso */ }, 100);
window.addEventListener('scroll', onScroll);
```

### Memoize com cache por argumentos
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

console.log(expensive(1000000));  // calculado
console.log(expensive(1000000));  // do cache
```

### Promise.all do zero
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
        .catch(reject);  // primeira rejeição rejeita tudo
    });
  });
}
```

### Recursos ES6+
```javascript
// Desestruturação com renomeação e valores padrão
const { name: userName = 'Anônimo', age = 0 } = user;
const [first, second = 'default', ...rest] = array;

// Optional chaining — curto-circuita em null/undefined
const city = user?.address?.city;
const firstTag = article?.tags?.[0];
const result = user?.getPreferences?.();

// Nullish coalescing — só usa fallback em null/undefined (não em 0, '', false)
const port = config.port ?? 3000;
const theme = userPreference ?? systemPreference ?? 'light';

// Atribuição lógica
user.name ??= 'Anônimo';    // atribui só se null/undefined
config.debug ||= false;      // atribui só se falsy
cache.data &&= transform(cache.data);  // atribui só se truthy

// Spread em vários contextos
const merged = { ...defaults, ...overrides };
const combined = [...arr1, ...arr2];
function log(...args) { console.log(...args); }
```

---

## 5. Erros Comuns e Armadilhas

> **`var` em loops**: `var` tem escopo de função, não de bloco. Todas as iterações do loop compartilham o mesmo binding. Use `let` ou `const` em loops — eles criam um novo binding por iteração.

> **Esquecer de `return` no `.then()`**: Se você esquecer de retornar um valor do callback de um `.then()`, o próximo `.then()` da cadeia recebe `undefined`.
> ```javascript
> fetch('/api/data')
>   .then((res) => res.json())  // precisa retornar a Promise
>   .then((data) => console.log(data));  // recebe o JSON parseado
>
> fetch('/api/data')
>   .then((res) => { res.json(); })  // sem return — undefined passa adiante!
>   .then((data) => console.log(data));  // data é undefined
> ```

> **Rejeições de Promise não tratadas**: Toda cadeia de Promise deve ter um `.catch()` ou estar dentro de um `try/catch` se usar `await`. Versões futuras do Node.js vão encerrar o processo em rejeições não tratadas.

> **`this` em callbacks**: Passar um método como callback perde o binding de `this`. Solução: use uma arrow function ou `.bind(this)`.
> ```javascript
> class Timer {
>   constructor() { this.count = 0; }
>   start() {
>     setInterval(this.tick, 1000);  // errado — this.tick perde `this`
>     setInterval(() => this.tick(), 1000);  // correto — arrow preserva this
>     setInterval(this.tick.bind(this), 1000);  // também correto
>   }
>   tick() { this.count++; }
> }
> ```

> **Função `async` sempre retorna uma Promise**: Mesmo que você faça `return 42` de uma função async, o chamador recebe `Promise<42>`. Você deve sempre `await` o resultado ou encadear `.then()`.

> **`==` vs `===`**: `==` realiza coerção de tipo (`0 == false` é `true`, `null == undefined` é `true`). `===` nunca coerce tipos. Sempre use `===` a menos que precise explicitamente de coerção de tipo (raro).

> **JSON.stringify com referências circulares**: Lança `TypeError: Converting circular structure to JSON`. Use uma biblioteca ou forneça uma função replacer.

---

## 6. Quando Usar / Não Usar

**Use `const` por padrão, `let` quando precisar de reatribuição, nunca `var`.**

**Use `async/await` em vez de cadeias brutas de `.then()`** para legibilidade, exceto quando precisar explicitamente do objeto Promise para combinadores.

**Use `Promise.all`** quando tiver múltiplas operações assíncronas independentes — não use `await` sequencial se elas podem rodar em paralelo.

**Use `Promise.allSettled`** quando precisar de todos os resultados independentemente de falhas individuais (ex.: operações em lote onde falha parcial é aceitável).

**Use delegação de eventos** ao adicionar listeners idênticos a muitos elementos — adicione um listener no pai em vez disso.

**Use closures** para encapsular estado privado, factory functions e memoização.

**Não use o objeto `arguments`** em código novo — use rest parameters `...args`. `arguments` não existe em arrow functions.

---

## 7. Cenário Real

### Implementando um cliente de API resiliente com retry

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
        signal: AbortSignal.timeout(10000),  // timeout de 10s
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

      // Retry em erros de rede (não em erros 4xx do cliente)
      if (err instanceof ApiError && err.status < 500) throw err;

      const delay = Math.min(1000 * 2 ** attempt, 30000);  // backoff exponencial
      console.warn(`Tentativa ${attempt} falhou, tentando novamente em ${delay}ms`);
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

// Uso
const api = new ApiClient('https://api.example.com', { maxRetries: 3 });

async function loadDashboard(userId) {
  // Busca paralela para dados independentes
  const [user, stats, notifications] = await Promise.all([
    api.get(`/users/${userId}`),
    api.get(`/users/${userId}/stats`),
    api.get(`/users/${userId}/notifications`),
  ]);

  return { user, stats, notifications };
}
```

---

## 8. Perguntas de Entrevista

**Q1: O que é o event loop do JavaScript?**

R: JavaScript é single-threaded. O event loop é o mecanismo que permite código assíncrono não-bloqueante. Ele tem uma call stack (onde a execução síncrona acontece), Web APIs (o navegador fornece setTimeout, fetch, eventos DOM — elas operam fora da thread JS), uma microtask queue (callbacks de Promise, queueMicrotask) e uma macrotask queue (setTimeout, setInterval, I/O). A regra do loop: roda código síncrono até a stack esvaziar → drena TODAS as microtasks → roda UMA macrotask → drena TODAS as microtasks → repete. É por isso que `Promise.resolve().then()` sempre dispara antes de `setTimeout(fn, 0)`.

---

**Q2: Qual a diferença entre `==` e `===`?**

R: `===` é igualdade estrita — sem coerção de tipo. Valor e tipo devem corresponder. `==` é igualdade solta — realiza coerção de tipo antes da comparação. Curiosidades: `0 == false` é `true`, `'' == false` é `true`, `null == undefined` é `true`, mas `null == 0` é `false`. Sempre use `===` para evitar coerções inesperadas. O único uso intencional de `==` é `x == null` para verificar tanto `null` quanto `undefined` de uma vez.

---

**Q3: O que é uma closure?**

R: Uma closure é uma função que mantém acesso a variáveis do escopo léxico externo mesmo após a função externa ter retornado. Isso funciona porque JavaScript usa escopo léxico — o escopo de uma função é determinado por onde ela é definida, não por onde é chamada. Usos práticos: encapsular estado privado (padrão módulo), factory functions que parametrizam comportamento, memoização e event handlers que precisam de acesso a dados de quando foram criados.

---

**Q4: Como `this` funciona em arrow functions?**

R: Arrow functions não têm seu próprio binding de `this`. Elas capturam `this` do contexto léxico que as envolve no momento da definição. Isso as torna ideais para callbacks e métodos que precisam preservar o `this` do objeto que as contém. `call`, `apply` e `bind` não podem alterar o `this` de uma arrow function — o argumento `this` é silenciosamente ignorado. Arrow functions não podem ser usadas como construtores com `new`.

---

**Q5: Qual a diferença entre `Promise.all` e `Promise.allSettled`?**

R: `Promise.all` rejeita imediatamente quando qualquer uma das Promises de entrada rejeita (fail-fast). Se uma falha, você recebe o motivo da primeira rejeição; os valores realizados são perdidos. Use quando todas as operações são necessárias para o resultado ser útil. `Promise.allSettled` espera por TODAS as Promises se resolverem (fulfilled ou rejected) e retorna um array de objetos com `{ status: 'fulfilled', value }` ou `{ status: 'rejected', reason }`. Use quando precisar de todos os resultados e quiser tratar falhas individuais — ex.: operações em lote onde sucesso parcial é aceitável.

---

**Q6: O que é a Temporal Dead Zone (TDZ)?**

R: A TDZ é o período desde o início de um escopo de bloco até o ponto onde uma variável `let` ou `const` é declarada. Durante esse período, a variável está no escopo (JS a conhece), mas acessá-la lança `ReferenceError`. Isso difere de `var`, que é inicializado com `undefined` quando elevado. A TDZ ajuda a capturar erros de programação como usar uma variável antes de configurá-la, o que o `undefined` silencioso do `var` mascararia.

---

**Q7: Como a herança prototípica difere da herança clássica?**

R: Herança clássica (Java, C++) é baseada em classes: classes são blueprints, instâncias são cópias. JavaScript usa herança prototípica: objetos herdam diretamente de outros objetos via cadeia de protótipos. Quando você acessa uma propriedade, o JS percorre os links `__proto__` até encontrar a propriedade ou chegar ao null. A sintaxe `class` do ES6 é açúcar sintático — por baixo dos panos, ainda usa cadeias de protótipos. Diferença chave: na herança prototípica, você pode modificar protótipos em tempo de execução (embora seja geralmente má prática); na herança clássica, a estrutura de classe é fixada em tempo de compilação.

---

**Q8: O que uma função `async` retorna?**

R: Uma função `async` sempre retorna uma Promise, independentemente do que você retorna explicitamente. Se você fizer `return 42`, o chamador recebe `Promise.resolve(42)`. Se a função async lança uma exceção, o chamador recebe uma Promise rejeitada. Isso significa que você deve sempre `await` o resultado de uma função async ou encadear `.then()` para obter o valor real. Esquecer isso — e tratar a função async como se retornasse de forma síncrona — é uma fonte comum de bugs.

---

**Q9: Explique a diferença entre filas de microtask e macrotask.**

R: Macrotasks (também chamadas tasks) vêm de: `setTimeout`, `setInterval`, `setImmediate` (Node), callbacks de I/O, renderização de UI e eventos de input do usuário. Microtasks vêm de: Promise `.then`/`.catch`/`.finally`, `queueMicrotask()` e `MutationObserver`. A diferença crítica: após cada macrotask, o engine drena a fila INTEIRA de microtasks antes de rodar a próxima macrotask. Isso significa que uma longa cadeia de callbacks Promise.then pode atrasar o próximo timeout e até bloquear a renderização da UI. Use `queueMicrotask()` para código que deve rodar o mais rápido possível após o código síncrono atual, mas antes de qualquer timer.

---

**Q10: O que é delegação de eventos?**

R: Delegação de eventos é adicionar um único event listener a um elemento pai em vez de múltiplos listeners em cada filho. Quando um filho é clicado, o evento borbulha (bubbles) até o pai, onde você verifica `event.target` para determinar qual filho disparou. Benefícios: funciona para filhos adicionados dinamicamente (não precisam de listeners próprios), usa menos memória, requer menos código de setup/teardown. Use `event.target.closest(selector)` para lidar com cliques em elementos dentro do target (ex.: clicar em um ícone dentro de um botão).

---

## 9. Exercícios

**Exercício 1 — Implementar debounce:**

Escreva uma função `debounce(fn, delay)` que:
- Retorna uma nova função que atrasa a chamada de `fn` por `delay` milissegundos
- Reinicia o delay se chamada novamente antes de disparar
- Preserva o contexto `this` e os argumentos
- Tem um método `cancel()` para cancelar a chamada pendente
- Tem um método `flush()` para invocar imediatamente a chamada pendente

Teste adicionando-a ao evento `input` de um campo de busca.

```javascript
// Dica
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
  debounced.flush = () => { /* exercício para o leitor */ };
  return debounced;
}
```

---

**Exercício 2 — Implementar memoize:**

Escreva `memoize(fn)` que:
- Cacheia resultados por argumentos serializados
- Funciona para funções com argumentos primitivos
- Trata casos extremos: valores de retorno `undefined`, funções chamadas sem args

Teste com uma função fibonacci e verifique se a contagem de chamadas cai.

---

**Exercício 3 — Implementar Promise.all do zero:**

Escreva `myPromiseAll(promises)` que:
- Retorna uma Promise que resolve com um array de valores na ordem original
- Rejeita imediatamente na primeira rejeição
- Trata array vazio (resolve com `[]`)
- Trata valores não-Promise no array (envolve com `Promise.resolve`)

---

**Exercício 4 — Corrija o bug clássico do loop:**

Dado este código:
```javascript
const fns = [];
for (var i = 0; i < 5; i++) {
  fns.push(function() { return i; });
}
console.log(fns.map((f) => f()));  // [5, 5, 5, 5, 5] — bug!
```

Corrija de três formas:
1. Usando `let` em vez de `var`
2. Usando uma IIFE (immediately invoked function expression)
3. Usando `Array.from` com índice

---

## 10. Leitura Adicional

- **MDN — Referência JavaScript**: https://developer.mozilla.org/pt-BR/docs/Web/JavaScript/Reference
- **"You Don't Know JS" de Kyle Simpson** (gratuito no GitHub): https://github.com/getify/You-Dont-Know-JS — a exploração mais profunda da mecânica do JavaScript
- **"JavaScript: The Good Parts" de Douglas Crockford** — curto mas importante para entender o que evitar
- **Visualizador do event loop**: https://www.jsv9000.app/ — visualizador interativo do event loop
- **"What the heck is the event loop anyway?"** (Philip Roberts, JSConf EU): https://www.youtube.com/watch?v=8aGhZQkoFbQ
- **Jake Archibald sobre tasks, microtasks, filas**: https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/
- **MDN — Closures**: https://developer.mozilla.org/pt-BR/docs/Web/JavaScript/Closures
- **Exploring ES2025+**: https://exploringjs.com/ — guias detalhados de cada edição do ECMAScript pelo Dr. Axel Rauschmayer
- **Documentação do event loop do Node.js**: https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick
