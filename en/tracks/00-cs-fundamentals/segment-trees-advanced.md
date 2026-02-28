# Segment Trees & Advanced Data Structures

## 1. What & Why

Segment trees and Fenwick trees (Binary Indexed Trees) solve a class of problems involving **range queries with updates** — operations that would otherwise require O(n) time per query on a plain array.

The canonical problem: given an array of n numbers, handle two types of operations:
1. Update the value at index i
2. Query the sum (or min/max) of elements in range [l, r]

Naive approach: O(1) update, O(n) query. Or precompute prefix sums: O(n) build, O(1) query, O(n) update. Neither is efficient when both operations are frequent.

Segment tree: O(log n) for both update and query. Fenwick tree: same complexity, simpler implementation for prefix sums.

These structures appear in competitive programming, database engines (range queries on B-trees), and computational geometry. They are advanced topics — typically asked only at senior-level or FAANG interviews.

---

## 2. Core Concepts

### Segment Tree

A segment tree is a full binary tree where:
- Each **leaf** stores a single element of the original array
- Each **internal node** stores the aggregate (sum, min, max, gcd, etc.) of the range its children cover
- The root covers the entire array [0, n-1]
- Node at index i covers range [l, r]; its left child covers [l, mid] and right child covers [mid+1, r]

**Storage:** Like heaps, segment trees can be stored as arrays. For a node at index i: left child = 2i, right child = 2i+1. Use 1-based indexing. Array size = 4n to handle all cases.

**Build:** O(n) — compute each node bottom-up.
**Point update:** O(log n) — update leaf, recompute ancestors.
**Range query:** O(log n) — combine at most O(log n) nodes.

### Fenwick Tree (Binary Indexed Tree / BIT)

A simpler structure for prefix sum queries with point updates. Uses the binary representation of indices to efficiently store partial sums. O(log n) update and prefix query. More space-efficient than a segment tree but less flexible (works well for prefix operations but harder to adapt for arbitrary range operations).

### Lazy Propagation

For **range updates** (e.g., add 5 to all elements in [l, r]), updating each leaf individually is O(n log n). Lazy propagation defers updates: mark a node as "pending update" and only propagate when a child needs to be accessed. Achieves O(log n) range update + O(log n) range query.

### Suffix Array

A sorted array of all suffixes of a string. Enables O(log n) exact substring search and O(n) LCP (Longest Common Prefix) computation. Used in bioinformatics, data compression, and string algorithms.

---

## 3. How It Works

