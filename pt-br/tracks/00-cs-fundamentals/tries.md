# Tries

## 1. O Que É e Por Que Importa

Uma trie (também chamada de árvore de prefixos ou árvore digital) é uma estrutura de dados em árvore onde cada caminho da raiz até um nó representa um prefixo de uma ou mais strings armazenadas na estrutura. A palavra "trie" vem de "retrieval" (recuperação).

Tries são excelentes em operações de prefixo de string que hashmaps não conseguem fazer eficientemente. Autocompletar, verificação ortográfica, roteamento IP e texto preditivo usam estruturas baseadas em trie. A vantagem chave: um hashmap pode dizer se uma string exata existe em O(1), mas uma trie pode dizer todas as strings que compartilham um dado prefixo em O(comprimento_do_prefixo + número_de_resultados).

---

## 2. Conceitos Fundamentais

### Estrutura da Trie

Cada nó representa um caractere e contém:
- Um mapa (ou array de tamanho fixo para ASCII) de nós filhos: `children: Map<string, TrieNode>`
- Uma flag indicando se esse nó marca o fim de uma palavra válida: `isEnd: boolean`
- Opcionalmente: uma contagem de palavras passando por esse nó, uma frequência ou a própria palavra

O nó raiz é vazio — não tem caractere associado. Todas as inserções começam a partir da raiz.

### Complexidade

- **Inserir:** O(m) onde m = comprimento da palavra
- **Busca (exata):** O(m)
- **StartsWith (busca por prefixo):** O(m)
- **Deletar:** O(m)
- **Espaço:** O(TAMANHO_DO_ALFABETO × N × M) onde N = número de palavras, M = comprimento médio de palavra. Na prática, prefixos compartilhados reduzem drasticamente o uso de espaço.

### Tries Comprimidas (Patricia Trees)

Tries padrão desperdiçam espaço quando um nó tem apenas um filho (longas cadeias sem bifurcação). Tries comprimidas mesclam tais cadeias em uma única aresta rotulada com a substring inteira. Usadas em radix trees, ternary search trees e suffix trees.

### Trie vs Hashmap

| | Trie | Hashmap |
|---|---|---|
| Lookup exato | O(m) | O(1) em média |
| Busca por prefixo | O(m + resultados) | O(total_strings × m) |
| Iteração ordenada | Natural (DFS) | Requer ordenação |
| Espaço | Prefixos compartilhados → eficiente para strings similares | Overhead por entrada |

---

## 3. Como Funciona

