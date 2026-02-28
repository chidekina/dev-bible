# Segment Trees e Estruturas de Dados Avançadas

## 1. O Que São e Por Que Importam

Segment trees e Fenwick trees (Árvores Binárias Indexadas / BIT) resolvem uma classe de problemas envolvendo **consultas de intervalo com atualizações** — operações que de outra forma exigiriam O(n) de tempo por consulta em um array simples.

O problema canônico: dado um array de n números, lidar com dois tipos de operações:
1. Atualizar o valor no índice i
2. Consultar a soma (ou min/max) dos elementos no intervalo [l, r]

Abordagem ingênua: O(1) atualização, O(n) consulta. Ou pré-computar somas de prefixo: O(n) build, O(1) consulta, O(n) atualização. Nenhuma é eficiente quando ambas as operações são frequentes.

Segment tree: O(log n) tanto para atualização quanto para consulta. Fenwick tree: mesma complexidade, implementação mais simples para somas de prefixo.

Essas estruturas aparecem em programação competitiva, engines de banco de dados (consultas de intervalo em B-trees) e geometria computacional. São tópicos avançados — tipicamente perguntados apenas em entrevistas de nível sênior ou FAANG.

---

## 2. Conceitos Fundamentais

### Segment Tree

Uma segment tree é uma árvore binária completa onde:
- Cada **folha** armazena um único elemento do array original
- Cada **nó interno** armazena o agregado (soma, min, max, mdc, etc.) do intervalo que seus filhos cobrem
- A raiz cobre todo o array [0, n-1]
- O nó no índice i cobre o intervalo [l, r]; seu filho esquerdo cobre [l, meio] e o filho direito cobre [meio+1, r]

**Armazenamento:** Como heaps, segment trees podem ser armazenadas como arrays. Para um nó no índice i: filho esquerdo = 2i, filho direito = 2i+1. Use indexação em 1. Tamanho do array = 4n para lidar com todos os casos.

**Build:** O(n) — computa cada nó de baixo para cima.
**Atualização pontual:** O(log n) — atualiza a folha, recomputa ancestrais.
**Consulta de intervalo:** O(log n) — combina no máximo O(log n) nós.

### Fenwick Tree (Árvore Binária Indexada / BIT)

Uma estrutura mais simples para consultas de soma de prefixo com atualizações pontuais. Usa a representação binária dos índices para armazenar eficientemente somas parciais. O(log n) para atualização e consulta de prefixo. Mais eficiente em espaço do que uma segment tree, mas menos flexível (funciona bem para operações de prefixo, mas mais difícil de adaptar para operações de intervalo arbitrárias).

### Propagação Preguiçosa (Lazy Propagation)

Para **atualizações de intervalo** (ex: adicionar 5 a todos os elementos em [l, r]), atualizar cada folha individualmente é O(n log n). Propagação preguiçosa adia atualizações: marca um nó como "atualização pendente" e só propaga quando um filho precisar ser acessado. Alcança O(log n) para atualização de intervalo + O(log n) para consulta de intervalo.

### Array de Sufixos

Um array ordenado de todos os sufixos de uma string. Permite busca exata de substring em O(log n) e computação de LCP (Maior Prefixo Comum) em O(n). Usado em bioinformática, compressão de dados e algoritmos de string.

---

## 3. Como Funciona

