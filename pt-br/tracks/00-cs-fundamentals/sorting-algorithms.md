# Algoritmos de Ordenação

## 1. O Que É e Por Que Importa

Ordenação é uma das operações mais fundamentais da ciência da computação. Dados ordenados habilitam busca binária (O(log n) vs O(n)), tornam duplicatas adjacentes (fácil deduplicação) e são pré-requisito para muitos algoritmos (merge de intervalos, técnica de dois ponteiros, Dijkstra).

Entender como diferentes algoritmos de ordenação funcionam — suas trocas em tempo, espaço, estabilidade e adaptabilidade — ajuda você a escolher a ferramenta certa e raciocinar sobre performance. Também fornece insight sobre padrões de design de algoritmos: divisão e conquista (merge sort), manipulação in-place (quicksort, heap sort) e exploração de propriedades dos dados (counting sort, radix sort).

---

## 2. Conceitos Fundamentais

### Estabilidade

Um sort é **estável** se preserva a ordem relativa de elementos com chaves iguais. Exemplo: ordenar uma lista de usuários por idade — se dois usuários têm 25 anos, sua ordem relativa da entrada é preservada.

Sorts estáveis: Merge sort, Timsort, Counting sort, Insertion sort.
Sorts instáveis: Quicksort, Heap sort, Selection sort.

Estabilidade importa ao ordenar por múltiplos critérios sequencialmente, ou quando a ordem original carrega significado.

### In-Place

Um sort é **in-place** se usa O(1) de espaço auxiliar (ignorando a pilha de chamadas). Quicksort e Heap sort são in-place. Merge sort requer O(n) de espaço extra para o merge.

### Limite Inferior Baseado em Comparação

Qualquer algoritmo de ordenação que apenas compara elementos requer pelo menos O(n log n) comparações no pior caso. Esse é um limite inferior matemático — você não pode fazer melhor apenas com ordenação por comparação. Prova: há n! possíveis ordenações; cada comparação elimina metade; você precisa de pelo menos log₂(n!) ≈ n log n comparações.

### Ordenação Linear

Algoritmos como Counting Sort, Radix Sort e Bucket Sort são O(n) ou O(n + k) mas funcionam explorando a **estrutura** dos dados (chaves inteiras com intervalo limitado), não comparações. Eles contornam o limite inferior O(n log n) porque ele se aplica apenas a sorts baseados em comparação.

---

## 3. Tabela Comparativa

| Algoritmo | Melhor | Médio | Pior | Espaço | Estável | In-Place | Notas |
|---|---|---|---|---|---|---|---|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Sim | Sim | Bom apenas com early-exit |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | Não | Sim | Minimiza trocas (bom para flash) |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Sim | Sim | Melhor para arrays quase ordenados ou pequenos |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Sim | Não | O(n log n) garantido, melhor para listas encadeadas |
| Quicksort | O(n log n) | O(n log n) | O(n²) | O(log n) | Não | Sim | Rápido na prática; pior caso com pivô ruim |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | Não | Sim | O(n log n) garantido, pior cache que quicksort |
| Timsort | O(n) | O(n log n) | O(n log n) | O(n) | Sim | Não | Usado em JS/Python; otimizado para dados reais |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Sim | Não | Apenas para chaves inteiras no intervalo [0, k] |
| Radix Sort | O(d(n+k)) | O(d(n+k)) | O(d(n+k)) | O(n+k) | Sim | Não | d = dígitos, k = base; para inteiros/strings |
| Bucket Sort | O(n+k) | O(n+k) | O(n²) | O(n+k) | Sim | Não | Para floats distribuídos uniformemente |

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Bubble Sort: O(n²) pior caso, O(n) melhor caso com early exit ---
function bubbleSort(arr: number[]): number[] {
  const a = [...arr];
  for (let i = 0; i < a.length; i++) {
    let swapped = false;
    for (let j = 0; j < a.length - i - 1; j++) {
      if (a[j] > a[j + 1]) {
        [a[j], a[j + 1]] = [a[j + 1], a[j]];
        swapped = true;
      }
    }
    if (!swapped) break; // já ordenado — O(n) melhor caso
  }
  return a;
}

// --- Selection Sort: O(n²) sempre, minimiza trocas ---
function selectionSort(arr: number[]): number[] {
  const a = [...arr];
  for (let i = 0; i < a.length; i++) {
    let minIdx = i;
    for (let j = i + 1; j < a.length; j++) {
      if (a[j] < a[minIdx]) minIdx = j;
    }
    if (minIdx !== i) [a[i], a[minIdx]] = [a[minIdx], a[i]];
  }
  return a;
}

