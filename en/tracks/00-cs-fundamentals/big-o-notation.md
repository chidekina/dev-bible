# Big O Notation

## 1. What & Why

Big O notation describes how an algorithm's time or space requirements grow as the input size grows. It is the language engineers use to compare algorithms and predict performance at scale. An algorithm that runs fine on 1,000 rows may grind to a halt on 10 million — Big O helps you spot this before it becomes a production incident.

Without Big O, you are guessing. With it, you can reason: "this endpoint does a nested loop over user records — at 100k users that is 10 billion operations — we need to fix this before launch." That is the kind of thinking that separates senior engineers from junior ones.

Big O is about **asymptotic behavior**: what happens to cost as n approaches infinity. Constant factors and lower-order terms are dropped because they become irrelevant at scale.

---

## 2. Core Concepts

### Complexity Classes

| Notation | Name | Example |
|---|---|---|
| O(1) | Constant | Array index access, hashmap lookup |
| O(log n) | Logarithmic | Binary search, balanced BST operations |
| O(n) | Linear | Linear scan, single loop |
| O(n log n) | Linearithmic | Merge sort, efficient sorting |
| O(n²) | Quadratic | Nested loops over the same data |
| O(2ⁿ) | Exponential | Recursive Fibonacci, subset generation |
| O(n!) | Factorial | Permutations, brute-force TSP |

### Simplification Rules

- **Drop constants:** O(3n) → O(n)
- **Drop non-dominant terms:** O(n² + n) → O(n²)
- **Sequential steps add:** O(a) then O(b) = O(a + b)
- **Nested steps multiply:** O(a) inside O(b) loop = O(a × b)

### Three Cases

- **Best case (Ω — Omega):** minimum operations for the most favorable input. Rarely useful in practice.
- **Average case (Θ — Theta):** expected operations over all inputs. Most realistic for planning.
- **Worst case (O — Big O):** maximum operations. The guarantee. What we quote by default.

### Amortized Analysis

The average cost per operation over a **sequence** of operations. Dynamic array `push` is O(1) amortized: most pushes are O(1), but occasionally the backing array doubles in size (O(n)), averaging out to O(1) per operation over n pushes.

### Space Complexity

Memory consumed by the algorithm — including auxiliary variables, data structures, and the call stack for recursive calls. O(1) space means constant memory regardless of input size. O(n) means memory grows linearly.

---

## 3. How It Works

```
Complexity growth comparison (n = 1,000):

O(1)       →            1 operation
O(log n)   →           10 operations
O(n)       →        1,000 operations
O(n log n) →       10,000 operations
O(n²)      →    1,000,000 operations
O(2ⁿ)      → 2^1000 operations (computationally impossible)
```

**The dominant term rule:** As n grows large, the fastest-growing term overwhelms all others. O(n³ + 1000n² + n) behaves like O(n³) for large n.

**Why constants don't matter in Big O:** If algorithm A takes 2n steps and algorithm B takes 1000n steps, A is faster by a constant factor. But O(n) vs O(n²) is a qualitative difference: at n = 1,000,000, n² is 10¹² operations while n is 10⁶ — a million times more work.

---

## 4. Code Examples (TypeScript)