```
Array: [2, 4, 5, 7, 2, 3, 1, 6]  (n = 8)

Segment Tree (soma):
                  [0,7] soma=30
              /                  \
        [0,3] soma=18          [4,7] soma=12
       /         \             /          \
  [0,1] soma=6  [2,3] soma=12  [4,5] soma=5  [6,7] soma=7
  /    \        /      \      /    \       /    \
[0]=2 [1]=4  [2]=5  [3]=7  [4]=2 [5]=3  [6]=1  [7]=6

Consulta soma(2, 5):
- Na raiz [0,7]: intervalo cobre consulta, vai para ambos os lados
- Em [0,3]: meio=1 → direita [2,3] totalmente dentro de [2,5] → retorna 12
- Em [4,7]: meio=5 → esquerda [4,5] totalmente dentro de [2,5] → retorna 5
- Total: 12 + 5 = 17 ✓

Atualiza arr[3] = 10 (era 7, delta = +3):
- Atualiza folha [3] = 10
- Atualiza pai [2,3] = 5 + 10 = 15
- Atualiza avô [0,3] = 6 + 15 = 21
- Atualiza raiz [0,7] = 21 + 12 = 33
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Segment Tree: Soma de Intervalo com Atualização Pontual ---
class SegmentTree {
  private tree: number[];
  private n: number;

  constructor(arr: number[]) {
    this.n = arr.length;
    this.tree = new Array(4 * this.n).fill(0);
    this.build(arr, 1, 0, this.n - 1);
  }

  // Build: O(n)
  private build(arr: number[], node: number, start: number, end: number): void {
    if (start === end) {
      this.tree[node] = arr[start];
      return;
    }
    const mid = Math.floor((start + end) / 2);
    this.build(arr, 2 * node, start, mid);
    this.build(arr, 2 * node + 1, mid + 1, end);
    this.tree[node] = this.tree[2 * node] + this.tree[2 * node + 1];
  }

  // Atualização pontual: define arr[index] = val. O(log n)
  update(index: number, val: number): void {
    this.updateHelper(1, 0, this.n - 1, index, val);
  }

  private updateHelper(node: number, start: number, end: number, index: number, val: number): void {
    if (start === end) {
      this.tree[node] = val;
      return;
    }
    const mid = Math.floor((start + end) / 2);
    if (index <= mid) this.updateHelper(2 * node, start, mid, index, val);
    else this.updateHelper(2 * node + 1, mid + 1, end, index, val);
    this.tree[node] = this.tree[2 * node] + this.tree[2 * node + 1];
  }

  // Consulta de soma de intervalo [left, right]: O(log n)
  query(left: number, right: number): number {
    return this.queryHelper(1, 0, this.n - 1, left, right);
  }

  private queryHelper(node: number, start: number, end: number, left: number, right: number): number {
    if (right < start || end < left) return 0;   // sem sobreposição
    if (left <= start && end <= right) return this.tree[node]; // sobreposição total

    // Sobreposição parcial: recursa em ambos os filhos
    const mid = Math.floor((start + end) / 2);
    return (
      this.queryHelper(2 * node, start, mid, left, right) +
      this.queryHelper(2 * node + 1, mid + 1, end, left, right)
    );
  }
}

// --- Segment Tree: Consulta de Mínimo de Intervalo ---
class RMQTree {
  private tree: number[];
  private n: number;

  constructor(arr: number[]) {
    this.n = arr.length;
    this.tree = new Array(4 * this.n).fill(Infinity);
    this.build(arr, 1, 0, this.n - 1);
  }

  private build(arr: number[], node: number, start: number, end: number): void {
    if (start === end) { this.tree[node] = arr[start]; return; }
    const mid = Math.floor((start + end) / 2);
    this.build(arr, 2 * node, start, mid);
    this.build(arr, 2 * node + 1, mid + 1, end);
    this.tree[node] = Math.min(this.tree[2 * node], this.tree[2 * node + 1]);
  }

  queryMin(left: number, right: number): number {
    return this.queryHelper(1, 0, this.n - 1, left, right);
  }

  private queryHelper(node: number, start: number, end: number, left: number, right: number): number {
    if (right < start || end < left) return Infinity;
    if (left <= start && end <= right) return this.tree[node];
    const mid = Math.floor((start + end) / 2);
    return Math.min(
      this.queryHelper(2 * node, start, mid, left, right),
      this.queryHelper(2 * node + 1, mid + 1, end, left, right)
    );
  }

  update(index: number, val: number): void {
    this.updateHelper(1, 0, this.n - 1, index, val);
  }

  private updateHelper(node: number, start: number, end: number, index: number, val: number): void {
    if (start === end) { this.tree[node] = val; return; }
    const mid = Math.floor((start + end) / 2);
    if (index <= mid) this.updateHelper(2 * node, start, mid, index, val);
    else this.updateHelper(2 * node + 1, mid + 1, end, index, val);
    this.tree[node] = Math.min(this.tree[2 * node], this.tree[2 * node + 1]);
  }
}

// --- Fenwick Tree (Árvore Binária Indexada): Somas de Prefixo ---
// Implementação mais simples para consultas de soma com atualizações pontuais
class FenwickTree {
  private tree: number[];
  private n: number;

  constructor(n: number) {
    this.n = n;
    this.tree = new Array(n + 1).fill(0); // indexação em 1
  }

  // Atualização pontual: adiciona delta ao índice i (indexação em 1). O(log n)
  update(i: number, delta: number): void {
    for (; i <= this.n; i += i & (-i)) { // i & (-i) isola o bit menos significativo
      this.tree[i] += delta;
    }
  }

  // Soma de prefixo [1, i]: O(log n)
  prefixSum(i: number): number {
    let sum = 0;
    for (; i > 0; i -= i & (-i)) {
      sum += this.tree[i];
    }
    return sum;
  }

  // Soma de intervalo [l, r] (indexação em 1): O(log n)
  rangeSum(l: number, r: number): number {
    return this.prefixSum(r) - this.prefixSum(l - 1);
  }

  // Build a partir de array: O(n log n)
  static fromArray(arr: number[]): FenwickTree {
    const bit = new FenwickTree(arr.length);
    for (let i = 0; i < arr.length; i++) {
      bit.update(i + 1, arr[i]); // converte para indexação em 1
    }
    return bit;
  }
}

// --- Contagem de Números Menores Após Si Mesmo usando BIT ---
// Para cada elemento, conta quantos elementos à sua direita são menores
// O(n log n)
function countSmaller(nums: number[]): number[] {
  // Compressão de coordenadas: mapeia valores para [1..n]
  const sorted = [...new Set(nums)].sort((a, b) => a - b);
  const rank = new Map(sorted.map((v, i) => [v, i + 1]));

  const bit = new FenwickTree(sorted.length);
  const result: number[] = new Array(nums.length);

  // Processa da direita para a esquerda
  for (let i = nums.length - 1; i >= 0; i--) {
    const r = rank.get(nums[i])!;
    result[i] = r > 1 ? bit.prefixSum(r - 1) : 0; // conta valores menores vistos até agora
    bit.update(r, 1);
  }
  return result;
}

// --- Segment Tree com Propagação Preguiçosa: Atualização de Intervalo + Soma de Intervalo ---
class LazySegTree {
  private tree: number[];
  private lazy: number[];
  private n: number;

  constructor(arr: number[]) {
    this.n = arr.length;
    this.tree = new Array(4 * this.n).fill(0);
    this.lazy = new Array(4 * this.n).fill(0);
    this.build(arr, 1, 0, this.n - 1);
  }

  private build(arr: number[], node: number, start: number, end: number): void {
    if (start === end) { this.tree[node] = arr[start]; return; }
    const mid = Math.floor((start + end) / 2);
    this.build(arr, 2 * node, start, mid);
    this.build(arr, 2 * node + 1, mid + 1, end);
    this.tree[node] = this.tree[2 * node] + this.tree[2 * node + 1];
  }

  private pushDown(node: number, start: number, end: number): void {
    if (this.lazy[node] === 0) return;
    const mid = Math.floor((start + end) / 2);
    // Aplica atualização pendente aos filhos
    this.tree[2 * node] += this.lazy[node] * (mid - start + 1);
    this.lazy[2 * node] += this.lazy[node];
    this.tree[2 * node + 1] += this.lazy[node] * (end - mid);
    this.lazy[2 * node + 1] += this.lazy[node];
    this.lazy[node] = 0;
  }

  // Atualização de intervalo: adiciona val a todos os elementos em [left, right]. O(log n)
  rangeUpdate(left: number, right: number, val: number): void {
    this.updateHelper(1, 0, this.n - 1, left, right, val);
  }

  private updateHelper(node: number, start: number, end: number, left: number, right: number, val: number): void {
    if (right < start || end < left) return;
    if (left <= start && end <= right) {
      this.tree[node] += val * (end - start + 1);
      this.lazy[node] += val;
      return;
    }
    this.pushDown(node, start, end);
    const mid = Math.floor((start + end) / 2);
    this.updateHelper(2 * node, start, mid, left, right, val);
    this.updateHelper(2 * node + 1, mid + 1, end, left, right, val);
    this.tree[node] = this.tree[2 * node] + this.tree[2 * node + 1];
  }

  // Consulta de soma de intervalo [left, right]. O(log n)
  query(left: number, right: number): number {
    return this.queryHelper(1, 0, this.n - 1, left, right);
  }

  private queryHelper(node: number, start: number, end: number, left: number, right: number): number {
    if (right < start || end < left) return 0;
    if (left <= start && end <= right) return this.tree[node];
    this.pushDown(node, start, end);
    const mid = Math.floor((start + end) / 2);
    return (
      this.queryHelper(2 * node, start, mid, left, right) +
      this.queryHelper(2 * node + 1, mid + 1, end, left, right)
    );
  }
}

// --- Array de soma de prefixo simples (quando não há atualizações necessárias) ---
// O(n) para build, O(1) para consulta — use quando atualizações são raras/ausentes
class PrefixSumArray {
  private prefix: number[];

  constructor(arr: number[]) {
    this.prefix = new Array(arr.length + 1).fill(0);
    for (let i = 0; i < arr.length; i++) {
      this.prefix[i + 1] = this.prefix[i] + arr[i];
    }
  }

  // Soma de intervalo [left, right] inclusive, indexação em 0: O(1)
  query(left: number, right: number): number {
    return this.prefix[right + 1] - this.prefix[left];
  }
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Tamanho do array da segment tree muito pequeno.** Aloque 4 × n elementos, não 2 × n. Para tamanhos de entrada que não são potências de 2, a árvore pode precisar de até 4n nós.

> ⚠️ **Off-by-one na indexação em 1 vs em 0.** Fenwick trees são naturalmente indexadas em 1 (`i & (-i)` quebra para i=0). Segment trees podem usar qualquer uma — seja consistente.

> ⚠️ **Esquecer de chamar pushDown antes de recursar em segment trees preguiçosas.** Se você recursa nos filhos sem primeiro aplicar a atualização preguiçosa pendente do pai, os filhos retornam valores desatualizados.

> ⚠️ **Usar uma segment tree quando um array de soma de prefixo simples é suficiente.** Se o array nunca muda após o build, `PrefixSumArray` é mais simples, mais rápido na prática e O(1) por consulta.

> ⚠️ **Fenwick tree: update é aditivo, não atribuição.** `bit.update(i, val)` adiciona val à posição i. Para definir a posição i como um novo valor, compute o delta: `bit.update(i, newVal - oldVal)`.

---

## 6. Quando Usar / Não Usar

**Use uma Segment Tree quando:**
- Consultas de intervalo (soma, min, max, mdc) com atualizações pontuais ou de intervalo frequentes
- A função de consulta é associativa (pode ser combinada de sub-intervalos)
- Programação competitiva, problemas baseados em intervalos

**Use uma Fenwick Tree quando:**
- Apenas consultas de soma de prefixo (não consultas de intervalo gerais)
- Implementação mais simples é preferida
- Contagem de inversões, problemas de compressão de coordenadas

**Use uma soma de prefixo simples quando:**
- O array é estático (sem atualizações após o build)
- Consultas O(1) necessárias, sem sobrecarga de atualização

**Pule estas quando:**
- n é pequeno — força bruta é mais limpa e rápida o suficiente
- O problema não envolve consultas de intervalo repetidas — apenas varre o array uma vez

---

## 7. Cenário do Mundo Real

Um dashboard financeiro mostra totais correntes para qualquer intervalo de datas e lida com atualizações em tempo real quando transações chegam:

```typescript
class TransactionLedger {
  private segTree: SegmentTree;
  private amounts: number[];