// --- Insertion Sort: O(n²) pior caso, O(n) melhor caso (quase ordenado) ---
// Melhor escolha para arrays pequenos (n < 20) ou entrada quase ordenada
function insertionSort(arr: number[]): number[] {
  const a = [...arr];
  for (let i = 1; i < a.length; i++) {
    const key = a[i];
    let j = i - 1;
    // Desloca elementos à direita até encontrar a posição correta para key
    while (j >= 0 && a[j] > key) {
      a[j + 1] = a[j];
      j--;
    }
    a[j + 1] = key;
  }
  return a;
}

// --- Merge Sort: O(n log n) sempre, estável, O(n) de espaço ---
function mergeSort(arr: number[]): number[] {
  if (arr.length <= 1) return arr;

  const mid = Math.floor(arr.length / 2);
  const left = mergeSort(arr.slice(0, mid));
  const right = mergeSort(arr.slice(mid));

  return merge(left, right);
}

function merge(left: number[], right: number[]): number[] {
  const result: number[] = [];
  let i = 0, j = 0;

  while (i < left.length && j < right.length) {
    if (left[i] <= right[j]) { // <= preserva estabilidade
      result.push(left[i++]);
    } else {
      result.push(right[j++]);
    }
  }
  // Anexa elementos restantes
  while (i < left.length) result.push(left[i++]);
  while (j < right.length) result.push(right[j++]);
  return result;
}

// Merge sort: versão iterativa bottom-up (O(1) de espaço na pilha)
function mergeSortIterative(arr: number[]): number[] {
  const a = [...arr];
  const n = a.length;
  // Mescla subarrays de tamanho crescente: 1, 2, 4, 8, ...
  for (let size = 1; size < n; size *= 2) {
    for (let start = 0; start < n; start += 2 * size) {
      const mid = Math.min(start + size, n);
      const end = Math.min(start + 2 * size, n);
      const merged = merge(a.slice(start, mid), a.slice(mid, end));
      a.splice(start, end - start, ...merged);
    }
  }
  return a;
}

// --- Quicksort: O(n log n) médio, O(n²) pior caso, in-place ---
function quicksort(arr: number[], low = 0, high = arr.length - 1): number[] {
  if (low < high) {
    const pivotIdx = partition(arr, low, high);
    quicksort(arr, low, pivotIdx - 1);
    quicksort(arr, pivotIdx + 1, high);
  }
  return arr;
}

// Esquema de partição de Lomuto
function partition(arr: number[], low: number, high: number): number {
  // Aleatoriza pivô para evitar pior caso O(n²) em entrada ordenada
  const randomIdx = low + Math.floor(Math.random() * (high - low + 1));
  [arr[randomIdx], arr[high]] = [arr[high], arr[randomIdx]];

  const pivot = arr[high];
  let i = low - 1;

  for (let j = low; j < high; j++) {
    if (arr[j] <= pivot) {
      i++;
      [arr[i], arr[j]] = [arr[j], arr[i]];
    }
  }
  [arr[i + 1], arr[high]] = [arr[high], arr[i + 1]];
  return i + 1;
}

// --- Quicksort: partição em 3 vias (trata duplicatas eficientemente) ---
// Útil quando há muitos elementos duplicados — reduz O(n log n) para O(n) em arrays iguais
function quicksort3Way(arr: number[], low = 0, high = arr.length - 1): void {
  if (low >= high) return;
  const pivot = arr[low];
  let lt = low, gt = high, i = low + 1;

  while (i <= gt) {
    if (arr[i] < pivot) { [arr[lt], arr[i]] = [arr[i], arr[lt]]; lt++; i++; }
    else if (arr[i] > pivot) { [arr[i], arr[gt]] = [arr[gt], arr[i]]; gt--; }
    else i++;
  }
  // arr[low..lt-1] < pivot, arr[lt..gt] == pivot, arr[gt+1..high] > pivot
  quicksort3Way(arr, low, lt - 1);
  quicksort3Way(arr, gt + 1, high);
}

// --- Counting Sort: O(n + k) onde k = intervalo de valores ---
// Funciona apenas para inteiros não-negativos com intervalo limitado
function countingSort(arr: number[], maxVal: number): number[] {
  const count = new Array(maxVal + 1).fill(0);
  for (const n of arr) count[n]++;

  // Acumula contagens (para sort estável — mantém ordem relativa)
  for (let i = 1; i <= maxVal; i++) count[i] += count[i - 1];

  const output = new Array(arr.length);
  // Constrói saída da direita para esquerda para estabilidade
  for (let i = arr.length - 1; i >= 0; i--) {
    output[--count[arr[i]]] = arr[i];
  }
  return output;
}

// --- Radix Sort: O(d × n) onde d = número de dígitos ---
// Ordena inteiros processando um dígito por vez (LSD para MSD)
function radixSort(arr: number[]): number[] {
  const maxVal = Math.max(...arr);
  let exp = 1;
  let a = [...arr];

  while (Math.floor(maxVal / exp) > 0) {
    a = countingSortByDigit(a, exp);
    exp *= 10;
  }
  return a;
}

