# Dynamic Programming

## 1. What & Why

Dynamic programming (DP) is an optimization technique for problems with two properties:
1. **Optimal substructure:** the optimal solution can be constructed from optimal solutions to subproblems
2. **Overlapping subproblems:** the same subproblems are solved multiple times in a naive recursive approach

DP eliminates redundant work by solving each subproblem once and caching the result. It is the difference between exponential and polynomial time on a huge class of problems: shortest paths, sequence alignment, resource allocation, and scheduling.

The "programming" in dynamic programming refers to tabulation (filling in a table), not writing code. The term was coined by Richard Bellman in the 1950s.

---

## 2. Core Concepts

### Top-Down (Memoization)

Write the recursive solution, add a cache. When a subproblem is first solved, store the result. When seen again, return from cache. Natural to write — just augment an existing recursive solution.

**Pros:** Only computes subproblems that are actually needed. Easier to implement when state space is large or sparsely populated.
**Cons:** Function call overhead. Recursive depth may cause stack overflow.

### Bottom-Up (Tabulation)

Fill a table starting from the smallest subproblems, building up to the final answer. Iterative — no recursion overhead. Often allows space optimization (only keep the last row/layer).

**Pros:** No recursion overhead. Often more space-efficient. Clear execution order.
**Cons:** Must solve all subproblems even if some are unnecessary. Less intuitive for complex state transitions.

### How to Approach a DP Problem

1. **Identify the state:** what information defines a subproblem? (e.g., index i, remaining capacity, number of items used)
2. **Define the recurrence:** express dp[state] in terms of smaller states
3. **Set base cases:** what is the value when there's no remaining work?
4. **Determine computation order:** bottom-up requires knowing the dependency direction
5. **Identify space optimization:** can you reduce from O(n²) to O(n)?

---

## 3. How It Works

