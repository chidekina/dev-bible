# Graphs

## 1. What & Why

A graph is a collection of nodes (vertices) connected by edges. Unlike trees, graphs have no root, no parent-child relationship, and can contain cycles. They model almost any relationship structure: social networks, web page links, road maps, dependency graphs, network topology, and state machines are all graphs.

Graph algorithms are among the most powerful in computer science. BFS finds shortest paths. DFS explores connected components. Topological sort orders tasks with dependencies. Dijkstra finds the cheapest route. Understanding these algorithms opens up a huge class of real-world problems.

---

## 2. Core Concepts

### Graph Types

| Property | Variants |
|---|---|
| Direction | Directed (edges have direction A→B) vs Undirected (edges are bidirectional) |
| Weight | Weighted (edges have cost) vs Unweighted (all edges equal) |
| Cycles | Cyclic (contains a cycle) vs Acyclic (no cycles) |
| Connectivity | Connected (path between any two nodes) vs Disconnected |

**DAG (Directed Acyclic Graph):** directed, no cycles. Models task dependencies, build systems, course prerequisites.

### Graph Representations

**Adjacency List:** `Map<Node, Node[]>` — for each node, list its neighbors. Space: O(V + E). Best for sparse graphs (most real-world graphs).

**Adjacency Matrix:** 2D array `matrix[i][j] = 1` if edge i→j exists. Space: O(V²). Best for dense graphs or when frequent edge existence checks are needed.

**Edge List:** list of `[from, to, weight]` tuples. Simple but O(E) edge lookup. Used in Kruskal's MST.

### Key Algorithms Overview

| Algorithm | Use Case | Complexity |
|---|---|---|
| BFS | Shortest path (unweighted), level traversal | O(V + E) |
| DFS | Path finding, cycle detection, connected components | O(V + E) |
| Topological Sort | Ordering with dependencies (DAGs) | O(V + E) |
| Dijkstra | Shortest path (weighted, non-negative) | O((V + E) log V) |
| Union-Find | Cycle detection, connected components, MST | O(α(n)) ≈ O(1) amortized |

---

## 3. How It Works

```
Graph example (undirected):
0 - 1 - 2
|   |
3   4

Adjacency List:
0: [1, 3]
1: [0, 2, 4]
2: [1]
3: [0]
4: [1]

BFS from 0: process level by level using a queue
Level 0: [0]
Level 1: [1, 3]
Level 2: [2, 4]
Visits: 0, 1, 3, 2, 4

DFS from 0: go as deep as possible before backtracking
Stack order: 0 → 1 → 2 (backtrack) → 4 (backtrack) → 3
Visits: 0, 1, 2, 4, 3
```

---

## 4. Code Examples (TypeScript)

