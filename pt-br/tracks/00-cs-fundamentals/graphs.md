# Grafos

## 1. O Que É e Por Que Importa

Um grafo é uma coleção de nós (vértices) conectados por arestas. Ao contrário de árvores, grafos não têm raiz, não têm relação pai-filho e podem conter ciclos. Eles modelam quase qualquer estrutura de relacionamento: redes sociais, links entre páginas web, mapas de estradas, grafos de dependência, topologia de rede e máquinas de estado são todos grafos.

Algoritmos de grafo estão entre os mais poderosos da ciência da computação. BFS encontra caminhos mais curtos. DFS explora componentes conectados. Ordenação topológica ordena tarefas com dependências. Dijkstra encontra a rota mais barata. Entender esses algoritmos abre uma enorme classe de problemas do mundo real.

---

## 2. Conceitos Fundamentais

### Tipos de Grafos

| Propriedade | Variantes |
|---|---|
| Direção | Dirigido (arestas têm direção A→B) vs Não-dirigido (arestas são bidirecionais) |
| Peso | Ponderado (arestas têm custo) vs Não-ponderado (todas as arestas iguais) |
| Ciclos | Cíclico (contém um ciclo) vs Acíclico (sem ciclos) |
| Conectividade | Conectado (caminho entre quaisquer dois nós) vs Desconectado |

**DAG (Directed Acyclic Graph — Grafo Dirigido Acíclico):** dirigido, sem ciclos. Modela dependências de tarefas, sistemas de build, pré-requisitos de disciplinas.

### Representações de Grafos

**Lista de Adjacência:** `Map<Nó, Nó[]>` — para cada nó, lista seus vizinhos. Espaço: O(V + E). Melhor para grafos esparsos (a maioria dos grafos do mundo real).

**Matriz de Adjacência:** array 2D `matrix[i][j] = 1` se a aresta i→j existe. Espaço: O(V²). Melhor para grafos densos ou quando verificações frequentes de existência de aresta são necessárias.

**Lista de Arestas:** lista de tuplas `[de, para, peso]`. Simples mas lookup de aresta em O(E). Usada no MST de Kruskal.

### Visão Geral dos Algoritmos Principais

| Algoritmo | Caso de Uso | Complexidade |
|---|---|---|
| BFS | Caminho mais curto (não-ponderado), travessia por nível | O(V + E) |
| DFS | Busca de caminho, detecção de ciclo, componentes conectados | O(V + E) |
| Ordenação Topológica | Ordenação com dependências (DAGs) | O(V + E) |
| Dijkstra | Caminho mais curto (ponderado, não-negativo) | O((V + E) log V) |
| Union-Find | Detecção de ciclo, componentes conectados, MST | O(α(n)) ≈ O(1) amortizado |

---

## 3. Como Funciona

