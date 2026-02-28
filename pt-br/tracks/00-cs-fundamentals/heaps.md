# Heaps

## 1. O Que É e Por Que Importa

Um heap é uma estrutura de dados especializada baseada em árvore que satisfaz a **propriedade de heap**: em um max-heap, todo pai é maior ou igual aos seus filhos; em um min-heap, todo pai é menor ou igual aos seus filhos. Essa propriedade torna encontrar o máximo (ou mínimo) uma operação O(1) e mantê-lo após inserções e remoções uma operação O(log n).

Heaps sustentam filas de prioridade — uma das estruturas de dados mais úteis para escalonamento, algoritmos de grafos (Dijkstra) e processamento de streams (encontrar os k maiores elementos). Entender heaps é essencial para entender como sistemas operacionais escalonam processos, como funciona o pathfinding A* e como operações de merge em listas ordenadas são feitas eficientemente.

---

## 2. Conceitos Fundamentais

### Heap como Árvore Binária Completa

Um heap é uma **árvore binária completa**: todos os níveis estão completamente preenchidos exceto possivelmente o último, que é preenchido da esquerda para a direita. Essa forma específica permite que o heap seja armazenado como um array plano — sem ponteiros necessários:

```
Para um nó no índice i (0-based):
  Pai:         Math.floor((i - 1) / 2)
  Filho esq.:  2 * i + 1
  Filho dir.:  2 * i + 2
```

### Min-Heap vs Max-Heap

- **Min-heap:** pai ≤ filhos. A raiz é sempre o mínimo. Usado para "k menores", algoritmo de Dijkstra.
- **Max-heap:** pai ≥ filhos. A raiz é sempre o máximo. Usado para "k maiores", heap sort.

### Operações Principais

- **insert (heapify up):** Adiciona ao final do array, depois sobe para restaurar a propriedade de heap. O(log n).
- **extractMin/extractMax (heapify down):** Remove a raiz, substitui com o último elemento, depois desce. O(log n).
- **peek:** Retorna a raiz sem remover. O(1).
- **buildHeap:** Converte um array arbitrário em um heap. O(n) — não O(n log n). A percepção chave: heapify começando do último nó não-folha para baixo.

### Por que buildHeap é O(n) e não O(n log n)

Intuitivamente, a maioria dos nós de um heap está perto do fundo e precisa descer apenas alguns níveis. O trabalho total é limitado pela soma das alturas de todos os nós, que é O(n) pelo argumento de série geométrica.

---

## 3. Como Funciona