```typescript
// O(1) — Constant time
// Execution time does not depend on input size
function getFirst<T>(arr: T[]): T {
  return arr[0];
}

function isEven(n: number): boolean {
  return n % 2 === 0; // always one arithmetic operation
}

// O(log n) — Logarithmic time
// Binary search: halves the search space each iteration
function binarySearch(arr: number[], target: number): number {
  let left = 0;
  let right = arr.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}

// O(n) — Linear time
// Visit each element exactly once
function findMax(arr: number[]): number {
  let max = arr[0];
  for (const n of arr) {
    if (n > max) max = n;
  }
  return max;
}

// O(n²) — Quadratic time: nested loops over the same data
function hasDuplicate(arr: number[]): boolean {
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j]) return true;
    }
  }
  return false;
}

// O(n) — Same problem solved in linear time using a Set
function hasDuplicateFast(arr: number[]): boolean {
  const seen = new Set<number>();
  for (const n of arr) {
    if (seen.has(n)) return true;
    seen.add(n);
  }
  return false;
}

// O(n log n) — Linearithmic time: merge sort
function mergeSort(arr: number[]): number[] {
  if (arr.length <= 1) return arr;

  const mid = Math.floor(arr.length / 2);
  const left = mergeSort(arr.slice(0, mid));  // O(log n) levels deep
  const right = mergeSort(arr.slice(mid));

  return merge(left, right); // O(n) work at each level
}

function merge(left: number[], right: number[]): number[] {
  const result: number[] = [];
  let i = 0, j = 0;
  while (i < left.length && j < right.length) {
    if (left[i] <= right[j]) result.push(left[i++]);
    else result.push(right[j++]);
  }
  return result.concat(left.slice(i)).concat(right.slice(j));
}

// O(2ⁿ) — Exponential time: naive recursive Fibonacci
function fib(n: number): number {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2); // two recursive calls per invocation
}

// O(n) — Memoized Fibonacci fixes exponential to linear
function fibMemo(n: number, memo = new Map<number, number>()): number {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n)!;
  const result = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  memo.set(n, result);
  return result;
}

// --- Space complexity examples ---

// O(1) space — in-place reversal
function reverseInPlace(arr: number[]): void {
  let left = 0, right = arr.length - 1;
  while (left < right) {
    [arr[left], arr[right]] = [arr[right], arr[left]];
    left++;
    right--;
  }
}

// O(n) space — allocates a new array
function reverseNew(arr: number[]): number[] {
  const result: number[] = [];
  for (let i = arr.length - 1; i >= 0; i--) {
    result.push(arr[i]); // allocates n elements
  }
  return result;
}

// O(log n) space — recursive binary search uses call stack
function binarySearchRecursive(
  arr: number[],
  target: number,
  left = 0,
  right = arr.length - 1
): number {
  if (left > right) return -1;
  const mid = Math.floor((left + right) / 2);
  if (arr[mid] === target) return mid;
  if (arr[mid] < target) return binarySearchRecursive(arr, target, mid + 1, right);
  return binarySearchRecursive(arr, target, left, mid - 1);
}

// --- Amortized O(1) push example ---
class DynamicArray<T> {
  private data: T[] = [];
  private capacity = 1;
  private size = 0;

  push(item: T): void {
    if (this.size === this.capacity) {
      // O(n) resize — but happens only O(log n) times during n pushes
      const newData = new Array<T>(this.capacity * 2);
      for (let i = 0; i < this.size; i++) newData[i] = this.data[i];
      this.data = newData;
      this.capacity *= 2;
    }
    this.data[this.size++] = item;
  }
  // Total resize work over n pushes: 1+2+4+...+n = 2n = O(n)
  // Amortized cost per push: O(n)/n = O(1)
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Confusing average vs worst case.** Quicksort is O(n log n) average but O(n²) worst case on already-sorted input with a bad pivot choice. Always specify which case you are quoting.

> ⚠️ **Ignoring space complexity.** Recursion uses O(depth) stack space. A recursive DFS on a graph with 100,000 nodes can cause a stack overflow before completing.

> ⚠️ **Hidden O(n) inside loops.** `Array.includes()`, `Array.indexOf()`, and `String.includes()` are all O(n). Calling them inside a loop produces O(n²) without it being obvious.

```typescript
// Looks O(n), actually O(n²):
function findCommon(a: number[], b: number[]): number[] {
  return a.filter(x => b.includes(x)); // b.includes() is O(n) per call
}

// Fixed: O(n)
function findCommonFast(a: number[], b: number[]): number[] {
  const setB = new Set(b);
  return a.filter(x => setB.has(x)); // Set.has() is O(1)
}
```

> ⚠️ **Array.shift() and Array.unshift() are O(n).** They must move every element. Use a pointer or a proper deque for O(1) front operations.

> ⚠️ **Object spread `{ ...obj }` in tight loops.** Spread is O(k) where k = number of keys. Inside an O(n) loop, this is O(n × k).

> ⚠️ **Premature optimization.** O(n²) on n = 100 runs in microseconds. Profile first. Fix what is measurably slow. Don't sacrifice readability for micro-optimizations that don't matter.

---

## 6. When to Use / Not Use

**Pay close attention to Big O when:**
- Data size can exceed 10,000 and the operation runs per-request
- Building a library or service consumed by many callers
- Diagnosing slow queries, timeouts, or CPU spikes in production
- In an interview — Big O analysis is expected for every proposed solution

**Don't over-optimize when:**
- n is small and bounded (config parsing, small dropdown options, a list of 10 items)
- The bottleneck is I/O — network, database, or disk latency dwarfs CPU time
- A simpler O(n²) solution is far more readable and n will never exceed a few hundred
- The constant factor matters more than the asymptotic class at the sizes you actually run

---

## 7. Real-World Scenario

A SaaS analytics platform has a "find users matching all selected tags" feature. The first version:

```typescript
// O(users × tags × avg_user_tags) per request
function findMatchingUsers(users: User[], tags: string[]): User[] {
  return users.filter(user =>
    tags.every(tag => user.tags.includes(tag)) // O(avg_user_tags) per tag
  );
}
```

With 50,000 users, 20 selected tags, each user having ~10 tags: 50,000 × 20 × 10 = 10,000,000 operations per request. At 100 req/s that is 1 billion operations per second. The server cannot cope.

Fix: pre-build an inverted index (tag → Set of user IDs) once when data loads:

```typescript
type UserIndex = Map<string, Set<string>>;

