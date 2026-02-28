# Padrões Comuns

## 1. O Que São e Por Que Importam

Padrões algorítmicos são templates reutilizáveis de resolução de problemas. Quando você reconhece que um problema se encaixa em um padrão, pode aplicar um esqueleto de solução conhecido em vez de resolver do zero. É assim que engenheiros experientes e programadores competitivos resolvem problemas desconhecidos rapidamente.

Os padrões importam porque os problemas de entrevista raramente são novos — eles são variações de cerca de uma dúzia de templates fundamentais. Conhecer os padrões, as pistas de reconhecimento e o esqueleto de código permite que você resolva 80% dos problemas médios do LeetCode com esforço moderado.

Este arquivo cobre os 10 padrões mais importantes com implementações completas em TypeScript e três exemplos de problemas cada.

---

## 2. Conceitos Fundamentais

Cada seção de padrão segue esta estrutura:
- **O que é** — uma definição em uma frase
- **Como reconhecer** — as pistas no enunciado do problema
- **Quando usar** — a condição estrutural que torna o padrão aplicável
- **Código template** — uma implementação TypeScript reutilizável
- **Exemplos de problemas** — três problemas que se encaixam no padrão

---

## 3. Padrões

---

### Padrão 1: Janela Deslizante (Tamanho Fixo)

**O que é:** Mantém uma janela de exatamente `k` elementos sobre um array. Desliza uma posição por vez adicionando o novo elemento à direita e removendo o elemento mais à esquerda.

**Como reconhecer:**
- "Subarray/substring de tamanho k"
- "Máx/mín/média sobre todas as janelas de tamanho k"

**Quando usar:** Quando a operação sobre a janela pode ser mantida incrementalmente (adicionar elemento da direita, remover da esquerda) em vez de recomputada do zero.

**Template:**

```typescript
function slidingWindowFixed<T>(arr: T[], k: number): T[][] {
  const windows: T[][] = [];
  if (arr.length < k) return windows;

  // Constrói a primeira janela
  let windowSum = 0;
  for (let i = 0; i < k; i++) {
    windowSum += arr[i] as unknown as number;
  }
  windows.push(arr.slice(0, k));

  // Desliza: remove o mais à esquerda, adiciona o próximo à direita
  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] as unknown as number;
    windowSum -= arr[i - k] as unknown as number;
    windows.push(arr.slice(i - k + 1, i + 1));
  }

  return windows;
}

// Problema: Soma máxima de subarray de tamanho k — O(n) de tempo, O(1) de espaço
function maxSumSubarray(arr: number[], k: number): number {
  if (arr.length < k) throw new Error("Array menor que k");

  let windowSum = 0;
  for (let i = 0; i < k; i++) windowSum += arr[i];

  let maxSum = windowSum;

  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k]; // desliza
    maxSum = Math.max(maxSum, windowSum);
  }

  return maxSum;
}

// Exemplo: [2,1,5,1,3,2], k=3 → 9 (5+1+3)
console.log(maxSumSubarray([2, 1, 5, 1, 3, 2], 3)); // 9

// Problema: Média de subarrays de tamanho k
function avgOfSubarrays(arr: number[], k: number): number[] {
  const result: number[] = [];
  let windowSum = 0;

  for (let i = 0; i < k; i++) windowSum += arr[i];
  result.push(windowSum / k);

  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k];
    result.push(windowSum / k);
  }

  return result;
}

// Problema: Contar elementos distintos em cada janela de tamanho k
function countDistinctInWindows(arr: number[], k: number): number[] {
  const result: number[] = [];
  const freq = new Map<number, number>();

  // Constrói a primeira janela
  for (let i = 0; i < k; i++) {
    freq.set(arr[i], (freq.get(arr[i]) ?? 0) + 1);
  }
  result.push(freq.size);

  // Desliza
  for (let i = k; i < arr.length; i++) {
    // Adiciona elemento à direita
    freq.set(arr[i], (freq.get(arr[i]) ?? 0) + 1);

    // Remove elemento à esquerda
    const leftVal = arr[i - k];
    const leftCount = freq.get(leftVal)!;
    if (leftCount === 1) freq.delete(leftVal);
    else freq.set(leftVal, leftCount - 1);

    result.push(freq.size);
  }

  return result;
}
```

**Exemplos de Problemas:**
1. Soma máxima de subarray de tamanho k
2. Média de todos os subarrays contíguos de tamanho k
3. Contar elementos distintos em cada janela de tamanho k

---

### Padrão 2: Janela Deslizante (Tamanho Variável)

**O que é:** Expande o ponteiro direito até que uma condição seja violada, depois encolhe o ponteiro esquerdo até que a condição seja restaurada. Rastreia a melhor janela válida vista.

**Como reconhecer:**
- "Maior/menor subarray/substring que satisfaz alguma condição"
- "Janela mínima contendo..."
- A condição pode ser verificada incrementalmente e é monótona (se uma janela de tamanho w falha, janelas menores podem passar)

**Quando usar:** Quando você precisa encontrar um subarray/substring ótimo com uma restrição sobre seu conteúdo (não tamanho fixo).

**Template:**

