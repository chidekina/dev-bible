# Searching Algorithms

## 1. What & Why

Searching — finding a target value or answering a yes/no question about data — is one of the most common operations in software. The choice of search algorithm can mean the difference between a feature that feels instant and one that brings down a server.

Linear search works on anything but is O(n). Binary search is O(log n) but requires sorted data. The real power of binary search extends beyond searching arrays: **binary search on the answer** applies binary search to a monotonic function, turning "find the minimum value x such that condition(x) is true" into an O(log(search_space)) problem. This pattern solves a wide variety of problems that don't look like array search problems at first glance.

---

## 2. Core Concepts

### Linear Search

Scan every element until found or the end is reached. O(n) time, O(1) space. Works on unsorted data, unindexed data, and any container. The baseline.

### Binary Search

Requires a **sorted array**. Halves the search space each iteration by comparing the target with the middle element. O(log n) time, O(1) space (iterative).

**Invariant:** The answer, if it exists, is always within [left, right]. Narrow this window by eliminating half at each step.

**Three templates:**
1. Exact match: find if target exists
2. Find first/leftmost position satisfying a condition
3. Find last/rightmost position satisfying a condition

Templates 2 and 3 are more versatile — they handle "find the leftmost element ≥ target" and similar queries.

### Binary Search on Answer

When you have a **monotonic condition** — a predicate that is false for small values and true for large values (or vice versa) — you can binary search on the answer space.

Pattern:
```
lo = minimum possible answer
hi = maximum possible answer
while lo < hi:
  mid = (lo + hi) / 2
  if condition(mid):
    hi = mid   (or lo = mid + 1 depending on direction)
  else:
    lo = mid + 1
return lo
```

### Ternary Search

Used for **unimodal functions** (functions that increase then decrease, or vice versa) to find the maximum or minimum. Narrows the search to 2/3 of the space each step. O(log n).

---

## 3. How It Works

