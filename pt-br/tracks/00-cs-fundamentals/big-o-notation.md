# Notação Big O

## 1. O Que É e Por Que Importa

A notação Big O descreve como os requisitos de tempo ou espaço de um algoritmo crescem conforme o tamanho da entrada aumenta. É a linguagem que engenheiros usam para comparar algoritmos e prever performance em escala. Um algoritmo que roda bem com 1.000 registros pode travar com 10 milhões — Big O ajuda você a identificar isso antes que vire um incidente em produção.

Sem Big O, você está adivinhando. Com ele, você consegue raciocinar: "este endpoint faz um loop aninhado sobre registros de usuários — com 100k usuários, isso são 10 bilhões de operações — precisamos resolver isso antes do lançamento." Esse é o tipo de raciocínio que diferencia engenheiros sênior de júniors.

Big O trata do **comportamento assintótico**: o que acontece com o custo quando n tende ao infinito. Fatores constantes e termos de menor ordem são descartados porque se tornam irrelevantes em escala.

---

## 2. Conceitos Fundamentais

### Classes de Complexidade

| Notação | Nome | Exemplo |
|---|---|---|
| O(1) | Constante | Acesso por índice em array, lookup em hashmap |
| O(log n) | Logarítmica | Busca binária, operações em BST balanceada |
| O(n) | Linear | Varredura linear, loop simples |
| O(n log n) | Linearítmica | Merge sort, ordenação eficiente |
| O(n²) | Quadrática | Loops aninhados sobre os mesmos dados |
| O(2ⁿ) | Exponencial | Fibonacci recursivo, geração de subconjuntos |
| O(n!) | Fatorial | Permutações, TSP por força bruta |

### Regras de Simplificação

- **Descartar constantes:** O(3n) → O(n)
- **Descartar termos não dominantes:** O(n² + n) → O(n²)
- **Passos sequenciais somam:** O(a) depois O(b) = O(a + b)
- **Passos aninhados multiplicam:** O(a) dentro de loop O(b) = O(a × b)

### Três Casos

- **Melhor caso (Ω — Omega):** mínimo de operações para a entrada mais favorável. Raramente útil na prática.
- **Caso médio (Θ — Theta):** operações esperadas ao longo de todas as entradas. O mais realista para planejamento.
- **Pior caso (O — Big O):** máximo de operações. A garantia. O que citamos por padrão.

### Análise Amortizada

O custo médio por operação ao longo de uma **sequência** de operações. O `push` em array dinâmico é O(1) amortizado: a maioria dos pushes é O(1), mas ocasionalmente o array de suporte dobra de tamanho (O(n)), resultando em média em O(1) por operação ao longo de n pushes.

### Complexidade de Espaço

Memória consumida pelo algoritmo — incluindo variáveis auxiliares, estruturas de dados e a pilha de chamadas para chamadas recursivas. O(1) de espaço significa memória constante independente do tamanho da entrada. O(n) significa que a memória cresce linearmente.

---

## 3. Como Funciona

```
Comparação de crescimento de complexidade (n = 1.000):

O(1)       →            1 operação
O(log n)   →           10 operações
O(n)       →        1.000 operações
O(n log n) →       10.000 operações
O(n²)      →    1.000.000 operações
O(2ⁿ)      → 2^1000 operações (computacionalmente impossível)
```

**A regra do termo dominante:** À medida que n cresce, o termo de maior crescimento domina todos os outros. O(n³ + 1000n² + n) se comporta como O(n³) para n grande.

**Por que constantes não importam em Big O:** Se o algoritmo A leva 2n passos e o algoritmo B leva 1000n passos, A é mais rápido por um fator constante. Mas O(n) vs O(n²) é uma diferença qualitativa: em n = 1.000.000, n² são 10¹² operações enquanto n são 10⁶ — um milhão de vezes mais trabalho.

---

## 4. Exemplos de Código (TypeScript)