  constructor(initialAmounts: number[]) {
    this.amounts = [...initialAmounts];
    this.segTree = new SegmentTree(this.amounts);
  }

  // Nova transação no índice (dia)
  addTransaction(dayIndex: number, amount: number): void {
    this.amounts[dayIndex] += amount;
    this.segTree.update(dayIndex, this.amounts[dayIndex]);
  }

  // Consulta total para intervalo de datas [startDay, endDay]
  totalForRange(startDay: number, endDay: number): number {
    return this.segTree.query(startDay, endDay);
  }
}
```

---

## 8. Perguntas de Entrevista

**P1: Quando você usaria uma segment tree em vez de um array de soma de prefixo simples?**
R: Quando o array é atualizado frequentemente junto com consultas de intervalo. Um array de soma de prefixo estático dá consultas O(1), mas atualizações O(n) (precisa reconstruir). Uma segment tree dá O(log n) para ambos. Se não há atualizações, use a soma de prefixo mais simples.

**P2: Qual é a diferença entre uma segment tree e uma Fenwick tree?**
R: Uma Fenwick tree (BIT) é mais simples de implementar e funciona naturalmente para consultas de soma de prefixo com atualizações pontuais — O(log n) cada. Uma segment tree é mais geral: pode lidar com qualquer operação associativa (min, max, mdc), atualizações de intervalo com propagação preguiçosa e consultas mais complexas. Fenwick tree tem melhores constantes na prática.

**P3: O que é propagação preguiçosa?**
R: Uma técnica para atualizações eficientes de intervalo em uma segment tree. Em vez de atualizar cada folha afetada imediatamente (O(n)), marque nós internos com uma "atualização pendente" (tag preguiçosa). A atualização só é aplicada quando um nó filho precisar ser acessado. Alcança O(log n) para atualização de intervalo + O(log n) para consulta.

**P4: Como funciona uma Fenwick tree?**
R: Cada índice i armazena a soma de um intervalo específico determinado pelo bit menos significativo de i. `i & (-i)` isola esse bit. Atualização: adiciona a i, depois salta para o próximo índice responsável. Consulta de prefixo: soma i, depois salta para baixo removendo o bit menos significativo. Ambas as operações são O(log n).

---

## 9. Exercícios

**Exercício 1:** Implemente consulta de soma de intervalo com atualizações pontuais usando tanto uma segment tree quanto uma Fenwick tree. Verifique que produzem resultados idênticos.

**Exercício 2:** Estenda a segment tree para suportar consulta de mínimo de intervalo (RMQ). Lide com atualizações pontuais e consultas de intervalo.

**Exercício 3:** Conte o número de elementos menores à direita de cada elemento em um array usando uma Fenwick tree com compressão de coordenadas. Esperado: O(n log n).
*Dica: processe da direita para a esquerda, use BIT para contar quantos elementos menores que o atual foram vistos.*

**Exercício 4:** Implemente propagação preguiçosa para suportar: (a) adicionar val a todos os elementos no intervalo [l, r], e (b) consultar soma dos elementos no intervalo [l, r]. Ambos em O(log n).

**Exercício 5:** Dado um array de inteiros, encontre o número de consultas de soma de intervalo onde a soma é zero. Use somas de prefixo + um hashmap.
*Dica: se prefixSum[i] == prefixSum[j], então soma(i+1..j) == 0.*

---

## 10. Leituras Complementares

- CLRS Capítulo 14 — Aumentando Estruturas de Dados
- [Segment Tree — cp-algorithms.com](https://cp-algorithms.com/data_structures/segment_tree.html)
- [Fenwick Tree — cp-algorithms.com](https://cp-algorithms.com/data_structures/fenwick.html)
- [Propagação Preguiçosa — codeforces.com](https://codeforces.com/blog/entry/18051)
- Problemas do LeetCode: #307 Range Sum Query Mutable (segment tree / BIT), #315 Count of Smaller Numbers After Self (BIT), #327 Count of Range Sum