```typescript
function slidingWindowVariable(arr: number[], condition: (window: number[]) => boolean): number {
  let left = 0;
  let maxLen = 0;

  for (let right = 0; right < arr.length; right++) {
    // Expande: adiciona arr[right] à janela
    // ... atualiza estado da janela

    // Encolhe: enquanto condição violada, move left
    while (/* condição violada */ false) {
      // remove arr[left] do estado da janela
      left++;
    }

    // Condição satisfeita — registra o melhor
    maxLen = Math.max(maxLen, right - left + 1);
  }

  return maxLen;
}

// Problema: Maior substring com no máximo k caracteres distintos — O(n)
function longestSubstringKDistinct(s: string, k: number): number {
  const freq = new Map<string, number>();
  let left = 0;
  let maxLen = 0;

  for (let right = 0; right < s.length; right++) {
    // Expande
    const c = s[right];
    freq.set(c, (freq.get(c) ?? 0) + 1);

    // Encolhe até ter no máximo k distintos
    while (freq.size > k) {
      const lc = s[left];
      const count = freq.get(lc)!;
      if (count === 1) freq.delete(lc);
      else freq.set(lc, count - 1);
      left++;
    }

    maxLen = Math.max(maxLen, right - left + 1);
  }

  return maxLen;
}

// Problema: Maior substring sem caracteres repetidos — O(n)
function longestUniqueSubstring(s: string): number {
  const lastSeen = new Map<string, number>();
  let left = 0;
  let maxLen = 0;

  for (let right = 0; right < s.length; right++) {
    const c = s[right];
    if (lastSeen.has(c) && lastSeen.get(c)! >= left) {
      left = lastSeen.get(c)! + 1; // salta left além da duplicata
    }
    lastSeen.set(c, right);
    maxLen = Math.max(maxLen, right - left + 1);
  }

  return maxLen;
}

// Problema: Janela mínima que contém todos os chars de t — O(n)
function minWindowSubstring(s: string, t: string): string {
  const need = new Map<string, number>();
  for (const c of t) need.set(c, (need.get(c) ?? 0) + 1);

  const have = new Map<string, number>();
  let formed = 0;
  const required = need.size;

  let left = 0;
  let minLen = Infinity;
  let minStart = 0;

  for (let right = 0; right < s.length; right++) {
    const c = s[right];
    have.set(c, (have.get(c) ?? 0) + 1);

    if (need.has(c) && have.get(c) === need.get(c)) formed++;

    while (formed === required) {
      // Registra o melhor
      if (right - left + 1 < minLen) {
        minLen = right - left + 1;
        minStart = left;
      }
      // Encolhe da esquerda
      const lc = s[left];
      have.set(lc, have.get(lc)! - 1);
      if (need.has(lc) && have.get(lc)! < need.get(lc)!) formed--;
      left++;
    }
  }

  return minLen === Infinity ? "" : s.slice(minStart, minStart + minLen);
}
```

**Exemplos de Problemas:**
1. Maior substring com no máximo k caracteres distintos
2. Maior substring sem caracteres repetidos
3. Substring mínima que contém todos os caracteres de t

---

### Padrão 3: Dois Ponteiros

**O que é:** Usa dois índices (geralmente `left` e `right`) que percorrem o array de extremidades opostas (ou na mesma direção) para encontrar pares ou particionar o array, evitando loops aninhados.

**Como reconhecer:**
- "Encontrar um par que soma..."
- "Remover duplicatas in-place"
- Entrada está ordenada (ou pode ser ordenada de forma barata)
- Precisa comparar elementos de ambas as extremidades

**Quando usar:** Quando a entrada está ordenada e você precisa encontrar pares, triplas ou particionar elementos. Reduz força bruta O(n²) para O(n).

**Template:**

```typescript
// Dois ponteiros em array ordenado
function twoPointerTemplate(arr: number[], target: number): [number, number] | null {
  let left = 0;
  let right = arr.length - 1;

  while (left < right) {
    const sum = arr[left] + arr[right];
    if (sum === target) return [left, right];
    if (sum < target) left++;   // precisa de soma maior
    else right--;               // precisa de soma menor
  }

  return null;
}

// Problema: Dois elementos em array ordenado que somam ao alvo — O(n) de tempo, O(1) de espaço
function twoSumSorted(nums: number[], target: number): [number, number] | null {
  let left = 0;
  let right = nums.length - 1;

  while (left < right) {
    const sum = nums[left] + nums[right];
    if (sum === target) return [left, right];
    if (sum < target) left++;
    else right--;
  }

  return null;
}

// Problema: Container com mais água — O(n)
function maxWater(height: number[]): number {
  let left = 0;
  let right = height.length - 1;
  let maxArea = 0;

  while (left < right) {
    const area = Math.min(height[left], height[right]) * (right - left);
    maxArea = Math.max(maxArea, area);

    // Move a parede mais curta — é a única forma de possivelmente encontrar mais água
    if (height[left] < height[right]) left++;
    else right--;
  }

  return maxArea;
}

// Problema: Três elementos que somam zero — O(n²)
function threeSum(nums: number[]): number[][] {
  nums.sort((a, b) => a - b);
  const result: number[][] = [];

  for (let i = 0; i < nums.length - 2; i++) {
    if (i > 0 && nums[i] === nums[i - 1]) continue; // pula duplicatas

    let left = i + 1;
    let right = nums.length - 1;

    while (left < right) {
      const sum = nums[i] + nums[left] + nums[right];
      if (sum === 0) {
        result.push([nums[i], nums[left], nums[right]]);
        while (left < right && nums[left] === nums[left + 1]) left++;
        while (left < right && nums[right] === nums[right - 1]) right--;
        left++;
        right--;
      } else if (sum < 0) {
        left++;
      } else {
        right--;
      }
    }
  }

  return result;
}
```