```
Após inserir: "car", "card", "care", "cat", "bat"

raiz
├── c
│   └── a
│       ├── r [FIM]
│       │   ├── d [FIM]
│       │   └── e [FIM]
│       └── t [FIM]
└── b
    └── a
        └── t [FIM]

Busca "card":
raiz → c → a → r → d [isEnd = true] → ENCONTRADO

StartsWith "car":
raiz → c → a → r → prefixo encontrado, coleta todas as palavras descendentes
Resultado: ["car", "card", "care"]

Note como "c", "a" são compartilhados entre "car*" e "cat" — essa é a economia de espaço.
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Implementação de TrieNode e Trie ---
class TrieNode {
  children: Map<string, TrieNode> = new Map();
  isEnd = false;
  // Opcional: conta palavras terminando aqui, ou armazena a própria palavra para recuperação
  word: string | null = null;
}

class Trie {
  private root = new TrieNode();

  // Inserir uma palavra: O(m) onde m = comprimento da palavra
  insert(word: string): void {
    let node = this.root;
    for (const char of word) {
      if (!node.children.has(char)) {
        node.children.set(char, new TrieNode());
      }
      node = node.children.get(char)!;
    }
    node.isEnd = true;
    node.word = word;
  }

  // Busca exata: O(m)
  search(word: string): boolean {
    const node = this.traverse(word);
    return node !== null && node.isEnd;
  }

  // Busca por prefixo: O(m)
  startsWith(prefix: string): boolean {
    return this.traverse(prefix) !== null;
  }

  // Obtém todas as palavras com dado prefixo: O(m + total_chars_nos_resultados)
  autocomplete(prefix: string): string[] {
    const node = this.traverse(prefix);
    if (node === null) return [];

    const results: string[] = [];
    this.collectWords(node, results);
    return results;
  }

  // Conta palavras com dado prefixo: O(m + tamanho_da_subárvore)
  countWordsWithPrefix(prefix: string): number {
    return this.autocomplete(prefix).length;
  }

  // Deletar uma palavra: O(m)
  delete(word: string): boolean {
    return this.deleteHelper(this.root, word, 0);
  }

  private deleteHelper(node: TrieNode, word: string, depth: number): boolean {
    if (depth === word.length) {
      if (!node.isEnd) return false; // palavra não existe
      node.isEnd = false;
      node.word = null;
      return node.children.size === 0; // seguro deletar este nó se não tem filhos
    }

    const char = word[depth];
    const child = node.children.get(char);
    if (!child) return false;

    const shouldDeleteChild = this.deleteHelper(child, word, depth + 1);
    if (shouldDeleteChild) {
      node.children.delete(char);
      return node.children.size === 0 && !node.isEnd;
    }
    return false;
  }

  private traverse(s: string): TrieNode | null {
    let node = this.root;
    for (const char of s) {
      if (!node.children.has(char)) return null;
      node = node.children.get(char)!;
    }
    return node;
  }

  private collectWords(node: TrieNode, results: string[]): void {
    if (node.isEnd && node.word !== null) results.push(node.word);
    for (const child of node.children.values()) {
      this.collectWords(child, results);
    }
  }
}

// --- Exemplo de uso ---
const trie = new Trie();
["apple", "app", "application", "apply", "apt", "banana"].forEach(w => trie.insert(w));

console.log(trie.search("app"));         // true
console.log(trie.search("ap"));          // false (não é uma palavra completa)
console.log(trie.startsWith("ap"));      // true
console.log(trie.autocomplete("app"));   // ["app", "apple", "application", "apply"]

// --- Busca de Palavras II: encontra todas as palavras de uma lista que existem em uma grade ---
// Usa uma trie para evitar buscas redundantes
function findWords(board: string[][], words: string[]): string[] {
  const trie = new Trie();
  for (const word of words) trie.insert(word);

  const rows = board.length, cols = board[0].length;
  const found = new Set<string>();
  const visited = Array.from({ length: rows }, () => new Array(cols).fill(false));

  function dfs(r: number, c: number, node: TrieNode): void {
    if (r < 0 || r >= rows || c < 0 || c >= cols || visited[r][c]) return;

    const char = board[r][c];
    const child = node.children.get(char);
    if (!child) return; // nenhuma palavra na trie começa com esse caminho

    if (child.isEnd && child.word !== null) found.add(child.word);

    visited[r][c] = true;
    dfs(r + 1, c, child);
    dfs(r - 1, c, child);
    dfs(r, c + 1, child);
    dfs(r, c - 1, child);
    visited[r][c] = false; // backtrack
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      dfs(r, c, (trie as any).root);
    }
  }

  return [...found];
}

// --- Replace Words: substitui palavras mais longas pela raiz mais curta ---
// Entrada: roots = ["cat", "bat", "rat"], sentence = "the cattle was rattled by the battery"
// Saída: "the cat was rat by the bat"
function replaceWords(dictionary: string[], sentence: string): string {
  const trie = new Trie();
  for (const root of dictionary) trie.insert(root);

  return sentence.split(" ").map(word => {
    // Encontra a raiz mais curta dessa palavra
    let node = (trie as any).root as TrieNode;
    for (let i = 0; i < word.length; i++) {
      const char = word[i];
      if (!node.children.has(char)) break; // nenhum prefixo de raiz existe
      node = node.children.get(char)!;
      if (node.isEnd) return word.slice(0, i + 1); // raiz encontrada — substitui
    }
    return word; // nenhuma raiz encontrada — mantém original
  }).join(" ");
}

// --- Sistema de autocompletar com rastreamento de frequência ---
class AutocompleteSystem {
  private trie: Map<string, Map<string, number>> = new Map(); // prefixo → {frase: contagem}
  private input = "";

  constructor(sentences: string[], times: number[]) {
    sentences.forEach((sentence, i) => {
      this.addSentence(sentence, times[i]);
    });
  }

  private addSentence(sentence: string, count: number): void {
    for (let i = 1; i <= sentence.length; i++) {
      const prefix = sentence.slice(0, i);
      if (!this.trie.has(prefix)) this.trie.set(prefix, new Map());
      const map = this.trie.get(prefix)!;
      map.set(sentence, (map.get(sentence) ?? 0) + count);
    }
  }

  input_char(c: string): string[] {
    if (c === "#") {
      this.addSentence(this.input, 1);
      this.input = "";
      return [];
    }

    this.input += c;
    const candidates = this.trie.get(this.input);
    if (!candidates) return [];

    // Ordena por frequência desc, depois alfabeticamente
    return [...candidates.entries()]
      .sort((a, b) => b[1] - a[1] || a[0].localeCompare(b[0]))
      .slice(0, 3)
      .map(([sentence]) => sentence);
  }
}

// --- Contar palavras com dado prefixo ---
class WordPrefixCounter {
  private prefixCounts = new Map<string, number>(); // prefixo → contagem

  insert(word: string): void {
    for (let i = 1; i <= word.length; i++) {
      const prefix = word.slice(0, i);
      this.prefixCounts.set(prefix, (this.prefixCounts.get(prefix) ?? 0) + 1);
    }
    // Conta o prefixo vazio como contagem total de palavras
    this.prefixCounts.set("", (this.prefixCounts.get("") ?? 0) + 1);
  }

  count(prefix: string): number {
    return this.prefixCounts.get(prefix) ?? 0;
  }
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Confundir `search` (correspondência exata, requer `isEnd`) com `startsWith` (correspondência de prefixo, apenas requer chegar ao último nó).** Essas são operações diferentes. Sempre defina `isEnd = true` na inserção e verifique-o na busca exata.

> ⚠️ **Explosão de memória com alfabetos grandes.** Se você usa um array de tamanho fixo de 26 para letras em inglês, cada nó aloca 26 slots mesmo que apenas 2 sejam usados. Use `Map<string, TrieNode>` para alfabetos dinâmicos ou quando memória é uma preocupação.

> ⚠️ **Não tratar a deleção corretamente.** Deletar uma palavra não deve remover nós que são compartilhados com outras palavras. Remova um nó apenas se não tiver filhos e não for fim de palavra para outra string.

> ⚠️ **Esquecer de tratar a string vazia.** Buscar por `""` geralmente deve retornar true se a string vazia foi inserida. Certifique-se de que seu código trata esse caso extremo sem erros de off-by-one.

> ⚠️ **Expor a raiz interna para problemas de DFS.** O padrão Word Search II requer iniciar o DFS a partir da raiz da trie. Exponha um método `getRoot()` ou reestruture o código para evitar problemas de encapsulamento.

---

## 6. Quando Usar / Não Usar

**Use uma trie quando:**
- Autocompletar / sugestões de digitação antecipada
- Verificação ortográfica e lookup de palavras
- Encontrar todas as strings com dado prefixo
- Tabelas de roteamento IP (usando trie binária nos bits de IP)
- Jogos de palavras (Boggle, validação de palavras no Scrabble)
- Substituir lookups repetidos de prefixo de string em um conjunto de strings

**NÃO use uma trie quando:**
- Precisa apenas de lookups exatos — hashmap é O(1) vs O(m) da trie
- As strings não têm nada em comum — sem prefixos compartilhados, sem benefício
- A memória é muito limitada e as strings são longas com poucos prefixos compartilhados
- Precisa de iteração ordenada por valor (não por ordem lexicográfica)

---

## 7. Cenário do Mundo Real

O recurso de autocompletar de um motor de busca precisa sugerir os 5 melhores completamentos para um prefixo digitado, ordenados por frequência de busca:

```typescript
class SearchAutocomplete {
  private root = new TrieNode();