function countingSortByDigit(arr: number[], exp: number): number[] {
  const count = new Array(10).fill(0);
  for (const n of arr) count[Math.floor(n / exp) % 10]++;
  for (let i = 1; i < 10; i++) count[i] += count[i - 1];

  const output = new Array(arr.length);
  for (let i = arr.length - 1; i >= 0; i--) {
    const digit = Math.floor(arr[i] / exp) % 10;
    output[--count[digit]] = arr[i];
  }
  return output;
}

// --- Dutch National Flag: Ordena 0s, 1s, 2s em uma passagem ---
// O(n) de tempo, O(1) de espaço — partição clássica em três vias
function dutchNationalFlag(arr: number[]): void {
  let low = 0, mid = 0, high = arr.length - 1;

  while (mid <= high) {
    if (arr[mid] === 0) {
      [arr[low], arr[mid]] = [arr[mid], arr[low]];
      low++; mid++;
    } else if (arr[mid] === 1) {
      mid++;
    } else { // arr[mid] === 2
      [arr[mid], arr[high]] = [arr[high], arr[mid]];
      high--; // não incrementa mid — novo arr[mid] é desconhecido
    }
  }
}

// --- Ordenar Lista Encadeada: Merge Sort — O(n log n), O(log n) de espaço ---
// Merge sort é preferido para listas encadeadas porque:
// 1. Acesso aleatório é O(n) — seleção de pivô no quicksort é complicada
// 2. O passo de merge do merge sort é O(1) de espaço para listas encadeadas (manipulação de ponteiros)
class ListNode {
  constructor(public val: number, public next: ListNode | null = null) {}
}

function sortList(head: ListNode | null): ListNode | null {
  if (head === null || head.next === null) return head;

  // Encontra o meio (ponteiro lento/rápido)
  let slow = head, fast: ListNode | null = head.next;
  while (fast !== null && fast.next !== null) {
    slow = slow.next!;
    fast = fast.next.next;
  }

  const mid = slow.next;
  slow.next = null; // divide

  const left = sortList(head);
  const right = sortList(mid);
  return mergeLinked(left, right);
}

