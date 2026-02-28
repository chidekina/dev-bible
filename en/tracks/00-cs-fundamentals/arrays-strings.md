# Arrays & Strings

## 1. What & Why

Arrays and strings are the most fundamental data structures in programming. Nearly every algorithm problem involves them, and nearly every production system manipulates them constantly. Understanding their memory layout, performance characteristics, and idiomatic patterns is essential before tackling more complex data structures.

Arrays give you O(1) indexed access and cache-friendly iteration — they are the fastest structure for sequential processing. Strings, being immutable sequences of characters in most languages (including JavaScript/TypeScript), have performance traps that catch experienced engineers off guard.

---

## 2. Core Concepts

### Static vs Dynamic Arrays

A **static array** has a fixed size set at creation. Memory is allocated once — a contiguous block. Accessing element `i` computes `base_address + i × element_size` in O(1).

A **dynamic array** (JavaScript's `Array`, Java's `ArrayList`, C++'s `vector`) grows automatically. Internally it maintains a backing static array. When capacity is exceeded:
1. A new array is allocated — typically 2× the old capacity
2. All elements are copied — O(n)
3. The old array is garbage collected

This doubling strategy keeps the **amortized cost of push at O(1)**. Over n pushes, total copy work is 1 + 2 + 4 + ... + n = 2n = O(n). Spread over n operations: O(1) each.

### String Immutability

In JavaScript/TypeScript, strings are **immutable**. Every operation that appears to "modify" a string actually creates a new one. This has a critical implication: concatenating in a loop is O(n²).

```
"a" + "b" = creates new string "ab"          (copies 2 chars)
"ab" + "c" = creates new string "abc"        (copies 3 chars)
"abc" + "d" = creates new string "abcd"      (copies 4 chars)
...
Total copies after n concatenations: 1+2+3+...+n = n(n+1)/2 = O(n²)
```

Fix: collect parts in an array and join once at the end.

### Memory Layout

Arrays store elements in **contiguous memory**. This means:
- Index access is O(1) — arithmetic on the base pointer
- Sequential iteration is cache-friendly — the CPU prefetches the next elements
- Insertion/deletion at the middle is O(n) — all subsequent elements shift

The cache-friendliness of arrays is why a simple array loop often outperforms a linked list traversal in practice, even for operations that are theoretically the same complexity.

---

## 3. How It Works

```
Array in memory (each element occupies 8 bytes on a 64-bit system):

Index:   0       1       2       3       4
       [100]   [200]   [300]   [400]   [500]
Addr:  1000    1008    1016    1024    1032

Access arr[3]: address = 1000 + 3 × 8 = 1024 → O(1)
```

**Dynamic array resizing:**
```
Initial: capacity=4, size=4  →  [a, b, c, d]
push(e): capacity full → allocate size 8, copy [a,b,c,d], add e
Result:  capacity=8, size=5  →  [a, b, c, d, e, _, _, _]
```

**String concatenation internals:**
```
s = ""
s += "hello"   // allocates "hello"               — 5 chars
s += " world"  // allocates "hello world"          — 11 chars (copies 5+6)
s += "!"       // allocates "hello world!"         — 12 chars (copies 11+1)
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- String concatenation: O(n²) trap vs O(n) fix ---

// Bad: O(n²) — creates n intermediate strings
function buildStringBad(parts: string[]): string {
  let result = "";
  for (const part of parts) {
    result += part; // allocates a new string every iteration
  }
  return result;
}

// Good: O(n) — one allocation at the end
function buildStringGood(parts: string[]): string {
  return parts.join(""); // internally: allocate once, fill in place
}

// --- Two-pointer technique ---

// Check if a string is a palindrome: O(n) time, O(1) space
function isPalindrome(s: string): boolean {
  let left = 0;
  let right = s.length - 1;
  while (left < right) {
    if (s[left] !== s[right]) return false;
    left++;
    right--;
  }
  return true;
}

// Two-sum on a sorted array: O(n) time, O(1) space
function twoSumSorted(arr: number[], target: number): [number, number] | null {
  let left = 0;
  let right = arr.length - 1;
  while (left < right) {
    const sum = arr[left] + arr[right];
    if (sum === target) return [left, right];
    if (sum < target) left++;   // need more, advance left
    else right--;               // too much, retreat right
  }
  return null;
}

// Reverse a string in-place (on char array): O(n) time, O(1) space
function reverseString(chars: string[]): void {
  let left = 0, right = chars.length - 1;
  while (left < right) {
    [chars[left], chars[right]] = [chars[right], chars[left]];
    left++;
    right--;
  }
}

// --- Sliding window ---

// Fixed window: maximum sum of subarray of size k — O(n)
function maxSumSubarray(arr: number[], k: number): number {
  if (arr.length < k) throw new Error("Array smaller than k");

  // Build initial window
  let windowSum = arr.slice(0, k).reduce((a, b) => a + b, 0);
  let maxSum = windowSum;

  // Slide: subtract element leaving, add element entering
  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k];
    maxSum = Math.max(maxSum, windowSum);
  }
  return maxSum;
}

// Variable window: longest substring without repeating characters — O(n)
function lengthOfLongestSubstring(s: string): number {
  const charIndex = new Map<string, number>(); // char -> last seen index
  let maxLen = 0;
  let left = 0;

  for (let right = 0; right < s.length; right++) {
    const char = s[right];
    // If char was seen and is within the current window, shrink left past it
    if (charIndex.has(char) && charIndex.get(char)! >= left) {
      left = charIndex.get(char)! + 1;
    }
    charIndex.set(char, right);
    maxLen = Math.max(maxLen, right - left + 1);
  }
  return maxLen;
}

// --- Anagram check: O(n) using frequency map ---
function isAnagram(s: string, t: string): boolean {
  if (s.length !== t.length) return false;
  const freq = new Map<string, number>();
  for (const c of s) freq.set(c, (freq.get(c) ?? 0) + 1);
  for (const c of t) {
    if (!freq.has(c) || freq.get(c)! === 0) return false;
    freq.set(c, freq.get(c)! - 1);
  }
  return true;
}

// --- indexOf from scratch: O(n × m) naive, O(n) with KMP ---
// Naive version (Brute force): O(n × m) where n = haystack length, m = needle length
function indexOfNaive(haystack: string, needle: string): number {
  if (needle.length === 0) return 0;
  for (let i = 0; i <= haystack.length - needle.length; i++) {
    let match = true;
    for (let j = 0; j < needle.length; j++) {
      if (haystack[i + j] !== needle[j]) { match = false; break; }
    }
    if (match) return i;
  }
  return -1;
}

// --- First non-repeating character: O(n) ---
function firstNonRepeating(s: string): string | null {
  const freq = new Map<string, number>();
  for (const c of s) freq.set(c, (freq.get(c) ?? 0) + 1);
  for (const c of s) {
    if (freq.get(c) === 1) return c;
  }
  return null;
}

// --- Reverse words in place (on array of chars): O(n) ---
// "the sky is blue" → "blue is sky the"
function reverseWords(s: string): string {
  const chars = s.trim().split(/\s+/); // split on any whitespace
  chars.reverse();                      // reverse word array
  return chars.join(" ");
}

// --- Find duplicate number in array [1..n]: O(n) time, O(1) space ---
// Uses the mathematical property: sum of 1..n = n(n+1)/2
function findDuplicate(nums: number[]): number {
  const n = nums.length - 1;
  const expectedSum = (n * (n + 1)) / 2;
  const actualSum = nums.reduce((a, b) => a + b, 0);
  return actualSum - expectedSum;
}
// Note: only works when exactly one duplicate exists.
// For cycle detection approach (works with any one duplicate), see linked-lists.md

// --- Prefix sum array: O(n) build, O(1) range query ---
function buildPrefixSum(arr: number[]): number[] {
  const prefix = new Array(arr.length + 1).fill(0);
  for (let i = 0; i < arr.length; i++) {
    prefix[i + 1] = prefix[i] + arr[i];
  }
  return prefix;
}

// Range sum [left, right] inclusive: O(1)
function rangeSum(prefix: number[], left: number, right: number): number {
  return prefix[right + 1] - prefix[left];
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Off-by-one errors.** The most common bug in array code. When iterating with indices, always double-check boundary conditions. A loop `for (let i = 0; i <= arr.length; i++)` reads one past the end.

> ⚠️ **Mutating an array while iterating over it.** Removing elements with `splice` inside a `for` loop shifts subsequent elements and causes skipped or double-processed items.

```typescript
// Wrong: splice shifts elements, causing skips
const arr = [1, 2, 3, 4, 5];
for (let i = 0; i < arr.length; i++) {
  if (arr[i] % 2 === 0) arr.splice(i, 1); // shifts everything left — skips next element
}

// Correct: iterate backwards when using splice
for (let i = arr.length - 1; i >= 0; i--) {
  if (arr[i] % 2 === 0) arr.splice(i, 1);
}

// Better: use filter (creates new array)
const evensRemoved = arr.filter(x => x % 2 !== 0);
```

> ⚠️ **String concatenation in a loop.** Always use `Array.join()` or `Array.push()` + `join()` for building strings iteratively.

> ⚠️ **Forgetting that `typeof arr[i]` can be `undefined`.** Accessing an out-of-bounds index returns `undefined` in JavaScript, not an error. This causes silent bugs.

> ⚠️ **Using `==` instead of `===` for string comparison.** JavaScript's `==` does type coercion. `"5" == 5` is true. Always use `===`.

> ⚠️ **`Array.slice()` vs `Array.splice()`.** `slice` is non-mutating; `splice` mutates in place. Getting them confused is a classic bug.

---

## 6. When to Use / Not Use

**Use arrays when:**
- You need O(1) indexed access
- You iterate sequentially (cache-friendly)
- The size is known or bounded
- You need contiguous memory for performance

**Consider alternatives when:**
- Frequent insertions/deletions at the front or middle (use linked list, deque)
- Membership testing for large collections (use Set — O(1) vs O(n))
- Key-value lookup (use Map/Object — O(1) vs O(n))
- The data is sorted and you need efficient lookup (use BST or sorted array + binary search)

---

## 7. Real-World Scenario

A logging system builds log messages by concatenating fields:

```typescript
// Bad: O(n²) — used in production, caused latency spikes at scale
function formatLogBad(fields: string[]): string {
  let log = "";
  for (const field of fields) {
    log += field + "|"; // new string allocation every iteration
  }
  return log;
}

// Good: O(n)
function formatLogGood(fields: string[]): string {
  return fields.join("|");
}
```

With 1,000 log entries per second and 20 fields per entry, the bad version does ~200 string allocations per entry = 200,000 allocations/second, each copying progressively more data. The garbage collector buries under pressure. The good version: 1 allocation per entry = 1,000/second. Problem solved.

Another common scenario: finding duplicates in user-submitted data.

```typescript
// Bad: O(n²) — used when data is "small enough" until it isn't
function findDuplicateEmails(emails: string[]): string[] {
  const duplicates: string[] = [];
  for (let i = 0; i < emails.length; i++) {
    for (let j = i + 1; j < emails.length; j++) {
      if (emails[i] === emails[j] && !duplicates.includes(emails[i])) {
        duplicates.push(emails[i]);
      }
    }
  }
  return duplicates;
}

// Good: O(n)
function findDuplicateEmailsFast(emails: string[]): string[] {
  const seen = new Set<string>();
  const duplicates = new Set<string>();
  for (const email of emails) {
    if (seen.has(email)) duplicates.add(email);
    else seen.add(email);
  }
  return [...duplicates];
}
```

---

## 8. Interview Questions

**Q1: Why is string concatenation in a loop O(n²)?**
A: Strings are immutable in JavaScript. Each `+=` allocates a new string and copies all previous characters plus the new ones. After k concatenations, total characters copied = 1 + 2 + ... + k = k(k+1)/2 = O(k²). Fix: use `Array.push()` then `.join("")` — one allocation at the end.

**Q2: What is the time complexity of Array.push() in JavaScript?**
A: O(1) amortized. Most pushes are O(1). Occasionally the backing array doubles — O(n) — but spread over n pushes this averages to O(1) per push.

**Q3: Implement indexOf from scratch.**
A: See the `indexOfNaive` function above. Naive is O(n × m). Mention KMP algorithm for O(n + m) if asked for optimal.

**Q4: Find the first non-repeating character in a string.**
A: Two passes: first pass builds a frequency map (O(n)), second pass finds the first character with count 1 (O(n)). Total: O(n) time, O(1) space (alphabet is fixed size).

**Q5: What is the difference between Array.slice() and Array.splice()?**
A: `slice(start, end)` returns a new array without modifying the original — O(k) where k = slice length. `splice(start, deleteCount, ...items)` mutates the original array in place — O(n) because it shifts elements.

**Q6: How would you find the longest substring without repeating characters?**
A: Sliding window with a Map tracking last-seen index. Expand right pointer each step; when a repeat is found within the window, advance left pointer past the previous occurrence. O(n) time, O(min(n, alphabet)) space.

**Q7: What is a prefix sum array and when do you use it?**
A: A prefix sum array stores cumulative sums: `prefix[i] = sum of arr[0..i-1]`. Once built in O(n), any range sum `arr[left..right]` = `prefix[right+1] - prefix[left]` in O(1). Used for repeated range sum queries.

**Q8: What does cache-friendly mean for arrays?**
A: CPUs load memory in cache lines (~64 bytes). Array elements sit contiguously in memory, so loading one element loads neighboring elements into cache. Sequential array access becomes very fast because most accesses hit the cache. Linked lists, with nodes scattered in memory, have poor cache locality.

---

## 9. Exercises

**Exercise 1:** Reverse the words in a sentence in-place. "the sky is blue" → "blue is sky the". Do it in O(n) time.
*Hint: reverse the entire string, then reverse each word individually.*

**Exercise 2:** Find the duplicate number in an array of integers `[1..n]` where one number appears twice. Achieve O(n) time and O(1) space.
*Hint: use the sum formula OR use Floyd's cycle detection (see linked-lists.md).*

**Exercise 3:** Find the length of the longest substring without repeating characters.
Input: `"abcabcbb"` → Output: `3` ("abc")
*Hint: sliding window with a Map or Set to track characters in the current window.*

**Exercise 4:** Given an array and a target sum k, find all pairs of indices `[i, j]` where `arr[i] + arr[j] === k`. Return in O(n) time.
*Hint: complement approach — for each element x, check if `k - x` is in a Set.*

**Exercise 5:** Implement a function that checks if one string is an anagram of another in O(n) time.
*Hint: character frequency map — increment for one string, decrement for the other, check all zeroes.*

---

## 10. Further Reading

- [Two Pointers Pattern — NeetCode](https://neetcode.io/roadmap)
- [Sliding Window — LeetCode Explore](https://leetcode.com/explore/)
- CLRS Chapter 2 — Insertion Sort and Loop Invariants (foundational array reasoning)
- [MDN — String methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String)
- [KMP Algorithm explained](https://cp-algorithms.com/string/prefix-function.html) — O(n) string search