**Exemplos de Problemas:**
1. Dois elementos em array ordenado que somam ao alvo
2. Container com mais água
3. Três elementos que somam zero (triplas)

---

### Padrão 4: Ponteiros Rápido e Lento

**O que é:** Dois ponteiros que se movem em velocidades diferentes por uma lista encadeada ou array. O ponteiro rápido se move 2 passos por iteração, o lento se move 1.

**Como reconhecer:**
- "Detectar ciclo em lista encadeada"
- "Encontrar o meio de uma lista encadeada"
- "Palíndromo em lista encadeada"
- Problemas em listas encadeadas onde você não pode usar espaço extra

**Quando usar:** Detecção de ciclos e encontrar o meio em listas encadeadas sem alocar memória extra.

**Template:**

```typescript
interface ListNode {
  val: number;
  next: ListNode | null;
}

// Detecção de ciclo — algoritmo de Floyd — O(n) de tempo, O(1) de espaço
function hasCycle(head: ListNode | null): boolean {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;

    if (slow === fast) return true; // encontraram dentro do ciclo
  }

  return false; // fast chegou ao fim — sem ciclo
}

// Encontrar o meio de uma lista encadeada — O(n) de tempo, O(1) de espaço
// Quando fast chega ao fim, slow está no meio
function findMiddle(head: ListNode | null): ListNode | null {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
  }

  return slow; // nó do meio
}

// Encontrar o início do ciclo — fase 2 de Floyd
function detectCycleStart(head: ListNode | null): ListNode | null {
  let slow = head;
  let fast = head;

  // Fase 1: detecta ciclo
  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) break;
  }

  if (fast === null || fast.next === null) return null; // sem ciclo

  // Fase 2: encontra ponto de entrada
  // Move um ponteiro para o início; ambos avançam na velocidade 1
  slow = head;
  while (slow !== fast) {
    slow = slow!.next;
    fast = fast!.next;
  }

  return slow; // início do ciclo
}

// Verificar se lista encadeada é palíndromo usando fast/slow + inversão — O(n), O(1)
function isPalindrome(head: ListNode | null): boolean {
  if (!head || !head.next) return true;

  // Encontra o meio
  let slow: ListNode | null = head;
  let fast: ListNode | null = head;
  while (fast && fast.next) {
    slow = slow!.next;
    fast = fast.next.next;
  }

  // Inverte a segunda metade
  let prev: ListNode | null = null;
  let curr: ListNode | null = slow;
  while (curr) {
    const next = curr.next;
    curr.next = prev;
    prev = curr;
    curr = next;
  }

  // Compara ambas as metades
  let left: ListNode | null = head;
  let right: ListNode | null = prev;
  while (right) {
    if (left!.val !== right.val) return false;
    left = left!.next;
    right = right.next;
  }

  return true;
}
```

**Exemplos de Problemas:**
1. Detectar ciclo em lista encadeada
2. Encontrar o meio de uma lista encadeada
3. Verificar palíndromo em lista encadeada

---

### Padrão 5: Mesclar Intervalos

**O que é:** Ordena intervalos por tempo de início, depois varre linearmente: se o intervalo atual se sobrepõe com o último intervalo mesclado, estende-o; caso contrário, adiciona como novo intervalo.

**Como reconhecer:**
- "Mesclar intervalos sobrepostos"
- "Encontrar todas as reuniões conflitantes"
- "Número mínimo de salas de reunião"
- Qualquer problema envolvendo intervalos que podem se sobrepor

**Quando usar:** Quando você precisa combinar ou contar intervalos de tempo, ranges ou segmentos sobrepostos.

**Template:**

```typescript
type Interval = [number, number]; // [início, fim]

// Mesclar intervalos sobrepostos — O(n log n)
function mergeIntervals(intervals: Interval[]): Interval[] {
  if (intervals.length === 0) return [];

  // Ordena por tempo de início
  intervals.sort((a, b) => a[0] - b[0]);

  const merged: Interval[] = [intervals[0]];

  for (let i = 1; i < intervals.length; i++) {
    const last = merged[merged.length - 1];
    const curr = intervals[i];

    if (curr[0] <= last[1]) {
      // Sobreposição — estende
      last[1] = Math.max(last[1], curr[1]);
    } else {
      // Sem sobreposição — adiciona novo
      merged.push(curr);
    }
  }

  return merged;
}

// Inserir um novo intervalo e mesclar — O(n)
function insertInterval(intervals: Interval[], newInterval: Interval): Interval[] {
  const result: Interval[] = [];
  let i = 0;

  // Adiciona todos os intervalos antes do newInterval
  while (i < intervals.length && intervals[i][1] < newInterval[0]) {
    result.push(intervals[i++]);
  }

  // Mescla intervalos sobrepostos com newInterval
  while (i < intervals.length && intervals[i][0] <= newInterval[1]) {
    newInterval[0] = Math.min(newInterval[0], intervals[i][0]);
    newInterval[1] = Math.max(newInterval[1], intervals[i][1]);
    i++;
  }
  result.push(newInterval);

  // Adiciona intervalos restantes
  while (i < intervals.length) result.push(intervals[i++]);

  return result;
}

// Mínimo de salas de reunião (sobreposição mínima em qualquer ponto) — O(n log n)
function minMeetingRooms(intervals: Interval[]): number {
  const starts = intervals.map(i => i[0]).sort((a, b) => a - b);
  const ends = intervals.map(i => i[1]).sort((a, b) => a - b);

  let rooms = 0;
  let endPtr = 0;

  for (let i = 0; i < starts.length; i++) {
    if (starts[i] < ends[endPtr]) {
      rooms++; // precisa de nova sala
    } else {
      endPtr++; // uma reunião terminou — reutiliza a sala
    }
  }

  return rooms;
}
```