```
Exemplo de grafo (não-dirigido):
0 - 1 - 2
|   |
3   4

Lista de Adjacência:
0: [1, 3]
1: [0, 2, 4]
2: [1]
3: [0]
4: [1]

BFS a partir de 0: processa nível por nível usando uma fila
Nível 0: [0]
Nível 1: [1, 3]
Nível 2: [2, 4]
Visitas: 0, 1, 3, 2, 4

DFS a partir de 0: vai o mais fundo possível antes de retroceder
Ordem na pilha: 0 → 1 → 2 (retrocede) → 4 (retrocede) → 3
Visitas: 0, 1, 2, 4, 3
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
type Graph = Map<number, number[]>;

// --- Helpers para construir grafos ---
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

// --- BFS: O(V + E) de tempo, O(V) de espaço ---
// Retorna o caminho mais curto do início ao fim em um grafo não-ponderado
function bfs(graph: Graph, start: number, end: number): number[] | null {
  const visited = new Set<number>();
  const queue: number[][] = [[start]]; // fila de caminhos
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

// BFS para distância mais curta (mais eficiente em espaço, sem rastreamento de caminho)
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

// --- DFS: O(V + E) de tempo, O(V) de espaço ---
// DFS recursivo — encontra todos os nós acessíveis a partir do início
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

// DFS iterativo — evita stack overflow em grafos profundos
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

// --- Número de Ilhas: O(V + E) = O(m × n) ---
// Trata a grade como grafo; DFS/BFS a partir de cada '1' não visitado
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

// --- Ordenação Topológica (Algoritmo de Kahn / baseado em BFS): O(V + E) ---
// Requer um DAG. Retorna a ordem linearizada ou null se houver ciclo.
function topologicalSort(numNodes: number, edges: [number, number][]): number[] | null {
  const graph: Map<number, number[]> = new Map();
  const inDegree = new Array(numNodes).fill(0);

  for (let i = 0; i < numNodes; i++) graph.set(i, []);
  for (const [u, v] of edges) {
    graph.get(u)!.push(v);
    inDegree[v]++;
  }

  // Começa com todos os nós sem dependências
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

  // Se nem todos os nós foram processados, há um ciclo
  return order.length === numNodes ? order : null;
}

// Cronograma de disciplinas: dá para concluir todas as disciplinas? (detecção de ciclo em grafo dirigido)
function canFinish(numCourses: number, prerequisites: [number, number][]): boolean {
  return topologicalSort(numCourses, prerequisites) !== null;
}

// --- Ordenação Topológica (baseada em DFS): O(V + E) ---
function topologicalSortDFS(numNodes: number, edges: [number, number][]): number[] | null {
  const graph: Map<number, number[]> = new Map();
  for (let i = 0; i < numNodes; i++) graph.set(i, []);
  for (const [u, v] of edges) graph.get(u)!.push(v);

  const WHITE = 0, GRAY = 1, BLACK = 2; // não visitado, em progresso, concluído
  const color = new Array(numNodes).fill(WHITE);
  const result: number[] = [];
  let hasCycle = false;

  function dfs(node: number): void {
    if (hasCycle) return;
    color[node] = GRAY; // marca como em progresso
    for (const neighbor of graph.get(node)!) {
      if (color[neighbor] === GRAY) { hasCycle = true; return; } // aresta de retorno = ciclo
      if (color[neighbor] === WHITE) dfs(neighbor);
    }
    color[node] = BLACK; // totalmente processado
    result.push(node); // insere após todos os descendentes
  }

  for (let i = 0; i < numNodes; i++) {
    if (color[i] === WHITE) dfs(i);
  }

  return hasCycle ? null : result.reverse();
}

// --- Algoritmo de Dijkstra: O((V + E) log V) ---
// Caminho mais curto em grafo ponderado com pesos não-negativos
type WeightedGraph = Map<number, [number, number][]>; // nó → [vizinho, peso][]

function dijkstra(graph: WeightedGraph, start: number, end: number): number {
  const dist = new Map<number, number>();
  // Min-heap: [distância, nó] — simplificado com sort de array (implementação real usa fila de prioridade)
  const heap: [number, number][] = [[0, start]];

  // Inicializa todas as distâncias com infinito
  for (const node of graph.keys()) dist.set(node, Infinity);
  dist.set(start, 0);

  while (heap.length > 0) {
    heap.sort((a, b) => a[0] - b[0]); // O(V log V) com min-heap real
    const [currentDist, node] = heap.shift()!;

    if (currentDist > dist.get(node)!) continue; // entrada desatualizada
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

// --- Union-Find (Conjunto Disjunto): O(α(n)) ≈ O(1) amortizado ---
// Usado para: detecção de ciclo, número de componentes, MST de Kruskal
class UnionFind {
  private parent: number[];
  private rank: number[];
  public components: number;

  constructor(n: number) {
    this.parent = Array.from({ length: n }, (_, i) => i);
    this.rank = new Array(n).fill(0);
    this.components = n;
  }

  // Find com compressão de caminho
  find(x: number): number {
    if (this.parent[x] !== x) {
      this.parent[x] = this.find(this.parent[x]); // compressão de caminho
    }
    return this.parent[x];
  }

  // Union por rank
  union(x: number, y: number): boolean {
    const rootX = this.find(x);
    const rootY = this.find(y);
    if (rootX === rootY) return false; // já conectados — adicionar essa aresta cria um ciclo

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

// Detectar ciclo em grafo não-dirigido usando Union-Find
function hasCycle(n: number, edges: [number, number][]): boolean {
  const uf = new UnionFind(n);
  for (const [u, v] of edges) {
    if (!uf.union(u, v)) return true; // aresta conecta nós já conectados = ciclo
  }
  return false;
}

// --- Clonar Grafo: O(V + E) ---
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

## 5. Erros Comuns e Armadilhas

> ⚠️ **Esquecer de marcar nós como visitados em BFS/DFS.** Sem um set de visitados, você vai looping infinitamente em grafos cíclicos ou revisitar nós de forma redundante. Marque como visitado ANTES de adicionar à fila/pilha, não depois de remover.

> ⚠️ **Usar `queue.shift()` para performance de BFS.** `Array.shift()` é O(n). Para grafos grandes, use uma implementação de fila adequada (lista encadeada ou baseada em ponteiro). Para fins de entrevista, `shift()` é aceitável.

> ⚠️ **Ordenação topológica em grafo com ciclos.** O algoritmo de Kahn detecta isso automaticamente — se o tamanho da saída ≠ número de nós, existe um ciclo. DFS-based topo sort detecta uma aresta de retorno (GRAY → GRAY) para sinalizar um ciclo.

> ⚠️ **Dijkstra com pesos negativos.** Dijkstra falha com pesos de aresta negativos. Use Bellman-Ford (O(VE)) em vez disso.

> ⚠️ **Não inicializar todos os nós na lista de adjacência.** Se um nó não tem arestas de saída, pode não aparecer como chave. Sempre inicialize todos os nós ao construir o grafo.

---

## 6. Quando Usar / Não Usar

**BFS quando:**
- Caminho mais curto em grafo não-ponderado
- Travessia nível por nível (escada de palavras, graus de separação em redes sociais)

**DFS quando:**
- Detecção de ciclo em grafo dirigido
- Ordenação topológica
- Componentes conectados
- Busca de caminho quando você precisa de qualquer caminho (não necessariamente o mais curto)
- Problemas de backtracking mapeados para um grafo

**Dijkstra quando:**
- Caminho mais curto em grafo ponderado com pesos não-negativos

**Union-Find quando:**
- Detectar se dois nós estão no mesmo componente conectado
- Detectar ciclos em grafos não-dirigidos
- Árvore geradora mínima de Kruskal

---

## 7. Cenário do Mundo Real

Um sistema de deploy precisa implantar microsserviços em ordem de dependência. Se o serviço A depende do serviço B, B deve ser implantado primeiro. Isso é ordenação topológica:

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
      // dep deve vir antes do service
      edges.push([nameToIndex.get(dep)!, nameToIndex.get(service.name)!]);
    }
  }

  const order = topologicalSort(services.length, edges);
  if (order === null) return null; // dependência circular!
  return order.map(i => services[i].name);
}
```

