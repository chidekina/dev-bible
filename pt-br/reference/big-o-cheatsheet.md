# Big-O Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## Classes de Complexidade (Do Melhor ao Pior)

| Notação | Nome | Exemplo |
|---------|------|---------|
| O(1) | Constante | Acesso por índice em array, lookup em hash map |
| O(log n) | Logarítmica | Busca binária, operações em BST balanceada |
| O(n) | Linear | Busca linear, loop único |
| O(n log n) | Linearítmica | Merge sort, heap sort |
| O(n²) | Quadrática | Bubble sort, loops aninhados |
| O(n³) | Cúbica | Multiplicação de matrizes ingênua |
| O(2ⁿ) | Exponencial | Fibonacci recursivo, conjunto potência |
| O(n!) | Fatorial | Geração de permutações, TSP por força bruta |

---

## Operações em Estruturas de Dados

### Array / Array Dinâmico (JavaScript Array, list do Python)

| Operação | Média | Pior caso |
|----------|-------|-----------|
| Acesso por índice | O(1) | O(1) |
| Busca (não ordenado) | O(n) | O(n) |
| Busca (ordenado, binária) | O(log n) | O(log n) |
| Inserção no final (amortizado) | O(1) | O(n) — resize |
| Inserção no início | O(n) | O(n) |
| Inserção em índice arbitrário | O(n) | O(n) |
| Remoção no final | O(1) | O(1) |
| Remoção no início | O(n) | O(n) |
| Remoção em índice arbitrário | O(n) | O(n) |

**Espaço:** O(n)

---

### Linked List (Simples / Dupla)

| Operação | Simples | Dupla |
|----------|---------|-------|
| Acesso por índice | O(n) | O(n) |
| Busca | O(n) | O(n) |
| Inserção no início | O(1) | O(1) |
| Inserção no final (com ponteiro) | O(1) | O(1) |
| Inserção em índice arbitrário | O(n) | O(n) |
| Remoção no início | O(1) | O(1) |
| Remoção no final | O(n) | O(1) |
| Remoção com referência ao nó | O(n) | O(1) |

**Espaço:** O(n)

---

### Stack / Queue

| Operação | Stack | Queue |
|----------|-------|-------|
| Push / Enqueue | O(1) | O(1) |
| Pop / Dequeue | O(1) | O(1) |
| Peek | O(1) | O(1) |
| Busca | O(n) | O(n) |

**Espaço:** O(n)
> Implemente Stack com array ou linked list. Implemente Queue com linked list dupla ou buffer circular para evitar Dequeue em O(n).

---

### Hash Map / Hash Set

| Operação | Média | Pior caso (todas colisões) |
|----------|-------|---------------------------|
| Inserção | O(1) | O(n) |
| Remoção | O(1) | O(n) |
| Lookup / Contains | O(1) | O(n) |

**Espaço:** O(n)
> O pior caso é raro com uma boa função hash. Load factor mantido < 0,75 aciona resize.

---

### Árvore de Busca Binária (BST)

| Operação | Média (balanceada) | Pior caso (degenerada/linear) |
|----------|--------------------|-------------------------------|
| Acesso / Busca | O(log n) | O(n) |
| Inserção | O(log n) | O(n) |
| Remoção | O(log n) | O(n) |

**Espaço:** O(n)
> Use variantes auto-balanceadas (AVL, Red-Black) para garantir O(log n).

---

### BST Balanceada (AVL / Red-Black Tree)

| Operação | Média | Pior caso |
|----------|-------|-----------|
| Acesso / Busca | O(log n) | O(log n) |
| Inserção | O(log n) | O(log n) |
| Remoção | O(log n) | O(log n) |

**Espaço:** O(n)

---

### Heap (Heap Binária)

| Operação | Tempo |
|----------|-------|
| Encontrar min/max | O(1) |
| Inserção | O(log n) |
| Remoção do min/max (extração) | O(log n) |
| Construir heap (heapify array) | O(n) |
| Remoção de elemento arbitrário | O(log n) |

**Espaço:** O(n)

---

### Trie (Árvore de Prefixos)

| Operação | Tempo |
|----------|-------|
| Inserção | O(m) — m = tamanho da chave |
| Busca | O(m) |
| Starts-with (prefixo) | O(m) |
| Remoção | O(m) |

**Espaço:** O(n × m) — n chaves, m comprimento médio

---

### Grafo

| Representação | Espaço | Adicionar Vértice | Adicionar Aresta | Remover Aresta | Consultar Aresta |
|---------------|--------|-------------------|------------------|----------------|-----------------|
| Matriz de Adjacência | O(V²) | O(V²) | O(1) | O(1) | O(1) |
| Lista de Adjacência | O(V+E) | O(1) | O(1) | O(E) | O(V) |

> V = vértices, E = arestas

---

## Algoritmos de Ordenação

| Algoritmo | Melhor | Médio | Pior | Espaço | Estável? |
|-----------|--------|-------|------|--------|---------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Sim |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | Não |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Sim |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Sim |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | Não |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | Não |
| Tim Sort | O(n) | O(n log n) | O(n log n) | O(n) | Sim |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Sim |
| Radix Sort | O(nk) | O(nk) | O(nk) | O(n+k) | Sim |
| Bucket Sort | O(n+k) | O(n+k) | O(n²) | O(n) | Sim |