  // Indexa uma consulta de busca com sua frequência
  index(query: string, frequency: number): void {
    let node = this.root;
    for (const char of query) {
      if (!node.children.has(char)) {
        node.children.set(char, new TrieNode());
      }
      node = node.children.get(char)!;
    }
    node.isEnd = true;
    node.word = query;
    // Armazena frequência no nó (estende TrieNode com campo freq)
    (node as any).freq = ((node as any).freq ?? 0) + frequency;
  }

  suggest(prefix: string, topK = 5): string[] {
    let node = this.root;
    for (const char of prefix) {
      if (!node.children.has(char)) return [];
      node = node.children.get(char)!;
    }

    const candidates: { word: string; freq: number }[] = [];
    this.collectWithFreq(node, candidates);

    return candidates
      .sort((a, b) => b.freq - a.freq || a.word.localeCompare(b.word))
      .slice(0, topK)
      .map(c => c.word);
  }

  private collectWithFreq(node: TrieNode, result: { word: string; freq: number }[]): void {
    if (node.isEnd && node.word) {
      result.push({ word: node.word, freq: (node as any).freq ?? 1 });
    }
    for (const child of node.children.values()) {
      this.collectWithFreq(child, result);
    }
  }
}
```

---

## 8. Perguntas de Entrevista

**Q1: Quando você usaria uma trie em vez de um hashmap?**
R: Use uma trie quando precisar de operações baseadas em prefixo: autocompletar, "alguma string armazenada começa com X?", coletar todas as strings com dado prefixo. Um hashmap apenas suporta lookup exato de chave em O(1) — não consegue responder consultas de prefixo eficientemente sem varrer todas as chaves.

**Q2: Implemente autocompletar. Dada uma lista de palavras e um prefixo, retorne todas as palavras começando com esse prefixo.**
R: Construa uma trie a partir de todas as palavras (O(total de chars)). Para uma consulta de prefixo, percorra a trie até o final do nó de prefixo — O(comprimento do prefixo). Depois colete com DFS todas as palavras na subárvore — O(tamanho da subárvore). Total: O(p + resultados).

**Q3: Qual é a complexidade de espaço de uma trie?**
R: O(TAMANHO_DO_ALFABETO × N × M) no pior caso onde N = número de palavras e M = comprimento médio. Na prática, prefixos compartilhados reduzem isso significativamente. Usar Maps em vez de arrays fixos evita alocar slots não usados.

**Q4: Como você deleta uma palavra de uma trie sem corromper outras palavras?**
R: Percorre recursivamente até o final da palavra, marca `isEnd = false`. No caminho de volta, se um nó não tem filhos e não é fim de outra palavra, pode ser seguramente deletado. Isso requer verificar condições antes de remover nós.

**Q5: O que é uma trie comprimida / Patricia tree?**
R: Uma trie comprimida mescla cadeias de filho único em arestas únicas rotuladas com a substring inteira (não apenas um caractere). Reduz a contagem de nós de O(total de chars) para O(número de palavras). Usada em suffix trees e radix trees.

---

## 9. Exercícios

**Exercício 1:** Implemente a classe Trie completa com `insert`, `search`, `startsWith` e `delete`. Teste com casos extremos: string vazia, prefixo de outra palavra, deletar um prefixo que também é uma palavra.

**Exercício 2:** Conte palavras com dado prefixo. Dada uma lista de palavras e consultas, para cada prefixo de consulta, conte quantas palavras começam com ele.
*Dica: armazene contagens de prefixo durante a inserção.*

**Exercício 3:** Implemente o problema Replace Words: dado um dicionário de palavras raiz, substitua qualquer palavra em uma frase pela raiz correspondente mais curta.

**Exercício 4:** Projete um "Magic Dictionary" — uma estrutura de dados que verifica se uma dada palavra existe no dicionário com exatamente um caractere alterado.
*Dica: trie + DFS com um contador de tolerância.*

**Exercício 5:** Implemente Boggle — encontre todas as palavras válidas em uma grade de caracteres onde palavras podem ser formadas por células adjacentes (horizontal/vertical/diagonal), sem reutilizar nenhuma célula.
*Dica: trie para lookup de palavras + DFS + backtracking com array de visitados.*

---

## 10. Leituras Complementares

- [Estrutura de dados Trie — Wikipedia](https://en.wikipedia.org/wiki/Trie)
- [Algoritmo Aho-Corasick](https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm) — busca multi-padrão de string usando trie com links de falha
- [Suffix Tree / Suffix Array](https://cp-algorithms.com/string/suffix-array.html)
- Problemas LeetCode: #208 Implement Trie, #212 Word Search II, #648 Replace Words, #745 Prefix and Suffix Search, #1268 Search Suggestions System
