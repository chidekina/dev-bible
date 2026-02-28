# Trees & Binary Search Trees

## 1. What & Why

Trees are hierarchical data structures where each node has at most one parent and zero or more children. They model real-world hierarchies naturally: file systems, org charts, DOM trees, decision trees, and database indexes all use tree structures.

Binary Search Trees (BSTs) add an ordering property that makes search O(log n) in balanced cases. When balanced, they provide O(log n) insert, delete, and search — better than arrays for dynamic sorted data. Self-balancing variants (AVL, Red-Black) guarantee this balance and power the sorted data structures in most standard libraries (`std::map` in C++, `TreeMap` in Java).

Understanding trees is essential because they underlie so many systems, and tree traversal problems are ubiquitous in interviews.

---

## 2. Core Concepts

### Terminology

- **Root:** the topmost node (no parent)
- **Node:** any element in the tree
- **Leaf:** a node with no children
- **Edge:** the connection between a parent and child
- **Height:** longest path from a node to a leaf (root height = tree height)
- **Depth:** distance from root to a given node
- **Subtree:** a node and all its descendants
- **Level:** all nodes at the same depth

### Binary Tree

Each node has at most 2 children: left and right. No ordering constraint. Special forms:
- **Full binary tree:** every node has 0 or 2 children
- **Complete binary tree:** all levels fully filled except possibly the last, filled left to right
- **Perfect binary tree:** all leaves at the same level, all internal nodes have 2 children
- **Balanced binary tree:** heights of left and right subtrees differ by at most 1 for all nodes

### Binary Search Tree (BST)

For every node N: all nodes in N's left subtree have values < N.value, and all nodes in N's right subtree have values > N.value.

This invariant makes search O(h) where h = height. For a balanced tree, h = O(log n). For a degenerate tree (all nodes on one side), h = O(n).

### Self-Balancing BSTs

**AVL Tree:** After each insert/delete, checks balance factors (height difference of subtrees). If violated, performs rotations to restore balance. Strictly balanced — heights differ by at most 1. Faster lookup than Red-Black but more rotations on insert/delete.

**Red-Black Tree:** Each node is colored red or black. Balance is maintained through a relaxed set of color rules that ensure the longest path is at most 2× the shortest path. Used in most language standard libraries due to fewer rotations on mutation.

---

## 3. How It Works

```
BST example:
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13

Search for 6:
1. Start at root 8. 6 < 8 → go left
2. At 3. 6 > 3 → go right
3. At 6. Found! O(log n) average

Insert 5:
1. 5 < 8 → left
2. 5 > 3 → right
3. 5 < 6 → left
4. 4 exists. 5 > 4 → right of 4. Insert!

Delete 6 (has two children):
Option: replace with in-order successor (7, smallest in right subtree)
→ node becomes 7, delete original 7
```