**Exemplos de Problemas:**
1. Mesclar intervalos sobrepostos
2. Inserir intervalo em lista ordenada e mesclar
3. Número mínimo de salas de reunião necessárias

---

### Padrão 6: Ordenação Cíclica

**O que é:** Quando um array contém números no intervalo `[1, n]` (ou `[0, n-1]`), você pode colocar cada número no seu índice correto em O(n) de tempo usando trocas, sem espaço extra.

**Como reconhecer:**
- Array contém números no intervalo `[1, n]` ou `[0, n-1]`
- "Encontrar número faltando", "Encontrar duplicata", "Encontrar todos os números faltando"
- Você precisa de O(n) de tempo com O(1) de espaço

**Quando usar:** Somente quando os números cabem em um intervalo de índice específico, permitindo que cada elemento seja sua própria "chave".

**Template:**

```typescript
// Coloca cada número em seu índice correto — O(n) de tempo, O(1) de espaço
function cyclicSort(nums: number[]): void {
  let i = 0;

  while (i < nums.length) {
    const correctIdx = nums[i] - 1; // número 1 pertence ao índice 0, etc.

    if (nums[i] !== nums[correctIdx]) {
      // Troca para posição correta
      [nums[i], nums[correctIdx]] = [nums[correctIdx], nums[i]];
    } else {
      i++; // já está na posição correta
    }
  }
}

// Encontrar número faltando em [1..n] — O(n), O(1) de espaço
function findMissingNumber(nums: number[]): number {
  let i = 0;

  while (i < nums.length) {
    const j = nums[i] - 1;
    if (nums[i] > 0 && nums[i] <= nums.length && nums[i] !== nums[j]) {
      [nums[i], nums[j]] = [nums[j], nums[i]];
    } else {
      i++;
    }
  }

  // Encontra a primeira posição onde o número não corresponde
  for (let i = 0; i < nums.length; i++) {
    if (nums[i] !== i + 1) return i + 1;
  }

  return nums.length + 1;
}

// Encontrar todas as duplicatas em array de tamanho n com valores em [1,n] — O(n), O(1) de espaço
function findAllDuplicates(nums: number[]): number[] {
  let i = 0;

  while (i < nums.length) {
    const j = nums[i] - 1;
    if (nums[i] !== nums[j]) {
      [nums[i], nums[j]] = [nums[j], nums[i]];
    } else {
      i++;
    }
  }

  const duplicates: number[] = [];
  for (let i = 0; i < nums.length; i++) {
    if (nums[i] !== i + 1) duplicates.push(nums[i]);
  }

  return duplicates;
}
```

**Exemplos de Problemas:**
1. Encontrar o número faltando em `[1..n]`
2. Encontrar todos os números duplicados em array de tamanho n com valores em `[1, n]`
3. Encontrar o menor inteiro positivo faltando

---

### Padrão 7: Inversão In-Place de Lista Encadeada

**O que é:** Inverte uma lista encadeada ou uma porção dela in-place usando três ponteiros: `prev`, `curr`, `next`. Sem estruturas de dados extras.

**Como reconhecer:**
- "Inverter uma lista encadeada"
- "Inverter a cada k nós"
- "Inverter uma porção (sublista)"

**Quando usar:** Qualquer inversão de lista encadeada com requisito de O(1) de espaço.

**Template:**

```typescript
interface ListNode {
  val: number;
  next: ListNode | null;
}

// Inverter toda a lista encadeada — O(n), O(1) de espaço
function reverseList(head: ListNode | null): ListNode | null {
  let prev: ListNode | null = null;
  let curr = head;

  while (curr !== null) {
    const next = curr.next; // salva próximo
    curr.next = prev;       // inverte ponteiro
    prev = curr;            // avança prev
    curr = next;            // avança curr
  }

  return prev; // nova cabeça
}

// Inverter sublista da posição left até right (indexação em 1) — O(n), O(1) de espaço
function reverseBetween(
  head: ListNode | null,
  left: number,
  right: number
): ListNode | null {
  if (!head || left === right) return head;

  const dummy: ListNode = { val: 0, next: head };
  let beforeLeft: ListNode = dummy;

  // Anda até o nó antes da posição `left`
  for (let i = 1; i < left; i++) beforeLeft = beforeLeft.next!;

  let prev: ListNode | null = null;
  let curr: ListNode | null = beforeLeft.next;

  // Inverte `right - left + 1` nós
  for (let i = 0; i <= right - left; i++) {
    const next = curr!.next;
    curr!.next = prev;
    prev = curr;
    curr = next;
  }

  // Reconecta
  beforeLeft.next!.next = curr;
  beforeLeft.next = prev;

  return dummy.next;
}

// Inverter a cada k nós — O(n), O(1) de espaço
function reverseKGroup(head: ListNode | null, k: number): ListNode | null {
  // Verifica se restam k nós
  let check: ListNode | null = head;
  let count = 0;
  while (check && count < k) {
    check = check.next;
    count++;
  }
  if (count < k) return head; // menos de k nós restam — não inverte

  // Inverte k nós
  let prev: ListNode | null = null;
  let curr: ListNode | null = head;
  for (let i = 0; i < k; i++) {
    const next = curr!.next;
    curr!.next = prev;
    prev = curr;
    curr = next;
  }

  // head agora é a cauda do grupo invertido — conecta ao próximo grupo
  head!.next = reverseKGroup(curr, k);

  return prev;
}
```

