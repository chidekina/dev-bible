# Common Patterns

## 1. What & Why

Algorithmic patterns are reusable problem-solving templates. Once you recognize that a problem fits a pattern, you can apply a known solution skeleton instead of solving from scratch. This is how experienced engineers and competitive programmers solve unfamiliar problems quickly.

Patterns matter because interview problems are rarely novel — they are variations on about a dozen fundamental templates. Knowing the patterns, the recognition cues, and the code skeleton lets you solve 80% of LeetCode medium problems with moderate effort.

This file covers the 10 most important patterns with full TypeScript implementations and three example problems each.

---

## 2. Core Concepts

Each pattern section follows this structure:
- **What it is** — a one-sentence definition
- **How to recognize it** — the cues in the problem statement
- **When to use it** — the structural condition that makes the pattern applicable
- **Template code** — a reusable TypeScript implementation
- **Example problems** — three problems that fit the pattern

---

## 3. Patterns

---

### Pattern 1: Sliding Window (Fixed Size)

**What it is:** Maintain a window of exactly `k` elements over an array. Slide it one position at a time by adding the new right element and removing the leftmost element.

**How to recognize it:**
- "Subarray/substring of size k"
- "Max/min/average over all windows of size k"

**When to use it:** The operation over the window can be maintained incrementally (add right element, remove left element) rather than recomputed from scratch.

**Template:**

```typescript
function slidingWindowFixed<T>(arr: T[], k: number): T[][] {
  const windows: T[][] = [];
  if (arr.length < k) return windows;

  // Build the first window
  let windowSum = 0;
  for (let i = 0; i < k; i++) {
    windowSum += arr[i] as unknown as number;
  }
  windows.push(arr.slice(0, k));

  // Slide: remove leftmost, add next right
  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] as unknown as number;
    windowSum -= arr[i - k] as unknown as number;
    windows.push(arr.slice(i - k + 1, i + 1));
  }

  return windows;
}

// Problem: Maximum sum subarray of size k — O(n) time, O(1) space
function maxSumSubarray(arr: number[], k: number): number {
  if (arr.length < k) throw new Error("Array smaller than k");

  let windowSum = 0;
  for (let i = 0; i < k; i++) windowSum += arr[i];

  let maxSum = windowSum;

  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k]; // slide
    maxSum = Math.max(maxSum, windowSum);
  }

  return maxSum;
}

// Example: [2,1,5,1,3,2], k=3 → 9 (5+1+3)
console.log(maxSumSubarray([2, 1, 5, 1, 3, 2], 3)); // 9

// Problem: Average of subarrays of size k
function avgOfSubarrays(arr: number[], k: number): number[] {
  const result: number[] = [];
  let windowSum = 0;

  for (let i = 0; i < k; i++) windowSum += arr[i];
  result.push(windowSum / k);

  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k];
    result.push(windowSum / k);
  }

  return result;
}

// Problem: Count distinct elements in every window of size k
function countDistinctInWindows(arr: number[], k: number): number[] {
  const result: number[] = [];
  const freq = new Map<number, number>();

  // Build first window
  for (let i = 0; i < k; i++) {
    freq.set(arr[i], (freq.get(arr[i]) ?? 0) + 1);
  }
  result.push(freq.size);

  // Slide
  for (let i = k; i < arr.length; i++) {
    // Add right element
    freq.set(arr[i], (freq.get(arr[i]) ?? 0) + 1);

    // Remove left element
    const leftVal = arr[i - k];
    const leftCount = freq.get(leftVal)!;
    if (leftCount === 1) freq.delete(leftVal);
    else freq.set(leftVal, leftCount - 1);

    result.push(freq.size);
  }

  return result;
}
```

**Example Problems:**
1. Maximum sum subarray of size k
2. Average of all contiguous subarrays of size k
3. Count distinct elements in every window of size k

---

### Pattern 2: Sliding Window (Variable Size)

**What it is:** Expand the right pointer until a condition breaks, then shrink the left pointer until the condition is restored. Track the best valid window seen.

**How to recognize it:**
- "Longest/shortest subarray/substring satisfying some condition"
- "Minimum window containing..."
- The condition can be checked incrementally and is monotonic (if a window of size w fails, smaller windows might pass)

**When to use it:** When you need to find an optimal subarray/substring with a constraint on its content (not a fixed size).

**Template:**

