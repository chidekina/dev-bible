# Árvores e Árvores Binárias de Busca

## 1. O Que É e Por Que Importa

Árvores são estruturas de dados hierárquicas onde cada nó tem no máximo um pai e zero ou mais filhos. Elas modelam hierarquias do mundo real de forma natural: sistemas de arquivos, organogramas, árvores DOM, árvores de decisão e índices de banco de dados todos usam estruturas de árvore.

Árvores Binárias de Busca (BSTs) adicionam uma propriedade de ordenação que torna a busca O(log n) em casos balanceados. Quando balanceadas, oferecem inserção, remoção e busca em O(log n) — melhor que arrays para dados ordenados dinâmicos. Variantes auto-balanceadas (AVL, Red-Black) garantem esse balanceamento e sustentam as estruturas de dados ordenadas na maioria das bibliotecas padrão (`std::map` em C++, `TreeMap` em Java).

Entender árvores é essencial porque elas sustentam tantos sistemas, e problemas de travessia de árvore são ubíquos em entrevistas.

---

## 2. Conceitos Fundamentais

### Terminologia

- **Raiz:** o nó mais alto (sem pai)
- **Nó:** qualquer elemento na árvore
- **Folha:** um nó sem filhos
- **Aresta:** a conexão entre pai e filho
- **Altura:** caminho mais longo de um nó até uma folha (altura da raiz = altura da árvore)
- **Profundidade:** distância da raiz até um dado nó
- **Subárvore:** um nó e todos os seus descendentes
- **Nível:** todos os nós na mesma profundidade

### Árvore Binária

Cada nó tem no máximo 2 filhos: esquerdo e direito. Sem restrição de ordenação. Formas especiais:
- **Árvore binária cheia:** todo nó tem 0 ou 2 filhos
- **Árvore binária completa:** todos os níveis completamente preenchidos exceto possivelmente o último, preenchido da esquerda para a direita
- **Árvore binária perfeita:** todas as folhas no mesmo nível, todos os nós internos têm 2 filhos
- **Árvore binária balanceada:** alturas das subárvores esquerda e direita diferem no máximo 1 para todos os nós

### Árvore Binária de Busca (BST)

Para todo nó N: todos os nós na subárvore esquerda de N têm valores < N.value, e todos os nós na subárvore direita têm valores > N.value.

Esse invariante torna a busca O(h) onde h = altura. Para uma árvore balanceada, h = O(log n). Para uma árvore degenerada (todos os nós de um lado), h = O(n).

### BSTs Auto-Balanceadas

**Árvore AVL:** Após cada inserção/remoção, verifica fatores de balanceamento (diferença de altura das subárvores). Se violado, realiza rotações para restaurar o balanceamento. Estritamente balanceada — alturas diferem no máximo 1. Lookup mais rápido que Red-Black, mas mais rotações em inserção/remoção.

**Árvore Red-Black:** Cada nó é colorido vermelho ou preto. O balanceamento é mantido por um conjunto relaxado de regras de cor que garantem que o caminho mais longo é no máximo 2× o mais curto. Usada na maioria das bibliotecas padrão de linguagens por causa de menos rotações em mutação.

---

## 3. Como Funciona

```
Exemplo de BST:
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13

Busca por 6:
1. Começa na raiz 8. 6 < 8 → vai para esquerda
2. Em 3. 6 > 3 → vai para direita
3. Em 6. Encontrado! O(log n) em média

Inserir 5:
1. 5 < 8 → esquerda
2. 5 > 3 → direita
3. 5 < 6 → esquerda
4. 4 existe. 5 > 4 → direita de 4. Inserir!

Remover 6 (tem dois filhos):
Opção: substituir pelo sucessor em ordem (7, o menor na subárvore direita)
→ nó se torna 7, remove o 7 original
```

