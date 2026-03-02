# CS Challenges

> Algorithm and data structure challenges covering the most common interview and competitive programming patterns. Mix of easy, medium, and hard problems. Solve each before reading the hints. Target the stated complexity — a brute-force that works is a starting point, not a finish line. Covers: arrays, strings, trees, graphs, dynamic programming, sliding window, and two pointers.

---

## Exercise 1 — Two Sum (Easy)

**Problem:** Given an array of integers `nums` and a target integer `target`, return the indices of the two numbers that add up to `target`. Assume exactly one solution exists. You may not use the same element twice.

**Constraints:**
- `2 <= nums.length <= 10^4`
- `-10^9 <= nums[i] <= 10^9`
- Exactly one valid answer exists

**Example:**
```
Input:  nums = [2, 7, 11, 15], target = 9
Output: [0, 1]   // nums[0] + nums[1] = 9

Input:  nums = [3, 2, 4], target = 6
Output: [1, 2]
```

**Hints:**
1. A nested loop O(n²) works but is not acceptable for large inputs.
2. For each element, you need the "complement": `complement = target - nums[i]`. Have you seen it before?
3. A hash map storing `{ value → index }` lets you answer that in O(1).
4. Scan once: for each element, check the map for its complement first, then insert the element.

**Target complexity:** Time O(n) | Space O(n)

---

## Exercise 2 — Valid Parentheses (Easy)

**Problem:** Given a string `s` containing only `(`, `)`, `{`, `}`, `[`, `]`, determine if the string is valid. A valid string has every open bracket closed by the same bracket type in the correct order.

**Constraints:**
- `1 <= s.length <= 10^4`
- `s` consists only of bracket characters

**Example:**
```
Input:  "()[]{}"  → true
Input:  "([)]"    → false
Input:  "{[]}"    → true
Input:  "("       → false
```

**Hints:**
1. When you see an opening bracket, you need to remember it for later.
2. A stack is a perfect fit: push opens, pop and verify on each closing bracket.
3. If the stack is empty when you encounter a close bracket, it's invalid.
4. After processing the whole string, the stack must be empty.

**Target complexity:** Time O(n) | Space O(n)

---

## Exercise 3 — Best Time to Buy and Sell Stock (Easy)

**Problem:** Given an array `prices` where `prices[i]` is the stock price on day `i`, find the maximum profit from a single buy-sell transaction. The sell day must come after the buy day. Return `0` if no profitable trade exists.

**Constraints:**
- `1 <= prices.length <= 10^5`
- `0 <= prices[i] <= 10^4`

**Example:**
```
Input:  [7, 1, 5, 3, 6, 4]  → Output: 5   // buy day 2 (1), sell day 5 (6)
Input:  [7, 6, 4, 3, 1]     → Output: 0   // prices only fall
```

**Hints:**
1. You want the largest difference where the larger element comes after the smaller one.
2. Scan left to right. Track two variables: `minPrice` and `maxProfit`.
3. At each step: `maxProfit = max(maxProfit, currentPrice - minPrice)`, then update `minPrice` if needed.
4. No need to look backward — `minPrice` already holds the best buy point seen so far.

**Target complexity:** Time O(n) | Space O(1)

---

## Exercise 4 — Longest Substring Without Repeating Characters (Medium)

**Problem:** Given a string `s`, find the length of the longest substring that contains no repeating characters.

**Constraints:**
- `0 <= s.length <= 5 * 10^4`
- `s` may contain English letters, digits, symbols, and spaces

**Example:**
```
Input:  "abcabcbb"  → Output: 3   // "abc"
Input:  "bbbbb"     → Output: 1   // "b"
Input:  "pwwkew"    → Output: 3   // "wke"
```

**Hints:**
1. This is a sliding window problem. Maintain a window `[left, right]` with no duplicates.
2. Use a map to store the last index each character was seen at.
3. When `s[right]` is already in the window, advance `left` to `max(left, lastSeen[s[right]] + 1)`.
4. The answer is `max(answer, right - left + 1)` after each step.

