# Recursion & Backtracking

## 1. What & Why

Recursion is a technique where a function calls itself to solve a smaller instance of the same problem. It is the natural way to express algorithms on recursive data structures (trees, graphs) and to implement divide-and-conquer strategies (merge sort, quicksort).

Backtracking is a refined form of recursion for **constraint satisfaction and enumeration problems**: explore all possible candidates, and when a partial candidate cannot lead to a valid solution, "undo" the last decision and try the next option. It is the foundation of algorithms for combinatorics (permutations, subsets), puzzle solving (N-queens, Sudoku), and path finding.

---

## 2. Core Concepts

### Recursion Anatomy

Every recursive function has two parts:
1. **Base case:** the condition under which the function stops calling itself (termination)
2. **Recursive case:** the call that reduces the problem toward the base case

A recursive function without a base case runs forever (stack overflow). A recursive case that doesn't reduce the problem also runs forever.

### Call Stack Behavior

Each recursive call creates a new **stack frame** holding the function's local variables and the return address. These frames accumulate on the call stack. When a base case is reached, frames start popping. The maximum depth of recursion determines how many frames are on the stack simultaneously — the space complexity.

### Tail Recursion

A recursive call is **tail recursive** if it is the last operation in the function — the caller doesn't need to do anything after the recursive call returns. Tail calls can theoretically be optimized by the runtime (tail call optimization / TCO): the current stack frame is reused instead of creating a new one. JavaScript supports TCO in strict mode per the ES2015 spec, but in practice most JS engines (including V8) don't implement it. Don't rely on TCO in TypeScript/JavaScript.

### Stack Overflow

JavaScript's call stack has a limit (~10,000–15,000 frames in V8). Deep recursion on large inputs will overflow. Solutions:
- Convert recursion to iteration with an explicit stack
- Use memoization to reduce calls
- Use trampolining (advanced: convert recursive calls to continuations)

### Memoization

Cache results of expensive recursive calls. When the same arguments are seen again, return the cached result instead of recomputing. This transforms exponential-time recursion (like naive Fibonacci) into linear time. This is **top-down dynamic programming**.

### Backtracking Template

```
function backtrack(state, candidates):
  if state is a complete solution:
    record/return state
    return

  for each choice in candidates:
    if choice is valid:
      make the choice (modify state)
      backtrack(updated state, remaining candidates)
      undo the choice (restore state)  // the "backtrack" step
```

The "undo" step is what makes backtracking different from plain DFS — you restore state so future iterations start fresh.

---

## 3. How It Works