**Exemplos de Problemas:**
1. Inverter toda a lista encadeada
2. Inverter uma sublista entre as posições left e right
3. Inverter a cada k nós em uma lista encadeada

---

### Padrão 8: Template BFS

**O que é:** Busca em Largura (BFS) explora nós nível por nível usando uma fila. Garante o caminho mais curto em grafos não ponderados e produz um percurso em ordem de nível em árvores.

**Como reconhecer:**
- "Caminho mais curto em grafo/grade não ponderado"
- "Percurso em ordem de nível"
- "Passos/saltos mínimos para chegar a..."
- "Encontrar todos os nós à distância k"

**Quando usar:** Problemas de caminho mais curto onde todas as arestas têm peso igual. Processamento nível a nível.

**Template:**

```typescript
// BFS genérico em um grafo — O(V + E)
function bfs(graph: Map<number, number[]>, start: number): number[] {
  const visited = new Set<number>();
  const queue: number[] = [start];
  visited.add(start);
  const order: number[] = [];

  while (queue.length > 0) {
    const node = queue.shift()!; // remove da frente
    order.push(node);

    for (const neighbor of graph.get(node) ?? []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push(neighbor);
      }
    }
  }

  return order;
}

// Caminho mais curto BFS em grafo não ponderado — O(V + E)
function shortestPath(
  graph: Map<number, number[]>,
  start: number,
  end: number
): number {
  if (start === end) return 0;

  const visited = new Set<number>([start]);
  const queue: Array<[number, number]> = [[start, 0]]; // [nó, distância]

  while (queue.length > 0) {
    const [node, dist] = queue.shift()!;

    for (const neighbor of graph.get(node) ?? []) {
      if (neighbor === end) return dist + 1;
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push([neighbor, dist + 1]);
      }
    }
  }

  return -1; // inalcançável
}

// Percurso em ordem de nível de árvore binária
interface TreeNode {
  val: number;
  left: TreeNode | null;
  right: TreeNode | null;
}

function levelOrder(root: TreeNode | null): number[][] {
  if (!root) return [];

  const result: number[][] = [];
  const queue: TreeNode[] = [root];

  while (queue.length > 0) {
    const levelSize = queue.length; // snapshot: todos os nós no nível atual
    const level: number[] = [];

    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift()!;
      level.push(node.val);

      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }

    result.push(level);
  }

  return result;
}

// BFS em grade 2D (número de ilhas) — O(m × n)
function numIslands(grid: string[][]): number {
  const rows = grid.length;
  const cols = grid[0].length;
  let count = 0;

  function bfsIsland(r: number, c: number): void {
    const queue: [number, number][] = [[r, c]];
    grid[r][c] = "0"; // marca como visitado mutando a grade

    const dirs = [[0, 1], [0, -1], [1, 0], [-1, 0]];

    while (queue.length > 0) {
      const [row, col] = queue.shift()!;
      for (const [dr, dc] of dirs) {
        const nr = row + dr;
        const nc = col + dc;
        if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] === "1") {
          grid[nr][nc] = "0";
          queue.push([nr, nc]);
        }
      }
    }
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === "1") {
        count++;
        bfsIsland(r, c);
      }
    }
  }

  return count;
}
```

**Exemplos de Problemas:**
1. Percurso em ordem de nível de uma árvore binária
2. Caminho mais curto em um grafo não ponderado
3. Número de ilhas em uma grade

---

### Padrão 9: Template DFS

**O que é:** Busca em Profundidade (DFS) explora o máximo possível ao longo de cada ramo antes de retroceder. Usa uma pilha (explícita ou a call stack via recursão).

**Como reconhecer:**
- "Encontrar todos os caminhos da origem ao destino"
- "Explorar todos os componentes conectados"
- "Verificar se existe caminho"
- "Percorrer uma árvore em pré-ordem/em-ordem/pós-ordem"

**Quando usar:** Quando você precisa explorar todas as possibilidades, encontrar componentes conectados ou processar nós de árvore em uma ordem específica de percurso.

**Template:**

```typescript
// DFS em grafo — recursivo — O(V + E)
function dfsRecursive(
  graph: Map<number, number[]>,
  node: number,
  visited = new Set<number>()
): void {
  visited.add(node);
  // processa nó

  for (const neighbor of graph.get(node) ?? []) {
    if (!visited.has(neighbor)) {
      dfsRecursive(graph, neighbor, visited);
    }
  }
}

// DFS em grafo — iterativo — O(V + E)
function dfsIterative(graph: Map<number, number[]>, start: number): void {
  const visited = new Set<number>();
  const stack: number[] = [start];

  while (stack.length > 0) {
    const node = stack.pop()!;
    if (visited.has(node)) continue;
    visited.add(node);
    // processa nó

    for (const neighbor of graph.get(node) ?? []) {
      if (!visited.has(neighbor)) {
        stack.push(neighbor);
      }
    }
  }
}

// DFS encontra todos os caminhos da origem ao destino — O(V + E)
function allPaths(
  graph: number[][],
  source: number,
  target: number
): number[][] {
  const result: number[][] = [];

  function dfs(node: number, path: number[]): void {
    if (node === target) {
      result.push([...path]);
      return;
    }

    for (const neighbor of graph[node]) {
      path.push(neighbor);
      dfs(neighbor, path);
      path.pop(); // backtrack
    }
  }

  dfs(source, [source]);
  return result;
}

// DFS flood fill (pinta região conectada) — O(m × n)
function floodFill(
  image: number[][],
  sr: number,
  sc: number,
  color: number
): number[][] {
  const originalColor = image[sr][sc];
  if (originalColor === color) return image;

  const rows = image.length;
  const cols = image[0].length;

  function dfs(r: number, c: number): void {
    if (r < 0 || r >= rows || c < 0 || c >= cols) return;
    if (image[r][c] !== originalColor) return;

    image[r][c] = color;
    dfs(r + 1, c);
    dfs(r - 1, c);
    dfs(r, c + 1);
    dfs(r, c - 1);
  }

  dfs(sr, sc);
  return image;
}
```