```
Min-heap como array: [1, 3, 2, 7, 6, 5, 4]

Como árvore:
        1          ← índice 0
       / \
      3   2        ← índices 1, 2
     / \ / \
    7  6 5  4     ← índices 3, 4, 5, 6

Inserir 0:
  Anexa: [1, 3, 2, 7, 6, 5, 4, 0]  (índice 7)
  Pai do 7 = índice 3 (valor 7). 0 < 7 → troca
  [1, 3, 2, 0, 6, 5, 4, 7]
  Pai do 3 = índice 1 (valor 3). 0 < 3 → troca
  [1, 0, 2, 3, 6, 5, 4, 7]
  Pai do 1 = índice 0 (valor 1). 0 < 1 → troca
  [0, 1, 2, 3, 6, 5, 4, 7]  ← concluído, 0 agora é a raiz

Extrair mínimo (remove raiz = 0):
  Substitui raiz com último elemento 7:
  [7, 1, 2, 3, 6, 5, 4]
  Desce: 7 > min(1, 2) = 1 → troca com filho esquerdo
  [1, 7, 2, 3, 6, 5, 4]
  7 > min(3, 6) = 3 → troca com filho esquerdo de 1
  [1, 3, 2, 7, 6, 5, 4]  ← concluído
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Implementação de MinHeap ---
class MinHeap {
  private heap: number[] = [];

  get size(): number { return this.heap.length; }
  peek(): number | undefined { return this.heap[0]; }

  insert(val: number): void {
    this.heap.push(val);
    this.heapifyUp(this.heap.length - 1);
  }

  extractMin(): number | undefined {
    if (this.heap.length === 0) return undefined;
    if (this.heap.length === 1) return this.heap.pop();

    const min = this.heap[0];
    this.heap[0] = this.heap.pop()!; // substitui raiz com último elemento
    this.heapifyDown(0);
    return min;
  }

  // Constrói heap a partir de array arbitrário: O(n)
  buildFrom(arr: number[]): void {
    this.heap = [...arr];
    // Começa do último nó não-folha, desce cada um
    const lastNonLeaf = Math.floor(this.heap.length / 2) - 1;
    for (let i = lastNonLeaf; i >= 0; i--) {
      this.heapifyDown(i);
    }
  }

  private heapifyUp(i: number): void {
    while (i > 0) {
      const parent = Math.floor((i - 1) / 2);
      if (this.heap[parent] <= this.heap[i]) break;
      [this.heap[parent], this.heap[i]] = [this.heap[i], this.heap[parent]];
      i = parent;
    }
  }

  private heapifyDown(i: number): void {
    const n = this.heap.length;
    while (true) {
      let smallest = i;
      const left = 2 * i + 1;
      const right = 2 * i + 2;

      if (left < n && this.heap[left] < this.heap[smallest]) smallest = left;
      if (right < n && this.heap[right] < this.heap[smallest]) smallest = right;

      if (smallest === i) break;
      [this.heap[smallest], this.heap[i]] = [this.heap[i], this.heap[smallest]];
      i = smallest;
    }
  }
}

// --- Fila de Prioridade Genérica (min ou max) ---
class PriorityQueue<T> {
  private heap: T[] = [];
  private comparator: (a: T, b: T) => number;

  constructor(comparator: (a: T, b: T) => number) {
    this.comparator = comparator;
  }

  push(item: T): void {
    this.heap.push(item);
    this.heapifyUp(this.heap.length - 1);
  }

  pop(): T | undefined {
    if (this.heap.length === 0) return undefined;
    if (this.heap.length === 1) return this.heap.pop();
    const top = this.heap[0];
    this.heap[0] = this.heap.pop()!;
    this.heapifyDown(0);
    return top;
  }

  peek(): T | undefined { return this.heap[0]; }
  get size(): number { return this.heap.length; }
  isEmpty(): boolean { return this.heap.length === 0; }

  private heapifyUp(i: number): void {
    while (i > 0) {
      const parent = Math.floor((i - 1) / 2);
      if (this.comparator(this.heap[parent], this.heap[i]) <= 0) break;
      [this.heap[parent], this.heap[i]] = [this.heap[i], this.heap[parent]];
      i = parent;
    }
  }

  private heapifyDown(i: number): void {
    const n = this.heap.length;
    while (true) {
      let best = i;
      const left = 2 * i + 1;
      const right = 2 * i + 2;
      if (left < n && this.comparator(this.heap[left], this.heap[best]) < 0) best = left;
      if (right < n && this.comparator(this.heap[right], this.heap[best]) < 0) best = right;
      if (best === i) break;
      [this.heap[best], this.heap[i]] = [this.heap[i], this.heap[best]];
      i = best;
    }
  }
}

// Uso:
const minPQ = new PriorityQueue<number>((a, b) => a - b); // min-heap
const maxPQ = new PriorityQueue<number>((a, b) => b - a); // max-heap
const taskPQ = new PriorityQueue<{ priority: number; task: string }>(
  (a, b) => a.priority - b.priority
);

// --- K Maiores Elementos: O(n log k) ---
// Usa um MIN-heap de tamanho k — se novo elemento > min do heap, substitui
function kLargest(nums: number[], k: number): number[] {
  const minHeap = new PriorityQueue<number>((a, b) => a - b);

  for (const num of nums) {
    minHeap.push(num);
    if (minHeap.size > k) minHeap.pop(); // remove o menor, mantendo o top k
  }

  const result: number[] = [];
  while (!minHeap.isEmpty()) result.push(minHeap.pop()!);
  return result.reverse(); // maior primeiro
}

// --- K Menores Elementos: O(n log k) ---
function kSmallest(nums: number[], k: number): number[] {
  const maxHeap = new PriorityQueue<number>((a, b) => b - a);
  for (const num of nums) {
    maxHeap.push(num);
    if (maxHeap.size > k) maxHeap.pop();
  }
  const result: number[] = [];
  while (!maxHeap.isEmpty()) result.push(maxHeap.pop()!);
  return result;
}

// --- Mesclar K Listas Ordenadas: O(n log k) onde n = total de elementos ---
class ListNode {
  constructor(public val: number, public next: ListNode | null = null) {}
}

function mergeKLists(lists: Array<ListNode | null>): ListNode | null {
  const pq = new PriorityQueue<ListNode>((a, b) => a.val - b.val);

  // Insere a cabeça de cada lista
  for (const head of lists) {
    if (head !== null) pq.push(head);
  }

  const dummy = new ListNode(0);
  let current = dummy;

  while (!pq.isEmpty()) {
    const node = pq.pop()!;
    current.next = node;
    current = current.next;
    if (node.next !== null) pq.push(node.next);
  }

  return dummy.next;
}

// --- Encontrar Mediana de Stream de Dados: O(log n) para adicionar, O(1) para mediana ---
// Dois heaps: maxHeap para metade inferior, minHeap para metade superior
class MedianFinder {
  private lower = new PriorityQueue<number>((a, b) => b - a); // max-heap (metade inferior)
  private upper = new PriorityQueue<number>((a, b) => a - b); // min-heap (metade superior)

  addNum(num: number): void {
    // Adiciona à metade inferior primeiro
    this.lower.push(num);

    // Balanceia: max da inferior deve ser ≤ min da superior
    if (!this.upper.isEmpty() && this.lower.peek()! > this.upper.peek()!) {
      this.upper.push(this.lower.pop()!);
    }

    // Balanceia tamanhos: inferior pode ser no máximo 1 maior que superior
    if (this.lower.size > this.upper.size + 1) {
      this.upper.push(this.lower.pop()!);
    } else if (this.upper.size > this.lower.size) {
      this.lower.push(this.upper.pop()!);
    }
  }

  findMedian(): number {
    if (this.lower.size > this.upper.size) return this.lower.peek()!;
    return (this.lower.peek()! + this.upper.peek()!) / 2;
  }
}

// --- Heap Sort: O(n log n) in-place ---
function heapSort(arr: number[]): number[] {
  const a = [...arr];
  const n = a.length;

  // Constrói max-heap
  for (let i = Math.floor(n / 2) - 1; i >= 0; i--) {
    siftDown(a, i, n);
  }

  // Extrai o máximo um por um, colocando no final
  for (let end = n - 1; end > 0; end--) {
    [a[0], a[end]] = [a[end], a[0]]; // move max para o final
    siftDown(a, 0, end);              // restaura heap no intervalo reduzido
  }
  return a;
}

function siftDown(arr: number[], i: number, n: number): void {
  while (true) {
    let largest = i;
    const left = 2 * i + 1, right = 2 * i + 2;
    if (left < n && arr[left] > arr[largest]) largest = left;
    if (right < n && arr[right] > arr[largest]) largest = right;
    if (largest === i) break;
    [arr[largest], arr[i]] = [arr[i], arr[largest]];
    i = largest;
  }
}

// --- K Pontos Mais Próximos da Origem: O(n log k) ---
function kClosest(points: [number, number][], k: number): [number, number][] {
  const distSq = ([x, y]: [number, number]) => x * x + y * y;
  // Max-heap de tamanho k — remove quando tamanho > k
  const maxHeap = new PriorityQueue<[number, number]>((a, b) => distSq(b) - distSq(a));

  for (const point of points) {
    maxHeap.push(point);
    if (maxHeap.size > k) maxHeap.pop();
  }

  const result: [number, number][] = [];
  while (!maxHeap.isEmpty()) result.push(maxHeap.pop()!);
  return result;
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Usar max-heap quando você precisa dos k menores (ou vice-versa).** Para "k maiores elementos", use um MIN-heap de tamanho k — remove o menor sempre que o tamanho exceder k, mantendo os k maiores. Contraintuitivo mas correto.

> ⚠️ **Off-by-one nas fórmulas de índice pai/filho.** Indexação 0-based: pai = `Math.floor((i - 1) / 2)`, esquerdo = `2i + 1`, direito = `2i + 2`. Para 1-based: pai = `Math.floor(i / 2)`, esquerdo = `2i`, direito = `2i + 1`.

> ⚠️ **Esquecer que JavaScript não tem fila de prioridade integrada.** Você deve implementar uma ou usar uma biblioteca. `Array.sort()` a cada vez é O(n log n) por operação — não O(log n).

> ⚠️ **Complexidade do buildHeap.** Construir um heap inserindo n elementos um por vez é O(n log n). O algoritmo buildHeap de Floyd (desce a partir do último nó não-folha) é O(n) — use-o ao converter um array existente.

> ⚠️ **Heap NÃO é um array ordenado.** Heap apenas garante que a raiz é min/max. O restante do array não está ordenado.

---

## 6. Quando Usar / Não Usar

**Use um heap quando:**
- Precisa extrair repetidamente o mínimo ou máximo de uma coleção dinâmica
- Encontrar os k maiores/menores elementos de um stream
- Mesclar k listas ordenadas
- Implementar Dijkstra ou o algoritmo de Prim
- Escalonar tarefas por prioridade
- Encontrar a mediana de um stream de dados

**NÃO use um heap quando:**
- Precisa buscar um elemento arbitrário — heap é O(n) para busca arbitrária
- Precisa de saída ordenada — use sort ou BST
- Precisa de acesso aleatório por índice — use um array

---

## 7. Cenário do Mundo Real

Um sistema de fila de jobs processa tarefas por prioridade. Novos jobs chegam constantemente; o sistema sempre executa a tarefa pendente de maior prioridade:

```typescript
interface Job {
  id: string;
  priority: number;
  execute: () => Promise<void>;
}