```
Fibonacci with DP:
State: dp[i] = ith Fibonacci number
Recurrence: dp[i] = dp[i-1] + dp[i-2]
Base cases: dp[0] = 0, dp[1] = 1

Filling the table bottom-up:
dp: [0, 1, 1, 2, 3, 5, 8, 13, ...]
    i=0 i=1 i=2 i=3 ...

Each value computed once: O(n) time, O(n) space.
Space optimization: only need last two values → O(1) space.

Coin Change:
coins=[1,5,10], amount=11
State: dp[i] = min coins to make amount i
dp[0] = 0 (base case)
dp[1] = min(dp[0]+1) = 1 (use coin 1)
dp[5] = min(dp[4]+1, dp[0]+1) = 1 (use coin 5)
dp[10] = min(dp[9]+1, dp[5]+1, dp[0]+1) = 1 (use coin 10)
dp[11] = min(dp[10]+1, dp[6]+1, dp[1]+1) = 2 (coin 10 + coin 1)
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- 1D DP Problems ---

// Climbing Stairs: count ways to climb n stairs (1 or 2 steps at a time)
// State: dp[i] = number of ways to reach stair i
function climbStairs(n: number): number {
  if (n <= 2) return n;
  let prev2 = 1, prev1 = 2;
  for (let i = 3; i <= n; i++) {
    const current = prev1 + prev2;
    prev2 = prev1;
    prev1 = current;
  }
  return prev1; // O(n) time, O(1) space
}

// House Robber: max money from non-adjacent houses
// State: dp[i] = max money from first i houses
function rob(nums: number[]): number {
  if (nums.length === 0) return 0;
  if (nums.length === 1) return nums[0];
  let prev2 = nums[0];
  let prev1 = Math.max(nums[0], nums[1]);
  for (let i = 2; i < nums.length; i++) {
    const current = Math.max(prev1, prev2 + nums[i]);
    prev2 = prev1;
    prev1 = current;
  }
  return prev1;
}

// Coin Change: minimum coins to make amount
// State: dp[i] = min coins to make amount i
function coinChange(coins: number[], amount: number): number {
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;

  for (let i = 1; i <= amount; i++) {
    for (const coin of coins) {
      if (coin <= i && dp[i - coin] !== Infinity) {
        dp[i] = Math.min(dp[i], dp[i - coin] + 1);
      }
    }
  }
  return dp[amount] === Infinity ? -1 : dp[amount];
}

// Word Break: can string s be segmented into words from wordDict?
// State: dp[i] = can s[0..i-1] be segmented?
function wordBreak(s: string, wordDict: string[]): boolean {
  const wordSet = new Set(wordDict);
  const dp = new Array(s.length + 1).fill(false);
  dp[0] = true; // empty string is always valid

  for (let i = 1; i <= s.length; i++) {
    for (let j = 0; j < i; j++) {
      if (dp[j] && wordSet.has(s.slice(j, i))) {
        dp[i] = true;
        break;
      }
    }
  }
  return dp[s.length];
}

// Maximum Product Subarray: O(n) time, O(1) space
// State: track both max and min (min can become max when multiplied by negative)
function maxProduct(nums: number[]): number {
  let maxSoFar = nums[0], minSoFar = nums[0], result = nums[0];

  for (let i = 1; i < nums.length; i++) {
    const candidates = [nums[i], maxSoFar * nums[i], minSoFar * nums[i]];
    maxSoFar = Math.max(...candidates);
    minSoFar = Math.min(...candidates);
    result = Math.max(result, maxSoFar);
  }
  return result;
}

// Longest Increasing Subsequence: O(n²) DP
// State: dp[i] = length of LIS ending at index i
function lengthOfLIS(nums: number[]): number {
  const dp = new Array(nums.length).fill(1);
  let maxLen = 1;

  for (let i = 1; i < nums.length; i++) {
    for (let j = 0; j < i; j++) {
      if (nums[j] < nums[i]) {
        dp[i] = Math.max(dp[i], dp[j] + 1);
      }
    }
    maxLen = Math.max(maxLen, dp[i]);
  }
  return maxLen;
}
// Note: O(n log n) is possible using patience sorting / binary search

// Decode Ways: number of ways to decode a digit string to letters
// 'A'=1, 'B'=2, ..., 'Z'=26
function numDecodings(s: string): number {
  const n = s.length;
  const dp = new Array(n + 1).fill(0);
  dp[0] = 1; // empty string: one way
  dp[1] = s[0] === "0" ? 0 : 1;

  for (let i = 2; i <= n; i++) {
    const oneDigit = parseInt(s[i - 1]);
    const twoDigits = parseInt(s.slice(i - 2, i));

    if (oneDigit >= 1) dp[i] += dp[i - 1];
    if (twoDigits >= 10 && twoDigits <= 26) dp[i] += dp[i - 2];
  }
  return dp[n];
}

// --- 2D DP Problems ---

// Longest Common Subsequence: O(m×n) time and space
// State: dp[i][j] = LCS length of s1[0..i-1] and s2[0..j-1]
function longestCommonSubsequence(text1: string, text2: string): number {
  const m = text1.length, n = text2.length;
  const dp: number[][] = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (text1[i - 1] === text2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1] + 1; // chars match — extend LCS
      } else {
        dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]); // skip one char
      }
    }
  }
  return dp[m][n];
}

// Edit Distance (Levenshtein): min operations to transform word1 to word2
// Operations: insert, delete, replace (each costs 1)
// State: dp[i][j] = edit distance between word1[0..i-1] and word2[0..j-1]
function minDistance(word1: string, word2: string): number {
  const m = word1.length, n = word2.length;
  const dp: number[][] = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));

  // Base cases: transform from/to empty string
  for (let i = 0; i <= m; i++) dp[i][0] = i; // delete all chars of word1
  for (let j = 0; j <= n; j++) dp[0][j] = j; // insert all chars of word2

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (word1[i - 1] === word2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1]; // no operation needed
      } else {
        dp[i][j] = 1 + Math.min(
          dp[i - 1][j],     // delete from word1
          dp[i][j - 1],     // insert into word1
          dp[i - 1][j - 1]  // replace
        );
      }
    }
  }
  return dp[m][n];
}

// 0/1 Knapsack: max value with weight capacity
// State: dp[i][w] = max value using first i items with capacity w
function knapsack(weights: number[], values: number[], capacity: number): number {
  const n = weights.length;
  // Space optimization: 1D dp (rolling array)
  const dp = new Array(capacity + 1).fill(0);

  for (let i = 0; i < n; i++) {
    // Iterate right to left to avoid using item i twice
    for (let w = capacity; w >= weights[i]; w--) {
      dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
    }
  }
  return dp[capacity];
}

// Unique Paths: robot in m×n grid, can only move right or down
// State: dp[i][j] = number of paths from top-left to (i, j)
function uniquePaths(m: number, n: number): number {
  const dp = new Array(n).fill(1); // first row: all 1s (only one way: all right)

  for (let i = 1; i < m; i++) {
    for (let j = 1; j < n; j++) {
      dp[j] += dp[j - 1]; // from top (dp[j]) + from left (dp[j-1])
    }
  }
  return dp[n - 1];
}

// Palindromic Substrings: count all palindromic substrings
// State: dp[i][j] = true if s[i..j] is a palindrome
function countSubstrings(s: string): number {
  const n = s.length;
  let count = 0;

  // Expand around center approach: O(n²) time, O(1) space
  function expandAroundCenter(left: number, right: number): void {
    while (left >= 0 && right < n && s[left] === s[right]) {
      count++;
      left--;
      right++;
    }
  }

  for (let i = 0; i < n; i++) {
    expandAroundCenter(i, i);     // odd length palindromes
    expandAroundCenter(i, i + 1); // even length palindromes
  }
  return count;
}

// Longest Palindromic Substring: O(n²) expand around center
function longestPalindrome(s: string): string {
  let start = 0, maxLen = 1;

  function expand(left: number, right: number): void {
    while (left >= 0 && right < s.length && s[left] === s[right]) {
      if (right - left + 1 > maxLen) {
        maxLen = right - left + 1;
        start = left;
      }
      left--; right++;
    }
  }

  for (let i = 0; i < s.length; i++) {
    expand(i, i);
    expand(i, i + 1);
  }
  return s.slice(start, start + maxLen);
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Wrong recurrence direction in 0/1 knapsack.** When using a 1D rolling array, iterate capacity from HIGH to LOW. Iterating low to high allows the same item to be used multiple times (becomes unbounded knapsack accidentally).

> ⚠️ **Off-by-one in dp array indexing.** Many DP problems use `dp[0]` as the base case for empty input. Then `dp[i]` represents the first i elements, not the ith element. Mixing these up by one index causes wrong results.

> ⚠️ **Mutating shared memoization cache between test cases.** If you define `memo` outside the function as a module-level variable, it persists between calls. Either create a new Map per call or use a closure.

> ⚠️ **Initializing dp array with 0 when the answer should be -Infinity or Infinity.** For "maximum product" or "minimum cost" problems, initialize with the correct neutral element.

> ⚠️ **Confusing LCS with LCS substring.** LCS (subsequence) can skip characters. LCS substring (contiguous) cannot. Different recurrences.

---

## 6. When to Use / Not Use

**DP applies when:**
- The problem asks for optimal value (min/max/count)
- The problem has optimal substructure (subproblem solutions combine to form full solution)
- The problem has overlapping subproblems (same subproblems appear repeatedly)
- The problem asks "how many ways" or "is it possible"

**DP does NOT apply when:**
- Greedy works (locally optimal = globally optimal) — greedy is simpler and faster
- The problem has no overlapping subproblems — plain recursion suffices
- Subproblems are independent — divide and conquer (not DP)

---

## 7. Real-World Scenario

Spell checkers use edit distance (Levenshtein) to suggest corrections:

```typescript
function spellSuggest(word: string, dictionary: string[], maxDistance = 2): string[] {
  return dictionary
    .map(dictWord => ({
      word: dictWord,
      dist: minDistance(word, dictWord)
    }))
    .filter(({ dist }) => dist <= maxDistance)
    .sort((a, b) => a.dist - b.dist)
    .map(({ word }) => word);
}
// "helo" → ["hello", "help", "hero"] (within edit distance 2)
```

A portfolio optimizer uses 0/1 knapsack to maximize return within a budget:

```typescript
interface Investment {
  name: string;
  cost: number;    // in thousands
  expectedReturn: number;
}