**Travessias:**
```
Inorder (esquerda, raiz, direita): 1, 3, 4, 6, 7, 8, 10, 13, 14 — saída ordenada para BST!
Preorder (raiz, esquerda, direita): 8, 3, 1, 6, 4, 7, 10, 14, 13
Postorder (esquerda, direita, raiz): 1, 4, 7, 6, 3, 13, 14, 10, 8
Level-order (BFS): 8, 3, 10, 1, 6, 14, 4, 7, 13
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Nó da Árvore ---
class TreeNode {
  constructor(
    public val: number,
    public left: TreeNode | null = null,
    public right: TreeNode | null = null
  ) {}
}

// --- Operações de BST ---
class BST {
  private root: TreeNode | null = null;

  // Inserir: O(log n) em média, O(n) no pior caso
  insert(val: number): void {
    this.root = this.insertNode(this.root, val);
  }

  private insertNode(node: TreeNode | null, val: number): TreeNode {
    if (node === null) return new TreeNode(val);
    if (val < node.val) node.left = this.insertNode(node.left, val);
    else if (val > node.val) node.right = this.insertNode(node.right, val);
    // Igual: ignora duplicatas (ou trata conforme necessário)
    return node;
  }

  // Busca: O(log n) em média
  search(val: number): TreeNode | null {
    return this.searchNode(this.root, val);
  }

  private searchNode(node: TreeNode | null, val: number): TreeNode | null {
    if (node === null || node.val === val) return node;
    return val < node.val
      ? this.searchNode(node.left, val)
      : this.searchNode(node.right, val);
  }

  // Remover: O(log n) em média
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
      // Encontrou o nó a remover
      if (node.left === null) return node.right; // sem filho esquerdo
      if (node.right === null) return node.left; // sem filho direito

      // Dois filhos: substitui pelo sucessor em ordem (mínimo da subárvore direita)
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

// --- Travessias ---

// Inorder: O(n) de tempo, O(h) de espaço (pilha de chamadas)
function inorder(root: TreeNode | null): number[] {
  if (root === null) return [];
  return [...inorder(root.left), root.val, ...inorder(root.right)];
}

// Inorder iterativo: O(n) de tempo, O(h) de espaço — evita limite de profundidade de recursão
function inorderIterative(root: TreeNode | null): number[] {
  const result: number[] = [];
  const stack: TreeNode[] = [];
  let current = root;

  while (current !== null || stack.length > 0) {
    // Vai o máximo possível para a esquerda
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

// --- Validar BST: O(n) ---
// Passa o intervalo permitido [min, max] para cada nó
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

// --- Profundidade Máxima: O(n) ---
function maxDepth(root: TreeNode | null): number {
  if (root === null) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}

// --- Menor Ancestral Comum: O(n) ---
// Para BST: usa a propriedade BST para O(log n)
function lowestCommonAncestorBST(
  root: TreeNode | null,
  p: number,
  q: number
): TreeNode | null {
  if (root === null) return null;
  if (p < root.val && q < root.val) return lowestCommonAncestorBST(root.left, p, q);
  if (p > root.val && q > root.val) return lowestCommonAncestorBST(root.right, p, q);
  return root; // raiz está entre p e q — é o LCA
}

// Para árvore binária geral: O(n)
function lowestCommonAncestor(
  root: TreeNode | null,
  p: number,
  q: number
): TreeNode | null {
  if (root === null || root.val === p || root.val === q) return root;
  const left = lowestCommonAncestor(root.left, p, q);
  const right = lowestCommonAncestor(root.right, p, q);
  if (left !== null && right !== null) return root; // p e q em lados diferentes
  return left ?? right;
}

// --- Serializar / Desserializar árvore binária: O(n) ---
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

// --- Construir BST a partir de array ordenado: O(n) ---
function sortedArrayToBST(nums: number[]): TreeNode | null {
  if (nums.length === 0) return null;
  const mid = Math.floor(nums.length / 2);
  const node = new TreeNode(nums[mid]);
  node.left = sortedArrayToBST(nums.slice(0, mid));
  node.right = sortedArrayToBST(nums.slice(mid + 1));
  return node;
}

// --- K-ésimo Menor em BST: O(n) pior caso, O(k + h) em média ---
// Travessia inorder dá ordem crescente; retorna o k-ésimo elemento
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

// --- Verificar se duas árvores são idênticas: O(n) ---
function isSameTree(p: TreeNode | null, q: TreeNode | null): boolean {
  if (p === null && q === null) return true;
  if (p === null || q === null) return false;
  return (
    p.val === q.val &&
    isSameTree(p.left, q.left) &&
    isSameTree(p.right, q.right)
  );
}

// --- Verificar se árvore é simétrica (espelho de si mesma): O(n) ---
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

## 5. Erros Comuns e Armadilhas

> ⚠️ **Assumir que uma BST está balanceada.** Uma BST construída inserindo dados já ordenados degenera em uma lista encadeada — O(n) para todas as operações. Sempre use uma BST auto-balanceada (AVL/Red-Black) para garantia de O(log n).

> ⚠️ **Confundir a direção da travessia inorder.** Inorder de uma BST dá ordem crescente apenas quando você visita ESQUERDA → RAIZ → DIREITA. Inverter a ordem dá ordem decrescente.

> ⚠️ **Off-by-one em level-order.** Ao agrupar nós por nível, você deve capturar `queue.length` no início de cada iteração de nível — a fila cresce enquanto você processa nós.

> ⚠️ **Não tratar o caso de dois filhos na remoção de BST.** O caso mais complexo é remover um nó com dois filhos. Você deve encontrar o sucessor em ordem (ou predecessor), copiar seu valor e então remover o sucessor.

> ⚠️ **Stack overflow em árvores profundas.** Travessia recursiva em uma BST degenerada (agindo como lista encadeada) causa profundidade de recursão O(n). Use travessia iterativa para código de produção.

---

## 6. Quando Usar / Não Usar

**Use uma BST quando:**
- Precisa de uma coleção dinâmica ordenada com inserção/remoção/busca em O(log n)
- Precisa encontrar o k-ésimo menor/maior elemento eficientemente
- Precisa de consultas de intervalo em dados ordenados
- Precisa encontrar o predecessor/sucessor de um valor

**Use uma BST auto-balanceada (AVL/Red-Black) quando:**
- Precisa de O(log n) garantido — use o sorted set/map integrado da linguagem
- Inserções e remoções são frequentes junto com lookups

**NÃO use uma BST quando:**
- Os dados são estáticos — use array ordenado + busca binária (melhor performance de cache)
- Precisa apenas de lookup O(1) — use um hashmap
- As chaves são strings arbitrárias ou objetos complexos — ordenação é custosa

---

## 7. Cenário do Mundo Real

Um sistema de negociação de ações precisa encontrar rapidamente todas as ordens entre os preços R$100 e R$200 (consulta de intervalo). Um array ordenado requer varredura O(n). Uma BST resolve em O(log n + k) onde k = número de resultados:

```typescript
function rangeQuery(
  root: TreeNode | null,
  low: number,
  high: number,
  result: number[] = []
): number[] {
  if (root === null) return result;

  // Poda: se raiz < low, apenas a subárvore direita pode ter nós válidos
  if (root.val >= low) rangeQuery(root.left, low, high, result);

  // Inclui nó atual se estiver no intervalo
  if (root.val >= low && root.val <= high) result.push(root.val);

  // Poda: se raiz > high, apenas a subárvore esquerda pode ter nós válidos
  if (root.val <= high) rangeQuery(root.right, low, high, result);

  return result;
}
```

---

## 8. Perguntas de Entrevista

**Q1: Qual é a diferença entre uma árvore binária e uma BST?**
R: Uma árvore binária é qualquer árvore onde cada nó tem no máximo 2 filhos — sem restrição de ordenação. Uma BST adiciona o invariante: valores da subárvore esquerda < valor do nó < valores da subárvore direita. Isso torna a busca O(log n) em média.

**Q2: Como funciona a remoção em uma BST?**
R: Três casos: (1) Folha — apenas remove. (2) Um filho — substitui o nó pelo seu filho. (3) Dois filhos — encontra o sucessor em ordem (mínimo da subárvore direita), copia seu valor para o nó atual, depois remove o sucessor (que tem no máximo um filho direito — caso 1 ou 2).

**Q3: Por que uma BST balanceada é importante?**
R: Uma BST não balanceada pode degenerar para O(n) em todas as operações (essencialmente uma lista encadeada). Uma BST balanceada garante altura O(log n), mantendo todas as operações em O(log n).

**Q4: Qual travessia produz saída ordenada para uma BST?**
R: Travessia inorder (esquerda → raiz → direita) visita nós em ordem crescente.

**Q5: Qual é a diferença entre árvores AVL e Red-Black?**
R: Árvores AVL são mais estritamente balanceadas (diferença de altura ≤ 1) — lookups mais rápidos mas mais rotações em inserção/remoção. Árvores Red-Black são menos estritamente balanceadas mas exigem menos rotações em mutações — melhores para cargas de trabalho com muitas escritas. A maioria das bibliotecas padrão usa árvores Red-Black.

**Q6: Como você verifica se uma árvore binária é uma BST válida?**
R: Passa um intervalo válido [min, max] para cada nó. O intervalo da raiz é (-∞, +∞). O intervalo do filho esquerdo é (min, pai.val). O intervalo do filho direito é (pai.val, max). Se algum nó violar seu intervalo, retorna false. O(n).

---

## 9. Exercícios

**Exercício 1:** Construa uma BST balanceada a partir de um array ordenado. A árvore resultante deve ter a altura mínima possível.
*Dica: sempre escolha o elemento do meio como raiz, recurse nas metades esquerda e direita.*

**Exercício 2:** Encontre o k-ésimo menor elemento em uma BST em O(k + h) de tempo.
*Dica: travessia inorder iterativa — pare quando você tiver visitado k nós.*

**Exercício 3:** Verifique se duas árvores binárias são idênticas (mesma estrutura e mesmos valores).

**Exercício 4:** Encontre todos os caminhos da raiz até uma folha onde a soma dos valores é igual a um alvo.
*Dica: DFS com uma soma corrente e rastreamento de caminho. Quando você chega a uma folha e a soma corresponde ao alvo, registre o caminho.*

**Exercício 5:** Achate uma árvore binária em uma lista encadeada in-place (apenas ponteiros direitos, esquerdo = null). A ordem deve corresponder à travessia preorder.
*Dica: abordagem de "Morris Traversal" ou postorder reverso.*

---

## 10. Leituras Complementares

- CLRS Capítulo 12 — Binary Search Trees, Capítulo 13 — Red-Black Trees
- [Visualgo — visualização de BST](https://visualgo.net/en/bst)
- [Rotações em AVL Tree explicadas](https://en.wikipedia.org/wiki/AVL_tree)
- Problemas LeetCode: #98 Validate BST, #102 Level Order Traversal, #105 Construct Binary Tree from Preorder/Inorder, #236 LCA, #297 Serialize/Deserialize