**Exemplos de Problemas:**
1. Encontrar todos os caminhos da origem ao destino em um DAG
2. Flood fill em uma região conectada de uma grade
3. Contar componentes conectados em um grafo não dirigido

---

### Padrão 10: Subsets / Combinações / Permutações (Backtracking)

**O que é:** Constrói uma solução incrementalmente — adiciona um candidato, recursa, depois desfaz (backtrack) para tentar o próximo candidato. Explora toda a árvore de busca de forma sistemática.

**Como reconhecer:**
- "Todos os subsets / conjunto potência"
- "Todas as combinações de tamanho k"
- "Todas as permutações"
- "Existe alguma atribuição válida?" (N-rainhas, Sudoku)

**Quando usar:** Quando você precisa enumerar todas as possibilidades ou encontrar qualquer configuração válida. Aceite tempo exponencial — é inevitável ao enumerar saídas exponenciais.

**Template:**

```typescript
// Template de backtracking
function backtrack(
  result: number[][],
  current: number[],
  start: number,
  candidates: number[],
  remaining: number
): void {
  if (/* caso base: solução completa */) {
    result.push([...current]); // copia antes de modificar
    return;
  }

  for (let i = start; i < candidates.length; i++) {
    // Pula escolhas inválidas (poda)
    if (/* condição de poda */) continue;

    current.push(candidates[i]); // escolhe
    backtrack(result, current, i + 1, candidates, remaining - candidates[i]); // explora
    current.pop(); // desfaz (backtrack)
  }
}

// Conjunto potência (todos os subsets) — O(2^n × n)
function powerSet(nums: number[]): number[][] {
  const result: number[][] = [];

  function backtrack(start: number, current: number[]): void {
    result.push([...current]); // todo estado é um subset válido

    for (let i = start; i < nums.length; i++) {
      current.push(nums[i]);
      backtrack(i + 1, current);
      current.pop();
    }
  }

  backtrack(0, []);
  return result;
}

// Combinações de tamanho k — O(C(n,k) × k)
function combinations(n: number, k: number): number[][] {
  const result: number[][] = [];

  function backtrack(start: number, current: number[]): void {
    if (current.length === k) {
      result.push([...current]);
      return;
    }

    // Poda: não há números suficientes para completar k
    for (let i = start; i <= n - (k - current.length) + 1; i++) {
      current.push(i);
      backtrack(i + 1, current);
      current.pop();
    }
  }

  backtrack(1, []);
  return result;
}

// Todas as permutações — O(n! × n)
function permutations(nums: number[]): number[][] {
  const result: number[][] = [];

  function backtrack(current: number[], remaining: number[]): void {
    if (remaining.length === 0) {
      result.push([...current]);
      return;
    }

    for (let i = 0; i < remaining.length; i++) {
      current.push(remaining[i]);
      backtrack(current, [...remaining.slice(0, i), ...remaining.slice(i + 1)]);
      current.pop();
    }
  }

  backtrack([], nums);
  return result;
}

// Soma de combinação (reutiliza elementos, soma ao alvo) — O(n^(alvo/min))
function combinationSum(candidates: number[], target: number): number[][] {
  const result: number[][] = [];
  candidates.sort((a, b) => a - b);

  function backtrack(start: number, current: number[], remaining: number): void {
    if (remaining === 0) {
      result.push([...current]);
      return;
    }

    for (let i = start; i < candidates.length; i++) {
      if (candidates[i] > remaining) break; // poda: ordenado, sem ponto em continuar

      current.push(candidates[i]);
      backtrack(i, current, remaining - candidates[i]); // i, não i+1 (pode reutilizar)
      current.pop();
    }
  }

  backtrack(0, [], target);
  return result;
}
```

**Exemplos de Problemas:**
1. Gerar todos os subsets (conjunto potência)
2. Todas as combinações de tamanho k de números 1 a n
3. Soma de combinação — encontrar todas as combinações que somam a um alvo

---

## 4. Tabela de Reconhecimento de Padrões

Use esta tabela quando você ler um problema e precisar identificar qual padrão tentar primeiro.

