# Sorting Algorithms

## 1. What & Why

Sorting is one of the most fundamental operations in computer science. Sorted data enables binary search (O(log n) vs O(n)), makes duplicates adjacent (easy deduplication), and is a prerequisite for many algorithms (merge intervals, two-pointer technique, Dijkstra).

Understanding how different sorting algorithms work — their trade-offs in time, space, stability, and adaptability — helps you choose the right tool and reason about performance. It also gives insight into algorithm design patterns: divide and conquer (merge sort), in-place manipulation (quicksort, heap sort), and exploitation of data properties (counting sort, radix sort).

---

## 2. Core Concepts

### Stability

A sort is **stable** if it preserves the relative order of elements with equal keys. Example: sorting a list of users by age — if two users are both age 25, their relative order from the input is preserved.

Stable sorts: Merge sort, Timsort, Counting sort, Insertion sort.
Unstable sorts: Quicksort, Heap sort, Selection sort.

Stability matters when sorting by multiple criteria sequentially, or when the original order carries meaning.

### In-Place

A sort is **in-place** if it uses O(1) auxiliary space (ignoring the call stack). Quicksort and Heap sort are in-place. Merge sort requires O(n) extra space for merging.

### Comparison-Based Lower Bound

Any sorting algorithm that only compares elements requires at least O(n log n) comparisons in the worst case. This is a mathematical lower bound — you cannot do better with comparison sorting alone. Proof: there are n! possible orderings; each comparison eliminates half; you need at least log₂(n!) ≈ n log n comparisons.

### Linear Sorting

Algorithms like Counting Sort, Radix Sort, and Bucket Sort are O(n) or O(n + k) but they work by exploiting the **structure** of the data (integer keys with bounded range), not comparisons. They sidestep the O(n log n) lower bound because it only applies to comparison-based sorts.

---

## 3. Comparison Table

| Algorithm | Best | Average | Worst | Space | Stable | In-Place | Notes |
|---|---|---|---|---|---|---|---|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Yes | Only good with early-exit optimization |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No | Yes | Minimizes swaps (good for flash memory) |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Yes | Best for nearly sorted or small arrays |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | No | Guaranteed O(n log n), best for linked lists |
| Quicksort | O(n log n) | O(n log n) | O(n²) | O(log n) | No | Yes | Fast in practice; worst case with bad pivot |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No | Yes | Guaranteed O(n log n), worse cache than quicksort |
| Timsort | O(n) | O(n log n) | O(n log n) | O(n) | Yes | No | Used in JS/Python; optimized for real-world data |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes | No | Only for integer keys in range [0, k] |
| Radix Sort | O(d(n+k)) | O(d(n+k)) | O(d(n+k)) | O(n+k) | Yes | No | d = digits, k = radix; for integers/strings |
| Bucket Sort | O(n+k) | O(n+k) | O(n²) | O(n+k) | Yes | No | For uniformly distributed floating points |

---

## 4. Code Examples (TypeScript)