```typescript
function slidingWindowVariable(arr: number[], condition: (window: number[]) => boolean): number {
  let left = 0;
  let maxLen = 0;

  for (let right = 0; right < arr.length; right++) {
    // Expand: add arr[right] to window
    // ... update window state

    // Shrink: while condition violated, move left
    while (/* condition violated */ false) {
      // remove arr[left] from window state
      left++;
    }

    // Condition satisfied — record best
    maxLen = Math.max(maxLen, right - left + 1);
  }

  return maxLen;
}

// Problem: Longest substring with at most k distinct characters — O(n)
function longestSubstringKDistinct(s: string, k: number): number {
  const freq = new Map<string, number>();
  let left = 0;
  let maxLen = 0;

  for (let right = 0; right < s.length; right++) {
    // Expand
    const c = s[right];
    freq.set(c, (freq.get(c) ?? 0) + 1);

    // Shrink until at most k distinct
    while (freq.size > k) {
      const lc = s[left];
      const count = freq.get(lc)!;
      if (count === 1) freq.delete(lc);
      else freq.set(lc, count - 1);
      left++;
    }

    maxLen = Math.max(maxLen, right - left + 1);
  }

  return maxLen;
}

// Problem: Longest substring without repeating characters — O(n)
function longestUniqueSubstring(s: string): number {
  const lastSeen = new Map<string, number>();
  let left = 0;
  let maxLen = 0;

  for (let right = 0; right < s.length; right++) {
    const c = s[right];
    if (lastSeen.has(c) && lastSeen.get(c)! >= left) {
      left = lastSeen.get(c)! + 1; // jump left past duplicate
    }
    lastSeen.set(c, right);
    maxLen = Math.max(maxLen, right - left + 1);
  }

  return maxLen;
}

// Problem: Minimum window substring containing all chars of t — O(n)
function minWindowSubstring(s: string, t: string): string {
  const need = new Map<string, number>();
  for (const c of t) need.set(c, (need.get(c) ?? 0) + 1);

  const have = new Map<string, number>();
  let formed = 0;
  const required = need.size;

  let left = 0;
  let minLen = Infinity;
  let minStart = 0;

  for (let right = 0; right < s.length; right++) {
    const c = s[right];
    have.set(c, (have.get(c) ?? 0) + 1);

    if (need.has(c) && have.get(c) === need.get(c)) formed++;

    while (formed === required) {
      // Record best
      if (right - left + 1 < minLen) {
        minLen = right - left + 1;
        minStart = left;
      }
      // Shrink left
      const lc = s[left];
      have.set(lc, have.get(lc)! - 1);
      if (need.has(lc) && have.get(lc)! < need.get(lc)!) formed--;
      left++;
    }
  }

  return minLen === Infinity ? "" : s.slice(minStart, minStart + minLen);
}
```

**Example Problems:**
1. Longest substring with at most k distinct characters
2. Longest substring without repeating characters
3. Minimum window substring

---

### Pattern 3: Two Pointers

**What it is:** Use two indices (usually `left` and `right`) that traverse the array from opposite ends (or in the same direction) to find pairs or partition the array, avoiding nested loops.

**How to recognize it:**
- "Find a pair that sums to..."
- "Remove duplicates in-place"
- Input is sorted (or can be sorted cheaply)
- Need to compare elements from both ends

**When to use it:** When the input is sorted and you need to find pairs, triplets, or partition elements. Reduces O(n²) brute force to O(n).

**Template:**

```typescript
// Two-pointer on sorted array
function twoPointerTemplate(arr: number[], target: number): [number, number] | null {
  let left = 0;
  let right = arr.length - 1;

  while (left < right) {
    const sum = arr[left] + arr[right];
    if (sum === target) return [left, right];
    if (sum < target) left++;   // need larger sum
    else right--;               // need smaller sum
  }

  return null;
}

// Problem: Two sum in sorted array — O(n) time, O(1) space
function twoSumSorted(nums: number[], target: number): [number, number] | null {
  let left = 0;
  let right = nums.length - 1;

  while (left < right) {
    const sum = nums[left] + nums[right];
    if (sum === target) return [left, right];
    if (sum < target) left++;
    else right--;
  }

  return null;
}

// Problem: Container with most water — O(n)
function maxWater(height: number[]): number {
  let left = 0;
  let right = height.length - 1;
  let maxArea = 0;

  while (left < right) {
    const area = Math.min(height[left], height[right]) * (right - left);
    maxArea = Math.max(maxArea, area);

    // Move the shorter wall — it's the only way to possibly find more water
    if (height[left] < height[right]) left++;
    else right--;
  }

  return maxArea;
}

// Problem: Three sum — O(n²)
function threeSum(nums: number[]): number[][] {
  nums.sort((a, b) => a - b);
  const result: number[][] = [];

  for (let i = 0; i < nums.length - 2; i++) {
    if (i > 0 && nums[i] === nums[i - 1]) continue; // skip duplicates

    let left = i + 1;
    let right = nums.length - 1;

    while (left < right) {
      const sum = nums[i] + nums[left] + nums[right];
      if (sum === 0) {
        result.push([nums[i], nums[left], nums[right]]);
        while (left < right && nums[left] === nums[left + 1]) left++;
        while (left < right && nums[right] === nums[right - 1]) right--;
        left++;
        right--;
      } else if (sum < 0) {
        left++;
      } else {
        right--;
      }
    }
  }

  return result;
}
```