**Target complexity:** Time O(n) | Space O(min(n, |alphabet|))

---

## Exercise 5 — Product of Array Except Self (Medium)

**Problem:** Given an integer array `nums`, return an array `output` where `output[i]` equals the product of all elements in `nums` except `nums[i]`. You must not use division, and you must run in O(n).

**Constraints:**
- `2 <= nums.length <= 10^5`
- `-30 <= nums[i] <= 30`
- The product of any prefix or suffix fits in a 32-bit integer

**Example:**
```
Input:  [1, 2, 3, 4]     → Output: [24, 12, 8, 6]
Input:  [-1, 1, 0, -3, 3] → Output: [0, 0, 9, 0, 0]
```

**Hints:**
1. `output[i] = (product of all elements to the left of i) * (product of all elements to the right of i)`.
2. First pass left to right: fill `output[i]` with the prefix product.
3. Second pass right to left: maintain a running suffix product and multiply it into `output[i]`.
4. No extra array needed — the output array doubles as the prefix array.

**Target complexity:** Time O(n) | Space O(1) extra (output array excluded)

---

## Exercise 6 — Maximum Depth of Binary Tree (Easy)

**Problem:** Given the root of a binary tree, return its maximum depth — the number of nodes along the longest path from root to the farthest leaf node.

**Constraints:**
- Number of nodes: `[0, 10^4]`
- `-100 <= Node.val <= 100`

**Example:**
```
Tree:      3
          / \
         9  20
            / \
           15   7

Output: 3
```

**Hints:**
1. Recursive approach: depth of a node is `1 + max(depth(left), depth(right))`.
2. Base case: `null` node has depth `0`.
3. Iterative alternative: BFS level-order traversal — count the number of levels.
4. DFS iterative with a stack of `(node, currentDepth)` pairs also works.

**Target complexity:** Time O(n) | Space O(h) where h = height (O(n) worst, O(log n) balanced)

---

## Exercise 7 — Lowest Common Ancestor of a BST (Medium)

**Problem:** Given a binary search tree and two nodes `p` and `q`, find their lowest common ancestor (LCA). The LCA is the deepest node that has both `p` and `q` as descendants (a node is a descendant of itself).

**Constraints:**
- All node values are unique
- `p` and `q` are guaranteed to exist in the BST
- `2 <= number of nodes <= 10^5`

**Example:**
```
BST:          6
             / \
            2   8
           / \ / \
          0  4 7  9
            / \
           3   5

LCA(2, 8) → 6
LCA(2, 4) → 2
LCA(0, 5) → 2
```

**Hints:**
1. Exploit the BST property: if both `p` and `q` are less than `node.val`, descend left. If both are greater, descend right. Otherwise, `node` is the LCA (the paths diverge here).
2. No recursion needed — iterate with a `while` loop.
3. This works because any node where `p.val <= node.val <= q.val` is necessarily the LCA.

**Target complexity:** Time O(h) | Space O(1)

---

## Exercise 8 — Number of Islands (Medium)

**Problem:** Given an `m x n` grid of `'1'` (land) and `'0'` (water), return the number of islands. An island is formed by connecting adjacent land cells horizontally or vertically, surrounded by water on all sides.

**Constraints:**
- `1 <= m, n <= 300`
- `grid[i][j]` is `'0'` or `'1'`

**Example:**
```
Grid 1:           Grid 2:
1 1 1 1 0         1 1 0 0 0
1 1 0 1 0         1 1 0 0 0
1 1 0 0 0         0 0 1 0 0
0 0 0 0 0         0 0 0 1 1

Output: 1         Output: 3
```

**Hints:**
1. Iterate over every cell. When you encounter a `'1'`, increment the island counter and flood-fill the entire island to mark it as visited.
2. Flood-fill: DFS or BFS from the starting cell. Mark each visited `'1'` as `'0'` (sink it) to avoid revisiting.
3. Directions: up, down, left, right. Guard against out-of-bounds indices.
4. The modification of the input grid is acceptable; if not, use a separate `visited` boolean grid.