> k = intervalo de valores. Tim Sort é utilizado em Python, Java e V8.

---

## Algoritmos de Busca

| Algoritmo | Melhor | Médio | Pior | Espaço | Requisito |
|-----------|--------|-------|------|--------|-----------|
| Busca Linear | O(1) | O(n) | O(n) | O(1) | Nenhum |
| Busca Binária | O(1) | O(log n) | O(log n) | O(1) | Array ordenado |
| Jump Search | O(1) | O(√n) | O(√n) | O(1) | Array ordenado |
| Busca por Interpolação | O(1) | O(log log n) | O(n) | O(1) | Ordenado, uniforme |
| Busca Exponencial | O(1) | O(log n) | O(log n) | O(1) | Array ordenado |

---

## Algoritmos de Grafo

| Algoritmo | Tempo | Espaço | Caso de Uso |
|-----------|-------|--------|-------------|
| BFS | O(V+E) | O(V) | Caminho mais curto (sem peso), ordem por nível |
| DFS | O(V+E) | O(V) | Detecção de ciclos, ordenação topológica |
| Dijkstra | O((V+E) log V) | O(V) | Caminho mais curto (pesos não-negativos) |
| Bellman-Ford | O(VE) | O(V) | Caminho mais curto (pesos negativos) |
| Floyd-Warshall | O(V³) | O(V²) | Caminho mais curto entre todos os pares |
| A* | O(E log V) | O(V) | Caminho mais curto heurístico |
| Kruskal MST | O(E log E) | O(V) | Árvore geradora mínima |
| Prim MST | O(E log V) | O(V) | Árvore geradora mínima |
| Ordenação Topológica | O(V+E) | O(V) | Ordenação de DAG (dependências) |

---

## Programação Dinâmica e Recursão

| Problema | Ingênuo | PD (memoização/tabulação) |
|----------|---------|--------------------------|
| Fibonacci | O(2ⁿ) | O(n) tempo, O(1) espaço (bottom-up) |
| Maior Subsequência Comum | O(2ⁿ) | O(m×n) |
| Mochila 0/1 | O(2ⁿ) | O(n×W) |
| Troco Mínimo | O(2ⁿ) | O(n×quantia) |
| Maior Subsequência Crescente | O(2ⁿ) | O(n²) ou O(n log n) |
| Distância de Edição | O(3ⁿ) | O(m×n) |
| Multiplicação em Cadeia de Matrizes | O(2ⁿ) | O(n³) |

---

## Complexidade de Espaço — Padrões Comuns

| Padrão | Espaço |
|--------|--------|
| Variáveis simples | O(1) |
| Array de tamanho n | O(n) |
| Matriz 2D n×m | O(n×m) |
| Profundidade de chamada recursiva d | O(d) na pilha de chamadas |
| DFS em grafo | O(V) — pilha de chamadas |
| BFS em grafo | O(V) — fila |
| Auxiliar do merge sort | O(n) |
| Pilha de chamadas do quick sort | O(log n) médio, O(n) pior caso |
| Tabela de memoização | O(subproblemas) |

---

## Análise Amortizada

| Operação | Amortizado | Por quê |
|----------|-----------|---------|
| Append em array dinâmico | O(1) | O(n) raro amortizado ao longo de n operações |
| Push em stack (com array redimensionável) | O(1) | Mesmo raciocínio acima |
| Operações em splay tree | O(log n) | Balanceamento auto-ajustável |
| Inserção em hash table (com resize) | O(1) | Resize dobra a capacidade |

---

## Regras Práticas

- **Ignorar constantes:** O(2n) = O(n)
- **Ignorar termos de menor ordem:** O(n² + n) = O(n²)
- **Múltiplos loops em sequência:** O(n) + O(n) = O(n)
- **Loops aninhados:** O(n) dentro de O(n) = O(n²)
- **Dividir e conquistar (divisão ao meio):** geralmente O(log n) ou O(n log n)
- **Recursão sem memoização em subproblemas sobrepostos:** frequentemente exponencial
- **Cada elemento interage com todos os outros:** O(n²)
- **Ordenação é o seu piso:** se precisa ordenar, não dá para vencer O(n log n) (baseado em comparação)

---

## Guia Rápido de Decisão

```
Precisa de lookup rápido?         → Hash Map O(1)
Precisa de ordem ordenada?        → BST balanceada O(log n) / array ordenado
Precisa de min/max rápido?        → Heap O(1) peek, O(log n) extração
Precisa de busca por prefixo?     → Trie O(m)
Precisa de caminho mais curto?    → BFS (sem peso) / Dijkstra (com peso)
Precisa de ordenação/dependências?→ Ordenação Topológica
Precisa buscar em array ordenado? → Busca Binária O(log n)
Precisa de sort estável?          → Merge Sort / Tim Sort
Precisa de sort in-place?         → Quick Sort / Heap Sort
```
