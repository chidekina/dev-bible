# Heaps

## 1. What & Why

A heap is a specialized tree-based data structure that satisfies the **heap property**: in a max-heap, every parent is greater than or equal to its children; in a min-heap, every parent is less than or equal to its children. This property makes finding the maximum (or minimum) an O(1) operation and maintaining it after insertions and deletions an O(log n) operation.

Heaps power priority queues — one of the most useful data structures for scheduling, graph algorithms (Dijkstra), and stream processing (find the k largest elements). Understanding heaps is essential for understanding how operating systems schedule processes, how A* pathfinding works, and how merge operations on sorted lists are done efficiently.

---

## 2. Core Concepts

### Heap as a Complete Binary Tree

A heap is a **complete binary tree**: all levels are fully filled except possibly the last, which is filled from left to right. This specific shape allows the heap to be stored as a flat array — no pointers needed:

```
For a node at index i (0-based):
  Parent:      Math.floor((i - 1) / 2)
  Left child:  2 * i + 1
  Right child: 2 * i + 2
```

### Min-Heap vs Max-Heap

- **Min-heap:** parent ≤ children. Root is always the minimum. Used for "k smallest", Dijkstra's algorithm.
- **Max-heap:** parent ≥ children. Root is always the maximum. Used for "k largest", heap sort.

### Core Operations

- **insert (heapify up):** Add to the end of the array, then bubble up to restore heap property. O(log n).
- **extractMin/extractMax (heapify down):** Remove root, replace with last element, then bubble down. O(log n).
- **peek:** Return root without removing. O(1).
- **buildHeap:** Convert an arbitrary array into a heap. O(n) — not O(n log n). The key insight: heapify starting from the last non-leaf node downward.

### Why buildHeap is O(n) not O(n log n)

Intuitively, most nodes in a heap are near the bottom and need to sift down only a few levels. The total work is bounded by the sum of heights of all nodes, which equals O(n) by the geometric series argument.

---

## 3. How It Works

```
Min-heap as array: [1, 3, 2, 7, 6, 5, 4]

As tree:
        1          ← index 0
       / \
      3   2        ← indices 1, 2
     / \ / \
    7  6 5  4     ← indices 3, 4, 5, 6

Insert 0:
  Append: [1, 3, 2, 7, 6, 5, 4, 0]  (index 7)
  Parent of 7 = index 3 (value 7). 0 < 7 → swap
  [1, 3, 2, 0, 6, 5, 4, 7]
  Parent of 3 = index 1 (value 3). 0 < 3 → swap
  [1, 0, 2, 3, 6, 5, 4, 7]
  Parent of 1 = index 0 (value 1). 0 < 1 → swap
  [0, 1, 2, 3, 6, 5, 4, 7]  ← done, 0 is now root

Extract min (remove root = 0):
  Replace root with last element 7:
  [7, 1, 2, 3, 6, 5, 4]
  Sift down: 7 > min(1, 2) = 1 → swap with left child
  [1, 7, 2, 3, 6, 5, 4]
  7 > min(3, 6) = 3 → swap with left child of 1
  [1, 3, 2, 7, 6, 5, 4]  ← done
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- MinHeap implementation ---
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
    this.heap[0] = this.heap.pop()!; // replace root with last element
    this.heapifyDown(0);
    return min;
  }

  // Build heap from arbitrary array: O(n)
  buildFrom(arr: number[]): void {
    this.heap = [...arr];
    // Start from last non-leaf node, sift down each
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

// --- Generic Priority Queue (min or max) ---
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

// Usage:
const minPQ = new PriorityQueue<number>((a, b) => a - b); // min-heap
const maxPQ = new PriorityQueue<number>((a, b) => b - a); // max-heap
const taskPQ = new PriorityQueue<{ priority: number; task: string }>(
  (a, b) => a.priority - b.priority
);

// --- K Largest Elements: O(n log k) ---
// Use a MIN-heap of size k — if new element > heap min, replace
function kLargest(nums: number[], k: number): number[] {
  const minHeap = new PriorityQueue<number>((a, b) => a - b);

  for (const num of nums) {
    minHeap.push(num);
    if (minHeap.size > k) minHeap.pop(); // remove smallest, keeping top k
  }

  const result: number[] = [];
  while (!minHeap.isEmpty()) result.push(minHeap.pop()!);
  return result.reverse(); // largest first
}

// --- K Smallest Elements: O(n log k) ---
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

// --- Merge K Sorted Lists: O(n log k) where n = total elements ---
class ListNode {
  constructor(public val: number, public next: ListNode | null = null) {}
}

function mergeKLists(lists: Array<ListNode | null>): ListNode | null {
  const pq = new PriorityQueue<ListNode>((a, b) => a.val - b.val);

  // Push the head of each list
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

// --- Find Median of Data Stream: O(log n) add, O(1) find median ---
// Two heaps: maxHeap for lower half, minHeap for upper half
class MedianFinder {
  private lower = new PriorityQueue<number>((a, b) => b - a); // max-heap (lower half)
  private upper = new PriorityQueue<number>((a, b) => a - b); // min-heap (upper half)

  addNum(num: number): void {
    // Add to lower half first
    this.lower.push(num);

    // Balance: lower max must be ≤ upper min
    if (!this.upper.isEmpty() && this.lower.peek()! > this.upper.peek()!) {
      this.upper.push(this.lower.pop()!);
    }

    // Balance sizes: lower can be at most 1 larger than upper
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

  // Build max-heap
  for (let i = Math.floor(n / 2) - 1; i >= 0; i--) {
    siftDown(a, i, n);
  }

  // Extract max one by one, placing at end
  for (let end = n - 1; end > 0; end--) {
    [a[0], a[end]] = [a[end], a[0]]; // move max to end
    siftDown(a, 0, end);              // restore heap on reduced range
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

// --- K Closest Points to Origin: O(n log k) ---
function kClosest(points: [number, number][], k: number): [number, number][] {
  const distSq = ([x, y]: [number, number]) => x * x + y * y;
  // Max-heap of size k — pop when size > k
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

## 5. Common Mistakes & Pitfalls

> ⚠️ **Using a max-heap when you need k smallest (or vice versa).** For "k largest elements", use a MIN-heap of size k — remove the smallest whenever size exceeds k, keeping the k largest. Counterintuitive but correct.

> ⚠️ **Off-by-one in parent/child index formulas.** 0-based indexing: parent = `Math.floor((i - 1) / 2)`, left = `2i + 1`, right = `2i + 2`. For 1-based: parent = `Math.floor(i / 2)`, left = `2i`, right = `2i + 1`.

> ⚠️ **Forgetting that JavaScript has no built-in priority queue.** You must implement one or use a library. `Array.sort()` each time is O(n log n) per operation — not O(log n).

> ⚠️ **buildHeap complexity.** Building a heap by inserting n elements one at a time is O(n log n). The Floyd buildHeap algorithm (sift down from last non-leaf) is O(n) — use it when converting an existing array.

> ⚠️ **Heap is NOT a sorted array.** Heap only guarantees the root is min/max. The rest of the array is not sorted.

---

## 6. When to Use / Not Use

**Use a heap when:**
- You need to repeatedly extract the min or max from a dynamic collection
- Finding the k largest/smallest elements from a stream
- Merging k sorted lists
- Implementing Dijkstra or Prim's algorithm
- Scheduling tasks by priority
- Finding median of a data stream

**Do NOT use a heap when:**
- You need to search for an arbitrary element — heap is O(n) for arbitrary search
- You need sorted output — use sort or BST
- You need random access by index — use an array

---

## 7. Real-World Scenario

A job queue system processes tasks by priority. New tasks arrive constantly; the system always executes the highest-priority pending task:

```typescript
interface Job {
  id: string;
  priority: number;
  execute: () => Promise<void>;
}