```typescript
// O(1) — Tempo constante
// O tempo de execução não depende do tamanho da entrada
function getFirst<T>(arr: T[]): T {
  return arr[0];
}

function isEven(n: number): boolean {
  return n % 2 === 0; // sempre uma operação aritmética
}

// O(log n) — Tempo logarítmico
// Busca binária: divide o espaço de busca pela metade a cada iteração
function binarySearch(arr: number[], target: number): number {
  let left = 0;
  let right = arr.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}

// O(n) — Tempo linear
// Visita cada elemento exatamente uma vez
function findMax(arr: number[]): number {
  let max = arr[0];
  for (const n of arr) {
    if (n > max) max = n;
  }
  return max;
}

// O(n²) — Tempo quadrático: loops aninhados sobre os mesmos dados
function hasDuplicate(arr: number[]): boolean {
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j]) return true;
    }
  }
  return false;
}

// O(n) — Mesmo problema resolvido em tempo linear usando um Set
function hasDuplicateFast(arr: number[]): boolean {
  const seen = new Set<number>();
  for (const n of arr) {
    if (seen.has(n)) return true;
    seen.add(n);
  }
  return false;
}

// O(n log n) — Tempo linearítmico: merge sort
function mergeSort(arr: number[]): number[] {
  if (arr.length <= 1) return arr;

  const mid = Math.floor(arr.length / 2);
  const left = mergeSort(arr.slice(0, mid));  // O(log n) níveis de profundidade
  const right = mergeSort(arr.slice(mid));

  return merge(left, right); // O(n) de trabalho em cada nível
}

function merge(left: number[], right: number[]): number[] {
  const result: number[] = [];
  let i = 0, j = 0;
  while (i < left.length && j < right.length) {
    if (left[i] <= right[j]) result.push(left[i++]);
    else result.push(right[j++]);
  }
  return result.concat(left.slice(i)).concat(right.slice(j));
}

// O(2ⁿ) — Tempo exponencial: Fibonacci recursivo ingênuo
function fib(n: number): number {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2); // duas chamadas recursivas por invocação
}

// O(n) — Fibonacci com memoização transforma exponencial em linear
function fibMemo(n: number, memo = new Map<number, number>()): number {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n)!;
  const result = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  memo.set(n, result);
  return result;
}

// --- Exemplos de complexidade de espaço ---

// O(1) de espaço — inversão in-place
function reverseInPlace(arr: number[]): void {
  let left = 0, right = arr.length - 1;
  while (left < right) {
    [arr[left], arr[right]] = [arr[right], arr[left]];
    left++;
    right--;
  }
}

// O(n) de espaço — aloca um novo array
function reverseNew(arr: number[]): number[] {
  const result: number[] = [];
  for (let i = arr.length - 1; i >= 0; i--) {
    result.push(arr[i]); // aloca n elementos
  }
  return result;
}

// O(log n) de espaço — busca binária recursiva usa pilha de chamadas
function binarySearchRecursive(
  arr: number[],
  target: number,
  left = 0,
  right = arr.length - 1
): number {
  if (left > right) return -1;
  const mid = Math.floor((left + right) / 2);
  if (arr[mid] === target) return mid;
  if (arr[mid] < target) return binarySearchRecursive(arr, target, mid + 1, right);
  return binarySearchRecursive(arr, target, left, mid - 1);
}

// --- Exemplo de push amortizado O(1) ---
class DynamicArray<T> {
  private data: T[] = [];
  private capacity = 1;
  private size = 0;

  push(item: T): void {
    if (this.size === this.capacity) {
      // O(n) redimensionamento — mas ocorre apenas O(log n) vezes durante n pushes
      const newData = new Array<T>(this.capacity * 2);
      for (let i = 0; i < this.size; i++) newData[i] = this.data[i];
      this.data = newData;
      this.capacity *= 2;
    }
    this.data[this.size++] = item;
  }
  // Trabalho total de redimensionamento ao longo de n pushes: 1+2+4+...+n = 2n = O(n)
  // Custo amortizado por push: O(n)/n = O(1)
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Confundir caso médio com pior caso.** Quicksort é O(n log n) no caso médio, mas O(n²) no pior caso em entrada já ordenada com uma má escolha de pivô. Sempre especifique qual caso está citando.

> ⚠️ **Ignorar complexidade de espaço.** Recursão usa O(profundidade) de espaço na pilha. Um DFS recursivo em um grafo com 100.000 nós pode causar stack overflow antes de terminar.

> ⚠️ **O(n) oculto dentro de loops.** `Array.includes()`, `Array.indexOf()` e `String.includes()` são todos O(n). Chamá-los dentro de um loop produz O(n²) sem que seja óbvio.

```typescript
// Parece O(n), na verdade é O(n²):
function findCommon(a: number[], b: number[]): number[] {
  return a.filter(x => b.includes(x)); // b.includes() é O(n) por chamada
}