function optimizePortfolio(investments: Investment[], budget: number): Investment[] {
  const dp = new Array(budget + 1).fill(0);
  const chosen: boolean[][] = Array.from({ length: investments.length }, () => new Array(budget + 1).fill(false));

  for (let i = 0; i < investments.length; i++) {
    const { cost, expectedReturn } = investments[i];
    for (let w = budget; w >= cost; w--) {
      if (dp[w - cost] + expectedReturn > dp[w]) {
        dp[w] = dp[w - cost] + expectedReturn;
        chosen[i][w] = true;
      }
    }
  }

  // Backtrack to find which investments were chosen
  const result: Investment[] = [];
  let remainingBudget = budget;
  for (let i = investments.length - 1; i >= 0; i--) {
    if (chosen[i][remainingBudget]) {
      result.push(investments[i]);
      remainingBudget -= investments[i].cost;
    }
  }
  return result;
}
```

---

## 8. Interview Questions

**Q1: How do you identify if a problem needs dynamic programming?**
A: Look for: (1) asks for optimal value (min/max), count of ways, or possibility, (2) the problem can be broken into subproblems of the same type, (3) the naive recursive solution recomputes the same inputs. Classic signals: "minimum number of...", "maximum sum/product", "count distinct ways", "longest/shortest subsequence".

**Q2: What is the difference between memoization (top-down) and tabulation (bottom-up)?**
A: Memoization: recursive, caches results as computed. Only solves needed subproblems. Stack overhead. Tabulation: iterative, fills a table from small to large. Solves all subproblems but enables space optimization. Usually faster in practice.

**Q3: Explain the state transition for coin change.**
A: State: `dp[i]` = minimum coins to make amount i. Base case: `dp[0] = 0`. Transition: for each coin c and each amount i ≥ c, `dp[i] = min(dp[i], dp[i-c] + 1)`. The idea: if we can make amount `i-c` optimally, we can make amount `i` by adding one coin of value `c`.

**Q4: How would you solve 0/1 knapsack with space optimization?**
A: Use a 1D array instead of 2D. For each item, iterate the capacity from high to low. This prevents using the same item twice (which would happen if iterating low to high, as dp[w-cost] would already include the current item).

**Q5: What is the difference between Longest Common Subsequence and Longest Common Substring?**
A: LCS subsequence allows gaps — characters don't need to be adjacent. Recurrence: if chars match, extend by 1; else take max of excluding either char. LCS substring requires contiguous characters — if chars don't match, reset to 0. Different recurrences produce different results.

---

## 9. Exercises

**Exercise 1:** House Robber II — same as house robber but houses are arranged in a circle (first and last houses are adjacent, so you can't rob both).
*Hint: run house robber twice — once on houses[0..n-2], once on houses[1..n-1]. Take the max.*

**Exercise 2:** Longest Palindromic Subsequence — find the length of the longest subsequence of s that is a palindrome.
*Hint: reverse s and find LCS of s and reversed(s).*

**Exercise 3:** Partition Equal Subset Sum — can you partition an array into two subsets with equal sum?
*Hint: reduce to 0/1 knapsack — can you achieve sum/2 using elements of the array?*

**Exercise 4:** Maximum sum of a non-empty contiguous subarray (Kadane's algorithm).
*Hint: dp[i] = max subarray sum ending at index i. dp[i] = max(nums[i], dp[i-1] + nums[i]).*

**Exercise 5:** Number of ways to climb n stairs if you can jump 1, 2, or 3 steps at a time.
*Hint: generalize climbing stairs. dp[i] = dp[i-1] + dp[i-2] + dp[i-3].*

---

## 10. Further Reading

- CLRS Chapter 15 — Dynamic Programming
- [DP patterns — NeetCode roadmap](https://neetcode.io/roadmap)
- [Thinking in DP — labuladong](https://labuladong.online/algo/dynamic_programming/)
- LeetCode: #70 Climbing Stairs, #198 House Robber, #322 Coin Change, #1143 LCS, #72 Edit Distance, #416 Partition Equal Subset Sum, #300 LIS