Se existe uma dependência circular (A depende de B, B depende de A), a função retorna null — detecte e reporte antes de tentar o deploy.

---

## 8. Perguntas de Entrevista

**Q1: Quando você usa BFS vs DFS?**
R: BFS garante o caminho mais curto em um grafo não-ponderado. Use BFS quando o comprimento do caminho importa ou você precisa de travessia nível por nível. DFS usa menos memória (O(h) vs O(w) para BFS onde w = largura máxima). Use DFS para detecção de ciclo, ordenação topológica, existência de caminho e backtracking.

**Q2: Como o algoritmo de Dijkstra funciona?**
R: Mantém um min-heap de (distância, nó). Começa com distância 0 para a fonte. Sempre processa o nó com a menor distância conhecida. Para cada vizinho, se o caminho pelo nó atual é mais curto que o conhecido, atualiza e insere no heap. Nunca processa um nó duas vezes (pula entradas desatualizadas do heap). O((V + E) log V) com um heap binário.

**Q3: O que é uma ordenação topológica e quando é válida?**
R: Uma linearização de um DAG onde cada aresta dirigida u→v significa que u aparece antes de v na ordenação. Válida apenas para DAGs (grafos dirigidos acíclicos). Usada para sistemas de build, programação de disciplinas, dependências de tarefas.

**Q4: Como você detecta um ciclo em um grafo dirigido?**
R: DFS com coloração de nós: BRANCO (não visitado), CINZA (atualmente na pilha DFS), PRETO (totalmente processado). Se você encontrar um nó CINZA durante o DFS, há uma aresta de retorno = ciclo.

**Q5: O que é Union-Find e quais problemas ele resolve?**
R: Uma estrutura de dados que rastreia quais elementos pertencem ao mesmo conjunto. Suporta `find(x)` (a qual conjunto x pertence?) e `union(x, y)` (mescla os conjuntos contendo x e y). Compressão de caminho + union por rank dá O(α(n)) ≈ O(1) amortizado. Usado para: componentes conectados, detecção de ciclo em grafos não-dirigidos, MST de Kruskal.

**Q6: Qual é a diferença entre lista de adjacência e matriz de adjacência?**
R: Lista de adjacência: O(V + E) de espaço, verificação de aresta O(grau), eficiente para grafos esparsos. Matriz de adjacência: O(V²) de espaço, verificação de aresta O(1), eficiente para grafos densos ou quando a existência de aresta é consultada frequentemente.

---

## 9. Exercícios

**Exercício 1:** Encontre todos os caminhos do nó fonte 0 ao nó alvo n-1 em um grafo acíclico dirigido.
*Dica: DFS a partir de 0, rastreie o caminho atual, registre quando chegar a n-1.*

**Exercício 2:** Detecte um ciclo em um grafo dirigido. Retorne true se um ciclo existir.
*Dica: DFS com marcação de três cores (branco/cinza/preto).*

**Exercício 3:** Encontre a árvore geradora mínima de um grafo ponderado não-dirigido conectado usando o algoritmo de Kruskal.
*Dica: ordene arestas por peso, use Union-Find para adicionar arestas sem criar ciclos.*

**Exercício 4:** Implemente word ladder — encontre a sequência de transformação mais curta de beginWord para endWord, mudando uma letra por vez.
*Dica: BFS onde cada nó é uma palavra e arestas conectam palavras que diferem em uma letra. Pré-compute vizinhos ou gere-os na hora.*

**Exercício 5:** Dado uma grade de '0' (água) e '1' (terra), encontre o número de ilhas. Uma ilha é cercada por água e formada conectando terras adjacentes horizontalmente ou verticalmente.
*Dica: DFS/BFS a partir de cada '1' não visitado, marque toda a terra conectada como visitada.*

---

## 10. Leituras Complementares

- CLRS Capítulo 22 — Elementary Graph Algorithms (BFS, DFS, ordenação topológica)
- CLRS Capítulo 24 — Single-Source Shortest Paths (Dijkstra, Bellman-Ford)
- [Algoritmos de grafo — cp-algorithms.com](https://cp-algorithms.com/)
- Problemas LeetCode: #200 Number of Islands, #207 Course Schedule, #743 Network Delay Time (Dijkstra), #323 Number of Connected Components, #684 Redundant Connection (Union-Find)