```
Binary search for 7 in [1, 3, 5, 7, 9, 11, 13]:

Iteration 1:
left=0, right=6, mid=3
arr[3] = 7 → Found! Return 3.

Binary search for 6 (not present):
Iteration 1: left=0, right=6, mid=3, arr[3]=7 > 6 → right=2
Iteration 2: left=0, right=2, mid=1, arr[1]=3 < 6 → left=2
Iteration 3: left=2, right=2, mid=2, arr[2]=5 < 6 → left=3
left > right → return -1

Binary search on answer example:
"What is the minimum number of days to ship all packages?"
Packages: [1,2,3,4,5,6,7,8,9,10], capacity limit per day

lo = max(packages) = 10 (minimum possible: ship one at a time)
hi = sum(packages) = 55 (maximum: ship all in one day)
Binary search: can we do it in 30 days? Check. Can we do it in 15 days? ...
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- Linear Search: O(n) ---
function linearSearch<T>(arr: T[], target: T): number {
  for (let i = 0; i < arr.length; i++) {
    if (arr[i] === target) return i;
  }
  return -1;
}

// --- Binary Search (exact match): O(log n) ---
function binarySearch(arr: number[], target: number): number {
  let left = 0, right = arr.length - 1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2); // avoid integer overflow
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}

// --- Binary Search: Find first position of target ---
// Returns the leftmost index where arr[i] === target
function findFirst(arr: number[], target: number): number {
  let left = 0, right = arr.length - 1;
  let result = -1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] === target) {
      result = mid;
      right = mid - 1; // keep searching left
    } else if (arr[mid] < target) {
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }
  return result;
}

// --- Binary Search: Find last position of target ---
function findLast(arr: number[], target: number): number {
  let left = 0, right = arr.length - 1;
  let result = -1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] === target) {
      result = mid;
      left = mid + 1; // keep searching right
    } else if (arr[mid] < target) {
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }
  return result;
}

// --- Binary Search: Lower bound (first position >= target) ---
// Standard library equivalent of std::lower_bound
function lowerBound(arr: number[], target: number): number {
  let left = 0, right = arr.length;
  while (left < right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] < target) left = mid + 1;
    else right = mid;
  }
  return left; // first index where arr[i] >= target
}

// --- Binary Search: Upper bound (first position > target) ---
function upperBound(arr: number[], target: number): number {
  let left = 0, right = arr.length;
  while (left < right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] <= target) left = mid + 1;
    else right = mid;
  }
  return left; // first index where arr[i] > target
}

// --- Search in Rotated Sorted Array: O(log n) ---
// [4, 5, 6, 7, 0, 1, 2], target = 0
function searchRotated(nums: number[], target: number): number {
  let left = 0, right = nums.length - 1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (nums[mid] === target) return mid;

    // Determine which half is sorted
    if (nums[left] <= nums[mid]) {
      // Left half is sorted
      if (target >= nums[left] && target < nums[mid]) {
        right = mid - 1; // target is in left half
      } else {
        left = mid + 1;
      }
    } else {
      // Right half is sorted
      if (target > nums[mid] && target <= nums[right]) {
        left = mid + 1; // target is in right half
      } else {
        right = mid - 1;
      }
    }
  }
  return -1;
}

// --- Find Peak Element: O(log n) ---
// A peak element is any element greater than its neighbors
function findPeakElement(nums: number[]): number {
  let left = 0, right = nums.length - 1;

  while (left < right) {
    const mid = left + Math.floor((right - left) / 2);
    if (nums[mid] > nums[mid + 1]) {
      right = mid; // peak is on the left (including mid)
    } else {
      left = mid + 1; // peak is on the right
    }
  }
  return left;
}

// --- Find Minimum in Rotated Sorted Array: O(log n) ---
function findMin(nums: number[]): number {
  let left = 0, right = nums.length - 1;

  while (left < right) {
    const mid = left + Math.floor((right - left) / 2);
    if (nums[mid] > nums[right]) {
      left = mid + 1; // minimum is in the right half
    } else {
      right = mid; // minimum is in the left half (including mid)
    }
  }
  return nums[left];
}

// --- Square root (integer part): O(log n) binary search on answer ---
function mySqrt(x: number): number {
  if (x < 2) return x;
  let left = 1, right = Math.floor(x / 2);
  let result = 1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (mid * mid === x) return mid;
    if (mid * mid < x) {
      result = mid;
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }
  return result;
}

// --- Binary Search on Answer: Capacity to Ship Within D Days ---
// Find minimum capacity to ship all packages within D days
function shipWithinDays(weights: number[], days: number): number {
  let lo = Math.max(...weights); // minimum: must carry heaviest package
  let hi = weights.reduce((a, b) => a + b, 0); // maximum: ship all in one day

  while (lo < hi) {
    const mid = lo + Math.floor((hi - lo) / 2);
    if (canShip(weights, days, mid)) {
      hi = mid; // mid works, try smaller
    } else {
      lo = mid + 1; // mid doesn't work, need more capacity
    }
  }
  return lo;
}

function canShip(weights: number[], days: number, capacity: number): boolean {
  let currentLoad = 0, daysNeeded = 1;
  for (const weight of weights) {
    if (currentLoad + weight > capacity) {
      daysNeeded++;
      currentLoad = 0;
    }
    currentLoad += weight;
  }
  return daysNeeded <= days;
}

// --- Binary Search on Answer: Koko Eating Bananas ---
// Find minimum eating speed k to eat all bananas in h hours
function minEatingSpeed(piles: number[], h: number): number {
  let lo = 1, hi = Math.max(...piles);

  while (lo < hi) {
    const mid = lo + Math.floor((hi - lo) / 2);
    const hoursNeeded = piles.reduce((sum, pile) => sum + Math.ceil(pile / mid), 0);
    if (hoursNeeded <= h) hi = mid;
    else lo = mid + 1;
  }
  return lo;
}

// --- Search a 2D Matrix: O(log(m × n)) ---
// Matrix rows sorted, first element of each row > last element of previous row
function searchMatrix(matrix: number[][], target: number): boolean {
  const m = matrix.length, n = matrix[0].length;
  let left = 0, right = m * n - 1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    const val = matrix[Math.floor(mid / n)][mid % n]; // convert 1D index to 2D
    if (val === target) return true;
    if (val < target) left = mid + 1;
    else right = mid - 1;
  }
  return false;
}

// --- Ternary Search: find maximum of unimodal function ---
// Works on any function f where f increases then decreases
function ternarySearchMax(f: (x: number) => number, lo: number, hi: number, eps = 1e-9): number {
  while (hi - lo > eps) {
    const m1 = lo + (hi - lo) / 3;
    const m2 = hi - (hi - lo) / 3;
    if (f(m1) < f(m2)) lo = m1;
    else hi = m2;
  }
  return (lo + hi) / 2;
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Integer overflow in `mid` calculation.** `(left + right) / 2` can overflow when left and right are large integers. Use `left + Math.floor((right - left) / 2)` instead. In TypeScript/JavaScript this is rarely an issue due to floating-point numbers, but it's a critical bug in Java/C++.

> ⚠️ **Infinite loops from incorrect boundary updates.** When searching for the first occurrence, ensure you do `right = mid` not `right = mid - 1` when `arr[mid] === target`. Using the wrong boundary update causes either missing valid answers or infinite loops.

> ⚠️ **Forgetting the "sorted" precondition.** Binary search silently produces wrong results on unsorted arrays. If the data is not sorted, sort it first — O(n log n) sort + O(log n) search is still better than O(n) per query when many queries are made.

> ⚠️ **Binary search on answer: wrong initial bounds.** `lo` should be the minimum possible answer (exclusive lower bound is wrong), `hi` should be the maximum possible answer (inclusive). Getting this wrong skips the actual answer.

> ⚠️ **Confusing `while (lo <= right)` vs `while (lo < right)`.** For exact match: use `<=`. For finding the first position satisfying a condition: use `<` and return `lo` after the loop. Mixing these up is a common source of off-by-one bugs.

---

## 6. When to Use / Not Use

**Linear search when:**
- Array is unsorted and not worth sorting
- n is small
- Data is in a linked list (no random access)
- You need to find all occurrences, not just one

**Binary search when:**
- Data is sorted (or you can sort it once for many queries)
- You need O(log n) per query
- The problem has a monotonic condition (binary search on answer)

**Binary search on answer when:**
- You need to find the minimum/maximum value satisfying a condition
- The condition is monotonic (if it holds for x, it holds for all x' > x)
- Common in optimization problems: "find minimum capacity", "find minimum days", "find minimum speed"

---

## 7. Real-World Scenario

A deployment pipeline needs to find which commit introduced a regression. Git's `git bisect` uses binary search:

```typescript
// Model of "git bisect" — find the first commit that introduced a bug
function gitBisect(
  commits: string[],
  isBuggy: (commitHash: string) => boolean
): string | null {
  // commits[0] is oldest, commits[n-1] is newest
  // Assumption: all commits before the bad one pass, all after fail (monotonic)
  let left = 0, right = commits.length - 1;
  let firstBad = -1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (isBuggy(commits[mid])) {
      firstBad = mid;
      right = mid - 1; // look for earlier bad commit
    } else {
      left = mid + 1;
    }
  }
  return firstBad === -1 ? null : commits[firstBad];
}
// Turns O(n) test-each-commit into O(log n) — critical for repositories with thousands of commits
```

A database query optimizer uses binary search to check index pages:

```typescript
// Simplified B-tree leaf page search
function findInBTreePage(page: number[], key: number): number {
  // Each page is a sorted array of keys
  return lowerBound(page, key); // Find position to insert/look up key
}
```

---

## 8. Interview Questions

**Q1: Why is binary search O(log n)?**
A: Each iteration halves the search space. Starting with n elements: after 1 step we have n/2, after 2 steps n/4, ..., after k steps n/2^k. We stop when n/2^k = 1, so k = log₂(n).

**Q2: How do you binary search on a rotated sorted array?**
A: At each step, determine which half is sorted by comparing `nums[left]` with `nums[mid]`. If the left half is sorted (`nums[left] <= nums[mid]`), check if target falls in the sorted range. Otherwise the right half is sorted — check that range. Eliminate the half that cannot contain the target. O(log n).

**Q3: How do you find the minimum in a rotated sorted array?**
A: Binary search where you compare `nums[mid]` with `nums[right]`. If `nums[mid] > nums[right]`, the minimum is in the right half. Otherwise it's in the left half (including mid). O(log n).

**Q4: Explain binary search on the answer with an example.**
A: The pattern: instead of searching for a value in an array, search the answer space for the minimum value satisfying a condition. Example: "minimum ship capacity to ship all packages in D days." Define `canShip(capacity)` — true if packages can be shipped in D days with given capacity. This function is monotonic (if capacity works, larger capacity also works). Binary search on [max_weight, sum_weights] to find the smallest capacity that works.

**Q5: What is the integer square root problem and how do you solve it with binary search?**
A: Find the largest integer k such that k² ≤ x. Binary search on [1, x/2] (or [0, x]). For each mid, check if mid² ≤ x. Narrow the search space. O(log x).

---

## 9. Exercises

**Exercise 1:** Find the square root of an integer (floor) without using `Math.sqrt`. Achieve O(log n).

**Exercise 2:** Search a 2D matrix where each row is sorted and the first element of each row is greater than the last element of the previous row. Find if a target exists. Achieve O(log(m×n)).

**Exercise 3:** Find the kth smallest element in a BST using binary search (not inorder traversal).
*Hint: binary search on the value range, counting elements ≤ mid using the BST structure.*

**Exercise 4:** Given an array of integers, find if there exist two elements a and b such that a² + b² = target using binary search.
*Hint: sort the array. For each element a, binary search for `target - a²` among squares.*

**Exercise 5:** Find the minimum number of days to make at least m bouquets from a garden of flowers that bloom on different days. Each bouquet requires k adjacent flowers.
*Hint: binary search on the answer (number of days). For a given day count, check if enough adjacent bloomed flowers exist.*

---

## 10. Further Reading

- CLRS Chapter 2.3 — Designing Algorithms (introduces binary search formally)
- [Binary Search — patterns explained](https://labuladong.online/algo/essential-technique/binary-search-framework/)
- [Binary Search on Answer — examples](https://cp-algorithms.com/num_methods/binary_search.html)
- LeetCode: #704 Binary Search, #33 Search in Rotated Sorted Array, #153 Find Minimum in Rotated Sorted Array, #875 Koko Eating Bananas, #1011 Capacity to Ship Packages
