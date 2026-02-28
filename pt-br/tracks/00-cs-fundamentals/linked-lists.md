# Listas Encadeadas

## 1. O Que É e Por Que Importa

Uma lista encadeada é uma estrutura de dados linear onde cada elemento (nó) contém um valor e um ponteiro para o próximo nó. Ao contrário de arrays, os nós de uma lista encadeada são espalhados pela memória — não há contiguidade garantida. Essa única diferença muda todo o perfil de performance.

A troca fundamental: listas encadeadas oferecem inserção e remoção em O(1) em qualquer posição para a qual você já tenha um ponteiro, enquanto arrays exigem O(n) de deslocamento. Mas listas encadeadas precisam de O(n) de tempo para encontrar uma posição (sem aritmética de índice), enquanto arrays oferecem acesso em O(1).

Listas encadeadas sustentam muitas estruturas de dados de nível superior: pilhas, filas, encadeamento de hash tables, listas de adjacência em grafos e o cache LRU.

---

## 2. Conceitos Fundamentais

### Tipos de Listas Encadeadas

**Lista Simplesmente Encadeada:** Cada nó tem `value` e `next`. A travessia é unidirecional. Simples e eficiente em memória.

**Lista Duplamente Encadeada:** Cada nó tem `value`, `next` e `prev`. Permite remoção em O(1) se você tiver o nó (sem precisar encontrar o nó anterior). Usada em caches LRU e histórico de navegador.

**Lista Encadeada Circular:** O `next` do último nó aponta de volta para a cabeça (ou algum outro nó). Usada em escalonamento round-robin e buffers circulares.

### Comparação de Complexidade

| Operação | Lista Encadeada | Array |
|---|---|---|
| Acesso por índice | O(n) | O(1) |
| Busca por valor | O(n) | O(n) |
| Inserir na cabeça | O(1) | O(n) |
| Inserir na cauda | O(1) com ponteiro para tail | O(1) amortizado |
| Inserir no meio (com ponteiro) | O(1) | O(n) |
| Remover da cabeça | O(1) | O(n) |
| Remover (com ponteiro) | O(1) dupla / O(n) simples | O(n) |

### Padrões Chave de Ponteiros

- **Ponteiros rápido/lento (algoritmo de Floyd):** Dois ponteiros avançando em velocidades diferentes — detecta ciclos, encontra pontos médios.
- **Rastreamento do ponteiro anterior:** Ao remover um nó em uma lista simplesmente encadeada, você precisa do nó anterior.
- **Nó sentinela (dummy head):** Um nó antes da cabeça real simplifica casos extremos na cabeça.

---

## 3. Como Funciona

```
Lista Simplesmente Encadeada: 1 → 2 → 3 → 4 → null

head → [1|next] → [2|next] → [3|next] → [4|null]

Memória: nós não são contíguos — podem estar em qualquer lugar no heap.
Acesso arr[2]: deve percorrer desde a cabeça: 1 → 2 → 3 — O(n)
```