**Example Problems:**
1. Two sum in sorted array
2. Container with most water
3. Three sum (triplets that sum to zero)

---

### Pattern 4: Fast & Slow Pointers

**What it is:** Two pointers that move at different speeds through a linked list or array. The fast pointer moves 2 steps per iteration, slow moves 1.

**How to recognize it:**
- "Detect cycle in linked list"
- "Find middle of linked list"
- "Linked list palindrome"
- Problems on linked lists where you cannot use extra space

**When to use it:** Cycle detection and middle-finding on linked lists without allocating extra memory.

**Template:**

```typescript
interface ListNode {
  val: number;
  next: ListNode | null;
}

// Cycle detection — Floyd's algorithm — O(n) time, O(1) space
function hasCycle(head: ListNode | null): boolean {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;

    if (slow === fast) return true; // met inside cycle
  }

  return false; // fast reached end — no cycle
}

// Find middle of linked list — O(n) time, O(1) space
// When fast reaches end, slow is at middle
function findMiddle(head: ListNode | null): ListNode | null {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
  }

  return slow; // middle node
}

// Find cycle start — Floyd's phase 2
function detectCycleStart(head: ListNode | null): ListNode | null {
  let slow = head;
  let fast = head;

  // Phase 1: detect cycle
  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) break;
  }

  if (fast === null || fast.next === null) return null; // no cycle

  // Phase 2: find entry point
  // Move one pointer to head; both advance at speed 1
  slow = head;
  while (slow !== fast) {
    slow = slow!.next;
    fast = fast!.next;
  }

  return slow; // cycle start
}

// Check if linked list is palindrome using fast/slow + reversal — O(n), O(1)
function isPalindrome(head: ListNode | null): boolean {
  if (!head || !head.next) return true;

  // Find middle
  let slow: ListNode | null = head;
  let fast: ListNode | null = head;
  while (fast && fast.next) {
    slow = slow!.next;
    fast = fast.next.next;
  }

  // Reverse second half
  let prev: ListNode | null = null;
  let curr: ListNode | null = slow;
  while (curr) {
    const next = curr.next;
    curr.next = prev;
    prev = curr;
    curr = next;
  }

  // Compare both halves
  let left: ListNode | null = head;
  let right: ListNode | null = prev;
  while (right) {
    if (left!.val !== right.val) return false;
    left = left!.next;
    right = right.next;
  }

  return true;
}
```

**Example Problems:**
1. Detect cycle in linked list
2. Find middle of linked list
3. Linked list palindrome check

---

### Pattern 5: Merge Intervals

**What it is:** Sort intervals by start time, then scan linearly: if the current interval overlaps with the last merged interval, extend it; otherwise, add it as a new interval.

**How to recognize it:**
- "Merge overlapping intervals"
- "Find all conflicting meetings"
- "Minimum number of meeting rooms"
- Any problem involving ranges that may overlap

**When to use it:** When you need to combine or count overlapping time ranges, intervals, or segments.

**Template:**

```typescript
type Interval = [number, number]; // [start, end]

// Merge overlapping intervals — O(n log n)
function mergeIntervals(intervals: Interval[]): Interval[] {
  if (intervals.length === 0) return [];

  // Sort by start time
  intervals.sort((a, b) => a[0] - b[0]);

  const merged: Interval[] = [intervals[0]];

  for (let i = 1; i < intervals.length; i++) {
    const last = merged[merged.length - 1];
    const curr = intervals[i];

    if (curr[0] <= last[1]) {
      // Overlapping — extend
      last[1] = Math.max(last[1], curr[1]);
    } else {
      // Non-overlapping — add new
      merged.push(curr);
    }
  }

  return merged;
}

// Insert a new interval and merge — O(n)
function insertInterval(intervals: Interval[], newInterval: Interval): Interval[] {
  const result: Interval[] = [];
  let i = 0;

  // Add all intervals before newInterval
  while (i < intervals.length && intervals[i][1] < newInterval[0]) {
    result.push(intervals[i++]);
  }

  // Merge overlapping intervals with newInterval
  while (i < intervals.length && intervals[i][0] <= newInterval[1]) {
    newInterval[0] = Math.min(newInterval[0], intervals[i][0]);
    newInterval[1] = Math.max(newInterval[1], intervals[i][1]);
    i++;
  }
  result.push(newInterval);

  // Add remaining intervals
  while (i < intervals.length) result.push(intervals[i++]);

  return result;
}

// Minimum meeting rooms (minimum overlap at any point) — O(n log n)
function minMeetingRooms(intervals: Interval[]): number {
  const starts = intervals.map(i => i[0]).sort((a, b) => a - b);
  const ends = intervals.map(i => i[1]).sort((a, b) => a - b);

  let rooms = 0;
  let endPtr = 0;

  for (let i = 0; i < starts.length; i++) {
    if (starts[i] < ends[endPtr]) {
      rooms++; // need a new room
    } else {
      endPtr++; // a meeting ended — reuse its room
    }
  }

  return rooms;
}
```

