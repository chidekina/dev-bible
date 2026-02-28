# Stacks & Queues

## 1. What & Why

Stacks and queues are restricted linear data structures that enforce specific access patterns. This restriction is their power: by limiting how you interact with the structure, they encode domain semantics directly into the data structure.

A **stack** is LIFO — Last In, First Out. Think of a call stack, undo history, or a stack of plates. The last thing you put in is the first thing you take out.

A **queue** is FIFO — First In, First Out. Think of a print queue, HTTP request queue, or BFS traversal. The first thing you put in is the first thing you get out.

These structures underlie many algorithms (BFS, DFS, expression parsing, backtracking) and system designs (task schedulers, rate limiters, event loops).

---

## 2. Core Concepts

### Stack (LIFO)

Operations:
- **push(item)** — add to top: O(1)
- **pop()** — remove from top: O(1)
- **peek()** — view top without removing: O(1)
- **isEmpty()** — check if empty: O(1)

### Queue (FIFO)

Operations:
- **enqueue(item)** — add to back: O(1)
- **dequeue()** — remove from front: O(1) with linked list, O(n) with naive array
- **peek()** — view front without removing: O(1)
- **isEmpty()** — check if empty: O(1)

### Deque (Double-Ended Queue)

Supports push/pop at both front and back in O(1). Used for:
- Sliding window maximum (monotonic deque)
- Implementing both stacks and queues with one structure
- Palindrome checking

### Monotonic Stack

A stack that maintains a monotonically increasing or decreasing order of elements. When pushing a new element, you pop all elements that violate the monotonic property. Used for "next greater element", "largest rectangle in histogram", "daily temperatures" problems.

---

## 3. How It Works

```
Stack (LIFO):
push(1) → [1]
push(2) → [1, 2]
push(3) → [1, 2, 3]
pop()   → returns 3, stack = [1, 2]
peek()  → returns 2, stack unchanged

Queue (FIFO):
enqueue(1) → [1]
enqueue(2) → [1, 2]
enqueue(3) → [1, 2, 3]
dequeue()  → returns 1, queue = [2, 3]
peek()     → returns 2, queue unchanged
```

**Why array-based queue has O(n) dequeue:**
When you `shift()` from a JavaScript array, every remaining element shifts left by one index — O(n). Fix: use a linked list, or use an index pointer (circular buffer technique).

---

## 4. Code Examples (TypeScript)