class JobScheduler {
  private queue = new PriorityQueue<Job>((a, b) => b.priority - a.priority); // max-heap por prioridade
  private running = false;

  submit(job: Job): void {
    this.queue.push(job);
    if (!this.running) this.processNext();
  }

  private async processNext(): Promise<void> {
    if (this.queue.isEmpty()) { this.running = false; return; }
    this.running = true;
    const job = this.queue.pop()!;
    try {
      await job.execute();
    } finally {
      this.processNext(); // processa o próximo job após o atual terminar
    }
  }
}
```

Um sistema de monitoramento precisa dos 10 endpoints de API mais lentos entre milhões de requisições:

```typescript
function top10Slowest(responseTimes: { endpoint: string; ms: number }[]) {
  const k = 10;
  const minHeap = new PriorityQueue<{ endpoint: string; ms: number }>(
    (a, b) => a.ms - b.ms // min-heap
  );

  for (const entry of responseTimes) {
    minHeap.push(entry);
    if (minHeap.size > k) minHeap.pop(); // descarta o mais rápido, mantém top 10 mais lentos
  }

  const result = [];
  while (!minHeap.isEmpty()) result.push(minHeap.pop()!);
  return result.reverse(); // mais lento primeiro
}
```

---

## 8. Perguntas de Entrevista

**Q1: Por que construir um heap é O(n) e não O(n log n)?**
R: Ao inserir n elementos um por um, cada inserção é O(log n) → total O(n log n). Mas o buildHeap de Floyd começa do último nó não-folha e desce. A maioria dos nós está perto do fundo e precisa de apenas O(1) de trabalho de descida. O trabalho total é limitado pela soma das alturas dos nós = O(n) por série geométrica.

**Q2: Como você implementaria uma fila de prioridade?**
R: Use um heap armazenado como array. Inserir: insere no final, heapify up — O(log n). Extrair min/max: troca raiz com o último, remove o último, heapify down — O(log n). Peek: retorna raiz — O(1).

**Q3: Como você encontra o k-ésimo maior elemento em um array?**
R: Use um min-heap de tamanho k. Para cada elemento, insira no heap; se tamanho > k, remove o mínimo. Após processar todos os elementos, o heap contém os k maiores; a raiz é o k-ésimo maior. O(n log k).

**Q4: Como você encontra a mediana de um stream de dados eficientemente?**
R: Dois heaps: um max-heap para a metade inferior e um min-heap para a metade superior. Mantenha-os balanceados (tamanhos diferem em no máximo 1). Após cada inserção, balanceie os heaps se necessário. A mediana é a raiz do heap maior (total ímpar) ou a média de ambas as raízes (total par). O(log n) para inserção, O(1) para mediana.

**Q5: O que é heap sort e quais são suas propriedades?**
R: Constrói um max-heap a partir do array (O(n)), depois extrai repetidamente o máximo e coloca no final (O(n log n)). Total: O(n log n). Propriedades: in-place (O(1) de espaço), não estável, O(n log n) no pior caso garantido (ao contrário do quicksort).

---

## 9. Exercícios

**Exercício 1:** Ordene um k-sorted array — um array onde cada elemento está no máximo k posições longe de sua posição correta ordenada. Alcance O(n log k).
*Dica: mantenha um min-heap de tamanho k+1. Para cada novo elemento, insira no heap e remova o mínimo para a saída.*

**Exercício 2:** Encontre os k pontos mais próximos da origem de uma lista de pontos 2D.
*Dica: max-heap de tamanho k, ordenado por distância ao quadrado — sem necessidade de sqrt.*

**Exercício 3:** Projete um escalonador de tarefas. Dadas tarefas com prioridades e tempos de execução, simule a execução de forma que a tarefa de maior prioridade rode primeiro. Quando uma tarefa termina, a próxima tarefa disponível de maior prioridade roda.

**Exercício 4:** Dada uma lista de inteiros, verifique se algum par soma um alvo k usando uma abordagem baseada em heap.

**Exercício 5:** Implemente heap sort do zero e verifique que produz a saída ordenada correta.

---

## 10. Leituras Complementares

- CLRS Capítulo 6 — Heapsort
- [Binary Heap — Wikipedia](https://en.wikipedia.org/wiki/Binary_heap)
- [Visualização de Heap](https://visualgo.net/en/heap)
- Problemas LeetCode: #215 Kth Largest Element, #23 Merge K Sorted Lists, #295 Find Median from Data Stream, #347 Top K Frequent Elements, #973 K Closest Points to Origin