**Example Problems:**
1. Merge overlapping intervals
2. Insert interval into sorted list and merge
3. Minimum number of meeting rooms needed

---

### Pattern 6: Cyclic Sort

**What it is:** When an array contains numbers in range `[1, n]` (or `[0, n-1]`), you can place each number at its correct index in O(n) time using swaps, without extra space.

**How to recognize it:**
- Array contains numbers in range `[1, n]` or `[0, n-1]`
- "Find missing number", "Find duplicate", "Find all missing numbers"
- You need O(n) time with O(1) space

**When to use it:** Only when numbers fit in a specific index range, allowing each element to be its own "key."

**Template:**

```typescript
// Place each number at its correct index — O(n) time, O(1) space
function cyclicSort(nums: number[]): void {
  let i = 0;

  while (i < nums.length) {
    const correctIdx = nums[i] - 1; // number 1 belongs at index 0, etc.

    if (nums[i] !== nums[correctIdx]) {
      // Swap to correct position
      [nums[i], nums[correctIdx]] = [nums[correctIdx], nums[i]];
    } else {
      i++; // already in correct position
    }
  }
}

// Find missing number in [1..n] — O(n), O(1) space
function findMissingNumber(nums: number[]): number {
  let i = 0;

  while (i < nums.length) {
    const j = nums[i] - 1;
    if (nums[i] > 0 && nums[i] <= nums.length && nums[i] !== nums[j]) {
      [nums[i], nums[j]] = [nums[j], nums[i]];
    } else {
      i++;
    }
  }

  // Find first position where number doesn't match
  for (let i = 0; i < nums.length; i++) {
    if (nums[i] !== i + 1) return i + 1;
  }

  return nums.length + 1;
}

// Find all duplicates in array of size n with values in [1,n] — O(n), O(1) space
function findAllDuplicates(nums: number[]): number[] {
  let i = 0;

  while (i < nums.length) {
    const j = nums[i] - 1;
    if (nums[i] !== nums[j]) {
      [nums[i], nums[j]] = [nums[j], nums[i]];
    } else {
      i++;
    }
  }

  const duplicates: number[] = [];
  for (let i = 0; i < nums.length; i++) {
    if (nums[i] !== i + 1) duplicates.push(nums[i]);
  }

  return duplicates;
}
```

**Example Problems:**
1. Find the missing number in `[1..n]`
2. Find all duplicate numbers in an array of size n with values in `[1, n]`
3. Find the smallest missing positive integer

---

### Pattern 7: In-Place Reversal of Linked List

**What it is:** Reverse a linked list or a portion of it in-place using three pointers: `prev`, `curr`, `next`. No extra data structures needed.

**How to recognize it:**
- "Reverse a linked list"
- "Reverse every k nodes"
- "Reverse a portion (sublist)"

**When to use it:** Any linked list reversal with O(1) space requirement.

**Template:**

```typescript
interface ListNode {
  val: number;
  next: ListNode | null;
}

// Reverse entire linked list — O(n), O(1) space
function reverseList(head: ListNode | null): ListNode | null {
  let prev: ListNode | null = null;
  let curr = head;

  while (curr !== null) {
    const next = curr.next; // save next
    curr.next = prev;       // reverse pointer
    prev = curr;            // advance prev
    curr = next;            // advance curr
  }

  return prev; // new head
}

// Reverse sublist from position left to right (1-indexed) — O(n), O(1) space
function reverseBetween(
  head: ListNode | null,
  left: number,
  right: number
): ListNode | null {
  if (!head || left === right) return head;

  const dummy: ListNode = { val: 0, next: head };
  let beforeLeft: ListNode = dummy;

  // Walk to node just before position `left`
  for (let i = 1; i < left; i++) beforeLeft = beforeLeft.next!;

  let prev: ListNode | null = null;
  let curr: ListNode | null = beforeLeft.next;

  // Reverse `right - left + 1` nodes
  for (let i = 0; i <= right - left; i++) {
    const next = curr!.next;
    curr!.next = prev;
    prev = curr;
    curr = next;
  }

  // Reconnect
  beforeLeft.next!.next = curr;
  beforeLeft.next = prev;

  return dummy.next;
}

// Reverse every k nodes — O(n), O(1) space
function reverseKGroup(head: ListNode | null, k: number): ListNode | null {
  // Check if k nodes remain
  let check: ListNode | null = head;
  let count = 0;
  while (check && count < k) {
    check = check.next;
    count++;
  }
  if (count < k) return head; // fewer than k nodes remain — don't reverse

  // Reverse k nodes
  let prev: ListNode | null = null;
  let curr: ListNode | null = head;
  for (let i = 0; i < k; i++) {
    const next = curr!.next;
    curr!.next = prev;
    prev = curr;
    curr = next;
  }

  // head is now the tail of reversed group — connect to next group
  head!.next = reverseKGroup(curr, k);

  return prev;
}
```