function buildIndex(users: User[]): UserIndex {
  const index: UserIndex = new Map();
  for (const user of users) {
    for (const tag of user.tags) {
      if (!index.has(tag)) index.set(tag, new Set());
      index.get(tag)!.add(user.id);
    }
  }
  return index; // O(users × avg_tags) — done once
}

function findMatchingFast(
  index: UserIndex,
  allUsers: Map<string, User>,
  tags: string[]
): User[] {
  if (tags.length === 0) return [];

  // Intersect sets, starting from the smallest for efficiency
  const sets = tags
    .map(tag => index.get(tag) ?? new Set<string>())
    .sort((a, b) => a.size - b.size);

  let result = new Set(sets[0]);
  for (let i = 1; i < sets.length; i++) {
    result = new Set([...result].filter(id => sets[i].has(id)));
  }

  return [...result].map(id => allUsers.get(id)!);
}
// Per request: O(tags × intersection_size) — typically O(1) for small tag counts
```

---

## 8. Interview Questions

**Q1: What is Big O notation and why do we use it?**
A: Big O describes the upper bound on an algorithm's growth in time or space relative to input size, dropping constants and lower-order terms. We use it to compare algorithms and predict performance at scale without benchmarking every input.

**Q2: What is the time complexity of accessing an array element by index?**
A: O(1). Arrays store elements in contiguous memory. The address is computed as `base + index × element_size` — a single arithmetic operation regardless of array size.

**Q3: Is O(2n) different from O(n)?**
A: No. Constants are dropped in Big O. Both describe linear growth — the ratio between 2n and n is constant regardless of n.

**Q4: What is the time complexity of JavaScript's Array.sort()?**
A: O(n log n). V8 uses Timsort — a hybrid of merge sort and insertion sort — which guarantees O(n log n) worst case and is stable since V8 7.0.

**Q5: A function has an O(n) loop then calls an O(n) helper. What is the total?**
A: O(n) + O(n) = O(2n) = O(n). Sequential independent operations add, then constants drop.

**Q6: What is the space complexity of a recursive Fibonacci function?**
A: O(n) space. The call stack grows to depth n before unwinding. Even though the call tree has O(2ⁿ) nodes, the maximum stack depth at any moment is n.

**Q7: When is O(n²) acceptable?**
A: When n is small (typically < 1,000) and the operation runs infrequently. Correctness first, profiled optimization second.

**Q8: What does "amortized O(1)" mean for dynamic array push?**
A: Most pushes are O(1). Occasionally the backing array doubles in size — O(n). Over n push operations, total resize work is 1 + 2 + 4 + ... + n = 2n = O(n), so the average per operation is O(n)/n = O(1).

**Q9: What is the complexity of Set.has() and Map.get()?**
A: O(1) average. Both use hash tables. Worst case is O(n) due to hash collisions, but this is extremely rare in practice.

**Q10: At what n does O(n log n) vs O(n²) matter?**
A: At n = 1,000: n log n ≈ 10,000 vs n² = 1,000,000 — 100× slower. At n = 10: negligible. The practical crossover is around n = 500–1,000 depending on constants.

---

## 9. Exercises

**Exercise 1:** Annotate the complexity of each line and the overall function:
```typescript
function mystery(arr: number[]): number {
  let result = 0;
  for (let i = 0; i < arr.length; i++) {       // outer loop: ?
    for (let j = 0; j < arr.length; j++) {     // inner loop: ?
      result += arr[i] * arr[j];               // body: ?
    }
  }
  return result;
}
```
*Hint: if arr.length = n, the inner body runs n × n times.*

**Exercise 2:** The function below is O(n²). Rewrite it in O(n) using a Set:
```typescript
function hasDuplicate(arr: number[]): boolean {
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j]) return true;
    }
  }
  return false;
}
```

**Exercise 3:** Given a sorted array, find if any two elements sum to a target. Achieve O(n) time and O(1) space.
*Hint: two-pointer — one at the start, one at the end, move based on whether the current sum is too high or too low.*

**Exercise 4:** Determine both the time AND space complexity:
```typescript
function fib(n: number): number {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}
```
*Hint: draw the call tree for fib(5). Count total nodes (time). Trace the maximum call stack depth at any moment (space).*

**Exercise 5:** You call `Array.includes(x)` inside a `for` loop iterating over the same array length. What is the overall complexity? Rewrite to achieve O(n).

---

## 10. Further Reading

- [Big O Cheat Sheet](https://www.bigocheatsheet.com/) — quick visual reference for all common data structures
- CLRS — Introduction to Algorithms, Chapter 3 (Growth of Functions)
- [NeetCode — Big O Crash Course](https://neetcode.io/) — animated, visual explanations
- [MDN — Array method complexities](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)
- [Timsort paper by Tim Peters](https://svn.python.org/projects/python/trunk/Objects/listsort.txt) — how V8's sort actually works