```
Array: [2, 4, 5, 7, 2, 3, 1, 6]  (n = 8)

Segment Tree (sum):
                  [0,7] sum=30
              /                  \
        [0,3] sum=18          [4,7] sum=12
       /         \             /          \
  [0,1] sum=6  [2,3] sum=12  [4,5] sum=5  [6,7] sum=7
  /    \        /      \      /    \       /    \
[0]=2 [1]=4  [2]=5  [3]=7  [4]=2 [5]=3  [6]=1  [7]=6

Query sum(2, 5):
- At root [0,7]: range spans query, go both ways
- At [0,3]: mid=1 → right [2,3] fully within [2,5] → return 12
- At [4,7]: mid=5 → left [4,5] fully within [2,5] → return 5
- Total: 12 + 5 = 17 ✓

Update arr[3] = 10 (was 7, delta = +3):
- Update leaf [3] = 10
- Update parent [2,3] = 5 + 10 = 15
- Update grandparent [0,3] = 6 + 15 = 21
- Update root [0,7] = 21 + 12 = 33
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- Segment Tree: Range Sum with Point Update ---
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

  // Point update: set arr[index] = val. O(log n)
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

  // Range sum query [left, right]: O(log n)
  query(left: number, right: number): number {
    return this.queryHelper(1, 0, this.n - 1, left, right);
  }

  private queryHelper(node: number, start: number, end: number, left: number, right: number): number {
    if (right < start || end < left) return 0;   // no overlap
    if (left <= start && end <= right) return this.tree[node]; // full overlap

    // Partial overlap: recurse on both children
    const mid = Math.floor((start + end) / 2);
    return (
      this.queryHelper(2 * node, start, mid, left, right) +
      this.queryHelper(2 * node + 1, mid + 1, end, left, right)
    );
  }
}

// --- Segment Tree: Range Minimum Query ---
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

// --- Fenwick Tree (Binary Indexed Tree): Prefix Sums ---
// Simpler implementation for sum queries with point updates
class FenwickTree {
  private tree: number[];
  private n: number;

  constructor(n: number) {
    this.n = n;
    this.tree = new Array(n + 1).fill(0); // 1-indexed
  }

  // Point update: add delta to index i (1-indexed). O(log n)
  update(i: number, delta: number): void {
    for (; i <= this.n; i += i & (-i)) { // i & (-i) isolates lowest set bit
      this.tree[i] += delta;
    }
  }

  // Prefix sum [1, i]: O(log n)
  prefixSum(i: number): number {
    let sum = 0;
    for (; i > 0; i -= i & (-i)) {
      sum += this.tree[i];
    }
    return sum;
  }

  // Range sum [l, r] (1-indexed): O(log n)
  rangeSum(l: number, r: number): number {
    return this.prefixSum(r) - this.prefixSum(l - 1);
  }

  // Build from array: O(n log n)
  static fromArray(arr: number[]): FenwickTree {
    const bit = new FenwickTree(arr.length);
    for (let i = 0; i < arr.length; i++) {
      bit.update(i + 1, arr[i]); // convert to 1-indexed
    }
    return bit;
  }
}

// --- Count of Smaller Numbers After Self using BIT ---
// For each element, count how many elements to its right are smaller
// O(n log n)
function countSmaller(nums: number[]): number[] {
  // Coordinate compression: map values to [1..n]
  const sorted = [...new Set(nums)].sort((a, b) => a - b);
  const rank = new Map(sorted.map((v, i) => [v, i + 1]));

  const bit = new FenwickTree(sorted.length);
  const result: number[] = new Array(nums.length);

  // Process from right to left
  for (let i = nums.length - 1; i >= 0; i--) {
    const r = rank.get(nums[i])!;
    result[i] = r > 1 ? bit.prefixSum(r - 1) : 0; // count smaller values seen so far
    bit.update(r, 1);
  }
  return result;
}

// --- Segment Tree with Lazy Propagation: Range Update + Range Sum ---
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
    // Apply pending update to children
    this.tree[2 * node] += this.lazy[node] * (mid - start + 1);
    this.lazy[2 * node] += this.lazy[node];
    this.tree[2 * node + 1] += this.lazy[node] * (end - mid);
    this.lazy[2 * node + 1] += this.lazy[node];
    this.lazy[node] = 0;
  }

  // Range update: add val to all elements in [left, right]. O(log n)
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

  // Range sum query [left, right]. O(log n)
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

// --- Simple prefix sum array (when no updates needed) ---
// O(n) build, O(1) query — use this when updates are rare/absent
class PrefixSumArray {
  private prefix: number[];

  constructor(arr: number[]) {
    this.prefix = new Array(arr.length + 1).fill(0);
    for (let i = 0; i < arr.length; i++) {
      this.prefix[i + 1] = this.prefix[i] + arr[i];
    }
  }

  // Range sum [left, right] inclusive, 0-indexed: O(1)
  query(left: number, right: number): number {
    return this.prefix[right + 1] - this.prefix[left];
  }
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Segment tree array size too small.** Allocate 4 × n elements, not 2 × n. For non-power-of-2 input sizes, the tree may need up to 4n nodes.

> ⚠️ **Off-by-one in 1-based vs 0-based indexing.** Fenwick trees are naturally 1-indexed (`i & (-i)` breaks for i=0). Segment trees can use either — be consistent.

> ⚠️ **Forgetting to call pushDown before recursing in lazy segment trees.** If you recurse into children without first applying the parent's pending lazy update, children return stale values.

> ⚠️ **Using a segment tree when a simple prefix sum suffices.** If the array never changes after build, `PrefixSumArray` is simpler, faster in practice, and O(1) per query.

> ⚠️ **Fenwick tree: update is additive, not assignment.** `bit.update(i, val)` adds val to position i. To set position i to a new value, compute the delta: `bit.update(i, newVal - oldVal)`.

---

## 6. When to Use / Not Use

**Use a Segment Tree when:**
- Range queries (sum, min, max, gcd) with frequent point or range updates
- The query function is associative (can be combined from sub-ranges)
- Competitive programming, interval-based problems

**Use a Fenwick Tree when:**
- Only prefix sum queries (not general range queries)
- Simpler implementation is preferred
- Counting inversions, coordinate compression problems

**Use a simple prefix sum when:**
- Array is static (no updates after build)
- O(1) queries needed, no update overhead

**Skip these when:**
- n is small — brute force is cleaner and fast enough
- The problem doesn't involve repeated range queries — just scan the array once

---

## 7. Real-World Scenario

A financial dashboard shows running totals for any date range and handles real-time updates when transactions arrive:

```typescript
class TransactionLedger {
  private segTree: SegmentTree;
  private amounts: number[];

