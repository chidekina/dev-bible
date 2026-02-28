# Pilhas e Filas

## 1. O Que É e Por Que Importa

Pilhas e filas são estruturas de dados lineares restritas que impõem padrões específicos de acesso. Essa restrição é o poder delas: ao limitar como você interage com a estrutura, elas codificam a semântica do domínio diretamente na estrutura de dados.

Uma **pilha** é LIFO — Last In, First Out (último a entrar, primeiro a sair). Pense na pilha de chamadas, histórico de desfazer ou uma pilha de pratos. A última coisa que você colocou é a primeira que você tira.

Uma **fila** é FIFO — First In, First Out (primeiro a entrar, primeiro a sair). Pense em uma fila de impressão, fila de requisições HTTP ou travessia BFS. A primeira coisa que você colocou é a primeira que você obtém.

Essas estruturas sustentam muitos algoritmos (BFS, DFS, análise de expressões, backtracking) e designs de sistemas (escalonadores de tarefas, rate limiters, event loops).

---

## 2. Conceitos Fundamentais

### Pilha (LIFO)

Operações:
- **push(item)** — adiciona ao topo: O(1)
- **pop()** — remove do topo: O(1)
- **peek()** — visualiza o topo sem remover: O(1)
- **isEmpty()** — verifica se está vazia: O(1)

### Fila (FIFO)

Operações:
- **enqueue(item)** — adiciona ao final: O(1)
- **dequeue()** — remove da frente: O(1) com lista encadeada, O(n) com array ingênuo
- **peek()** — visualiza a frente sem remover: O(1)
- **isEmpty()** — verifica se está vazia: O(1)

### Deque (Fila de Duas Extremidades)

Suporta push/pop em ambas as extremidades (frente e fundo) em O(1). Usada para:
- Máximo em janela deslizante (deque monotônico)
- Implementar pilhas e filas com uma única estrutura
- Verificação de palíndromo

### Pilha Monotônica

Uma pilha que mantém ordem monotonicamente crescente ou decrescente de elementos. Ao inserir um novo elemento, você remove todos os elementos que violam a propriedade monotônica. Usada para problemas de "próximo elemento maior", "maior retângulo em histograma", "temperaturas diárias".

---

## 3. Como Funciona

```
Pilha (LIFO):
push(1) → [1]
push(2) → [1, 2]
push(3) → [1, 2, 3]
pop()   → retorna 3, pilha = [1, 2]
peek()  → retorna 2, pilha inalterada

Fila (FIFO):
enqueue(1) → [1]
enqueue(2) → [1, 2]
enqueue(3) → [1, 2, 3]
dequeue()  → retorna 1, fila = [2, 3]
peek()     → retorna 2, fila inalterada
```

**Por que fila baseada em array tem dequeue O(n):**
Quando você faz `shift()` em um array JavaScript, todos os elementos restantes se deslocam um índice para a esquerda — O(n). Solução: use uma lista encadeada, ou use um ponteiro de índice (técnica de buffer circular).

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Pilha: implementação baseada em array ---
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item); // Array.push é O(1) amortizado
  }

  pop(): T | undefined {
    return this.items.pop(); // Array.pop é O(1)
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  get size(): number {
    return this.items.length;
  }
}

// --- Fila: baseada em lista encadeada (O(1) enqueue e dequeue) ---
class QueueNode<T> {
  constructor(public value: T, public next: QueueNode<T> | null = null) {}
}

class Queue<T> {
  private head: QueueNode<T> | null = null;
  private tail: QueueNode<T> | null = null;
  private _size = 0;

  enqueue(item: T): void {
    const node = new QueueNode(item);
    if (this.tail === null) {
      this.head = this.tail = node;
    } else {
      this.tail.next = node;
      this.tail = node;
    }
    this._size++;
  }

  dequeue(): T | undefined {
    if (this.head === null) return undefined;
    const value = this.head.value;
    this.head = this.head.next;
    if (this.head === null) this.tail = null;
    this._size--;
    return value;
  }

  peek(): T | undefined {
    return this.head?.value;
  }

  isEmpty(): boolean {
    return this._size === 0;
  }

  get size(): number {
    return this._size;
  }
}

// --- Deque: implementação duplamente encadeada ---
class DequeNode<T> {
  constructor(
    public value: T,
    public prev: DequeNode<T> | null = null,
    public next: DequeNode<T> | null = null
  ) {}
}

class Deque<T> {
  private head: DequeNode<T> | null = null;
  private tail: DequeNode<T> | null = null;

  pushFront(value: T): void {
    const node = new DequeNode(value, null, this.head);
    if (this.head) this.head.prev = node;
    this.head = node;
    if (this.tail === null) this.tail = node;
  }