**Target complexity:** Time O(m * n) | Space O(m * n) worst-case DFS stack

---

## Exercise 9 — Coin Change (Medium)

**Problem:** Given an array `coins` of coin denominations and an integer `amount`, return the minimum number of coins needed to reach exactly that amount. If it is impossible, return `-1`. You have unlimited coins of each denomination.

**Constraints:**
- `1 <= coins.length <= 12`
- `1 <= coins[i] <= 2^31 - 1`
- `0 <= amount <= 10^4`

**Example:**
```
Input:  coins = [1, 5, 11], amount = 15
Output: 3   // 5 + 5 + 5

Input:  coins = [2], amount = 3
Output: -1

Input:  coins = [1], amount = 0
Output: 0
```

**Hints:**
1. Bottom-up DP. Define `dp[i]` = minimum coins to make amount `i`.
2. Initialize: `dp[0] = 0`, all other entries to `Infinity`.
3. For each `i` from 1 to `amount`, and for each coin `c` where `c <= i`: `dp[i] = min(dp[i], dp[i - c] + 1)`.
4. Return `dp[amount] === Infinity ? -1 : dp[amount]`.

**Target complexity:** Time O(amount * n) | Space O(amount)

---

## Exercise 10 — Merge K Sorted Lists (Hard)

**Problem:** You are given an array of `k` linked lists, each sorted in ascending order. Merge all lists into one sorted linked list and return it.

**Constraints:**
- `0 <= k <= 10^4`
- `0 <= lists[i].length <= 500`
- Total nodes: at most `10^4`
- Node values: `-10^4 <= val <= 10^4`

**Example:**
```
Input:  [[1,4,5], [1,3,4], [2,6]]
Output: [1, 1, 2, 3, 4, 4, 5, 6]
```

**Hints:**
1. Naive merge one-by-one: O(kN) — acceptable for small k but not optimal.
2. Divide and conquer: pair up lists and merge each pair, then repeat. O(N log k) time.
3. Min-heap: seed the heap with the head node of each list. Pop the minimum, push its `next` into the heap. Repeat until heap is empty.
4. In JS/TS there is no built-in heap — implement a simple binary min-heap or use a sorted array with binary insertion for small k.

**Target complexity:** Time O(N log k) | Space O(k)

---

## Exercise 11 — Trapping Rain Water (Hard)

**Problem:** Given `n` non-negative integers representing an elevation map (each bar has width 1), compute how much water can be trapped after rain.

**Constraints:**
- `n == height.length`
- `1 <= n <= 2 * 10^4`
- `0 <= height[i] <= 10^5`

**Example:**
```
Input:  [0,1,0,2,1,0,1,3,2,1,2,1]
Output: 6

Input:  [4,2,0,3,2,5]
Output: 9
```

**Hints:**
1. Water at index `i` = `min(maxLeft[i], maxRight[i]) - height[i]`, floored at 0.
2. Precompute `maxLeft` (left scan) and `maxRight` (right scan) arrays — O(n) time, O(n) space.
3. Optimize to O(1) space with two pointers. Close in from both ends. Move the side with the smaller max — that side's contribution is fully determined.
4. The invariant: `water[i] = min(leftMax, rightMax) - height[i]`.

**Target complexity:** Time O(n) | Space O(1) (two-pointer approach)

---

## Exercise 12 — Word Break (Medium)

**Problem:** Given a string `s` and a dictionary `wordDict`, return `true` if `s` can be segmented into a space-separated sequence of dictionary words.

**Constraints:**
- `1 <= s.length <= 300`
- `1 <= wordDict.length <= 1000`
- All strings use lowercase English letters
- Words in `wordDict` are unique

**Example:**
```
Input:  s = "leetcode",      wordDict = ["leet","code"]          → true
Input:  s = "applepenapple", wordDict = ["apple","pen"]          → true
Input:  s = "catsandog",     wordDict = ["cats","dog","sand","and","cat"] → false
```