**Example Problems:**
1. Reverse entire linked list
2. Reverse a sublist between positions left and right
3. Reverse every k nodes in a linked list

---

### Pattern 8: BFS Template

**What it is:** Breadth-First Search explores nodes level by level using a queue. It guarantees the shortest path in unweighted graphs and produces a level-order traversal of trees.

**How to recognize it:**
- "Shortest path in unweighted graph/grid"
- "Level order traversal"
- "Minimum steps/hops to reach..."
- "Find all nodes at distance k"

**When to use it:** Shortest path problems where all edges have equal weight. Level-by-level processing.

**Template:**

```typescript
// Generic BFS on a graph — O(V + E)
function bfs(graph: Map<number, number[]>, start: number): number[] {
  const visited = new Set<number>();
  const queue: number[] = [start];
  visited.add(start);
  const order: number[] = [];

  while (queue.length > 0) {
    const node = queue.shift()!; // dequeue from front
    order.push(node);

    for (const neighbor of graph.get(node) ?? []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push(neighbor);
      }
    }
  }

  return order;
}

// BFS shortest path in unweighted graph — O(V + E)
function shortestPath(
  graph: Map<number, number[]>,
  start: number,
  end: number
): number {
  if (start === end) return 0;

  const visited = new Set<number>([start]);
  const queue: Array<[number, number]> = [[start, 0]]; // [node, distance]

  while (queue.length > 0) {
    const [node, dist] = queue.shift()!;

    for (const neighbor of graph.get(node) ?? []) {
      if (neighbor === end) return dist + 1;
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push([neighbor, dist + 1]);
      }
    }
  }

  return -1; // unreachable
}

// BFS level-order traversal of binary tree
interface TreeNode {
  val: number;
  left: TreeNode | null;
  right: TreeNode | null;
}

function levelOrder(root: TreeNode | null): number[][] {
  if (!root) return [];

  const result: number[][] = [];
  const queue: TreeNode[] = [root];

  while (queue.length > 0) {
    const levelSize = queue.length; // snapshot: all nodes at current level
    const level: number[] = [];

    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift()!;
      level.push(node.val);

      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }

    result.push(level);
  }

  return result;
}

// BFS on 2D grid (number of islands) — O(m × n)
function numIslands(grid: string[][]): number {
  const rows = grid.length;
  const cols = grid[0].length;
  let count = 0;

  function bfsIsland(r: number, c: number): void {
    const queue: [number, number][] = [[r, c]];
    grid[r][c] = "0"; // mark visited by mutating grid

    const dirs = [[0, 1], [0, -1], [1, 0], [-1, 0]];

    while (queue.length > 0) {
      const [row, col] = queue.shift()!;
      for (const [dr, dc] of dirs) {
        const nr = row + dr;
        const nc = col + dc;
        if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] === "1") {
          grid[nr][nc] = "0";
          queue.push([nr, nc]);
        }
      }
    }
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === "1") {
        count++;
        bfsIsland(r, c);
      }
    }
  }

  return count;
}
```

**Example Problems:**
1. Level-order traversal of a binary tree
2. Shortest path in an unweighted graph
3. Number of islands in a grid

---

### Pattern 9: DFS Template

**What it is:** Depth-First Search explores as far as possible along each branch before backtracking. Uses a stack (explicit or the call stack via recursion).

**How to recognize it:**
- "Find all paths from source to target"
- "Explore all connected components"
- "Check if path exists"
- "Traverse a tree in preorder/inorder/postorder"

**When to use it:** When you need to explore all possibilities, find connected components, or process tree nodes in a specific traversal order.

**Template:**