```typescript
// --- Bubble Sort: O(n²) worst, O(n) best with early exit ---
function bubbleSort(arr: number[]): number[] {
  const a = [...arr];
  for (let i = 0; i < a.length; i++) {
    let swapped = false;
    for (let j = 0; j < a.length - i - 1; j++) {
      if (a[j] > a[j + 1]) {
        [a[j], a[j + 1]] = [a[j + 1], a[j]];
        swapped = true;
      }
    }
    if (!swapped) break; // already sorted — O(n) best case
  }
  return a;
}

// --- Selection Sort: O(n²) always, minimizes swaps ---
function selectionSort(arr: number[]): number[] {
  const a = [...arr];
  for (let i = 0; i < a.length; i++) {
    let minIdx = i;
    for (let j = i + 1; j < a.length; j++) {
      if (a[j] < a[minIdx]) minIdx = j;
    }
    if (minIdx !== i) [a[i], a[minIdx]] = [a[minIdx], a[i]];
  }
  return a;
}

// --- Insertion Sort: O(n²) worst, O(n) best (nearly sorted) ---
// Best choice for small arrays (n < 20) or nearly sorted input
function insertionSort(arr: number[]): number[] {
  const a = [...arr];
  for (let i = 1; i < a.length; i++) {
    const key = a[i];
    let j = i - 1;
    // Shift elements right until we find the correct position for key
    while (j >= 0 && a[j] > key) {
      a[j + 1] = a[j];
      j--;
    }
    a[j + 1] = key;
  }
  return a;
}

// --- Merge Sort: O(n log n) always, stable, O(n) space ---
function mergeSort(arr: number[]): number[] {
  if (arr.length <= 1) return arr;

  const mid = Math.floor(arr.length / 2);
  const left = mergeSort(arr.slice(0, mid));
  const right = mergeSort(arr.slice(mid));

  return merge(left, right);
}

function merge(left: number[], right: number[]): number[] {
  const result: number[] = [];
  let i = 0, j = 0;

  while (i < left.length && j < right.length) {
    if (left[i] <= right[j]) { // <= preserves stability
      result.push(left[i++]);
    } else {
      result.push(right[j++]);
    }
  }
  // Append remaining elements
  while (i < left.length) result.push(left[i++]);
  while (j < right.length) result.push(right[j++]);
  return result;
}

// Merge sort: bottom-up iterative version (O(1) stack space)
function mergeSortIterative(arr: number[]): number[] {
  const a = [...arr];
  const n = a.length;
  // Merge subarrays of increasing size: 1, 2, 4, 8, ...
  for (let size = 1; size < n; size *= 2) {
    for (let start = 0; start < n; start += 2 * size) {
      const mid = Math.min(start + size, n);
      const end = Math.min(start + 2 * size, n);
      const merged = merge(a.slice(start, mid), a.slice(mid, end));
      a.splice(start, end - start, ...merged);
    }
  }
  return a;
}

// --- Quicksort: O(n log n) average, O(n²) worst, in-place ---
function quicksort(arr: number[], low = 0, high = arr.length - 1): number[] {
  if (low < high) {
    const pivotIdx = partition(arr, low, high);
    quicksort(arr, low, pivotIdx - 1);
    quicksort(arr, pivotIdx + 1, high);
  }
  return arr;
}

// Lomuto partition scheme
function partition(arr: number[], low: number, high: number): number {
  // Randomize pivot to avoid O(n²) worst case on sorted input
  const randomIdx = low + Math.floor(Math.random() * (high - low + 1));
  [arr[randomIdx], arr[high]] = [arr[high], arr[randomIdx]];

  const pivot = arr[high];
  let i = low - 1;

  for (let j = low; j < high; j++) {
    if (arr[j] <= pivot) {
      i++;
      [arr[i], arr[j]] = [arr[j], arr[i]];
    }
  }
  [arr[i + 1], arr[high]] = [arr[high], arr[i + 1]];
  return i + 1;
}

// --- Quicksort: 3-way partition (handles duplicates efficiently) ---
// Useful when many duplicate elements — reduces O(n log n) to O(n) for all-same arrays
function quicksort3Way(arr: number[], low = 0, high = arr.length - 1): void {
  if (low >= high) return;
  const pivot = arr[low];
  let lt = low, gt = high, i = low + 1;

  while (i <= gt) {
    if (arr[i] < pivot) { [arr[lt], arr[i]] = [arr[i], arr[lt]]; lt++; i++; }
    else if (arr[i] > pivot) { [arr[i], arr[gt]] = [arr[gt], arr[i]]; gt--; }
    else i++;
  }
  // arr[low..lt-1] < pivot, arr[lt..gt] == pivot, arr[gt+1..high] > pivot
  quicksort3Way(arr, low, lt - 1);
  quicksort3Way(arr, gt + 1, high);
}

// --- Counting Sort: O(n + k) where k = range of values ---
// Only works for non-negative integers with bounded range
function countingSort(arr: number[], maxVal: number): number[] {
  const count = new Array(maxVal + 1).fill(0);
  for (const n of arr) count[n]++;

  // Accumulate counts (for stable sort — maintains relative order)
  for (let i = 1; i <= maxVal; i++) count[i] += count[i - 1];

  const output = new Array(arr.length);
  // Build output right-to-left for stability
  for (let i = arr.length - 1; i >= 0; i--) {
    output[--count[arr[i]]] = arr[i];
  }
  return output;
}

// --- Radix Sort: O(d × n) where d = number of digits ---
// Sorts integers by processing one digit at a time (LSD to MSD)
function radixSort(arr: number[]): number[] {
  const maxVal = Math.max(...arr);
  let exp = 1;
  let a = [...arr];

  while (Math.floor(maxVal / exp) > 0) {
    a = countingSortByDigit(a, exp);
    exp *= 10;
  }
  return a;
}

function countingSortByDigit(arr: number[], exp: number): number[] {
  const count = new Array(10).fill(0);
  for (const n of arr) count[Math.floor(n / exp) % 10]++;
  for (let i = 1; i < 10; i++) count[i] += count[i - 1];

  const output = new Array(arr.length);
  for (let i = arr.length - 1; i >= 0; i--) {
    const digit = Math.floor(arr[i] / exp) % 10;
    output[--count[digit]] = arr[i];
  }
  return output;
}

// --- Dutch National Flag: Sort 0s, 1s, 2s in one pass ---
// O(n) time, O(1) space — classic three-way partition
function dutchNationalFlag(arr: number[]): void {
  let low = 0, mid = 0, high = arr.length - 1;

  while (mid <= high) {
    if (arr[mid] === 0) {
      [arr[low], arr[mid]] = [arr[mid], arr[low]];
      low++; mid++;
    } else if (arr[mid] === 1) {
      mid++;
    } else { // arr[mid] === 2
      [arr[mid], arr[high]] = [arr[high], arr[mid]];
      high--; // don't increment mid — new arr[mid] is unknown
    }
  }
}

// --- Sort Linked List: Merge Sort — O(n log n), O(log n) space ---
// Merge sort is preferred for linked lists because:
// 1. Random access is O(n) — quicksort's pivot selection is awkward
// 2. Merge sort's merge step is O(1) space for linked lists (pointer manipulation)
class ListNode {
  constructor(public val: number, public next: ListNode | null = null) {}
}

function sortList(head: ListNode | null): ListNode | null {
  if (head === null || head.next === null) return head;

  // Find middle (slow/fast pointer)
  let slow = head, fast: ListNode | null = head.next;
  while (fast !== null && fast.next !== null) {
    slow = slow.next!;
    fast = fast.next.next;
  }

  const mid = slow.next;
  slow.next = null; // split

  const left = sortList(head);
  const right = sortList(mid);
  return mergeLinked(left, right);
}

function mergeLinked(l1: ListNode | null, l2: ListNode | null): ListNode | null {
  const dummy = new ListNode(0);
  let cur = dummy;
  while (l1 !== null && l2 !== null) {
    if (l1.val <= l2.val) { cur.next = l1; l1 = l1.next; }
    else { cur.next = l2; l2 = l2.next; }
    cur = cur.next;
  }
  cur.next = l1 ?? l2;
  return dummy.next;
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Quicksort on sorted input without random pivot.** Using the last or first element as pivot on an already-sorted array gives O(n²) performance. Always randomize the pivot selection.

> ⚠️ **Assuming JavaScript's Array.sort() is unstable.** Since V8 7.0 (Node.js 11+), `Array.sort()` uses Timsort and is stable. For older environments or cross-browser safety, be explicit if stability matters.

> ⚠️ **Using Array.sort() without a comparator for numbers.** `[10, 2, 1].sort()` → `[1, 10, 2]` (lexicographic). Always pass a comparator: `.sort((a, b) => a - b)`.

> ⚠️ **Merge sort on linked lists vs arrays.** For arrays, merge sort's merge step requires O(n) extra space. For linked lists, the merge is in-place (O(1) space) — just rewire pointers. This is why merge sort is preferred for linked lists.

> ⚠️ **Counting sort with large k.** If values range from 0 to 10^9, you cannot allocate an array of 10^9. Use radix sort or comparison sort instead.

---

## 6. When to Use / Not Use

**Insertion sort when:**
- Array is small (n < 20) — low overhead, cache-friendly
- Array is nearly sorted — O(n) best case

**Merge sort when:**
- Stability is required
- Sorting a linked list (in-place merge, no random access needed)
- External sorting (data doesn't fit in memory — merge chunks from disk)

**Quicksort when:**
- Average performance matters, not worst case
- In-place sorting preferred (memory constrained)
- Data has many distinct values (avoid worst case with 3-way partitioning for duplicates)

**Heap sort when:**
- Guaranteed O(n log n) worst case with O(1) space
- Not concerned with cache performance

**Counting/Radix sort when:**
- Integer keys with known bounded range
- Need linear time sort

---

## 7. Real-World Scenario

Timsort — the algorithm used in JavaScript's `Array.sort()` and Python's `sorted()` — is a hybrid that combines merge sort and insertion sort. It exploits naturally occurring runs (already sorted subsequences) in real-world data:

1. Scan the array for "natural runs" (already sorted or reverse-sorted subsequences)
2. Extend short runs to a minimum size using insertion sort (insertion sort is fast on small arrays)
3. Merge runs using merge sort's merge step

Real-world data is rarely random — user records often have portions already sorted by date, product IDs are often nearly monotonic. Timsort's O(n) best case for already-sorted input is not just theoretical — it happens constantly in practice.

---

## 8. Interview Questions

**Q1: Why is merge sort preferred for linked lists?**
A: Quicksort needs random access for efficient pivot selection, and linked list access is O(n). Merge sort only needs sequential access. Also, the merge step for linked lists is O(1) space (just rewire pointers), whereas merging arrays requires O(n) extra space.

**Q2: When would you choose quicksort over merge sort?**
A: When you need in-place sorting and average-case performance matters more than worst-case guarantees. Quicksort has better cache performance in practice (sequential memory access pattern) and lower constant factors. With random pivot, the O(n²) worst case is astronomically unlikely.

**Q3: Why is counting sort O(n) not a contradiction of the O(n log n) lower bound?**
A: The O(n log n) lower bound applies specifically to comparison-based sorting — algorithms that only use element comparisons. Counting sort doesn't compare elements; it exploits the integer key structure to directly compute positions. It's a different model of computation, so the lower bound doesn't apply.

**Q4: What is Timsort?**
A: A hybrid sorting algorithm used in Python and JavaScript. Combines insertion sort (for small runs) with merge sort (for merging runs). Detects naturally sorted subsequences in the data, making it O(n) for already-sorted input and O(n log n) worst case. Stable.

**Q5: Sort an array containing only 0s, 1s, and 2s in one pass.**
A: Dutch National Flag algorithm by Dijkstra. Three pointers: low, mid, high. Invariant: everything before low is 0, between low and mid is 1, after high is 2. O(n) time, O(1) space.

**Q6: What makes a sorting algorithm stable and why does it matter?**
A: Stable sort preserves the original relative order of equal elements. Matters when sorting on multiple keys: first sort by secondary key (stable sort), then sort by primary key — the secondary order is preserved within equal primary keys. Unstable sort would scramble the secondary order.

---

## 9. Exercises

**Exercise 1:** Sort an array of 0s, 1s, and 2s in a single pass without using a sorting library. The Dutch National Flag problem.
*Hint: three pointers — low (boundary of 0s), mid (current), high (boundary of 2s).*

**Exercise 2:** Find the kth largest element in an unsorted array in O(n) average time.
*Hint: Quickselect — partial quicksort that only recurses on the relevant partition.*

**Exercise 3:** Sort a linked list in O(n log n) time and O(log n) space.
*Hint: merge sort — find middle with fast/slow pointer, split, sort each half, merge.*

**Exercise 4:** Implement counting sort for an array of objects sorted by a numeric field (e.g., age). Ensure the sort is stable.

**Exercise 5:** Given an array of integers, find the smallest positive integer gap between any two elements. Do it in O(n) using bucket/counting sort concepts.
*Hint: Pigeonhole principle — with n elements in range [min, max], the max gap is at least (max - min) / (n - 1). Use buckets of that size.*

---

## 10. Further Reading

- CLRS Chapter 6 (Heapsort), Chapter 7 (Quicksort), Chapter 8 (Linear Sorting)
- [Timsort — Tim Peters' original description](https://svn.python.org/projects/python/trunk/Objects/listsort.txt)
- [Sorting visualizer](https://visualgo.net/en/sorting)
- LeetCode: #912 Sort an Array, #75 Sort Colors (Dutch National Flag), #148 Sort List, #215 Kth Largest Element (Quickselect)