```typescript
// --- Stack: array-based implementation ---
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item); // Array.push is O(1) amortized
  }

  pop(): T | undefined {
    return this.items.pop(); // Array.pop is O(1)
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

// --- Queue: linked list-based (O(1) enqueue and dequeue) ---
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

// --- Deque: doubly-linked implementation ---
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

// --- Balanced brackets checker: classic stack problem ---
function isBalanced(s: string): boolean {
  const stack = new Stack<string>();
  const pairs: Record<string, string> = { ")": "(", "]": "[", "}": "{" };

  for (const char of s) {
    if ("([{".includes(char)) {
      stack.push(char);
    } else if (")]}" .includes(char)) {
      if (stack.isEmpty() || stack.pop() !== pairs[char]) return false;
    }
  }
  return stack.isEmpty(); // all opened must be closed
}

// --- Evaluate Reverse Polish Notation: O(n) ---
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
      else stack.push(Math.trunc(a / b)); // truncate toward zero
    } else {
      stack.push(parseInt(token, 10));
    }
  }
  return stack.pop()!;
}

// --- Min Stack: O(1) push, pop, peek, getMin ---
// Maintain a parallel "min stack" that tracks minimums
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

// --- Monotonic Stack: Next Greater Element ---
// For each element, find the next element to its right that is greater
// Returns -1 if no such element exists
// O(n) — each element is pushed and popped at most once
function nextGreaterElement(nums: number[]): number[] {
  const result = new Array(nums.length).fill(-1);
  const stack = new Stack<number>(); // stores indices

  for (let i = 0; i < nums.length; i++) {
    // Pop all elements smaller than current — current is their "next greater"
    while (!stack.isEmpty() && nums[stack.peek()!] < nums[i]) {
      const idx = stack.pop()!;
      result[idx] = nums[i];
    }
    stack.push(i);
  }

  return result;
}

// Example: [2, 1, 2, 4, 3] → [4, 2, 4, -1, -1]

// --- Daily Temperatures: Monotonic Stack ---
// Find how many days until a warmer temperature
// O(n) — each day is pushed and popped at most once
function dailyTemperatures(temps: number[]): number[] {
  const result = new Array(temps.length).fill(0);
  const stack = new Stack<number>(); // indices

  for (let i = 0; i < temps.length; i++) {
    while (!stack.isEmpty() && temps[stack.peek()!] < temps[i]) {
      const idx = stack.pop()!;
      result[idx] = i - idx; // days to wait
    }
    stack.push(i);
  }
  return result;
}

// --- Queue using two stacks ---
// Push is O(1), dequeue is amortized O(1)
// The trick: pour stack1 into stack2 only when stack2 is empty
class QueueFromStacks<T> {
  private inbox = new Stack<T>();  // push here
  private outbox = new Stack<T>(); // pop from here

  enqueue(item: T): void {
    this.inbox.push(item);
  }

  dequeue(): T | undefined {
    if (this.outbox.isEmpty()) {
      // Transfer all elements — reverses order = FIFO
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

// --- Sliding Window Maximum using Deque: O(n) ---
// For each window of size k, find the maximum
function slidingWindowMax(nums: number[], k: number): number[] {
  const result: number[] = [];
  const deque = new Deque<number>(); // stores indices, front = max

  for (let i = 0; i < nums.length; i++) {
    // Remove indices outside current window
    while (!deque.isEmpty() && deque.peekFront()! <= i - k) {
      deque.popFront();
    }
    // Remove indices of smaller elements (they can never be the max)
    while (!deque.isEmpty() && nums[deque.peekBack()!] <= nums[i]) {
      deque.popBack();
    }
    deque.pushBack(i);
    // Window is fully formed
    if (i >= k - 1) result.push(nums[deque.peekFront()!]);
  }
  return result;
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Using Array.shift() for queue dequeue.** `shift()` is O(n) because it shifts all elements. Use a linked list queue or the two-pointer/circular buffer technique instead.

> ⚠️ **Not handling the empty stack case.** Calling `pop()` or `peek()` on an empty stack returns `undefined` (or throws). Always check `isEmpty()` before accessing.

> ⚠️ **Monotonic stack direction.** Decide upfront: do you want the next **greater** or next **smaller** element? Monotonic decreasing stack → next greater. Monotonic increasing stack → next smaller. Getting this backwards produces wrong results.

> ⚠️ **Forgetting to drain remaining elements in the monotonic stack.** After processing all elements, items still on the stack have no "next greater" — fill them with -1 or 0 depending on the problem.

> ⚠️ **Circular queue off-by-one.** When implementing a circular buffer, the "full" condition is `(rear + 1) % capacity === front`, not `rear === front - 1`. Getting this wrong causes one slot to appear empty or overwritten.

---

## 6. When to Use / Not Use

**Use a Stack when:**
- Tracking function calls (call stack analog)
- Undoable operations (editors, graphics software)
- Parsing nested structures (brackets, XML, JSON)
- DFS traversal (iterative version)
- Evaluating expressions (RPN, infix → postfix)

**Use a Queue when:**
- BFS traversal
- Task scheduling (first submitted = first processed)
- Rate limiters and request buffers
- Event loops and message queues

**Use a Deque when:**
- Sliding window maximum/minimum
- Palindrome checking
- Implementing both stack and queue operations

**Use a Monotonic Stack when:**
- "Next greater/smaller element" problems
- "Largest rectangle in histogram"
- "Trapping rain water"
- Any problem where you need the nearest element satisfying an ordering condition

---

## 7. Real-World Scenario

A text editor's undo/redo system uses two stacks:

```typescript
class TextEditor {
  private undoStack: string[] = []; // stack of states
  private redoStack: string[] = [];
  private current = "";