```typescript
// DFS on graph — recursive — O(V + E)
function dfsRecursive(
  graph: Map<number, number[]>,
  node: number,
  visited = new Set<number>()
): void {
  visited.add(node);
  // process node

  for (const neighbor of graph.get(node) ?? []) {
    if (!visited.has(neighbor)) {
      dfsRecursive(graph, neighbor, visited);
    }
  }
}

// DFS on graph — iterative — O(V + E)
function dfsIterative(graph: Map<number, number[]>, start: number): void {
  const visited = new Set<number>();
  const stack: number[] = [start];

  while (stack.length > 0) {
    const node = stack.pop()!;
    if (visited.has(node)) continue;
    visited.add(node);
    // process node

    for (const neighbor of graph.get(node) ?? []) {
      if (!visited.has(neighbor)) {
        stack.push(neighbor);
      }
    }
  }
}

// DFS find all paths from source to target — O(V + E)
function allPaths(
  graph: number[][],
  source: number,
  target: number
): number[][] {
  const result: number[][] = [];

  function dfs(node: number, path: number[]): void {
    if (node === target) {
      result.push([...path]);
      return;
    }

    for (const neighbor of graph[node]) {
      path.push(neighbor);
      dfs(neighbor, path);
      path.pop(); // backtrack
    }
  }

  dfs(source, [source]);
  return result;
}

// DFS flood fill (paint connected region) — O(m × n)
function floodFill(
  image: number[][],
  sr: number,
  sc: number,
  color: number
): number[][] {
  const originalColor = image[sr][sc];
  if (originalColor === color) return image;

  const rows = image.length;
  const cols = image[0].length;

  function dfs(r: number, c: number): void {
    if (r < 0 || r >= rows || c < 0 || c >= cols) return;
    if (image[r][c] !== originalColor) return;

    image[r][c] = color;
    dfs(r + 1, c);
    dfs(r - 1, c);
    dfs(r, c + 1);
    dfs(r, c - 1);
  }

  dfs(sr, sc);
  return image;
}
```

**Example Problems:**
1. Find all paths from source to target in a DAG
2. Flood fill a connected region in a grid
3. Count connected components in an undirected graph

---

### Pattern 10: Subsets / Combinations / Permutations (Backtracking)

**What it is:** Build a solution incrementally — add a candidate, recurse, then undo (backtrack) to try the next candidate. Explores the entire search tree systematically.

**How to recognize it:**
- "All subsets / power set"
- "All combinations of size k"
- "All permutations"
- "Is there any valid assignment?" (N-queens, Sudoku)

**When to use it:** When you need to enumerate all possibilities or find any valid configuration. Accept exponential time — it's unavoidable when enumerating exponential outputs.

**Template:**

```typescript
// Backtracking template
function backtrack(
  result: number[][],
  current: number[],
  start: number,
  candidates: number[],
  remaining: number
): void {
  if (/* base case: solution complete */) {
    result.push([...current]); // copy before modifying
    return;
  }

  for (let i = start; i < candidates.length; i++) {
    // Skip invalid choices (pruning)
    if (/* pruning condition */) continue;

    current.push(candidates[i]); // choose
    backtrack(result, current, i + 1, candidates, remaining - candidates[i]); // explore
    current.pop(); // unchoose (backtrack)
  }
}

// Power set (all subsets) — O(2^n × n)
function powerSet(nums: number[]): number[][] {
  const result: number[][] = [];

  function backtrack(start: number, current: number[]): void {
    result.push([...current]); // every state is a valid subset

    for (let i = start; i < nums.length; i++) {
      current.push(nums[i]);
      backtrack(i + 1, current);
      current.pop();
    }
  }

  backtrack(0, []);
  return result;
}

// Combinations of size k — O(C(n,k) × k)
function combinations(n: number, k: number): number[][] {
  const result: number[][] = [];

  function backtrack(start: number, current: number[]): void {
    if (current.length === k) {
      result.push([...current]);
      return;
    }

    // Pruning: not enough numbers left to fill k
    for (let i = start; i <= n - (k - current.length) + 1; i++) {
      current.push(i);
      backtrack(i + 1, current);
      current.pop();
    }
  }

  backtrack(1, []);
  return result;
}

// All permutations — O(n! × n)
function permutations(nums: number[]): number[][] {
  const result: number[][] = [];

  function backtrack(current: number[], remaining: number[]): void {
    if (remaining.length === 0) {
      result.push([...current]);
      return;
    }

    for (let i = 0; i < remaining.length; i++) {
      current.push(remaining[i]);
      backtrack(current, [...remaining.slice(0, i), ...remaining.slice(i + 1)]);
      current.pop();
    }
  }

  backtrack([], nums);
  return result;
}

// Combination sum (reuse elements, sum to target) — O(n^(target/min))
function combinationSum(candidates: number[], target: number): number[][] {
  const result: number[][] = [];
  candidates.sort((a, b) => a - b);

  function backtrack(start: number, current: number[], remaining: number): void {
    if (remaining === 0) {
      result.push([...current]);
      return;
    }

    for (let i = start; i < candidates.length; i++) {
      if (candidates[i] > remaining) break; // pruning: sorted, no point continuing

      current.push(candidates[i]);
      backtrack(i, current, remaining - candidates[i]); // i, not i+1 (can reuse)
      current.pop();
    }
  }

  backtrack(0, [], target);
  return result;
}
```