```typescript
type Graph = Map<number, number[]>;

// --- Build graph helpers ---
function buildUndirectedGraph(edges: [number, number][], n: number): Graph {
  const graph: Graph = new Map();
  for (let i = 0; i < n; i++) graph.set(i, []);
  for (const [u, v] of edges) {
    graph.get(u)!.push(v);
    graph.get(v)!.push(u);
  }
  return graph;
}

function buildDirectedGraph(edges: [number, number][], n: number): Graph {
  const graph: Graph = new Map();
  for (let i = 0; i < n; i++) graph.set(i, []);
  for (const [u, v] of edges) {
    graph.get(u)!.push(v);
  }
  return graph;
}

// --- BFS: O(V + E) time, O(V) space ---
// Returns shortest path from start to end in an unweighted graph
function bfs(graph: Graph, start: number, end: number): number[] | null {
  const visited = new Set<number>();
  const queue: number[][] = [[start]]; // queue of paths
  visited.add(start);

  while (queue.length > 0) {
    const path = queue.shift()!;
    const node = path[path.length - 1];

    if (node === end) return path;

    for (const neighbor of graph.get(node) ?? []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push([...path, neighbor]);
      }
    }
  }
  return null;
}

// BFS for shortest distance (more space-efficient, no path tracking)
function bfsDistance(graph: Graph, start: number): Map<number, number> {
  const dist = new Map<number, number>();
  dist.set(start, 0);
  const queue = [start];

  while (queue.length > 0) {
    const node = queue.shift()!;
    for (const neighbor of graph.get(node) ?? []) {
      if (!dist.has(neighbor)) {
        dist.set(neighbor, dist.get(node)! + 1);
        queue.push(neighbor);
      }
    }
  }
  return dist;
}

// --- DFS: O(V + E) time, O(V) space ---
// Recursive DFS — finds all nodes reachable from start
function dfsRecursive(
  graph: Graph,
  node: number,
  visited = new Set<number>()
): number[] {
  if (visited.has(node)) return [];
  visited.add(node);
  const result = [node];
  for (const neighbor of graph.get(node) ?? []) {
    result.push(...dfsRecursive(graph, neighbor, visited));
  }
  return result;
}

// Iterative DFS — avoids stack overflow for deep graphs
function dfsIterative(graph: Graph, start: number): number[] {
  const visited = new Set<number>();
  const stack = [start];
  const result: number[] = [];

  while (stack.length > 0) {
    const node = stack.pop()!;
    if (visited.has(node)) continue;
    visited.add(node);
    result.push(node);
    for (const neighbor of graph.get(node) ?? []) {
      if (!visited.has(neighbor)) stack.push(neighbor);
    }
  }
  return result;
}

// --- Number of Islands: O(V + E) = O(m × n) ---
// Treat grid as graph; DFS/BFS from each unvisited '1'
function numIslands(grid: string[][]): number {
  const rows = grid.length, cols = grid[0].length;
  const visited = Array.from({ length: rows }, () => new Array(cols).fill(false));
  let count = 0;

  function dfs(r: number, c: number): void {
    if (r < 0 || r >= rows || c < 0 || c >= cols) return;
    if (visited[r][c] || grid[r][c] === "0") return;
    visited[r][c] = true;
    dfs(r + 1, c); dfs(r - 1, c);
    dfs(r, c + 1); dfs(r, c - 1);
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === "1" && !visited[r][c]) {
        dfs(r, c);
        count++;
      }
    }
  }
  return count;
}

// --- Topological Sort (Kahn's Algorithm / BFS-based): O(V + E) ---
// Requires a DAG. Returns linearized order or null if cycle exists.
function topologicalSort(numNodes: number, edges: [number, number][]): number[] | null {
  const graph: Map<number, number[]> = new Map();
  const inDegree = new Array(numNodes).fill(0);

  for (let i = 0; i < numNodes; i++) graph.set(i, []);
  for (const [u, v] of edges) {
    graph.get(u)!.push(v);
    inDegree[v]++;
  }

  // Start with all nodes that have no dependencies
  const queue: number[] = [];
  for (let i = 0; i < numNodes; i++) {
    if (inDegree[i] === 0) queue.push(i);
  }

  const order: number[] = [];
  while (queue.length > 0) {
    const node = queue.shift()!;
    order.push(node);
    for (const neighbor of graph.get(node)!) {
      inDegree[neighbor]--;
      if (inDegree[neighbor] === 0) queue.push(neighbor);
    }
  }

  // If not all nodes processed, there's a cycle
  return order.length === numNodes ? order : null;
}

// Course Schedule: can you finish all courses? (cycle detection in directed graph)
function canFinish(numCourses: number, prerequisites: [number, number][]): boolean {
  return topologicalSort(numCourses, prerequisites) !== null;
}

// --- Topological Sort (DFS-based): O(V + E) ---
function topologicalSortDFS(numNodes: number, edges: [number, number][]): number[] | null {
  const graph: Map<number, number[]> = new Map();
  for (let i = 0; i < numNodes; i++) graph.set(i, []);
  for (const [u, v] of edges) graph.get(u)!.push(v);

  const WHITE = 0, GRAY = 1, BLACK = 2; // unvisited, in-progress, done
  const color = new Array(numNodes).fill(WHITE);
  const result: number[] = [];
  let hasCycle = false;

  function dfs(node: number): void {
    if (hasCycle) return;
    color[node] = GRAY; // mark as in-progress
    for (const neighbor of graph.get(node)!) {
      if (color[neighbor] === GRAY) { hasCycle = true; return; } // back edge = cycle
      if (color[neighbor] === WHITE) dfs(neighbor);
    }
    color[node] = BLACK; // fully processed
    result.push(node); // push after all descendants
  }

  for (let i = 0; i < numNodes; i++) {
    if (color[i] === WHITE) dfs(i);
  }

  return hasCycle ? null : result.reverse();
}

// --- Dijkstra's Algorithm: O((V + E) log V) ---
// Shortest path in weighted graph with non-negative weights
type WeightedGraph = Map<number, [number, number][]>; // node → [neighbor, weight][]

function dijkstra(graph: WeightedGraph, start: number, end: number): number {
  const dist = new Map<number, number>();
  // Min-heap: [distance, node] — simplified with array sort (real impl uses priority queue)
  const heap: [number, number][] = [[0, start]];

  // Initialize all distances to infinity
  for (const node of graph.keys()) dist.set(node, Infinity);
  dist.set(start, 0);

  while (heap.length > 0) {
    heap.sort((a, b) => a[0] - b[0]); // O(V log V) with real min-heap
    const [currentDist, node] = heap.shift()!;

    if (currentDist > dist.get(node)!) continue; // stale entry
    if (node === end) return currentDist;

    for (const [neighbor, weight] of graph.get(node) ?? []) {
      const newDist = currentDist + weight;
      if (newDist < dist.get(neighbor)!) {
        dist.set(neighbor, newDist);
        heap.push([newDist, neighbor]);
      }
    }
  }
  return dist.get(end) ?? Infinity;
}

// --- Union-Find (Disjoint Set Union): O(α(n)) ≈ O(1) amortized ---
// Used for: cycle detection, number of components, Kruskal's MST
class UnionFind {
  private parent: number[];
  private rank: number[];
  public components: number;

  constructor(n: number) {
    this.parent = Array.from({ length: n }, (_, i) => i);
    this.rank = new Array(n).fill(0);
    this.components = n;
  }

  // Find with path compression
  find(x: number): number {
    if (this.parent[x] !== x) {
      this.parent[x] = this.find(this.parent[x]); // path compression
    }
    return this.parent[x];
  }

  // Union by rank
  union(x: number, y: number): boolean {
    const rootX = this.find(x);
    const rootY = this.find(y);
    if (rootX === rootY) return false; // already connected — adding this edge creates a cycle

    if (this.rank[rootX] < this.rank[rootY]) {
      this.parent[rootX] = rootY;
    } else if (this.rank[rootX] > this.rank[rootY]) {
      this.parent[rootY] = rootX;
    } else {
      this.parent[rootY] = rootX;
      this.rank[rootX]++;
    }
    this.components--;
    return true;
  }

  connected(x: number, y: number): boolean {
    return this.find(x) === this.find(y);
  }
}

// Detect cycle in undirected graph using Union-Find
function hasCycle(n: number, edges: [number, number][]): boolean {
  const uf = new UnionFind(n);
  for (const [u, v] of edges) {
    if (!uf.union(u, v)) return true; // edge connects already-connected nodes = cycle
  }
  return false;
}

// --- Clone Graph: O(V + E) ---
class GraphNode {
  val: number;
  neighbors: GraphNode[];
  constructor(val = 0, neighbors: GraphNode[] = []) {
    this.val = val;
    this.neighbors = neighbors;
  }
}

function cloneGraph(node: GraphNode | null): GraphNode | null {
  if (node === null) return null;
  const clones = new Map<GraphNode, GraphNode>();

  function clone(n: GraphNode): GraphNode {
    if (clones.has(n)) return clones.get(n)!;
    const copy = new GraphNode(n.val);
    clones.set(n, copy);
    copy.neighbors = n.neighbors.map(neighbor => clone(neighbor));
    return copy;
  }
  return clone(node);
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Forgetting to mark nodes as visited in BFS/DFS.** Without a visited set, you'll loop infinitely in cyclic graphs or redundantly revisit nodes. Mark as visited BEFORE adding to the queue/stack, not after dequeuing.

> ⚠️ **Using `queue.shift()` for BFS performance.** `Array.shift()` is O(n). For large graphs, use a proper queue implementation (linked list or pointer-based). For interview purposes, `shift()` is acceptable.

> ⚠️ **Topological sort on a graph with cycles.** Kahn's algorithm detects this automatically — if the output size ≠ number of nodes, a cycle exists. DFS-based topo sort detects a back edge (GRAY → GRAY) to signal a cycle.

> ⚠️ **Dijkstra with negative weights.** Dijkstra fails with negative edge weights. Use Bellman-Ford (O(VE)) instead.

> ⚠️ **Not initializing all nodes in the adjacency list.** If a node has no outgoing edges, it might not appear as a key. Always initialize all nodes when building the graph.

---

## 6. When to Use / Not Use

**BFS when:**
- Shortest path in an unweighted graph
- Level-by-level traversal (word ladder, social network degrees)

**DFS when:**
- Cycle detection in a directed graph
- Topological sort
- Connected components
- Path finding when you need any path (not necessarily shortest)
- Backtracking problems mapped to a graph

**Dijkstra when:**
- Shortest path in a weighted graph with non-negative weights

**Union-Find when:**
- Detecting if two nodes are in the same connected component
- Detecting cycles in undirected graphs
- Kruskal's minimum spanning tree

---

## 7. Real-World Scenario

A deployment system needs to deploy microservices in dependency order. If service A depends on service B, B must be deployed first. This is topological sort:

```typescript
interface Service {
  name: string;
  dependencies: string[];
}