**Traversals:**
```
Inorder (left, root, right): 1, 3, 4, 6, 7, 8, 10, 13, 14 — sorted output for BST!
Preorder (root, left, right): 8, 3, 1, 6, 4, 7, 10, 14, 13
Postorder (left, right, root): 1, 4, 7, 6, 3, 13, 14, 10, 8
Level-order (BFS): 8, 3, 10, 1, 6, 14, 4, 7, 13
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- Tree Node ---
class TreeNode {
  constructor(
    public val: number,
    public left: TreeNode | null = null,
    public right: TreeNode | null = null
  ) {}
}

// --- BST operations ---
class BST {
  private root: TreeNode | null = null;

  // Insert: O(log n) average, O(n) worst
  insert(val: number): void {
    this.root = this.insertNode(this.root, val);
  }

  private insertNode(node: TreeNode | null, val: number): TreeNode {
    if (node === null) return new TreeNode(val);
    if (val < node.val) node.left = this.insertNode(node.left, val);
    else if (val > node.val) node.right = this.insertNode(node.right, val);
    // Equal: ignore duplicates (or handle as needed)
    return node;
  }

  // Search: O(log n) average
  search(val: number): TreeNode | null {
    return this.searchNode(this.root, val);
  }

  private searchNode(node: TreeNode | null, val: number): TreeNode | null {
    if (node === null || node.val === val) return node;
    return val < node.val
      ? this.searchNode(node.left, val)
      : this.searchNode(node.right, val);
  }

  // Delete: O(log n) average
  delete(val: number): void {
    this.root = this.deleteNode(this.root, val);
  }

  private deleteNode(node: TreeNode | null, val: number): TreeNode | null {
    if (node === null) return null;

    if (val < node.val) {
      node.left = this.deleteNode(node.left, val);
    } else if (val > node.val) {
      node.right = this.deleteNode(node.right, val);
    } else {
      // Found the node to delete
      if (node.left === null) return node.right; // no left child
      if (node.right === null) return node.left; // no right child

      // Two children: replace with in-order successor (min of right subtree)
      const successor = this.findMin(node.right);
      node.val = successor.val;
      node.right = this.deleteNode(node.right, successor.val);
    }
    return node;
  }

  private findMin(node: TreeNode): TreeNode {
    while (node.left !== null) node = node.left;
    return node;
  }
}

// --- Traversals ---

// Inorder: O(n) time, O(h) space (call stack)
function inorder(root: TreeNode | null): number[] {
  if (root === null) return [];
  return [...inorder(root.left), root.val, ...inorder(root.right)];
}

// Iterative inorder: O(n) time, O(h) space — avoids recursion stack depth limit
function inorderIterative(root: TreeNode | null): number[] {
  const result: number[] = [];
  const stack: TreeNode[] = [];
  let current = root;

  while (current !== null || stack.length > 0) {
    // Go as far left as possible
    while (current !== null) {
      stack.push(current);
      current = current.left;
    }
    current = stack.pop()!;
    result.push(current.val);
    current = current.right;
  }
  return result;
}

// Preorder: O(n)
function preorder(root: TreeNode | null): number[] {
  if (root === null) return [];
  return [root.val, ...preorder(root.left), ...preorder(root.right)];
}

// Postorder: O(n)
function postorder(root: TreeNode | null): number[] {
  if (root === null) return [];
  return [...postorder(root.left), ...postorder(root.right), root.val];
}

// Level-order (BFS): O(n)
function levelOrder(root: TreeNode | null): number[][] {
  if (root === null) return [];
  const result: number[][] = [];
  const queue: TreeNode[] = [root];

  while (queue.length > 0) {
    const levelSize = queue.length;
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

// --- Validate BST: O(n) ---
// Pass down allowed [min, max] range for each node
function isValidBST(
  root: TreeNode | null,
  min = -Infinity,
  max = Infinity
): boolean {
  if (root === null) return true;
  if (root.val <= min || root.val >= max) return false;
  return (
    isValidBST(root.left, min, root.val) &&
    isValidBST(root.right, root.val, max)
  );
}

// --- Max Depth: O(n) ---
function maxDepth(root: TreeNode | null): number {
  if (root === null) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}

// --- Lowest Common Ancestor: O(n) ---
// For BST: use BST property for O(log n)
function lowestCommonAncestorBST(
  root: TreeNode | null,
  p: number,
  q: number
): TreeNode | null {
  if (root === null) return null;
  if (p < root.val && q < root.val) return lowestCommonAncestorBST(root.left, p, q);
  if (p > root.val && q > root.val) return lowestCommonAncestorBST(root.right, p, q);
  return root; // root is between p and q — it's the LCA
}

// For general binary tree: O(n)
function lowestCommonAncestor(
  root: TreeNode | null,
  p: number,
  q: number
): TreeNode | null {
  if (root === null || root.val === p || root.val === q) return root;
  const left = lowestCommonAncestor(root.left, p, q);
  const right = lowestCommonAncestor(root.right, p, q);
  if (left !== null && right !== null) return root; // p and q on different sides
  return left ?? right;
}

// --- Serialize / Deserialize binary tree: O(n) ---
function serialize(root: TreeNode | null): string {
  if (root === null) return "null";
  return `${root.val},${serialize(root.left)},${serialize(root.right)}`;
}

function deserialize(data: string): TreeNode | null {
  const nodes = data.split(",");
  let index = 0;

  function buildTree(): TreeNode | null {
    if (nodes[index] === "null") { index++; return null; }
    const node = new TreeNode(parseInt(nodes[index++]));
    node.left = buildTree();
    node.right = buildTree();
    return node;
  }
  return buildTree();
}

// --- Build BST from sorted array: O(n) ---
function sortedArrayToBST(nums: number[]): TreeNode | null {
  if (nums.length === 0) return null;
  const mid = Math.floor(nums.length / 2);
  const node = new TreeNode(nums[mid]);
  node.left = sortedArrayToBST(nums.slice(0, mid));
  node.right = sortedArrayToBST(nums.slice(mid + 1));
  return node;
}

// --- Kth Smallest in BST: O(n) worst, O(k + h) average ---
// Inorder traversal gives sorted order; return kth element
function kthSmallest(root: TreeNode | null, k: number): number {
  const stack: TreeNode[] = [];
  let current = root;
  let count = 0;

  while (current !== null || stack.length > 0) {
    while (current !== null) {
      stack.push(current);
      current = current.left;
    }
    current = stack.pop()!;
    count++;
    if (count === k) return current.val;
    current = current.right;
  }
  return -1;
}

// --- Check if two trees are identical: O(n) ---
function isSameTree(p: TreeNode | null, q: TreeNode | null): boolean {
  if (p === null && q === null) return true;
  if (p === null || q === null) return false;
  return (
    p.val === q.val &&
    isSameTree(p.left, q.left) &&
    isSameTree(p.right, q.right)
  );
}

// --- Check if tree is symmetric (mirror of itself): O(n) ---
function isSymmetric(root: TreeNode | null): boolean {
  function isMirror(left: TreeNode | null, right: TreeNode | null): boolean {
    if (left === null && right === null) return true;
    if (left === null || right === null) return false;
    return (
      left.val === right.val &&
      isMirror(left.left, right.right) &&
      isMirror(left.right, right.left)
    );
  }
  return isMirror(root?.left ?? null, root?.right ?? null);
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Assuming a BST is balanced.** A BST built by inserting already-sorted data degenerates into a linked list — O(n) for all operations. Always use a self-balancing BST (AVL/Red-Black) for guaranteed O(log n).

> ⚠️ **Confusing inorder traversal direction.** Inorder of a BST gives sorted ascending order only when you visit LEFT → ROOT → RIGHT. Reversing the order gives descending.

> ⚠️ **Off-by-one in level-order.** When grouping nodes by level, you must capture `queue.length` at the start of each level iteration — the queue grows as you process nodes.

> ⚠️ **Not handling the two-children case in BST deletion.** The most complex case is deleting a node with two children. You must find the in-order successor (or predecessor), copy its value, then delete the successor.

> ⚠️ **Stack overflow on deep trees.** Recursive traversal on a degenerate BST (acting as a linked list) causes O(n) recursion depth. Use iterative traversal for production code.

---

## 6. When to Use / Not Use

**Use a BST when:**
- You need a sorted, dynamic collection with O(log n) insert/delete/search
- You need to find the kth smallest/largest element efficiently
- You need range queries on sorted data
- You need to find the predecessor/successor of a value

**Use a self-balancing BST (AVL/Red-Black) when:**
- You need guaranteed O(log n) — use the language's built-in sorted set/map
- Insertions and deletions are frequent alongside lookups

**Do NOT use a BST when:**
- Data is static — use a sorted array + binary search (better cache performance)
- You only need O(1) lookup — use a hashmap
- Keys are arbitrary strings or complex objects — ordering is expensive

---

## 7. Real-World Scenario

A stock trading system needs to quickly find all orders between price $100 and $200 (range query). A sorted array requires O(n) scan. A BST solves it in O(log n + k) where k = number of results:

```typescript
function rangeQuery(
  root: TreeNode | null,
  low: number,
  high: number,
  result: number[] = []
): number[] {
  if (root === null) return result;

  // Prune: if root < low, only right subtree can have valid nodes
  if (root.val >= low) rangeQuery(root.left, low, high, result);

  // Include current node if in range
  if (root.val >= low && root.val <= high) result.push(root.val);

  // Prune: if root > high, only left subtree can have valid nodes
  if (root.val <= high) rangeQuery(root.right, low, high, result);

  return result;
}
```

---

## 8. Interview Questions

**Q1: What is the difference between a binary tree and a BST?**
A: A binary tree is any tree where each node has at most 2 children — no ordering constraint. A BST adds the invariant: left subtree values < node value < right subtree values. This makes search O(log n) on average.

**Q2: How does deletion work in a BST?**
A: Three cases: (1) Leaf — just remove. (2) One child — replace node with its child. (3) Two children — find the in-order successor (minimum of the right subtree), copy its value to the current node, then delete the successor (which has at most one right child — case 1 or 2).

**Q3: Why is a balanced BST important?**
A: An unbalanced BST can degenerate to O(n) for all operations (essentially a linked list). A balanced BST guarantees O(log n) height, keeping all operations at O(log n).

**Q4: What traversal gives sorted output for a BST?**
A: Inorder traversal (left → root → right) visits nodes in ascending sorted order.

**Q5: What is the difference between AVL and Red-Black trees?**
A: AVL trees are more strictly balanced (height difference ≤ 1) — faster lookups but more rotations on insert/delete. Red-Black trees are less strictly balanced but require fewer rotations on mutations — better for write-heavy workloads. Most language standard libraries use Red-Black trees.

**Q6: How do you check if a binary tree is a valid BST?**
A: Pass down a valid range [min, max] for each node. The root's range is (-∞, +∞). Left child's range is (min, parent.val). Right child's range is (parent.val, max). If any node violates its range, return false. O(n).

---

## 9. Exercises

**Exercise 1:** Build a balanced BST from a sorted array. The resulting tree should be the minimum possible height.
*Hint: always pick the middle element as root, recurse on left and right halves.*

**Exercise 2:** Find the kth smallest element in a BST in O(k + h) time.
*Hint: iterative inorder traversal — stop when you've visited k nodes.*

**Exercise 3:** Check if two binary trees are identical (same structure and same values).

**Exercise 4:** Find all paths from root to leaf where the sum of values equals a target.
*Hint: DFS with a running sum and path tracking. When you reach a leaf and sum matches target, record the path.*

**Exercise 5:** Flatten a binary tree into a linked list in-place (right pointers only, left = null). The order should match preorder traversal.
*Hint: "Morris Traversal" or reverse postorder approach.*

---

## 10. Further Reading

- CLRS Chapter 12 — Binary Search Trees, Chapter 13 — Red-Black Trees
- [Visualgo — BST visualization](https://visualgo.net/en/bst)
- [AVL Tree rotations explained](https://en.wikipedia.org/wiki/AVL_tree)
- LeetCode problems: #98 Validate BST, #102 Level Order Traversal, #105 Construct Binary Tree from Preorder/Inorder, #236 LCA, #297 Serialize/Deserialize