```
Recursion trace for factorial(4):
factorial(4)
  → 4 × factorial(3)
        → 3 × factorial(2)
              → 2 × factorial(1)
                    → 1 × factorial(0) = 1  ← base case
                  = 1 × 1 = 1
            = 2 × 1 = 2
        = 3 × 2 = 6
  = 4 × 6 = 24

Call stack depth = 5 frames (0 to 4)

Backtracking trace for permutations([1,2,3]):
Start: current=[], remaining=[1,2,3]
  Choose 1: current=[1], remaining=[2,3]
    Choose 2: current=[1,2], remaining=[3]
      Choose 3: current=[1,2,3] → RECORD [1,2,3]
      Undo 3: current=[1,2]
    Undo 2: current=[1]
    Choose 3: current=[1,3], remaining=[2]
      Choose 2: current=[1,3,2] → RECORD [1,3,2]
      ...
  Undo 1: current=[]
  Choose 2: ...
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- Basic Recursion ---

// Factorial: O(n) time, O(n) space (call stack)
function factorial(n: number): number {
  if (n === 0) return 1; // base case
  return n * factorial(n - 1);
}

// Fibonacci: naive O(2ⁿ) — call tree has 2^n nodes
function fibNaive(n: number): number {
  if (n <= 1) return n;
  return fibNaive(n - 1) + fibNaive(n - 2);
}

// Fibonacci: memoized — O(n) time, O(n) space
function fibMemo(n: number, memo = new Map<number, number>()): number {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n)!;
  const result = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  memo.set(n, result);
  return result;
}

// Fibonacci: iterative — O(n) time, O(1) space (no recursion)
function fibIterative(n: number): number {
  if (n <= 1) return n;
  let a = 0, b = 1;
  for (let i = 2; i <= n; i++) {
    [a, b] = [b, a + b];
  }
  return b;
}

// Power function: O(log n) using fast exponentiation
function power(base: number, exp: number): number {
  if (exp === 0) return 1;
  if (exp % 2 === 0) {
    const half = power(base, exp / 2);
    return half * half; // avoid recomputing
  }
  return base * power(base, exp - 1);
}

// Flatten a deeply nested object recursively
function flattenObject(obj: Record<string, unknown>, prefix = ""): Record<string, unknown> {
  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(obj)) {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === "object" && value !== null && !Array.isArray(value)) {
      Object.assign(result, flattenObject(value as Record<string, unknown>, fullKey));
    } else {
      result[fullKey] = value;
    }
  }
  return result;
}
// { a: { b: { c: 1 } } } → { "a.b.c": 1 }

// --- Convert recursion to iteration using explicit stack ---
// Tree inorder traversal: recursive vs iterative
class TreeNode {
  constructor(public val: number, public left: TreeNode | null = null, public right: TreeNode | null = null) {}
}

// Recursive: risks stack overflow on degenerate trees
function inorderRecursive(root: TreeNode | null): number[] {
  if (!root) return [];
  return [...inorderRecursive(root.left), root.val, ...inorderRecursive(root.right)];
}

// Iterative: safe for any depth
function inorderIterative(root: TreeNode | null): number[] {
  const result: number[] = [];
  const stack: TreeNode[] = [];
  let current = root;
  while (current !== null || stack.length > 0) {
    while (current !== null) { stack.push(current); current = current.left; }
    current = stack.pop()!;
    result.push(current.val);
    current = current.right;
  }
  return result;
}

// --- Backtracking Template ---

// Generate all permutations of an array: O(n! × n) time
function permutations<T>(arr: T[]): T[][] {
  const result: T[][] = [];

  function backtrack(current: T[], remaining: T[]): void {
    if (remaining.length === 0) {
      result.push([...current]); // snapshot — current is mutated
      return;
    }
    for (let i = 0; i < remaining.length; i++) {
      current.push(remaining[i]);                           // choose
      backtrack(current, [...remaining.slice(0, i), ...remaining.slice(i + 1)]);
      current.pop();                                        // undo
    }
  }

  backtrack([], arr);
  return result;
}

// Generate all subsets (power set): O(2ⁿ × n) time
function subsets<T>(arr: T[]): T[][] {
  const result: T[][] = [];

  function backtrack(start: number, current: T[]): void {
    result.push([...current]); // every state is a valid subset
    for (let i = start; i < arr.length; i++) {
      current.push(arr[i]);       // include arr[i]
      backtrack(i + 1, current);  // recurse on remaining elements
      current.pop();              // undo
    }
  }

  backtrack(0, []);
  return result;
}

// Generate all valid parentheses combinations: O(4ⁿ / √n) — Catalan number
function generateParentheses(n: number): string[] {
  const result: string[] = [];

  function backtrack(current: string, open: number, close: number): void {
    if (current.length === 2 * n) {
      result.push(current);
      return;
    }
    if (open < n) backtrack(current + "(", open + 1, close);
    if (close < open) backtrack(current + ")", open, close + 1);
  }

  backtrack("", 0, 0);
  return result;
}

// N-Queens: place N queens on N×N board so no two attack each other
function solveNQueens(n: number): string[][] {
  const result: string[][] = [];
  const board = Array.from({ length: n }, () => new Array(n).fill("."));

  // Track attacked columns and diagonals
  const cols = new Set<number>();
  const diag1 = new Set<number>(); // row - col (top-left to bottom-right)
  const diag2 = new Set<number>(); // row + col (top-right to bottom-left)

  function backtrack(row: number): void {
    if (row === n) {
      result.push(board.map(r => r.join("")));
      return;
    }
    for (let col = 0; col < n; col++) {
      if (cols.has(col) || diag1.has(row - col) || diag2.has(row + col)) continue;

      // Place queen
      board[row][col] = "Q";
      cols.add(col); diag1.add(row - col); diag2.add(row + col);

      backtrack(row + 1);

      // Remove queen (backtrack)
      board[row][col] = ".";
      cols.delete(col); diag1.delete(row - col); diag2.delete(row + col);
    }
  }

  backtrack(0);
  return result;
}

// Sudoku Solver
function solveSudoku(board: string[][]): boolean {
  const [row, col] = findEmpty(board);
  if (row === -1) return true; // no empty cell — solved!

  for (let num = 1; num <= 9; num++) {
    const c = num.toString();
    if (isValid(board, row, col, c)) {
      board[row][col] = c;         // place number
      if (solveSudoku(board)) return true;
      board[row][col] = ".";       // backtrack
    }
  }
  return false; // no valid number for this cell
}

function findEmpty(board: string[][]): [number, number] {
  for (let r = 0; r < 9; r++) {
    for (let c = 0; c < 9; c++) {
      if (board[r][c] === ".") return [r, c];
    }
  }
  return [-1, -1];
}

function isValid(board: string[][], row: number, col: number, num: string): boolean {
  // Check row
  if (board[row].includes(num)) return false;
  // Check column
  for (let r = 0; r < 9; r++) { if (board[r][col] === num) return false; }
  // Check 3×3 box
  const boxRow = Math.floor(row / 3) * 3;
  const boxCol = Math.floor(col / 3) * 3;
  for (let r = boxRow; r < boxRow + 3; r++) {
    for (let c = boxCol; c < boxCol + 3; c++) {
      if (board[r][c] === num) return false;
    }
  }
  return true;
}

// Word Search in Grid: does a word exist as a path of adjacent cells?
function wordSearch(board: string[][], word: string): boolean {
  const rows = board.length, cols = board[0].length;

  function dfs(r: number, c: number, index: number): boolean {
    if (index === word.length) return true; // all characters matched
    if (r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] !== word[index]) return false;

    const temp = board[r][c];
    board[r][c] = "#"; // mark as visited

    const found = dfs(r+1,c,index+1) || dfs(r-1,c,index+1) ||
                  dfs(r,c+1,index+1) || dfs(r,c-1,index+1);

    board[r][c] = temp; // restore (backtrack)
    return found;
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (dfs(r, c, 0)) return true;
    }
  }
  return false;
}

// Combination Sum: find all combinations that sum to target (can reuse elements)
function combinationSum(candidates: number[], target: number): number[][] {
  const result: number[][] = [];

  function backtrack(start: number, current: number[], remaining: number): void {
    if (remaining === 0) { result.push([...current]); return; }
    if (remaining < 0) return;

    for (let i = start; i < candidates.length; i++) {
      current.push(candidates[i]);
      backtrack(i, current, remaining - candidates[i]); // i, not i+1, allows reuse
      current.pop();
    }
  }

  candidates.sort((a, b) => a - b); // sort for early pruning
  backtrack(0, [], target);
  return result;
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Missing or wrong base case.** The most common recursion bug. A base case that is never reached causes infinite recursion and stack overflow. A base case that fires too early returns wrong results.

> ⚠️ **Not "undoing" the choice in backtracking.** If you modify state (push to array, mark visited, etc.) before recursing, you must reverse that modification after returning. Forgetting this corrupts state for sibling branches.

> ⚠️ **Pushing `current` directly instead of a copy.** `result.push(current)` adds a reference to the same array. When `current` is later modified (elements popped), all previously recorded entries in `result` change. Always push a copy: `result.push([...current])`.

> ⚠️ **Stack overflow on large inputs.** Recursive DFS on a graph with 100,000 nodes, or permutations of a large array, will overflow the call stack. Convert to iterative with an explicit stack, or increase Node.js stack size with `--stack-size`.

> ⚠️ **Exponential time without memoization.** If the same subproblems appear multiple times (overlapping subproblems), pure recursion is exponential. Add memoization or switch to bottom-up DP.

---

## 6. When to Use / Not Use

**Use recursion when:**
- The problem is defined recursively (trees, graphs, divide and conquer)
- The code is significantly cleaner recursive than iterative
- Depth is bounded and not large (< 1,000 levels)

**Use backtracking when:**
- Enumerating all solutions satisfying constraints
- Finding any solution that satisfies constraints
- Problems involving combinations, permutations, or arrangements with validity checks

**Convert to iteration when:**
- Recursion depth can be large (graphs, large datasets)
- Tail recursive and you need performance
- You've hit stack overflow

**Add memoization when:**
- Subproblems repeat (same arguments appear multiple times in the call tree)
- If most subproblems repeat, consider converting to bottom-up DP instead

---

## 7. Real-World Scenario

A file system traversal recursively processes all files in nested directories:

```typescript
import * as fs from "fs";
import * as path from "path";