**Example Problems:**
1. Generate all subsets (power set)
2. All combinations of size k from numbers 1 to n
3. Combination sum — find all combinations that sum to a target

---

## 4. Pattern Recognition Cheat Sheet

Use this table when you read a problem and need to identify which pattern to try first.

| Problem Characteristics | Pattern to Try |
|---|---|
| Fixed window size over array/string | Sliding Window (Fixed) |
| Longest/shortest subarray satisfying a condition | Sliding Window (Variable) |
| Sorted array, find pair/triplet summing to target | Two Pointers |
| "Container with most water", "trap rainwater" | Two Pointers |
| Cycle detection in linked list | Fast & Slow Pointers |
| Find middle of linked list | Fast & Slow Pointers |
| Overlapping intervals, meeting rooms | Merge Intervals |
| Array with values in range `[1, n]`, find missing/duplicate | Cyclic Sort |
| Reverse linked list or portion of it | In-Place Reversal |
| Shortest path in unweighted graph or grid | BFS |
| Level-order traversal of tree | BFS |
| Find all paths, explore all components | DFS |
| Connected region in grid (flood fill, islands) | DFS |
| All subsets, combinations, permutations | Backtracking |
| "Is there a valid arrangement?" (N-queens, Sudoku) | Backtracking |
| Optimal substructure + overlapping subproblems | Dynamic Programming |
| Locally optimal choice leads to global optimum | Greedy |
| Sorted array, "find position of X" | Binary Search |
| "Minimum/maximum satisfying condition" over monotonic function | Binary Search on Answer |

> 💡 **Pattern stacking:** Many hard problems combine two patterns. "Sliding window max" uses a sliding window + a monotonic deque. "Trapping rainwater" can be solved with two pointers or with a stack. Always consider whether the problem has two distinct sub-problems that each fit a different pattern.

> ⚠️ **Beware of mislabeling.** A problem asking "find if any pair sums to target" in an **unsorted** array is a hashmap problem, not two pointers. Two pointers requires sortedness. Always check the input constraints before pattern-matching.

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Forgetting to copy the current array before pushing to results.** In backtracking, `result.push(current)` pushes a reference. Always `result.push([...current])`.

> ⚠️ **Using `queue.shift()` for BFS in performance-critical code.** `Array.shift()` is O(n). For large graphs, use a proper deque or implement a circular buffer queue.

> ⚠️ **Mutating the input grid in BFS/DFS.** Marking visited cells by overwriting `grid[r][c]` is efficient but makes the function impure and non-reusable. For production code, use a separate `visited` Set.

> ⚠️ **Off-by-one in sliding window.** Window of size k: indices `[i-k, i-1]`. When sliding, remove `arr[i-k]` not `arr[i-k-1]`. Always verify with a small example (n=3, k=2).

> ⚠️ **Forgetting to skip duplicates in combination/permutation problems.** When the input has duplicates and you need unique results, sort first, then skip `if (i > start && nums[i] === nums[i-1]) continue`.

---

## 6. When to Use / Not Use

**These patterns solve almost every LeetCode medium — but they are not appropriate for all production code:**

- **Sliding window** is appropriate in production for streaming aggregations, rate limiters, and real-time analytics over time windows.
- **Two pointers** applies to sorted data deduplication and range merging in real systems.
- **BFS/DFS** patterns apply directly to graph traversal in dependency resolution, network routing, and UI tree rendering.
- **Backtracking** is rarely used directly in production — O(n!) inputs never reach servers. But the template applies to configuration generators, test matrix generators, and constraint-satisfaction systems.

---

## 7. Real-World Scenario

A shipping company needs to assign delivery drivers to time slots, ensuring no two drivers with the same route overlap. The problem reduces to: given a list of time intervals, find the minimum number of groups (rooms) such that no two intervals in the same group overlap.

This is exactly the **Merge Intervals** "minimum meeting rooms" problem:

```typescript
function minDriverGroups(shifts: Interval[]): number {
  const starts = shifts.map(s => s[0]).sort((a, b) => a - b);
  const ends = shifts.map(s => s[1]).sort((a, b) => a - b);

  let groups = 0;
  let endPtr = 0;

  for (let i = 0; i < starts.length; i++) {
    if (starts[i] < ends[endPtr]) {
      groups++; // new shift starts before any ends — need new group
    } else {
      endPtr++; // a shift ended — reuse its group slot
    }
  }

  return groups;
}
```

Recognizing the pattern cut implementation time from hours (brute force simulation) to minutes.

---

## 8. Interview Questions

