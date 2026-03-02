# Big-O Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## Complexity Classes (Best to Worst)

| Notation | Name | Example |
|----------|------|---------|
| O(1) | Constant | Array index access, hash map lookup |
| O(log n) | Logarithmic | Binary search, balanced BST ops |
| O(n) | Linear | Linear search, single loop |
| O(n log n) | Linearithmic | Merge sort, heap sort |
| O(n²) | Quadratic | Bubble sort, nested loops |
| O(n³) | Cubic | Naive matrix multiplication |
| O(2ⁿ) | Exponential | Recursive Fibonacci, power set |
| O(n!) | Factorial | Permutation generation, brute-force TSP |

---

## Data Structure Operations

### Array / Dynamic Array (JavaScript Array, Python list)

| Operation | Average | Worst |
|-----------|---------|-------|
| Access by index | O(1) | O(1) |
| Search (unsorted) | O(n) | O(n) |
| Search (sorted, binary) | O(log n) | O(log n) |
| Insert at end (amortized) | O(1) | O(n) — resize |
| Insert at beginning | O(n) | O(n) |
| Insert at arbitrary index | O(n) | O(n) |
| Delete at end | O(1) | O(1) |
| Delete at beginning | O(n) | O(n) |
| Delete at arbitrary index | O(n) | O(n) |

**Space:** O(n)

---

### Linked List (Singly / Doubly)

| Operation | Singly | Doubly |
|-----------|--------|--------|
| Access by index | O(n) | O(n) |
| Search | O(n) | O(n) |
| Insert at head | O(1) | O(1) |
| Insert at tail (with tail ptr) | O(1) | O(1) |
| Insert at arbitrary index | O(n) | O(n) |
| Delete at head | O(1) | O(1) |
| Delete at tail | O(n) | O(1) |
| Delete with node reference | O(n) | O(1) |

**Space:** O(n)

---

### Stack / Queue

| Operation | Stack | Queue |
|-----------|-------|-------|
| Push / Enqueue | O(1) | O(1) |
| Pop / Dequeue | O(1) | O(1) |
| Peek | O(1) | O(1) |
| Search | O(n) | O(n) |

**Space:** O(n)
> Implement Stack with array or linked list. Implement Queue with doubly linked list or circular buffer to avoid O(n) dequeue.

---

### Hash Map / Hash Set

| Operation | Average | Worst (all collisions) |
|-----------|---------|------------------------|
| Insert | O(1) | O(n) |
| Delete | O(1) | O(n) |
| Lookup / Contains | O(1) | O(n) |

**Space:** O(n)
> Worst case is rare with a good hash function. Load factor kept < 0.75 triggers resize.

---

### Binary Search Tree (BST)

| Operation | Average (balanced) | Worst (degenerate/linear) |
|-----------|--------------------|---------------------------|
| Access / Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |

**Space:** O(n)
> Use self-balancing variants (AVL, Red-Black) to guarantee O(log n).

---

### Balanced BST (AVL / Red-Black Tree)

| Operation | Average | Worst |
|-----------|---------|-------|
| Access / Search | O(log n) | O(log n) |
| Insert | O(log n) | O(log n) |
| Delete | O(log n) | O(log n) |

**Space:** O(n)

---

### Heap (Binary Heap)

| Operation | Time |
|-----------|------|
| Find min/max | O(1) |
| Insert | O(log n) |
| Delete min/max (extract) | O(log n) |
| Build heap (heapify array) | O(n) |
| Delete arbitrary element | O(log n) |

**Space:** O(n)

---

### Trie (Prefix Tree)

| Operation | Time |
|-----------|------|
| Insert | O(m) — m = key length |
| Search | O(m) |
| Starts-with prefix | O(m) |
| Delete | O(m) |

**Space:** O(n × m) — n keys, m average length

---

### Graph

| Representation | Space | Add Vertex | Add Edge | Remove Edge | Query Edge |
|----------------|-------|------------|----------|-------------|------------|
| Adjacency Matrix | O(V²) | O(V²) | O(1) | O(1) | O(1) |
| Adjacency List | O(V+E) | O(1) | O(1) | O(E) | O(V) |

> V = vertices, E = edges

---

## Sorting Algorithms

| Algorithm | Best | Average | Worst | Space | Stable? |
|-----------|------|---------|-------|-------|---------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Tim Sort | O(n) | O(n log n) | O(n log n) | O(n) | Yes |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes |
| Radix Sort | O(nk) | O(nk) | O(nk) | O(n+k) | Yes |
| Bucket Sort | O(n+k) | O(n+k) | O(n²) | O(n) | Yes |

