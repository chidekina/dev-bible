# Linked Lists

## 1. What & Why

A linked list is a linear data structure where each element (node) contains a value and a pointer to the next node. Unlike arrays, linked list nodes are scattered throughout memory — there is no guaranteed contiguity. This one difference changes the entire performance profile.

The key trade-off: linked lists offer O(1) insertion and deletion at any position you already have a pointer to, while arrays require O(n) shifting. But linked lists require O(n) time to find a position (no index arithmetic), while arrays offer O(1) access.

Linked lists power many higher-level data structures: stacks, queues, hash table chaining, adjacency lists in graphs, and the LRU cache.

---

## 2. Core Concepts

### Types of Linked Lists

**Singly Linked List:** Each node has `value` and `next`. Traversal is one-directional. Simple and memory-efficient.

**Doubly Linked List:** Each node has `value`, `next`, and `prev`. Enables O(1) deletion if you have the node (no need to find the previous node). Used in LRU caches and browser history.

**Circular Linked List:** The last node's `next` points back to the head (or some other node). Used in round-robin scheduling, circular buffers.

### Complexity Comparison

| Operation | Linked List | Array |
|---|---|---|
| Access by index | O(n) | O(1) |
| Search by value | O(n) | O(n) |
| Insert at head | O(1) | O(n) |
| Insert at tail | O(1) with tail pointer | O(1) amortized |
| Insert at middle (given pointer) | O(1) | O(n) |
| Delete at head | O(1) | O(n) |
| Delete (given pointer) | O(1) doubly / O(n) singly | O(n) |

### Key Pointer Patterns