| Características do Problema | Padrão a Tentar |
|---|---|
| Janela de tamanho fixo sobre array/string | Janela Deslizante (Fixa) |
| Maior/menor subarray que satisfaz uma condição | Janela Deslizante (Variável) |
| Array ordenado, encontrar par/tripla que soma ao alvo | Dois Ponteiros |
| "Container com mais água", "capturar água da chuva" | Dois Ponteiros |
| Detecção de ciclo em lista encadeada | Ponteiros Rápido e Lento |
| Encontrar o meio de uma lista encadeada | Ponteiros Rápido e Lento |
| Intervalos sobrepostos, salas de reunião | Mesclar Intervalos |
| Array com valores no intervalo `[1, n]`, encontrar faltando/duplicado | Ordenação Cíclica |
| Inverter lista encadeada ou porção dela | Inversão In-Place |
| Caminho mais curto em grafo ou grade não ponderado | BFS |
| Percurso em ordem de nível de árvore | BFS |
| Encontrar todos os caminhos, explorar todos os componentes | DFS |
| Região conectada em grade (flood fill, ilhas) | DFS |
| Todos os subsets, combinações, permutações | Backtracking |
| "Existe um arranjo válido?" (N-rainhas, Sudoku) | Backtracking |
| Subestrutura ótima + subproblemas sobrepostos | Programação Dinâmica |
| Escolha localmente ótima leva ao ótimo global | Greedy |
| Array ordenado, "encontrar posição de X" | Busca Binária |
| "Mínimo/máximo que satisfaz condição" sobre função monótona | Busca Binária na Resposta |

> **Empilhamento de padrões:** Muitos problemas difíceis combinam dois padrões. "Máximo da janela deslizante" usa janela deslizante + deque monótono. "Capturar água da chuva" pode ser resolvido com dois ponteiros ou com uma pilha. Sempre considere se o problema tem dois subproblemas distintos que se encaixam em padrões diferentes.

> ⚠️ **Cuidado com rótulos errados.** Um problema que pede "verificar se algum par soma ao alvo" em um array **não ordenado** é um problema de hashmap, não de dois ponteiros. Dois ponteiros requer ordenação. Sempre verifique as restrições de entrada antes de identificar o padrão.

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Esquecer de copiar o array atual antes de fazer push para os resultados.** No backtracking, `result.push(current)` faz push de uma referência. Sempre use `result.push([...current])`.

> ⚠️ **Usar `queue.shift()` para BFS em código com requisitos de performance.** `Array.shift()` é O(n). Para grafos grandes, use uma deque adequada ou implemente uma fila circular.

> ⚠️ **Mutar a grade de entrada no BFS/DFS.** Marcar células visitadas sobrescrevendo `grid[r][c]` é eficiente, mas torna a função impura e não reutilizável. Para código de produção, use um Set `visited` separado.

> ⚠️ **Off-by-one na janela deslizante.** Janela de tamanho k: índices `[i-k, i-1]`. Ao deslizar, remove `arr[i-k]` e não `arr[i-k-1]`. Sempre verifique com um exemplo pequeno (n=3, k=2).

> ⚠️ **Esquecer de pular duplicatas em problemas de combinação/permutação.** Quando a entrada tem duplicatas e você precisa de resultados únicos, ordene primeiro, depois pule com `if (i > start && nums[i] === nums[i-1]) continue`.

---

## 6. Quando Usar / Não Usar

**Esses padrões resolvem quase todos os problemas médios do LeetCode — mas não são apropriados para todo código de produção:**

- **Janela deslizante** é apropriada em produção para agregações de streaming, limitadores de taxa e análise em tempo real sobre janelas de tempo.
- **Dois ponteiros** se aplica a deduplicação de dados ordenados e mesclagem de intervalos em sistemas reais.
- **BFS/DFS** aplicam-se diretamente a travessia de grafos em resolução de dependências, roteamento de rede e renderização de árvore de UI.
- **Backtracking** raramente é usado diretamente em produção — entradas O(n!) nunca chegam a servidores. Mas o template se aplica a geradores de configuração, geradores de matriz de testes e sistemas de satisfação de restrições.

---

## 7. Cenário do Mundo Real

Uma empresa de entregas precisa atribuir motoristas a turnos de horário, garantindo que dois motoristas com a mesma rota não se sobreponham. O problema se reduz a: dado uma lista de intervalos de tempo, encontrar o número mínimo de grupos (salas) tal que nenhum dois intervalos no mesmo grupo se sobreponham.

Isso é exatamente o problema de "mínimo de salas de reunião" do **Mesclar Intervalos**:

```typescript
function minDriverGroups(shifts: Interval[]): number {
  const starts = shifts.map(s => s[0]).sort((a, b) => a - b);
  const ends = shifts.map(s => s[1]).sort((a, b) => a - b);

  let groups = 0;
  let endPtr = 0;

  for (let i = 0; i < starts.length; i++) {
    if (starts[i] < ends[endPtr]) {
      groups++; // novo turno começa antes de algum terminar — precisa de novo grupo
    } else {
      endPtr++; // um turno terminou — reutiliza o slot do grupo
    }
  }

  return groups;
}
```

Reconhecer o padrão reduziu o tempo de implementação de horas (simulação por força bruta) para minutos.

---

## 8. Perguntas de Entrevista

**P1: Como você reconhece um problema de janela deslizante?**
R: O problema pede um subarray ou substring ótimo (máx/mín/maior/menor) contíguo. Se o tamanho da janela é fixo, use a variante fixa. Se é variável com uma restrição (no máximo k distintos, soma igual ao alvo), use a variante variável com condição de encolhimento.