> k = range of values. Tim Sort is used in Python, Java, V8.

---

## Searching Algorithms

| Algorithm | Best | Average | Worst | Space | Requirement |
|-----------|------|---------|-------|-------|-------------|
| Linear Search | O(1) | O(n) | O(n) | O(1) | None |
| Binary Search | O(1) | O(log n) | O(log n) | O(1) | Sorted array |
| Jump Search | O(1) | O(√n) | O(√n) | O(1) | Sorted array |
| Interpolation Search | O(1) | O(log log n) | O(n) | O(1) | Sorted, uniform |
| Exponential Search | O(1) | O(log n) | O(log n) | O(1) | Sorted array |

---

## Graph Algorithms

| Algorithm | Time | Space | Use Case |
|-----------|------|-------|---------|
| BFS | O(V+E) | O(V) | Shortest path (unweighted), level order |
| DFS | O(V+E) | O(V) | Cycle detection, topological sort |
| Dijkstra | O((V+E) log V) | O(V) | Shortest path (non-negative weights) |
| Bellman-Ford | O(VE) | O(V) | Shortest path (negative weights) |
| Floyd-Warshall | O(V³) | O(V²) | All-pairs shortest path |
| A* | O(E log V) | O(V) | Heuristic shortest path |
| Kruskal MST | O(E log E) | O(V) | Minimum spanning tree |
| Prim MST | O(E log V) | O(V) | Minimum spanning tree |
| Topological Sort | O(V+E) | O(V) | DAG ordering (dependencies) |

---

## Dynamic Programming & Recursion

| Problem | Naive | DP (memo/tabulation) |
|---------|-------|----------------------|
| Fibonacci | O(2ⁿ) | O(n) time, O(1) space (bottom-up) |
| Longest Common Subsequence | O(2ⁿ) | O(m×n) |
| 0/1 Knapsack | O(2ⁿ) | O(n×W) |
| Coin Change (min coins) | O(2ⁿ) | O(n×amount) |
| Longest Increasing Subsequence | O(2ⁿ) | O(n²) or O(n log n) |
| Edit Distance | O(3ⁿ) | O(m×n) |
| Matrix Chain Multiplication | O(2ⁿ) | O(n³) |

---

## Space Complexity — Common Patterns

| Pattern | Space |
|---------|-------|
| Simple variables | O(1) |
| Array of size n | O(n) |
| 2D matrix n×m | O(n×m) |
| Recursive call depth d | O(d) on call stack |
| DFS on graph | O(V) — call stack |
| BFS on graph | O(V) — queue |
| Merge sort auxiliary | O(n) |
| Quick sort call stack | O(log n) average, O(n) worst |
| Memoization table | O(subproblems) |

---

## Amortized Analysis

| Operation | Amortized | Why |
|-----------|-----------|-----|
| Dynamic array append | O(1) | Rare O(n) resize amortized over n ops |
| Stack push (with resizing array) | O(1) | Same as above |
| Splay tree operations | O(log n) | Self-adjusting balancing |
| Hash table insert (resize) | O(1) | Resize doubles capacity |

---

## Rules of Thumb

- **Drop constants:** O(2n) = O(n)
- **Drop lower-order terms:** O(n² + n) = O(n²)
- **Multiple loops in sequence:** O(n) + O(n) = O(n)
- **Nested loops:** O(n) inside O(n) = O(n²)
- **Divide and conquer (halving):** usually O(log n) or O(n log n)
- **Recursion without memoization on overlapping subproblems:** often exponential
- **Every element touches every other:** O(n²)
- **Sorting is your floor:** if you need to sort, you can't beat O(n log n) (comparison-based)

---

## Quick Decision Guide

```
Need fast lookup?           → Hash Map O(1)
Need sorted order?          → Balanced BST O(log n) / sorted array
Need min/max fast?          → Heap O(1) peek, O(log n) extract
Need prefix search?         → Trie O(m)
Need shortest path?         → BFS (unweighted) / Dijkstra (weighted)
Need ordering/dependencies? → Topological Sort
Need to search sorted array? → Binary Search O(log n)
Need stable sort?           → Merge Sort / Tim Sort
Need in-place sort?         → Quick Sort / Heap Sort
```