function mergeLinked(l1: ListNode | null, l2: ListNode | null): ListNode | null {
  const dummy = new ListNode(0);
  let cur = dummy;
  while (l1 !== null && l2 !== null) {
    if (l1.val <= l2.val) { cur.next = l1; l1 = l1.next; }
    else { cur.next = l2; l2 = l2.next; }
    cur = cur.next;
  }
  cur.next = l1 ?? l2;
  return dummy.next;
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Quicksort em entrada ordenada sem pivô aleatório.** Usar o último ou primeiro elemento como pivô em um array já ordenado resulta em performance O(n²). Sempre aleatorize a seleção do pivô.

> ⚠️ **Assumir que `Array.sort()` do JavaScript é instável.** Desde o V8 7.0 (Node.js 11+), `Array.sort()` usa Timsort e é estável. Para ambientes mais antigos ou segurança cross-browser, seja explícito se estabilidade importa.

> ⚠️ **Usar `Array.sort()` sem comparador para números.** `[10, 2, 1].sort()` → `[1, 10, 2]` (lexicográfico). Sempre passe um comparador: `.sort((a, b) => a - b)`.

> ⚠️ **Merge sort em listas encadeadas vs arrays.** Para arrays, o passo de merge do merge sort requer O(n) de espaço extra. Para listas encadeadas, o merge é in-place (O(1) de espaço) — apenas reconecta ponteiros. Por isso o merge sort é preferido para listas encadeadas.

> ⚠️ **Counting sort com k grande.** Se os valores variam de 0 a 10⁹, você não pode alocar um array de 10⁹. Use radix sort ou sort por comparação em vez disso.

---

## 6. Quando Usar / Não Usar

**Insertion sort quando:**
- Array é pequeno (n < 20) — baixo overhead, amigável ao cache
- Array está quase ordenado — O(n) melhor caso

**Merge sort quando:**
- Estabilidade é necessária
- Ordenando uma lista encadeada (merge in-place, sem acesso aleatório necessário)
- Ordenação externa (dados não cabem na memória — mescla chunks do disco)

**Quicksort quando:**
- Performance média importa, não pior caso
- Ordenação in-place preferida (memória limitada)
- Dados têm muitos valores distintos (evita pior caso com partição em 3 vias para duplicatas)

**Heap sort quando:**
- O(n log n) garantido no pior caso com O(1) de espaço
- Não se preocupa com performance de cache

**Counting/Radix sort quando:**
- Chaves inteiras com intervalo limitado conhecido
- Precisa de sort em tempo linear

---

## 7. Cenário do Mundo Real

Timsort — o algoritmo usado no `Array.sort()` do JavaScript e no `sorted()` do Python — é um híbrido que combina merge sort e insertion sort. Ele explora "runs" naturalmente ocorrentes (subsequências já ordenadas) em dados do mundo real:

1. Varre o array por "runs naturais" (subsequências já ordenadas ou ordenadas inversamente)
2. Estende runs curtos a um tamanho mínimo usando insertion sort (insertion sort é rápido em arrays pequenos)
3. Mescla runs usando o passo de merge do merge sort

Dados do mundo real raramente são aleatórios — registros de usuários frequentemente têm porções já ordenadas por data, IDs de produtos são frequentemente quase monotônicos. O melhor caso O(n) do Timsort para entrada já ordenada não é apenas teórico — acontece constantemente na prática.

---

## 8. Perguntas de Entrevista

**Q1: Por que merge sort é preferido para listas encadeadas?**
R: Quicksort precisa de acesso aleatório para seleção eficiente de pivô, e o acesso em lista encadeada é O(n). Merge sort precisa apenas de acesso sequencial. Além disso, o passo de merge para listas encadeadas é O(1) de espaço (apenas reconecta ponteiros), enquanto mesclar arrays requer O(n) de espaço extra.

**Q2: Quando você escolheria quicksort sobre merge sort?**
R: Quando você precisa de ordenação in-place e a performance do caso médio importa mais do que garantias de pior caso. Quicksort tem melhor performance de cache na prática (padrão de acesso sequencial à memória) e fatores constantes menores. Com pivô aleatório, o pior caso O(n²) é astronomicamente improvável.

**Q3: Por que counting sort ser O(n) não é uma contradição do limite inferior O(n log n)?**
R: O limite inferior O(n log n) se aplica especificamente à ordenação baseada em comparação — algoritmos que apenas usam comparações de elementos. Counting sort não compara elementos; ele explora a estrutura de chave inteira para calcular posições diretamente. É um modelo de computação diferente, então o limite inferior não se aplica.

**Q4: O que é Timsort?**
R: Um algoritmo de ordenação híbrido usado em Python e JavaScript. Combina insertion sort (para runs pequenos) com merge sort (para mesclar runs). Detecta subsequências naturalmente ordenadas nos dados, tornando-o O(n) para entrada já ordenada e O(n log n) no pior caso. Estável.

**Q5: Ordene um array contendo apenas 0s, 1s e 2s em uma passagem.**
R: Algoritmo da Bandeira Nacional Holandesa de Dijkstra. Três ponteiros: low, mid, high. Invariante: tudo antes de low é 0, entre low e mid é 1, após high é 2. O(n) de tempo, O(1) de espaço.

**Q6: O que torna um algoritmo de ordenação estável e por que importa?**
R: Sort estável preserva a ordem relativa original de elementos iguais. Importa ao ordenar por múltiplas chaves: primeiro ordena pela chave secundária (sort estável), depois ordena pela chave primária — a ordem secundária é preservada dentro de chaves primárias iguais. Sort instável embaralharia a ordem secundária.

---

## 9. Exercícios

**Exercício 1:** Ordene um array de 0s, 1s e 2s em uma única passagem sem usar biblioteca de ordenação. O problema Dutch National Flag.
*Dica: três ponteiros — low (fronteira dos 0s), mid (atual), high (fronteira dos 2s).*

**Exercício 2:** Encontre o k-ésimo maior elemento em um array não ordenado em O(n) de tempo médio.
*Dica: Quickselect — quicksort parcial que recorre apenas na partição relevante.*

**Exercício 3:** Ordene uma lista encadeada em O(n log n) de tempo e O(log n) de espaço.
*Dica: merge sort — encontra o meio com ponteiro rápido/lento, divide, ordena cada metade, mescla.*

**Exercício 4:** Implemente counting sort para um array de objetos ordenados por um campo numérico (ex.: idade). Garanta que o sort seja estável.

**Exercício 5:** Dado um array de inteiros, encontre o menor gap positivo entre quaisquer dois elementos. Faça em O(n) usando conceitos de bucket/counting sort.
*Dica: princípio do pombal — com n elementos no intervalo [min, max], o gap máximo é pelo menos (max - min) / (n - 1). Use buckets desse tamanho.*

---

## 10. Leituras Complementares

- CLRS Capítulo 6 (Heapsort), Capítulo 7 (Quicksort), Capítulo 8 (Linear Sorting)
- [Timsort — descrição original de Tim Peters](https://svn.python.org/projects/python/trunk/Objects/listsort.txt)
- [Visualizador de ordenação](https://visualgo.net/en/sorting)
- LeetCode: #912 Sort an Array, #75 Sort Colors (Dutch National Flag), #148 Sort List, #215 Kth Largest Element (Quickselect)