**Hints:**
1. Define `dp[i]` = `true` if `s[0..i-1]` can be segmented using the dictionary.
2. Base case: `dp[0] = true` (empty prefix).
3. For each position `i`, try every split point `j < i`: if `dp[j]` is true and `s[j..i-1]` is in the dictionary, set `dp[i] = true`.
4. Convert `wordDict` to a `Set` for O(1) lookup. Return `dp[s.length]`.

**Target complexity:** Time O(n² * m) where m = avg word length | Space O(n)

---

## Exercise 13 — Serialize and Deserialize Binary Tree (Hard)

**Problem:** Design an algorithm to serialize a binary tree to a string, and deserialize that string back to the original tree. There is no restriction on format — you define it.

**Constraints:**
- Number of nodes: `[0, 10^4]`
- `-1000 <= Node.val <= 1000`
- The deserializer must produce a tree structurally identical to the original

**Example:**
```
Tree:      1
          / \
         2   3
            / \
           4   5

Serialized (one valid format): "1,2,X,X,3,4,X,X,5,X,X"
Deserialized: same tree
```

**Hints:**
1. Use preorder traversal (root → left → right). Represent null children with a sentinel like `"X"`.
2. Serialization: recursive preorder. Append `node.val` or `"X"` at each step, joined by commas.
3. Deserialization: split on comma, use a pointer or queue. Consume tokens in preorder: read a token, if it's `"X"` return null, else create a node and recurse for left then right.
4. BFS level-order also works and produces a more readable format, but preorder has simpler recursion.

**Target complexity:** Time O(n) | Space O(n)

---

## Exercise 14 — Find Median from Data Stream (Hard)

**Problem:** Design a data structure supporting `addNum(num)` and `findMedian()`. `findMedian` returns the median of all numbers added so far (average of the two middle values if even count).

**Constraints:**
- `-10^5 <= num <= 10^5`
- At most `5 * 10^4` calls total
- At least one number added before calling `findMedian`

**Example:**
```
addNum(1) → addNum(2) → findMedian() = 1.5
addNum(3) → findMedian() = 2.0
```

**Hints:**
1. Maintain two heaps: a **max-heap** for the lower half, a **min-heap** for the upper half.
2. Keep sizes balanced: `|maxHeap.size - minHeap.size| <= 1`.
3. Add to the max-heap first. If its top exceeds the min-heap's top, move the max-heap's top to the min-heap. Then rebalance sizes.
4. `findMedian`: if sizes are equal, average both tops. Otherwise, return the top of the larger heap.
5. JS has no built-in heap — implement a `MinHeap` class with `push` and `pop` in O(log n).

**Target complexity:** addNum O(log n) | findMedian O(1) | Space O(n)

---

## Exercise 15 — Alien Dictionary (Hard)

**Problem:** You receive a list of words sorted according to an alien language's character ordering (using the same letters as English). Derive the order of characters. If no valid order exists (cycle detected), return `""`. If multiple valid orders exist, return any.

**Constraints:**
- `1 <= words.length <= 100`
- `1 <= words[i].length <= 100`
- All characters are lowercase English letters

**Example:**
```
Input:  ["wrt","wrf","er","ett","rftt"]   → Output: "wertf"
Input:  ["z","x"]                          → Output: "zx"
Input:  ["z","x","z"]                      → Output: ""   // cycle
Input:  ["abc","ab"]                       → Output: ""   // invalid prefix ordering
```

**Hints:**
1. Compare each pair of adjacent words character by character. The first differing character pair tells you one ordering rule: `charA` comes before `charB`.
2. Build a directed graph with edges `charA → charB`. Track in-degrees for each character.
3. Apply Kahn's topological sort: start with all nodes of in-degree 0, process in BFS order.
4. If the sorted output length does not equal the number of unique characters, a cycle exists — return `""`.
5. Edge case: if `words[i]` is a prefix of `words[i-1]`, the input itself is invalid.

**Target complexity:** Time O(C) where C = total characters | Space O(1) (alphabet size is constant at 26)