**P2: Quando dois ponteiros não funciona?**
R: Quando o array não está ordenado e você quer evitar o custo de ordenação O(n log n). Nesse caso, use um hashmap para encontrar pares em O(n). Dois ponteiros requer comportamento monótono: mover um ponteiro deve aumentar ou diminuir monotonicamente a quantidade relevante.

**P3: Qual é a diferença entre BFS e DFS para caminho mais curto?**
R: BFS garante o caminho mais curto em grafos não ponderados porque explora todos os nós à distância d antes de qualquer nó à distância d+1. DFS encontra um caminho, mas não necessariamente o mais curto — ele vai fundo primeiro e pode encontrar um caminho mais longo.

**P4: Por que backtracking tem complexidade exponencial?**
R: A árvore de busca tem fator de ramificação b e profundidade d, dando b^d nós. Para subsets: fator de ramificação 2, profundidade n → O(2^n). Para permutações: fator de ramificação n, depois n-1, etc. → O(n!). Poda reduz a constante, mas não a classe assintótica no pior caso.

**P5: Como ponteiros rápido/lento provam detecção de ciclo em O(n)?**
R: Se existe um ciclo de comprimento c, o ponteiro rápido alcança o lento dentro do ciclo. Uma vez que ambos estão no ciclo, a distância entre eles diminui 1 por passo (o rápido ganha 1 passo por iteração). Eles se encontram dentro de c iterações, e as iterações totais antes de se encontrarem são O(n).

**P6: Qual é o invariante no algoritmo de mesclar intervalos?**
R: O array de resultado é sempre uma lista de intervalos não sobrepostos e ordenados. Ao processar um novo intervalo: se ele se sobrepõe com o último intervalo mesclado (novo.início <= último.fim), estende; caso contrário, acrescenta. A ordenação por tempo de início garante que só precisamos olhar para o último intervalo.

**P7: Você pode fazer janela deslizante em matrizes 2D?**
R: Sim, mas torna-se uma janela deslizante 2D. Para uma soma de submatriz k×k fixa, pré-compute um array de soma de prefixo 2D: `prefix[i][j]` = soma de todos os elementos no retângulo superior esquerdo. Qualquer soma de submatriz k×k é então O(1): `prefix[r+k][c+k] - prefix[r][c+k] - prefix[r+k][c] + prefix[r][c]`.

**P8: O que distingue a ordenação cíclica da ordenação regular?**
R: A ordenação cíclica explora a restrição de que os valores estão em `[1, n]` — cada valor codifica sua própria posição correta. Isso permite ordenação O(n) com O(1) de espaço. A ordenação por comparação regular requer O(n log n) porque não faz suposições sobre intervalos de valores.

---

## 9. Exercícios

**Exercício 1 (Janela Deslizante Fixa):** Dado um array de inteiros e um número k, encontre a soma máxima de qualquer subarray contíguo de tamanho k. Depois estenda: encontre todos os subarrays de comprimento k cuja soma seja igual a um alvo.

*Dica: Mantenha uma soma corrente. Deslize adicionando `arr[direita]` e subtraindo `arr[direita - k]`.*

**Exercício 2 (Janela Deslizante Variável):** Dada uma string, encontre o comprimento da maior substring que contém no máximo 2 caracteres distintos.

*Dica: Use um mapa de frequência. Quando `freq.size > 2`, encolha pela esquerda até que seja 2 novamente.*

**Exercício 3 (Dois Ponteiros):** Dado um array ordenado, remova duplicatas in-place e retorne o novo comprimento. Use O(1) de espaço extra.

*Dica: O ponteiro `esquerda` rastreia a posição do último elemento único. `direita` escaneia à frente.*

**Exercício 4 (Mesclar Intervalos):** Dada uma lista de intervalos representando turnos de funcionários, encontre todos os intervalos de "tempo livre" onde nenhum funcionário está trabalhando.

*Dica: Achate todos os intervalos em uma lista, ordene, mescle intervalos sobrepostos, depois encontre lacunas entre intervalos mesclados.*

**Exercício 5 (Backtracking):** Gere todas as combinações válidas de parênteses para n pares. Para n=3: `["((()))","(()())","(())()","()(())","()()()"]`.

*Dica: A cada passo você pode adicionar `(` se a contagem de abertos < n, ou `)` se a contagem de fechados < abertos.*

**Exercício 6 (BFS):** Dada uma palavra e uma lista de palavras, encontre o comprimento da sequência de transformação mais curta onde cada passo muda exatamente uma letra e as palavras intermediárias devem estar na lista de palavras (escada de palavras).

*Dica: Modele como BFS — cada palavra é um nó, arestas conectam palavras que diferem por um caractere.*

---

## 10. Leituras Complementares

- [NeetCode Roadmap](https://neetcode.io/roadmap) — problemas organizados por padrão, com explicações em vídeo
- [LeetCode Patterns](https://seanprashad.com/leetcode-patterns/) — listas de problemas curadas por padrão
- [Grokking the Coding Interview](https://www.designgurus.io/course/grokking-the-coding-interview) — o curso original baseado em padrões
- [Blind 75](https://leetcode.com/discuss/general-discussion/460599/blind-75-leetcode-questions) — os 75 problemas que cobrem todos os padrões críticos
- [Template de backtracking — LeetCode discuss](https://leetcode.com/problems/combination-sum/solutions/16502/a-general-approach-to-backtracking-questions-in-java-subsets-permutations-combination-sum-palindrome-partitioning/)
- *Algorithm Design Manual* por Steven Skiena — Capítulo 7 (Busca Combinatória e Métodos Heurísticos)