  constructor(initialAmounts: number[]) {
    this.amounts = [...initialAmounts];
    this.segTree = new SegmentTree(this.amounts);
  }

  // New transaction at index (day)
  addTransaction(dayIndex: number, amount: number): void {
    this.amounts[dayIndex] += amount;
    this.segTree.update(dayIndex, this.amounts[dayIndex]);
  }

  // Query total for date range [startDay, endDay]
  totalForRange(startDay: number, endDay: number): number {
    return this.segTree.query(startDay, endDay);
  }
}
```

---

## 8. Interview Questions

**Q1: When would you use a segment tree over a simple prefix sum array?**
A: When the array is updated frequently alongside range queries. A static prefix sum array gives O(1) queries but O(n) updates (need to rebuild). A segment tree gives O(log n) for both. If there are no updates, use the simpler prefix sum.

**Q2: What is the difference between a segment tree and a Fenwick tree?**
A: A Fenwick tree (BIT) is simpler to implement and works naturally for prefix sum queries with point updates — O(log n) each. A segment tree is more general: it can handle any associative operation (min, max, gcd), range updates with lazy propagation, and more complex queries. Fenwick tree has better constants in practice.

**Q3: What is lazy propagation?**
A: A technique for efficient range updates on a segment tree. Instead of updating every affected leaf immediately (O(n)), mark internal nodes with a "pending update" (lazy tag). The update is only applied when a child node needs to be accessed. Achieves O(log n) range update + O(log n) query.

**Q4: How does a Fenwick tree work?**
A: Each index i stores the sum of a specific range determined by the lowest set bit of i. `i & (-i)` isolates that bit. Update: add to i, then jump to the next responsible index. Prefix query: sum i, then jump down by removing the lowest set bit. Both operations are O(log n).

---

## 9. Exercises

**Exercise 1:** Implement range sum query with point updates using both a segment tree and a Fenwick tree. Verify they produce identical results.

**Exercise 2:** Extend the segment tree to support range minimum query (RMQ). Handle both point updates and range queries.

**Exercise 3:** Count the number of smaller elements to the right of each element in an array using a Fenwick tree with coordinate compression. Expected: O(n log n).
*Hint: process from right to left, use BIT to count how many elements less than current have been seen.*

**Exercise 4:** Implement lazy propagation to support: (a) add val to all elements in range [l, r], and (b) query sum of elements in range [l, r]. Both in O(log n).

**Exercise 5:** Given an array of integers, find the number of range sum queries where the sum equals zero. Use prefix sums + a hashmap.
*Hint: if prefixSum[i] == prefixSum[j], then sum(i+1..j) == 0.*

---

## 10. Further Reading

- CLRS Chapter 14 — Augmenting Data Structures
- [Segment Tree — cp-algorithms.com](https://cp-algorithms.com/data_structures/segment_tree.html)
- [Fenwick Tree — cp-algorithms.com](https://cp-algorithms.com/data_structures/fenwick.html)
- [Lazy Propagation — codeforces.com](https://codeforces.com/blog/entry/18051)
- LeetCode problems: #307 Range Sum Query Mutable (segment tree / BIT), #315 Count of Smaller Numbers After Self (BIT), #327 Count of Range Sum