- **Fast/slow pointers (Floyd's algorithm):** Two pointers advancing at different speeds — detects cycles, finds midpoints.
- **Previous pointer tracking:** When deleting a node in a singly linked list, you need the previous node.
- **Dummy head node:** A sentinel node before the real head simplifies edge cases at the head.

---

## 3. How It Works

```
Singly Linked List: 1 → 2 → 3 → 4 → null

head → [1|next] → [2|next] → [3|next] → [4|null]

Memory: nodes are not contiguous — they can be anywhere on the heap.
Access arr[2]: must traverse from head: 1 → 2 → 3 — O(n)
```

**Cycle detection (Floyd's tortoise and hare):**
```
slow moves 1 step per iteration
fast moves 2 steps per iteration

If there is a cycle, fast will eventually "lap" slow — they meet inside the cycle.
If no cycle, fast reaches null first.
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- Node and List definitions ---

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

  // Insert at head: O(1)
  prepend(value: T): void {
    const node = new ListNode(value, this.head);
    this.head = node;
    if (this.tail === null) this.tail = node;
    this.size++;
  }

  // Insert at tail: O(1) with tail pointer
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

  // Delete by value: O(n)
  delete(value: T): boolean {
    if (this.head === null) return false;

    // Special case: deleting the head
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

  // Search: O(n)
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

// --- Reverse a linked list in-place: O(n) time, O(1) space ---
function reverseList<T>(head: ListNode<T> | null): ListNode<T> | null {
  let prev: ListNode<T> | null = null;
  let current = head;

  while (current !== null) {
    const next = current.next; // save next before overwriting
    current.next = prev;       // reverse the pointer
    prev = current;            // advance prev
    current = next;            // advance current
  }

  return prev; // prev is the new head
}

// --- Detect cycle (Floyd's tortoise and hare): O(n) time, O(1) space ---
function hasCycle<T>(head: ListNode<T> | null): boolean {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;         // moves 1 step
    fast = fast.next.next;     // moves 2 steps

    if (slow === fast) return true; // they met — cycle exists
  }
  return false; // fast reached null — no cycle
}

// --- Find cycle start (Floyd's extended): O(n) time, O(1) space ---
function detectCycleStart<T>(head: ListNode<T> | null): ListNode<T> | null {
  let slow = head;
  let fast = head;

  // Phase 1: detect meeting point inside cycle
  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) break;
  }

  if (fast === null || fast.next === null) return null; // no cycle

  // Phase 2: move one pointer to head, advance both at same speed
  // They will meet at the cycle start — mathematical proof via distances
  slow = head;
  while (slow !== fast) {
    slow = slow!.next;
    fast = fast!.next;
  }
  return slow;
}

// --- Find middle of linked list: O(n) time, O(1) space ---
// Fast/slow pointer: when fast reaches end, slow is at middle
function findMiddle<T>(head: ListNode<T> | null): ListNode<T> | null {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
  }
  return slow; // for even length, returns second middle node
}

// --- Merge two sorted linked lists: O(n + m) time, O(1) space ---
function mergeSorted(
  l1: ListNode<number> | null,
  l2: ListNode<number> | null
): ListNode<number> | null {
  const dummy = new ListNode<number>(0); // sentinel to avoid head special case
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

  current.next = l1 ?? l2; // attach remaining list
  return dummy.next;
}

// --- Find nth node from end: O(n) time, O(1) space ---
// Two pointers: advance first pointer n steps, then move both until first reaches end
function nthFromEnd<T>(head: ListNode<T> | null, n: number): ListNode<T> | null {
  let ahead = head;
  let behind = head;

  // Advance 'ahead' n steps
  for (let i = 0; i < n; i++) {
    if (ahead === null) return null; // n > list length
    ahead = ahead.next;
  }

  // Move both until ahead reaches null
  while (ahead !== null) {
    ahead = ahead.next;
    behind = behind!.next;
  }

  return behind;
}

// --- Check if linked list is a palindrome: O(n) time, O(1) space ---
function isPalindromeList(head: ListNode<number> | null): boolean {
  if (head === null || head.next === null) return true;

  // Step 1: find middle
  let slow = head;
  let fast = head;
  while (fast.next !== null && fast.next.next !== null) {
    slow = slow.next!;
    fast = fast.next.next;
  }

  // Step 2: reverse second half
  let secondHalf = reverseList(slow.next);

  // Step 3: compare first and second halves
  let left: ListNode<number> | null = head;
  let right = secondHalf;
  let isPalin = true;
  while (right !== null) {
    if (left!.value !== right.value) { isPalin = false; break; }
    left = left!.next;
    right = right.next;
  }

  // Step 4: restore list (optional, for non-destructive behavior)
  slow.next = reverseList(secondHalf);

  return isPalin;
}

// --- Doubly Linked List ---
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

  // O(1) delete given the node itself
  deleteNode(node: DListNode<T>): void {
    if (node.prev) node.prev.next = node.next;
    else this.head = node.next; // node was head

    if (node.next) node.next.prev = node.prev;
    else this.tail = node.prev; // node was tail
  }
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Forgetting to handle the head node specially in singly linked lists.** Deletion and insertion at the head cannot use the "find previous" pattern. Use a dummy sentinel node to unify all cases.

> ⚠️ **Not checking `fast.next !== null` in fast/slow pointer loops.** If the list length is even, `fast.next.next` can NPE before `fast` itself is null.

> ⚠️ **Losing the reference to the next node when reversing.** Always save `current.next` in a temp variable before overwriting `current.next = prev`.

> ⚠️ **Forgetting to update the tail pointer.** When deleting the last node or appending, many implementations forget to update `tail`, causing bugs on subsequent tail operations.

> ⚠️ **Assuming `O(1)` deletion from a singly linked list.** You need the *previous* node to perform deletion. If you only have the current node, you must traverse from head — O(n). Doubly linked lists solve this.

---

## 6. When to Use / Not Use

**Use a linked list when:**
- You need O(1) insertions/deletions at a known position (e.g., LRU cache eviction)
- You are implementing a stack or queue and want O(1) push/pop at both ends
- The list size fluctuates dramatically (no wasted capacity like dynamic arrays)
- You are implementing hash table chaining

**Do NOT use a linked list when:**
- You need indexed access — O(n) traversal is much slower than O(1) array access
- Cache performance matters — non-contiguous memory kills CPU prefetch
- Memory is tight — each node carries pointer overhead (8 bytes per pointer on 64-bit)
- The data is fixed-size and sequential — a plain array is almost always faster in practice

---

## 7. Real-World Scenario

An LRU (Least Recently Used) cache requires O(1) access, O(1) insert, and O(1) eviction of the oldest entry. This is impossible with an array alone (O(n) delete), but a combination of a hashmap + doubly linked list achieves all three:

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
    // Move to front (most recently used)
    this.list.deleteNode(node);
    const newNode = this.list.append(node.value);
    this.map.set(key, newNode);
    return node.value.value;
  }

  put(key: number, value: number): void {
    if (this.map.has(key)) {
      this.list.deleteNode(this.map.get(key)!);
    } else if (this.map.size >= this.capacity) {
      // Evict LRU (tail of list — or head, depending on your orientation)
      // Implementation detail: track head/tail appropriately
    }
    const node = this.list.append({ key, value });
    this.map.set(key, node);
  }
}
```

---

## 8. Interview Questions

**Q1: How do you detect a cycle in a linked list?**
A: Floyd's tortoise and hare algorithm. Two pointers — slow moves 1 step, fast moves 2 steps per iteration. If there is a cycle, fast will eventually catch slow (they meet inside the cycle). If no cycle, fast reaches null. O(n) time, O(1) space.

**Q2: How do you reverse a linked list in-place?**
A: Iterate with three pointers: `prev = null`, `current = head`, and `next`. In each step: save `current.next`, set `current.next = prev`, advance `prev = current`, advance `current = next`. Return `prev` as the new head. O(n) time, O(1) space.

**Q3: How do you find the nth node from the end of a linked list?**
A: Two-pointer technique. Advance the first pointer n steps ahead. Then move both pointers together until the first reaches null. The second pointer is now at the nth node from the end. O(n) time, O(1) space, single pass.

**Q4: How do you find the middle of a linked list?**
A: Fast/slow pointer. Move slow 1 step, fast 2 steps. When fast reaches the end, slow is at the middle. O(n) time, O(1) space.

**Q5: What is the difference between a singly and doubly linked list?**
A: Singly has only a `next` pointer — O(n) deletion in general (must find previous node). Doubly has `prev` and `next` — O(1) deletion given the node itself, but each node uses more memory (one extra pointer).

**Q6: When would you prefer a linked list over an array?**
A: When insertions/deletions at arbitrary positions are frequent and you already have a pointer to the position. The LRU cache is the canonical example. In practice, linked lists are rarely faster than arrays for sequential workloads due to cache behavior.

---

## 9. Exercises

**Exercise 1:** Implement a doubly linked list with `append`, `prepend`, `deleteNode`, and `toArray` operations. Test all edge cases: empty list, single element, deleting head, deleting tail.

**Exercise 2:** Given a linked list that may contain a cycle, detect the cycle and remove it (set the node that points back to `null`).
*Hint: use Floyd's algorithm to find the cycle start, then find the node whose next is the cycle start.*

**Exercise 3:** Check if a singly linked list is a palindrome in O(n) time and O(1) space.
*Hint: find the middle, reverse the second half, compare, then restore.*

**Exercise 4:** Merge k sorted linked lists into one sorted linked list.
*Hint: use a min-heap (priority queue) of size k — always extract the minimum, then push the next node from that list. O(n log k) where n = total nodes.*

**Exercise 5:** Remove all occurrences of a given value from a linked list. Return the new head.
*Hint: use a dummy node before the head to handle removal of head nodes cleanly.*

---

## 10. Further Reading

- CLRS Chapter 10 — Elementary Data Structures (Linked Lists)
- [NeetCode — Linked List playlist](https://neetcode.io/roadmap)
- [Floyd's Cycle Detection — proof](https://en.wikipedia.org/wiki/Cycle_detection#Floyd's_tortoise_and_hare)
- LeetCode problems: #206 Reverse Linked List, #141 Linked List Cycle, #21 Merge Two Sorted Lists, #146 LRU Cache