// Corrigido: O(n)
function findCommonFast(a: number[], b: number[]): number[] {
  const setB = new Set(b);
  return a.filter(x => setB.has(x)); // Set.has() é O(1)
}
```

> ⚠️ **`Array.shift()` e `Array.unshift()` são O(n).** Eles precisam mover todos os elementos. Use um ponteiro ou um deque adequado para operações O(1) na frente.

> ⚠️ **Spread de objeto `{ ...obj }` em loops apertados.** Spread é O(k) onde k = número de chaves. Dentro de um loop O(n), isso é O(n × k).

> ⚠️ **Otimização prematura.** O(n²) em n = 100 roda em microssegundos. Performe primeiro. Corrija o que é mensuravelmente lento. Não sacrifique legibilidade por micro-otimizações irrelevantes.

---

## 6. Quando Usar / Não Usar

**Preste muita atenção ao Big O quando:**
- O tamanho dos dados pode exceder 10.000 e a operação roda por requisição
- Está construindo uma biblioteca ou serviço consumido por muitos chamadores
- Diagnosticando queries lentas, timeouts ou picos de CPU em produção
- Em uma entrevista — análise de Big O é esperada para toda solução proposta

**Não otimize em excesso quando:**
- n é pequeno e limitado (parsing de config, poucas opções de dropdown, lista de 10 itens)
- O gargalo é I/O — latência de rede, banco de dados ou disco eclipsa o tempo de CPU
- Uma solução O(n²) mais simples é muito mais legível e n nunca vai exceder algumas centenas
- O fator constante importa mais do que a classe assintótica nos tamanhos que você realmente opera

---

## 7. Cenário do Mundo Real

Uma plataforma SaaS de analytics tem o recurso "encontrar usuários que correspondem a todas as tags selecionadas". A primeira versão:

```typescript
// O(usuários × tags × media_tags_por_usuario) por requisição
function findMatchingUsers(users: User[], tags: string[]): User[] {
  return users.filter(user =>
    tags.every(tag => user.tags.includes(tag)) // O(media_tags) por tag
  );
}
```

Com 50.000 usuários, 20 tags selecionadas, cada usuário tendo ~10 tags: 50.000 × 20 × 10 = 10.000.000 operações por requisição. A 100 req/s isso são 1 bilhão de operações por segundo. O servidor não aguenta.

Solução: construir um índice invertido (tag → Set de IDs de usuário) uma vez ao carregar os dados:

```typescript
type UserIndex = Map<string, Set<string>>;

function buildIndex(users: User[]): UserIndex {
  const index: UserIndex = new Map();
  for (const user of users) {
    for (const tag of user.tags) {
      if (!index.has(tag)) index.set(tag, new Set());
      index.get(tag)!.add(user.id);
    }
  }
  return index; // O(usuários × media_tags) — feito uma vez
}