function deploymentOrder(services: Service[]): string[] | null {
  const nameToIndex = new Map<string, number>();
  services.forEach((s, i) => nameToIndex.set(s.name, i));

  const edges: [number, number][] = [];
  for (const service of services) {
    for (const dep of service.dependencies) {
      // dep must come before service
      edges.push([nameToIndex.get(dep)!, nameToIndex.get(service.name)!]);
    }
  }

  const order = topologicalSort(services.length, edges);
  if (order === null) return null; // circular dependency!
  return order.map(i => services[i].name);
}
```

If a circular dependency exists (A depends on B, B depends on A), the function returns null — detect and report before attempting deployment.

---

## 8. Interview Questions

**Q1: When do you use BFS vs DFS?**
A: BFS guarantees the shortest path in an unweighted graph. Use BFS when path length matters or you need level-order traversal. DFS uses less memory (O(h) vs O(w) for BFS where w = max width). Use DFS for cycle detection, topological sort, path existence, and backtracking.

**Q2: How does Dijkstra's algorithm work?**
A: Maintain a min-heap of (distance, node). Start with distance 0 for the source. Always process the node with the smallest known distance. For each neighbor, if the path through the current node is shorter than known, update and push to heap. Never process a node twice (skip stale heap entries). O((V + E) log V) with a binary heap.

**Q3: What is a topological sort and when is it valid?**
A: A linearization of a DAG where every directed edge u→v means u appears before v in the ordering. Only valid for DAGs (directed acyclic graphs). Used for build systems, course scheduling, task dependencies.

**Q4: How do you detect a cycle in a directed graph?**
A: DFS with node coloring: WHITE (unvisited), GRAY (currently on the DFS stack), BLACK (fully processed). If you encounter a GRAY node during DFS, there's a back edge = cycle.

**Q5: What is Union-Find and what problems does it solve?**
A: A data structure tracking which elements belong to the same set. Supports `find(x)` (which set does x belong to?) and `union(x, y)` (merge the sets containing x and y). Path compression + union by rank gives O(α(n)) ≈ O(1) amortized. Used for: connected components, cycle detection in undirected graphs, Kruskal's MST.

**Q6: What is the difference between adjacency list and adjacency matrix?**
A: Adjacency list: O(V + E) space, O(degree) edge check, efficient for sparse graphs. Adjacency matrix: O(V²) space, O(1) edge check, efficient for dense graphs or when edge existence is frequently queried.

---

## 9. Exercises

**Exercise 1:** Find all paths from source node 0 to target node n-1 in a directed acyclic graph.
*Hint: DFS from 0, track the current path, record when you reach n-1.*

**Exercise 2:** Detect a cycle in a directed graph. Return true if a cycle exists.
*Hint: DFS with three-color marking (white/gray/black).*

**Exercise 3:** Find the minimum spanning tree of a connected weighted undirected graph using Kruskal's algorithm.
*Hint: sort edges by weight, use Union-Find to add edges without creating cycles.*

**Exercise 4:** Implement word ladder — find the shortest transformation sequence from beginWord to endWord, changing one letter at a time.
*Hint: BFS where each node is a word and edges connect words that differ by one letter. Precompute neighbors or generate them on the fly.*

**Exercise 5:** Given a grid of '0' (water) and '1' (land), find the number of islands. An island is surrounded by water and formed by connecting adjacent lands horizontally or vertically.
*Hint: DFS/BFS from each unvisited '1', mark all connected land as visited.*

---

## 10. Further Reading

- CLRS Chapter 22 — Elementary Graph Algorithms (BFS, DFS, topological sort)
- CLRS Chapter 24 — Single-Source Shortest Paths (Dijkstra, Bellman-Ford)
- [Graph algorithms — cp-algorithms.com](https://cp-algorithms.com/)
- LeetCode problems: #200 Number of Islands, #207 Course Schedule, #743 Network Delay Time (Dijkstra), #323 Number of Connected Components, #684 Redundant Connection (Union-Find)