  pushBack(value: T): void {
    const node = new DequeNode(value, this.tail);
    if (this.tail) this.tail.next = node;
    this.tail = node;
    if (this.head === null) this.head = node;
  }

  popFront(): T | undefined {
    if (!this.head) return undefined;
    const value = this.head.value;
    this.head = this.head.next;
    if (this.head) this.head.prev = null;
    else this.tail = null;
    return value;
  }

  popBack(): T | undefined {
    if (!this.tail) return undefined;
    const value = this.tail.value;
    this.tail = this.tail.prev;
    if (this.tail) this.tail.next = null;
    else this.head = null;
    return value;
  }

  peekFront(): T | undefined { return this.head?.value; }
  peekBack(): T | undefined { return this.tail?.value; }
  isEmpty(): boolean { return this.head === null; }
}

// --- Verificador de colchetes balanceados: problema clássico de pilha ---
function isBalanced(s: string): boolean {
  const stack = new Stack<string>();
  const pairs: Record<string, string> = { ")": "(", "]": "[", "}": "{" };

  for (const char of s) {
    if ("([{".includes(char)) {
      stack.push(char);
    } else if (")]}".includes(char)) {
      if (stack.isEmpty() || stack.pop() !== pairs[char]) return false;
    }
  }
  return stack.isEmpty(); // todos os abertos devem ser fechados
}

// --- Avaliar Notação Polonesa Reversa: O(n) ---
// "2 1 + 3 *" → ((2 + 1) * 3) = 9
function evalRPN(tokens: string[]): number {
  const stack = new Stack<number>();
  const ops = new Set(["+", "-", "*", "/"]);

  for (const token of tokens) {
    if (ops.has(token)) {
      const b = stack.pop()!;
      const a = stack.pop()!;
      if (token === "+") stack.push(a + b);
      else if (token === "-") stack.push(a - b);
      else if (token === "*") stack.push(a * b);
      else stack.push(Math.trunc(a / b)); // trunca em direção ao zero
    } else {
      stack.push(parseInt(token, 10));
    }
  }
  return stack.pop()!;
}

// --- Min Stack: O(1) push, pop, peek, getMin ---
// Mantém uma "min stack" paralela que rastreia os mínimos
class MinStack {
  private stack: number[] = [];
  private minStack: number[] = [];

  push(val: number): void {
    this.stack.push(val);
    const currentMin = this.minStack.length === 0
      ? val
      : Math.min(val, this.minStack[this.minStack.length - 1]);
    this.minStack.push(currentMin);
  }

  pop(): void {
    this.stack.pop();
    this.minStack.pop();
  }

  top(): number {
    return this.stack[this.stack.length - 1];
  }

  getMin(): number {
    return this.minStack[this.minStack.length - 1];
  }
}

// --- Pilha Monotônica: Próximo Elemento Maior ---
// Para cada elemento, encontra o próximo à direita que é maior
// Retorna -1 se não existir tal elemento
// O(n) — cada elemento é inserido e removido no máximo uma vez
function nextGreaterElement(nums: number[]): number[] {
  const result = new Array(nums.length).fill(-1);
  const stack = new Stack<number>(); // armazena índices

  for (let i = 0; i < nums.length; i++) {
    // Remove todos elementos menores que o atual — o atual é o "próximo maior" deles
    while (!stack.isEmpty() && nums[stack.peek()!] < nums[i]) {
      const idx = stack.pop()!;
      result[idx] = nums[i];
    }
    stack.push(i);
  }

  return result;
}

// Exemplo: [2, 1, 2, 4, 3] → [4, 2, 4, -1, -1]

// --- Temperaturas Diárias: Pilha Monotônica ---
// Encontra quantos dias até uma temperatura mais quente
// O(n) — cada dia é inserido e removido no máximo uma vez
function dailyTemperatures(temps: number[]): number[] {
  const result = new Array(temps.length).fill(0);
  const stack = new Stack<number>(); // índices

  for (let i = 0; i < temps.length; i++) {
    while (!stack.isEmpty() && temps[stack.peek()!] < temps[i]) {
      const idx = stack.pop()!;
      result[idx] = i - idx; // dias de espera
    }
    stack.push(i);
  }
  return result;
}

// --- Fila usando duas pilhas ---
// Push é O(1), dequeue é O(1) amortizado
// O truque: derrama stack1 em stack2 somente quando stack2 está vazia
class QueueFromStacks<T> {
  private inbox = new Stack<T>();  // insere aqui
  private outbox = new Stack<T>(); // remove daqui