  type(text: string): void {
    this.undoStack.push(this.current); // save current state before change
    this.redoStack.length = 0;         // any new action clears redo history
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
editor.type("Hello");
editor.type(" World");
console.log(editor.getText()); // "Hello World"
editor.undo();
console.log(editor.getText()); // "Hello"
editor.redo();
console.log(editor.getText()); // "Hello World"
```

A BFS-based shortest path finder for an unweighted graph (e.g., finding the shortest route between two pages on Wikipedia) uses a queue:

```typescript
function shortestPath(graph: Map<string, string[]>, start: string, end: string): string[] | null {
  const queue = new Queue<string[]>(); // queue of paths
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

## 8. Interview Questions

**Q1: What is the difference between a stack and a queue?**
A: Stack is LIFO (last in, first out) — used for DFS, undo/redo, call stack emulation. Queue is FIFO (first in, first out) — used for BFS, task scheduling, message buffers.

**Q2: Implement a min-stack that returns the minimum value in O(1).**
A: Maintain a parallel min-stack alongside the main stack. When pushing x, also push `min(x, minStack.top)` to the min-stack. When popping, pop from both. `getMin()` peeks the top of the min-stack. All operations O(1).

**Q3: Implement a queue using two stacks.**
A: Use `inbox` (push new items) and `outbox` (pop items from). Dequeue from outbox; if outbox is empty, pour everything from inbox into outbox first (reversing order = FIFO). Amortized O(1) dequeue because each element moves at most twice.

**Q4: What is a monotonic stack and when do you use it?**
A: A stack maintained in strictly increasing or decreasing order. When pushing, pop all elements that violate the order. Used for "next greater element", "largest rectangle in histogram", "trapping rain water". Achieves O(n) where naive nested loops would be O(n²).

**Q5: Evaluate a Reverse Polish Notation expression.**
A: Process tokens left to right. Push numbers. On an operator, pop two numbers, apply the operator, push result. Final answer is the only element remaining.

**Q6: How would you implement a browser's back/forward navigation?**
A: Two stacks — `backStack` and `forwardStack`. Navigate to a page: push current to backStack, clear forwardStack. Go back: push current to forwardStack, pop from backStack. Go forward: push current to backStack, pop from forwardStack.

---

## 9. Exercises

**Exercise 1:** Design a browser history system that supports:
- `visit(url)` — navigate to a URL
- `back(steps)` — go back up to `steps` pages
- `forward(steps)` — go forward up to `steps` pages

*Hint: two stacks — back history and forward history.*

**Exercise 2:** Implement a circular queue (fixed capacity ring buffer) with O(1) enqueue and dequeue.
*Hint: use an array with `head` and `tail` pointers, track `size` to differentiate full from empty.*

**Exercise 3:** Find the largest rectangle area in a histogram.
Input: `[2, 1, 5, 6, 2, 3]` → Output: `10`
*Hint: monotonic increasing stack — when you encounter a shorter bar, pop and calculate the area for each popped bar using the current index as the right boundary.*

**Exercise 4:** Implement a stack that supports:
- `push(val)`
- `pop()`
- `getMax()` in O(1)

*Hint: parallel max-stack, same approach as min-stack.*

**Exercise 5:** Given a string of parentheses, find the length of the longest valid (well-formed) parentheses substring.
Input: `")()())"` → Output: `4` ("()()")
*Hint: stack that stores indices.*

---

## 10. Further Reading

- CLRS Chapter 10.1 — Stacks and Queues
- [Monotonic Stack — patterns explained](https://labuladong.online/algo/data-structure/monotonic-stack/)
- LeetCode problems: #20 Valid Parentheses, #155 Min Stack, #232 Queue using Stacks, #239 Sliding Window Maximum, #84 Largest Rectangle in Histogram
- [Circular Buffer — Wikipedia](https://en.wikipedia.org/wiki/Circular_buffer)