**Detecção de ciclo (tartaruga e lebre de Floyd):**
```
slow avança 1 passo por iteração
fast avança 2 passos por iteração

Se há um ciclo, fast eventualmente "alcança" slow — eles se encontram dentro do ciclo.
Se não há ciclo, fast chega ao null primeiro.
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Definições de Nó e Lista ---

class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null
  ) {}
}

class SinglyLinkedList<T> {
  private head: ListNode<T> | null = null;
  private tail: ListNode<T> | null = null;
  public size = 0;

  // Inserir na cabeça: O(1)
  prepend(value: T): void {
    const node = new ListNode(value, this.head);
    this.head = node;
    if (this.tail === null) this.tail = node;
    this.size++;
  }

  // Inserir na cauda: O(1) com ponteiro para tail
  append(value: T): void {
    const node = new ListNode(value);
    if (this.tail === null) {
      this.head = this.tail = node;
    } else {
      this.tail.next = node;
      this.tail = node;
    }
    this.size++;
  }

  // Remover por valor: O(n)
  delete(value: T): boolean {
    if (this.head === null) return false;

    // Caso especial: removendo a cabeça
    if (this.head.value === value) {
      this.head = this.head.next;
      if (this.head === null) this.tail = null;
      this.size--;
      return true;
    }

    let current = this.head;
    while (current.next !== null) {
      if (current.next.value === value) {
        if (current.next === this.tail) this.tail = current;
        current.next = current.next.next;
        this.size--;
        return true;
      }
      current = current.next;
    }
    return false;
  }

  // Busca: O(n)
  find(value: T): ListNode<T> | null {
    let current = this.head;
    while (current !== null) {
      if (current.value === value) return current;
      current = current.next;
    }
    return null;
  }

  toArray(): T[] {
    const result: T[] = [];
    let current = this.head;
    while (current !== null) {
      result.push(current.value);
      current = current.next;
    }
    return result;
  }
}

// --- Inverter lista encadeada in-place: O(n) de tempo, O(1) de espaço ---
function reverseList<T>(head: ListNode<T> | null): ListNode<T> | null {
  let prev: ListNode<T> | null = null;
  let current = head;

  while (current !== null) {
    const next = current.next; // salva next antes de sobrescrever
    current.next = prev;       // inverte o ponteiro
    prev = current;            // avança prev
    current = next;            // avança current
  }

  return prev; // prev é a nova cabeça
}

// --- Detectar ciclo (tartaruga e lebre de Floyd): O(n) de tempo, O(1) de espaço ---
function hasCycle<T>(head: ListNode<T> | null): boolean {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;         // avança 1 passo
    fast = fast.next.next;     // avança 2 passos

    if (slow === fast) return true; // se encontraram — há ciclo
  }
  return false; // fast chegou ao null — sem ciclo
}

// --- Encontrar início do ciclo (Floyd estendido): O(n) de tempo, O(1) de espaço ---
function detectCycleStart<T>(head: ListNode<T> | null): ListNode<T> | null {
  let slow = head;
  let fast = head;

  // Fase 1: detectar ponto de encontro dentro do ciclo
  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) break;
  }

  if (fast === null || fast.next === null) return null; // sem ciclo

  // Fase 2: move um ponteiro para a cabeça, avança ambos na mesma velocidade
  // Eles se encontrarão no início do ciclo — prova matemática via distâncias
  slow = head;
  while (slow !== fast) {
    slow = slow!.next;
    fast = fast!.next;
  }
  return slow;
}

// --- Encontrar meio da lista encadeada: O(n) de tempo, O(1) de espaço ---
// Ponteiro rápido/lento: quando fast chega ao fim, slow está no meio
function findMiddle<T>(head: ListNode<T> | null): ListNode<T> | null {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
  }
  return slow; // para comprimento par, retorna o segundo nó do meio
}

// --- Mesclar duas listas encadeadas ordenadas: O(n + m) de tempo, O(1) de espaço ---
function mergeSorted(
  l1: ListNode<number> | null,
  l2: ListNode<number> | null
): ListNode<number> | null {
  const dummy = new ListNode<number>(0); // sentinela para evitar caso especial da cabeça
  let current = dummy;

  while (l1 !== null && l2 !== null) {
    if (l1.value <= l2.value) {
      current.next = l1;
      l1 = l1.next;
    } else {
      current.next = l2;
      l2 = l2.next;
    }
    current = current.next;
  }

  current.next = l1 ?? l2; // anexa a lista restante
  return dummy.next;
}

// --- Encontrar n-ésimo nó a partir do fim: O(n) de tempo, O(1) de espaço ---
// Dois ponteiros: avança o primeiro n passos, depois move ambos até o primeiro chegar ao fim
function nthFromEnd<T>(head: ListNode<T> | null, n: number): ListNode<T> | null {
  let ahead = head;
  let behind = head;

  // Avança 'ahead' n passos
  for (let i = 0; i < n; i++) {
    if (ahead === null) return null; // n > comprimento da lista
    ahead = ahead.next;
  }

  // Move ambos até ahead chegar ao null
  while (ahead !== null) {
    ahead = ahead.next;
    behind = behind!.next;
  }

  return behind;
}

// --- Verificar se lista encadeada é palíndromo: O(n) de tempo, O(1) de espaço ---
function isPalindromeList(head: ListNode<number> | null): boolean {
  if (head === null || head.next === null) return true;

  // Passo 1: encontrar meio
  let slow = head;
  let fast = head;
  while (fast.next !== null && fast.next.next !== null) {
    slow = slow.next!;
    fast = fast.next.next;
  }

  // Passo 2: inverter segunda metade
  let secondHalf = reverseList(slow.next);

  // Passo 3: comparar primeira e segunda metades
  let left: ListNode<number> | null = head;
  let right = secondHalf;
  let isPalin = true;
  while (right !== null) {
    if (left!.value !== right.value) { isPalin = false; break; }
    left = left!.next;
    right = right.next;
  }

  // Passo 4: restaurar lista (opcional, para comportamento não destrutivo)
  slow.next = reverseList(secondHalf);

  return isPalin;
}

// --- Lista Duplamente Encadeada ---
class DListNode<T> {
  constructor(
    public value: T,
    public prev: DListNode<T> | null = null,
    public next: DListNode<T> | null = null
  ) {}
}

class DoublyLinkedList<T> {
  private head: DListNode<T> | null = null;
  private tail: DListNode<T> | null = null;

  append(value: T): DListNode<T> {
    const node = new DListNode(value);
    if (this.tail === null) {
      this.head = this.tail = node;
    } else {
      node.prev = this.tail;
      this.tail.next = node;
      this.tail = node;
    }
    return node;
  }

  // O(1) remoção dado o próprio nó
  deleteNode(node: DListNode<T>): void {
    if (node.prev) node.prev.next = node.next;
    else this.head = node.next; // nó era a cabeça

    if (node.next) node.next.prev = node.prev;
    else this.tail = node.prev; // nó era a cauda
  }
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Esquecer de tratar o nó cabeça de forma especial em listas simplesmente encadeadas.** Remoção e inserção na cabeça não podem usar o padrão "encontrar anterior". Use um nó sentinela para unificar todos os casos.

> ⚠️ **Não verificar `fast.next !== null` em loops de ponteiro rápido/lento.** Se o comprimento da lista é par, `fast.next.next` pode lançar NPE antes de `fast` em si ser null.

> ⚠️ **Perder a referência para o próximo nó ao inverter.** Sempre salve `current.next` em uma variável temporária antes de sobrescrever `current.next = prev`.

> ⚠️ **Esquecer de atualizar o ponteiro para tail.** Ao remover o último nó ou ao anexar, muitas implementações esquecem de atualizar `tail`, causando bugs em operações subsequentes na cauda.

> ⚠️ **Assumir remoção O(1) de lista simplesmente encadeada.** Você precisa do nó *anterior* para realizar a remoção. Se você tem apenas o nó atual, deve percorrer desde a cabeça — O(n). Listas duplamente encadeadas resolvem isso.

---

## 6. Quando Usar / Não Usar

**Use lista encadeada quando:**
- Precisa de inserções/remoções em O(1) em uma posição conhecida (ex.: evicção em cache LRU)
- Está implementando uma pilha ou fila e quer push/pop em O(1) em ambas as extremidades
- O tamanho da lista varia drasticamente (sem capacidade desperdiçada como em arrays dinâmicos)
- Está implementando encadeamento em hash table

**NÃO use lista encadeada quando:**
- Precisa de acesso indexado — travessia O(n) é muito mais lenta que acesso O(1) em array
- Performance de cache importa — memória não contígua prejudica o prefetch da CPU
- Memória é limitada — cada nó carrega overhead de ponteiro (8 bytes por ponteiro em 64-bit)
- Os dados têm tamanho fixo e são sequenciais — um array simples é quase sempre mais rápido na prática

---

## 7. Cenário do Mundo Real

Um cache LRU (Least Recently Used — Menos Recentemente Usado) requer acesso em O(1), inserção em O(1) e evicção em O(1) do elemento mais antigo. Isso é impossível apenas com array (O(n) para deletar), mas a combinação hashmap + lista duplamente encadeada alcança os três:

```typescript
class LRUCache {
  private capacity: number;
  private map: Map<number, DListNode<{ key: number; value: number }>>;
  private list: DoublyLinkedList<{ key: number; value: number }>;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.map = new Map();
    this.list = new DoublyLinkedList();
  }

  get(key: number): number {
    if (!this.map.has(key)) return -1;
    const node = this.map.get(key)!;
    // Move para a frente (mais recentemente usado)
    this.list.deleteNode(node);
    const newNode = this.list.append(node.value);
    this.map.set(key, newNode);
    return node.value.value;
  }

  put(key: number, value: number): void {
    if (this.map.has(key)) {
      this.list.deleteNode(this.map.get(key)!);
    } else if (this.map.size >= this.capacity) {
      // Evictar LRU (cauda da lista — ou cabeça, dependendo da orientação)
      // Detalhe de implementação: rastrear head/tail adequadamente
    }
    const node = this.list.append({ key, value });
    this.map.set(key, node);
  }
}
```

---

## 8. Perguntas de Entrevista

**Q1: Como você detecta um ciclo em uma lista encadeada?**
R: Algoritmo da tartaruga e lebre de Floyd. Dois ponteiros — slow avança 1 passo, fast avança 2 passos por iteração. Se há um ciclo, fast eventualmente alcança slow (eles se encontram dentro do ciclo). Se não há ciclo, fast chega ao null. O(n) de tempo, O(1) de espaço.

**Q2: Como você inverte uma lista encadeada in-place?**
R: Itera com três ponteiros: `prev = null`, `current = head` e `next`. A cada passo: salva `current.next`, define `current.next = prev`, avança `prev = current`, avança `current = next`. Retorna `prev` como a nova cabeça. O(n) de tempo, O(1) de espaço.

**Q3: Como você encontra o n-ésimo nó a partir do fim de uma lista encadeada?**
R: Técnica de dois ponteiros. Avança o primeiro ponteiro n passos à frente. Depois move ambos os ponteiros juntos até o primeiro chegar ao null. O segundo ponteiro está agora no n-ésimo nó a partir do fim. O(n) de tempo, O(1) de espaço, passagem única.

**Q4: Como você encontra o meio de uma lista encadeada?**
R: Ponteiro rápido/lento. Move slow 1 passo, fast 2 passos. Quando fast chega ao fim, slow está no meio. O(n) de tempo, O(1) de espaço.

**Q5: Qual é a diferença entre lista simplesmente e duplamente encadeada?**
R: A simples tem apenas ponteiro `next` — remoção O(n) em geral (deve encontrar o nó anterior). A dupla tem `prev` e `next` — remoção O(1) dado o próprio nó, mas cada nó usa mais memória (um ponteiro extra).

**Q6: Quando você preferiria uma lista encadeada a um array?**
R: Quando inserções/remoções em posições arbitrárias são frequentes e você já tem um ponteiro para a posição. O cache LRU é o exemplo canônico. Na prática, listas encadeadas raramente são mais rápidas que arrays para cargas de trabalho sequenciais devido ao comportamento de cache.

---

## 9. Exercícios

**Exercício 1:** Implemente uma lista duplamente encadeada com operações `append`, `prepend`, `deleteNode` e `toArray`. Teste todos os casos extremos: lista vazia, elemento único, remover cabeça, remover cauda.

**Exercício 2:** Dada uma lista encadeada que pode conter um ciclo, detecte e remova o ciclo (defina `null` no nó que aponta de volta).
*Dica: use o algoritmo de Floyd para encontrar o início do ciclo, depois encontre o nó cujo next é o início do ciclo.*

**Exercício 3:** Verifique se uma lista simplesmente encadeada é palíndromo em O(n) de tempo e O(1) de espaço.
*Dica: encontre o meio, inverta a segunda metade, compare, depois restaure.*

**Exercício 4:** Mescle k listas encadeadas ordenadas em uma única lista ordenada.
*Dica: use um min-heap (fila de prioridade) de tamanho k — sempre extraia o mínimo, depois insira o próximo nó dessa lista. O(n log k) onde n = total de nós.*

**Exercício 5:** Remova todas as ocorrências de um dado valor de uma lista encadeada. Retorne a nova cabeça.
*Dica: use um nó dummy antes da cabeça para tratar remoções de nós na cabeça de forma limpa.*

---

## 10. Leituras Complementares

- CLRS Capítulo 10 — Estruturas de Dados Elementares (Listas Encadeadas)
- [NeetCode — playlist de Linked List](https://neetcode.io/roadmap)
- [Detecção de Ciclo de Floyd — prova](https://en.wikipedia.org/wiki/Cycle_detection#Floyd's_tortoise_and_hare)
- Problemas LeetCode: #206 Reverse Linked List, #141 Linked List Cycle, #21 Merge Two Sorted Lists, #146 LRU Cache