function findFilesRecursive(dir: string, extension: string): string[] {
  const results: string[] = [];

  function traverse(currentDir: string): void {
    const entries = fs.readdirSync(currentDir, { withFileTypes: true });
    for (const entry of entries) {
      const fullPath = path.join(currentDir, entry.name);
      if (entry.isDirectory()) {
        traverse(fullPath); // recurse into subdirectory
      } else if (entry.name.endsWith(extension)) {
        results.push(fullPath);
      }
    }
  }

  traverse(dir);
  return results;
}
```

A configuration parser handles arbitrarily nested JSON:

```typescript
function validateConfig(config: unknown, schema: Record<string, unknown>): boolean {
  if (typeof config !== "object" || config === null) return false;
  const cfg = config as Record<string, unknown>;

  for (const [key, expectedType] of Object.entries(schema)) {
    if (!(key in cfg)) return false;
    if (typeof expectedType === "object") {
      // Recurse into nested schema
      if (!validateConfig(cfg[key], expectedType as Record<string, unknown>)) return false;
    } else {
      if (typeof cfg[key] !== expectedType) return false;
    }
  }
  return true;
}
```

---

## 8. Interview Questions

**Q1: What is the difference between recursion and iteration?**
A: Recursion uses the call stack to track state across repeated function calls. Iteration uses explicit loop variables. Recursion is more expressive for recursive structures (trees, graphs) but uses O(depth) stack space. Iteration is more memory-efficient. Any recursive algorithm can be converted to iterative using an explicit stack.

**Q2: How would you flatten a deeply nested object recursively?**
A: Recurse through the object's entries. If a value is a non-null object (and not an array), recurse into it with the current path prepended. Otherwise, add the key-value pair to the result. See the `flattenObject` function above.

**Q3: What is the time complexity of generating all permutations?**
A: O(n! × n). There are n! permutations, and each takes O(n) to copy into the result. The recursion tree has n! leaves and O(n × n!) total nodes.

**Q4: What is the key difference between backtracking and plain DFS?**
A: Both explore a decision tree. DFS typically marks nodes as visited permanently. Backtracking undoes changes after exploring a branch, restoring state for sibling branches. This is essential when the same node can appear in multiple solutions or when you need to explore all valid combinations.

**Q5: When does recursion lead to exponential time? How do you fix it?**
A: When subproblems overlap — the same inputs are computed multiple times (e.g., naive Fibonacci: fib(n-1) and fib(n-2) both compute fib(n-2)). Fix: memoize (top-down DP) to cache results, or rewrite as bottom-up DP.

---

## 9. Exercises

**Exercise 1:** Generate all valid combinations of n pairs of parentheses.
Input: n = 3 → Output: ["((()))", "(()())", "(())()", "()(())", "()()()"]
*Hint: track count of open and close parentheses. Only add "(" if open < n, only add ")" if close < open.*

**Exercise 2:** Given a phone number string, return all possible letter combinations (like old T9 keyboards).
Input: "23" → Output: ["ad", "ae", "af", "bd", "be", "bf", "cd", "ce", "cf"]
*Hint: backtracking over digit-to-letters map.*

**Exercise 3:** Find all combinations of numbers in an array that sum to a target (elements can be reused).
*Hint: sorted candidates + backtrack with start index. Allow same index to be used again.*

**Exercise 4:** Solve the N-Queens problem — place N queens on an N×N chessboard so that no two queens attack each other. Return all solutions.

**Exercise 5:** Implement a recursive deep clone of a JavaScript object (handling nested objects, arrays, and primitive values). Handle circular references.

---

## 10. Further Reading

- CLRS Chapter 4 — Divide and Conquer
- [Backtracking — patterns and templates](https://labuladong.online/algo/essential-technique/backtrack-framework/)
- [The Recursive Book of Recursion](https://inventwithpython.com/recursion/) — by Al Sweigart (free online)
- LeetCode: #46 Permutations, #78 Subsets, #22 Generate Parentheses, #51 N-Queens, #37 Sudoku Solver, #79 Word Search