class JobScheduler {
  private queue = new PriorityQueue<Job>((a, b) => b.priority - a.priority); // max-heap by priority
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
      this.processNext(); // process next job after current completes
    }
  }
}
```

A monitoring system needs the top 10 slowest API endpoints across millions of requests:

```typescript
function top10Slowest(responseTimes: { endpoint: string; ms: number }[]) {
  const k = 10;
  const minHeap = new PriorityQueue<{ endpoint: string; ms: number }>(
    (a, b) => a.ms - b.ms // min-heap
  );

  for (const entry of responseTimes) {
    minHeap.push(entry);
    if (minHeap.size > k) minHeap.pop(); // drop fastest, keep top 10 slowest
  }

  const result = [];
  while (!minHeap.isEmpty()) result.push(minHeap.pop()!);
  return result.reverse(); // slowest first
}
```

---

## 8. Interview Questions

**Q1: Why is building a heap O(n) and not O(n log n)?**
A: When inserting n elements one-by-one, each insert is O(log n) → total O(n log n). But Floyd's buildHeap starts from the last non-leaf and sifts down. Most nodes are near the bottom and need only O(1) sift-down work. The total work is bounded by the sum of node heights = O(n) by geometric series.

**Q2: How would you implement a priority queue?**
A: Use a heap stored as an array. Insert: push to end, heapify up — O(log n). Extract min/max: swap root with last, remove last, heapify down — O(log n). Peek: return root — O(1).

**Q3: How do you find the kth largest element in an array?**
A: Use a min-heap of size k. For each element, push to heap; if size > k, pop the min. After processing all elements, the heap contains the k largest; the root is the kth largest. O(n log k).

**Q4: How do you find the median of a data stream efficiently?**
A: Two heaps: a max-heap for the lower half and a min-heap for the upper half. Maintain them balanced (sizes differ by at most 1). After each insert, balance the heaps if needed. Median is the root of the larger heap (odd total) or average of both roots (even total). O(log n) insert, O(1) median.

**Q5: What is heap sort and what are its properties?**
A: Build a max-heap from the array (O(n)), then repeatedly extract the max and place at the end (O(n log n)). Total: O(n log n). Properties: in-place (O(1) space), not stable, O(n log n) worst case guaranteed (unlike quicksort).

---

## 9. Exercises

**Exercise 1:** Sort a k-sorted array — an array where each element is at most k positions away from its correct sorted position. Achieve O(n log k).
*Hint: maintain a min-heap of size k+1. For each new element, push to heap and pop the min to output.*

**Exercise 2:** Find the k closest points to the origin from a list of 2D points.
*Hint: max-heap of size k, ordered by distance squared — no need for sqrt.*

**Exercise 3:** Design a task scheduler. Given tasks with priorities and execution times, simulate execution such that the highest-priority task runs first. When a task completes, the next highest-priority ready task runs.

**Exercise 4:** Given a list of integers, find if any two numbers sum to a given target k using a heap-based approach.

**Exercise 5:** Implement heap sort from scratch and verify it produces the correct sorted output.

---

## 10. Further Reading

- CLRS Chapter 6 — Heapsort
- [Binary Heap — Wikipedia](https://en.wikipedia.org/wiki/Binary_heap)
- [Heap visualization](https://visualgo.net/en/heap)
- LeetCode problems: #215 Kth Largest Element, #23 Merge K Sorted Lists, #295 Find Median from Data Stream, #347 Top K Frequent Elements, #973 K Closest Points to Origin