  enqueue(item: T): void {
    this.inbox.push(item);
  }

  dequeue(): T | undefined {
    if (this.outbox.isEmpty()) {
      // Transfere todos os elementos — inverte a ordem = FIFO
      while (!this.inbox.isEmpty()) {
        this.outbox.push(this.inbox.pop()!);
      }
    }
    return this.outbox.pop();
  }

  peek(): T | undefined {
    if (this.outbox.isEmpty()) {
      while (!this.inbox.isEmpty()) {
        this.outbox.push(this.inbox.pop()!);
      }
    }
    return this.outbox.peek();
  }
}

// --- Máximo em Janela Deslizante usando Deque: O(n) ---
// Para cada janela de tamanho k, encontra o máximo
function slidingWindowMax(nums: number[], k: number): number[] {
  const result: number[] = [];
  const deque = new Deque<number>(); // armazena índices, frente = máximo

  for (let i = 0; i < nums.length; i++) {
    // Remove índices fora da janela atual
    while (!deque.isEmpty() && deque.peekFront()! <= i - k) {
      deque.popFront();
    }
    // Remove índices de elementos menores (nunca serão o máximo)
    while (!deque.isEmpty() && nums[deque.peekBack()!] <= nums[i]) {
      deque.popBack();
    }
    deque.pushBack(i);
    // Janela está completamente formada
    if (i >= k - 1) result.push(nums[deque.peekFront()!]);
  }
  return result;
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Usar `Array.shift()` para dequeue de fila.** `shift()` é O(n) porque desloca todos os elementos. Use uma fila baseada em lista encadeada ou a técnica de ponteiro/buffer circular.

> ⚠️ **Não tratar o caso de pilha vazia.** Chamar `pop()` ou `peek()` em uma pilha vazia retorna `undefined` (ou lança exceção). Sempre verifique `isEmpty()` antes de acessar.

> ⚠️ **Direção da pilha monotônica.** Decida antecipadamente: você quer o próximo elemento **maior** ou **menor**? Pilha monotônica decrescente → próximo maior. Pilha monotônica crescente → próximo menor. Errar isso produz resultados incorretos.

> ⚠️ **Esquecer de drenar os elementos restantes na pilha monotônica.** Após processar todos os elementos, itens ainda na pilha não têm "próximo maior" — preencha-os com -1 ou 0 dependendo do problema.

> ⚠️ **Off-by-one em fila circular.** Ao implementar um buffer circular, a condição "cheio" é `(rear + 1) % capacity === front`, não `rear === front - 1`. Errar isso faz um slot parecer vazio ou ser sobrescrito.

---

## 6. Quando Usar / Não Usar

**Use uma Pilha quando:**
- Rastrear chamadas de função (análogo à call stack)
- Operações desfazíveis (editores, software gráfico)
- Analisar estruturas aninhadas (colchetes, XML, JSON)
- Travessia DFS (versão iterativa)
- Avaliar expressões (RPN, infix → postfix)

**Use uma Fila quando:**
- Travessia BFS
- Escalonamento de tarefas (primeiro submetido = primeiro processado)
- Rate limiters e buffers de requisição
- Event loops e filas de mensagens

**Use um Deque quando:**
- Máximo/mínimo em janela deslizante
- Verificação de palíndromo
- Implementar operações de pilha e fila simultaneamente

**Use uma Pilha Monotônica quando:**
- Problemas de "próximo elemento maior/menor"
- "Maior retângulo em histograma"
- "Acumulação de água da chuva" (trapping rain water)
- Qualquer problema onde você precisa do elemento mais próximo satisfazendo uma condição de ordenação

---

## 7. Cenário do Mundo Real

Um sistema de desfazer/refazer de editor de texto usa duas pilhas:

```typescript
class TextEditor {
  private undoStack: string[] = []; // pilha de estados
  private redoStack: string[] = [];
  private current = "";

  type(text: string): void {
    this.undoStack.push(this.current); // salva estado atual antes de mudar
    this.redoStack.length = 0;         // qualquer nova ação limpa o histórico de redo
    this.current += text;
  }

  undo(): void {
    if (this.undoStack.length === 0) return;
    this.redoStack.push(this.current);
    this.current = this.undoStack.pop()!;
  }

  redo(): void {
    if (this.redoStack.length === 0) return;
    this.undoStack.push(this.current);
    this.current = this.redoStack.pop()!;
  }

  getText(): string {
    return this.current;
  }
}

const editor = new TextEditor();
editor.type("Olá");
editor.type(" Mundo");
console.log(editor.getText()); // "Olá Mundo"
editor.undo();
console.log(editor.getText()); // "Olá"
editor.redo();
console.log(editor.getText()); // "Olá Mundo"
```

Um encontrador de caminho mais curto baseado em BFS para um grafo não ponderado (ex.: encontrar a rota mais curta entre duas páginas na Wikipedia) usa uma fila:

```typescript
function shortestPath(graph: Map<string, string[]>, start: string, end: string): string[] | null {
  const queue = new Queue<string[]>(); // fila de caminhos
  const visited = new Set<string>();

  queue.enqueue([start]);
  visited.add(start);

  while (!queue.isEmpty()) {
    const path = queue.dequeue()!;
    const node = path[path.length - 1];

    if (node === end) return path;

    for (const neighbor of graph.get(node) ?? []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.enqueue([...path, neighbor]);
      }
    }
  }
  return null;
}
```

---

## 8. Perguntas de Entrevista

**Q1: Qual é a diferença entre pilha e fila?**
R: Pilha é LIFO (último a entrar, primeiro a sair) — usada para DFS, desfazer/refazer, emulação de call stack. Fila é FIFO (primeiro a entrar, primeiro a sair) — usada para BFS, escalonamento de tarefas, buffers de mensagem.

**Q2: Implemente uma min-stack que retorne o valor mínimo em O(1).**
R: Mantenha uma min-stack paralela ao lado da pilha principal. Ao inserir x, também insira `min(x, minStack.top)` na min-stack. Ao remover, remova de ambas. `getMin()` espiona o topo da min-stack. Todas as operações em O(1).

**Q3: Implemente uma fila usando duas pilhas.**
R: Use `inbox` (insere novos itens) e `outbox` (remove itens). Dequeue da outbox; se outbox estiver vazia, derrame tudo de inbox para outbox primeiro (invertendo a ordem = FIFO). Dequeue amortizado em O(1) porque cada elemento se move no máximo duas vezes.

**Q4: O que é uma pilha monotônica e quando você a usa?**
R: Uma pilha mantida em ordem estritamente crescente ou decrescente. Ao inserir, remove todos os elementos que violam a ordem. Usada para "próximo elemento maior", "maior retângulo em histograma", "acumulação de água". Alcança O(n) onde loops aninhados ingênuos seriam O(n²).

**Q5: Avalie uma expressão em Notação Polonesa Reversa.**
R: Processa tokens da esquerda para a direita. Insere números. Em um operador, remove dois números, aplica o operador, insere o resultado. A resposta final é o único elemento restante.

**Q6: Como você implementaria a navegação para trás/frente de um navegador?**
R: Duas pilhas — `backStack` e `forwardStack`. Navegar para uma página: insere atual em backStack, limpa forwardStack. Voltar: insere atual em forwardStack, remove de backStack. Avançar: insere atual em backStack, remove de forwardStack.

---

## 9. Exercícios

**Exercício 1:** Projete um sistema de histórico de navegador que suporte:
- `visit(url)` — navega para uma URL
- `back(steps)` — volta até `steps` páginas
- `forward(steps)` — avança até `steps` páginas

*Dica: duas pilhas — histórico para trás e histórico para frente.*

**Exercício 2:** Implemente uma fila circular (ring buffer de capacidade fixa) com enqueue e dequeue em O(1).
*Dica: use um array com ponteiros `head` e `tail`, rastreie `size` para diferenciar cheio de vazio.*

**Exercício 3:** Encontre a maior área de retângulo em um histograma.
Entrada: `[2, 1, 5, 6, 2, 3]` → Saída: `10`
*Dica: pilha monotônica crescente — quando você encontra uma barra mais curta, remove e calcula a área para cada barra removida usando o índice atual como fronteira direita.*

**Exercício 4:** Implemente uma pilha que suporte:
- `push(val)`
- `pop()`
- `getMax()` em O(1)

*Dica: max-stack paralela, mesma abordagem que a min-stack.*

**Exercício 5:** Dada uma string de parênteses, encontre o comprimento da maior substring de parênteses válida (bem formada).
Entrada: `")()())"` → Saída: `4` ("()()")
*Dica: pilha que armazena índices.*

---

## 10. Leituras Complementares

- CLRS Capítulo 10.1 — Pilhas e Filas
- [Pilha Monotônica — padrões explicados](https://labuladong.online/algo/data-structure/monotonic-stack/)
- Problemas LeetCode: #20 Valid Parentheses, #155 Min Stack, #232 Queue using Stacks, #239 Sliding Window Maximum, #84 Largest Rectangle in Histogram
- [Buffer Circular — Wikipedia](https://en.wikipedia.org/wiki/Circular_buffer)