function findMatchingFast(
  index: UserIndex,
  allUsers: Map<string, User>,
  tags: string[]
): User[] {
  if (tags.length === 0) return [];

  // Intersectar sets, começando pelo menor para eficiência
  const sets = tags
    .map(tag => index.get(tag) ?? new Set<string>())
    .sort((a, b) => a.size - b.size);

  let result = new Set(sets[0]);
  for (let i = 1; i < sets.length; i++) {
    result = new Set([...result].filter(id => sets[i].has(id)));
  }

  return [...result].map(id => allUsers.get(id)!);
}
// Por requisição: O(tags × tamanho_intersecção) — tipicamente O(1) para poucas tags
```

---

## 8. Perguntas de Entrevista

**Q1: O que é a notação Big O e por que a usamos?**
R: Big O descreve o limite superior do crescimento de um algoritmo em tempo ou espaço em relação ao tamanho da entrada, descartando constantes e termos de menor ordem. Usamos para comparar algoritmos e prever performance em escala sem precisar fazer benchmark de cada entrada.

**Q2: Qual é a complexidade de tempo para acessar um elemento de array por índice?**
R: O(1). Arrays armazenam elementos em memória contígua. O endereço é calculado como `base + índice × tamanho_do_elemento` — uma única operação aritmética independente do tamanho do array.

**Q3: O(2n) é diferente de O(n)?**
R: Não. Constantes são descartadas em Big O. Ambos descrevem crescimento linear — a razão entre 2n e n é constante independente de n.

**Q4: Qual é a complexidade de tempo do `Array.sort()` do JavaScript?**
R: O(n log n). O V8 usa Timsort — um híbrido de merge sort e insertion sort — que garante O(n log n) no pior caso e é estável desde o V8 7.0.

**Q5: Uma função tem um loop O(n) e depois chama um helper O(n). Qual é o total?**
R: O(n) + O(n) = O(2n) = O(n). Operações independentes sequenciais somam e então as constantes são descartadas.

**Q6: Qual é a complexidade de espaço de uma função Fibonacci recursiva?**
R: O(n) de espaço. A pilha de chamadas cresce até profundidade n antes de desenrolar. Mesmo que a árvore de chamadas tenha O(2ⁿ) nós, a profundidade máxima da pilha em qualquer momento é n.

**Q7: Quando O(n²) é aceitável?**
R: Quando n é pequeno (tipicamente < 1.000) e a operação é executada com pouca frequência. Corretude primeiro, otimização baseada em profiling depois.

**Q8: O que significa "O(1) amortizado" para push em array dinâmico?**
R: A maioria dos pushes é O(1). Ocasionalmente o array de suporte dobra de tamanho — O(n). Ao longo de n operações de push, o trabalho total de redimensionamento é 1 + 2 + 4 + ... + n = 2n = O(n), então a média por operação é O(n)/n = O(1).

**Q9: Qual é a complexidade de `Set.has()` e `Map.get()`?**
R: O(1) em média. Ambos usam hash tables. O pior caso é O(n) devido a colisões de hash, mas isso é extremamente raro na prática.

**Q10: Em que valor de n O(n log n) vs O(n²) começa a importar?**
R: Em n = 1.000: n log n ≈ 10.000 vs n² = 1.000.000 — 100× mais lento. Em n = 10: desprezível. O cruzamento prático fica em torno de n = 500–1.000, dependendo das constantes.

---

## 9. Exercícios

**Exercício 1:** Anote a complexidade de cada linha e da função como um todo:
```typescript
function mystery(arr: number[]): number {
  let result = 0;
  for (let i = 0; i < arr.length; i++) {       // loop externo: ?
    for (let j = 0; j < arr.length; j++) {     // loop interno: ?
      result += arr[i] * arr[j];               // corpo: ?
    }
  }
  return result;
}
```
*Dica: se arr.length = n, o corpo interno roda n × n vezes.*

**Exercício 2:** A função abaixo é O(n²). Reescreva em O(n) usando um Set:
```typescript
function hasDuplicate(arr: number[]): boolean {
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j]) return true;
    }
  }
  return false;
}
```

**Exercício 3:** Dado um array ordenado, verifique se algum par de elementos soma um alvo. Alcance O(n) de tempo e O(1) de espaço.
*Dica: dois ponteiros — um no início, outro no fim, mova com base em se a soma atual é muito alta ou muito baixa.*

**Exercício 4:** Determine tanto a complexidade de TEMPO quanto de ESPAÇO:
```typescript
function fib(n: number): number {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}
```
*Dica: desenhe a árvore de chamadas para fib(5). Conte o total de nós (tempo). Trace a profundidade máxima da pilha em qualquer momento (espaço).*

**Exercício 5:** Você chama `Array.includes(x)` dentro de um loop `for` que itera sobre o mesmo comprimento do array. Qual é a complexidade geral? Reescreva para alcançar O(n).

---

## 10. Leituras Complementares

- [Big O Cheat Sheet](https://www.bigocheatsheet.com/) — referência visual rápida para todas as estruturas de dados comuns
- CLRS — Introduction to Algorithms, Capítulo 3 (Growth of Functions)
- [NeetCode — Big O Crash Course](https://neetcode.io/) — explicações animadas e visuais
- [MDN — Complexidades dos métodos de Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)
- [Paper do Timsort por Tim Peters](https://svn.python.org/projects/python/trunk/Objects/listsort.txt) — como o sort do V8 realmente funciona