**Q1: How do you recognize a sliding window problem?**
A: The problem asks for an optimal (max/min/longest/shortest) contiguous subarray or substring. If the window size is fixed, use the fixed variant. If it's variable with a constraint (at most k distinct, sum equals target), use the variable variant with a shrink condition.

**Q2: When does two pointers not work?**
A: When the array is not sorted and you need to avoid the O(n log n) sort cost. In that case, use a hashmap for O(n) pair finding. Two pointers requires monotonic behavior: moving a pointer must monotonically increase or decrease the relevant quantity.

**Q3: What is the difference between BFS and DFS for shortest path?**
A: BFS guarantees the shortest path in unweighted graphs because it explores all nodes at distance d before any at distance d+1. DFS finds a path but not necessarily the shortest one — it goes deep first and might find a longer path.

**Q4: Why does backtracking have exponential complexity?**
A: The search tree has branching factor b and depth d, giving b^d nodes. For subsets: branching factor 2, depth n → O(2^n). For permutations: branching factor n, then n-1, etc. → O(n!). Pruning reduces the constant but not the asymptotic class in the worst case.

**Q5: How do fast/slow pointers prove cycle detection in O(n)?**
A: If a cycle of length c exists, the fast pointer laps the slow pointer inside the cycle. Once both are in the cycle, their distance closes by 1 per step (fast gains 1 step per iteration). They meet within c iterations, and total iterations before meeting is O(n).

**Q6: What is the invariant in the merge intervals algorithm?**
A: The result array is always a list of non-overlapping, sorted intervals. When processing a new interval: if it overlaps with the last merged interval (new.start <= last.end), extend; otherwise append. The sort by start time ensures we only need to look at the last interval.

**Q7: Can you do sliding window on 2D matrices?**
A: Yes, but it becomes a 2D sliding window. For a fixed k×k submatrix sum, precompute a 2D prefix sum array: `prefix[i][j]` = sum of all elements in top-left rectangle. Any k×k submatrix sum is then O(1): `prefix[r+k][c+k] - prefix[r][c+k] - prefix[r+k][c] + prefix[r][c]`.

**Q8: What distinguishes cyclic sort from regular sorting?**
A: Cyclic sort exploits the constraint that values are in `[1, n]` — each value encodes its own correct position. This allows O(n) sorting with O(1) space. Regular comparison sort requires O(n log n) because it makes no assumptions about value ranges.

---

## 9. Exercises

**Exercise 1 (Sliding Window Fixed):** Given an array of integers and a number k, find the maximum sum of any contiguous subarray of size k. Then extend it: find all k-length subarrays whose sum equals a target.

*Hint: Maintain a running sum. Slide by adding `arr[right]` and subtracting `arr[right - k]`.*

**Exercise 2 (Sliding Window Variable):** Given a string, find the length of the longest substring that contains at most 2 distinct characters.

*Hint: Use a frequency map. When `freq.size > 2`, shrink from the left until it's 2 again.*

**Exercise 3 (Two Pointers):** Given a sorted array, remove duplicates in-place and return the new length. Use O(1) extra space.

*Hint: `left` pointer tracks the position of the last unique element. `right` scans ahead.*

**Exercise 4 (Merge Intervals):** Given a list of intervals representing employee work shifts, find all "free time" intervals where no employee is working.

*Hint: Flatten all intervals into one list, sort, merge overlapping intervals, then find gaps between merged intervals.*

**Exercise 5 (Backtracking):** Generate all valid combinations of parentheses for n pairs. For n=3: `["((()))","(()())","(())()","()(())","()()()"]`.

*Hint: At each step you can add `(` if open count < n, or `)` if close count < open count.*

**Exercise 6 (BFS):** Given a word and a word list, find the length of the shortest transformation sequence where each step changes exactly one letter and the intermediate words must be in the word list (word ladder).

*Hint: Model as BFS — each word is a node, edges connect words that differ by one character.*

---

## 10. Further Reading

- [NeetCode Roadmap](https://neetcode.io/roadmap) — problems organized by pattern, with video explanations
- [LeetCode Patterns](https://seanprashad.com/leetcode-patterns/) — curated problem lists by pattern
- [Grokking the Coding Interview](https://www.designgurus.io/course/grokking-the-coding-interview) — the original pattern-based course
- [Blind 75](https://leetcode.com/discuss/general-discussion/460599/blind-75-leetcode-questions) — the 75 problems that cover all critical patterns
- [Backtracking template — LeetCode discuss](https://leetcode.com/problems/combination-sum/solutions/16502/a-general-approach-to-backtracking-questions-in-java-subsets-permutations-combination-sum-palindrome-partitioning/)
- *Algorithm Design Manual* by Steven Skiena — Chapter 7 (Combinatorial Search and Heuristic Methods)
