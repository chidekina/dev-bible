# Tries

## 1. What & Why

A trie (also called prefix tree or digital tree) is a tree data structure where each path from root to a node represents a prefix of one or more strings stored in the structure. The word "trie" comes from "retrieval."

Tries excel at string prefix operations that hashmaps cannot do efficiently. Autocomplete, spell checking, IP routing, and predictive text all use trie-based data structures. The key advantage: a hashmap can tell you if an exact string exists in O(1), but a trie can tell you all strings sharing a given prefix in O(prefix_length + number_of_results).

---

## 2. Core Concepts

### Trie Structure

Each node represents a character and contains:
- A map (or fixed-size array for ASCII) of child nodes: `children: Map<string, TrieNode>`
- A flag indicating whether this node marks the end of a valid word: `isEnd: boolean`
- Optionally: a count of words passing through this node, a frequency, or the word itself

The root node is empty — it has no associated character. All insertions start from the root.

### Complexity

- **Insert:** O(m) where m = length of the word
- **Search (exact):** O(m)
- **StartsWith (prefix search):** O(m)
- **Delete:** O(m)
- **Space:** O(ALPHABET_SIZE × N × M) where N = number of words, M = average word length. In practice, shared prefixes dramatically reduce space usage.

### Compressed Tries (Patricia Trees)

Standard tries waste space when a node has only one child (long chains with no branching). Compressed tries merge such chains into a single edge labeled with the entire substring. Used in radix trees, ternary search trees, and suffix trees.

### vs Hashmap

| | Trie | Hashmap |
|---|---|---|
| Exact lookup | O(m) | O(1) average |
| Prefix search | O(m + results) | O(total_strings × m) |
| Sorted iteration | Natural (DFS) | Requires sorting |
| Space | Shared prefixes → efficient for similar strings | Per-entry overhead |

---

## 3. How It Works

```
After inserting: "car", "card", "care", "cat", "bat"

root
├── c
│   └── a
│       ├── r [END]
│       │   ├── d [END]
│       │   └── e [END]
│       └── t [END]
└── b
    └── a
        └── t [END]

Search "card":
root → c → a → r → d [isEnd = true] → FOUND

StartsWith "car":
root → c → a → r → found prefix, collect all descendant words
Result: ["car", "card", "care"]

Note how "c", "a" are shared between "car*" and "cat" — this is the space saving.
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- TrieNode and Trie implementation ---
class TrieNode {
  children: Map<string, TrieNode> = new Map();
  isEnd = false;
  // Optional: count words ending here, or store the word itself for retrieval
  word: string | null = null;
}

class Trie {
  private root = new TrieNode();

  // Insert a word: O(m) where m = word length
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

  // Exact search: O(m)
  search(word: string): boolean {
    const node = this.traverse(word);
    return node !== null && node.isEnd;
  }

  // Prefix search: O(m)
  startsWith(prefix: string): boolean {
    return this.traverse(prefix) !== null;
  }

  // Get all words with given prefix: O(m + total_chars_in_results)
  autocomplete(prefix: string): string[] {
    const node = this.traverse(prefix);
    if (node === null) return [];

    const results: string[] = [];
    this.collectWords(node, results);
    return results;
  }

  // Count words with given prefix: O(m + subtree_size)
  countWordsWithPrefix(prefix: string): number {
    return this.autocomplete(prefix).length;
  }

  // Delete a word: O(m)
  delete(word: string): boolean {
    return this.deleteHelper(this.root, word, 0);
  }

  private deleteHelper(node: TrieNode, word: string, depth: number): boolean {
    if (depth === word.length) {
      if (!node.isEnd) return false; // word doesn't exist
      node.isEnd = false;
      node.word = null;
      return node.children.size === 0; // safe to delete this node if no children
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

// --- Usage example ---
const trie = new Trie();
["apple", "app", "application", "apply", "apt", "banana"].forEach(w => trie.insert(w));

console.log(trie.search("app"));         // true
console.log(trie.search("ap"));          // false (not a full word)
console.log(trie.startsWith("ap"));      // true
console.log(trie.autocomplete("app"));   // ["app", "apple", "application", "apply"]

// --- Word Search II: find all words from a list that exist in a grid ---
// Uses a trie to avoid redundant searches
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
    if (!child) return; // no word in trie starts with this path

    if (child.isEnd && child.word !== null) found.add(child.word);

    visited[r][c] = true;
    dfs(r + 1, c, child);
    dfs(r - 1, c, child);
    dfs(r, c + 1, child);
    dfs(r, c - 1, child);
    visited[r][c] = false; // backtrack
  }

  // Access root directly — we need the internal root
  // (for this, expose a method or adapt the class)
  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      // Start DFS from each cell
      dfs(r, c, (trie as any).root);
    }
  }

  return [...found];
}

// --- Replace Words: given a dictionary of roots, replace longer words with their shortest root ---
// Input: roots = ["cat", "bat", "rat"], sentence = "the cattle was rattled by the battery"
// Output: "the cat was rat by the bat"
function replaceWords(dictionary: string[], sentence: string): string {
  const trie = new Trie();
  for (const root of dictionary) trie.insert(root);

  return sentence.split(" ").map(word => {
    // Find the shortest root of this word
    let node = (trie as any).root as TrieNode;
    for (let i = 0; i < word.length; i++) {
      const char = word[i];
      if (!node.children.has(char)) break; // no root prefix exists
      node = node.children.get(char)!;
      if (node.isEnd) return word.slice(0, i + 1); // found root — replace
    }
    return word; // no root found — keep original
  }).join(" ");
}

// --- Autocomplete system with frequency tracking ---
class AutocompleteSystem {
  private trie: Map<string, Map<string, number>> = new Map(); // prefix → {sentence: count}
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

    // Sort by frequency desc, then alphabetically
    return [...candidates.entries()]
      .sort((a, b) => b[1] - a[1] || a[0].localeCompare(b[0]))
      .slice(0, 3)
      .map(([sentence]) => sentence);
  }
}

// --- Count words with a given prefix ---
class WordPrefixCounter {
  private prefixCounts = new Map<string, number>(); // prefix → count

  insert(word: string): void {
    for (let i = 1; i <= word.length; i++) {
      const prefix = word.slice(0, i);
      this.prefixCounts.set(prefix, (this.prefixCounts.get(prefix) ?? 0) + 1);
    }
    // Count the empty prefix as total word count
    this.prefixCounts.set("", (this.prefixCounts.get("") ?? 0) + 1);
  }

  count(prefix: string): number {
    return this.prefixCounts.get(prefix) ?? 0;
  }
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Confusing `search` (exact match, requires `isEnd`) with `startsWith` (prefix match, just requires reaching the last node).** These are different operations. Always set `isEnd = true` on insert and check it on exact search.

> ⚠️ **Memory explosion with large alphabets.** If you use a fixed-size array of 26 for English letters, each node allocates 26 slots even if only 2 are used. Use a `Map<string, TrieNode>` for dynamic alphabets or when memory is a concern.

> ⚠️ **Not handling deletion correctly.** Deleting a word must not remove nodes that are shared with other words. Only remove a node if it has no children and is not an end-of-word for another string.

> ⚠️ **Forgetting to handle the empty string.** Searching for `""` should typically return true if the empty string was inserted. Ensure your code handles this edge case without off-by-one errors.

> ⚠️ **Exposing internal root for DFS problems.** The Word Search II pattern requires starting DFS from the trie's root. Either expose a `getRoot()` method or restructure the code to avoid encapsulation issues.

---

## 6. When to Use / Not Use

**Use a trie when:**
- Autocomplete / typeahead suggestions
- Spell checking and word lookup
- Finding all strings with a given prefix
- IP routing tables (using binary trie on IP bits)
- Word games (Boggle, Scrabble word validation)
- Replacing repeated string prefix lookups in a set of strings

**Do NOT use a trie when:**
- You only need exact lookups — a hashmap is O(1) vs trie's O(m)
- The strings have nothing in common — no shared prefixes, no benefit
- Memory is very tight and strings are long with few shared prefixes
- You need sorted iteration by value (not lexicographic order)

---

## 7. Real-World Scenario

A search engine's autocomplete feature needs to suggest the top 5 completions for a typed prefix, ordered by search frequency:

```typescript
class SearchAutocomplete {
  private root = new TrieNode();

  // Index a search query with its frequency
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
    // Store frequency in the node (extend TrieNode with a freq field)
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

## 8. Interview Questions

**Q1: When would you use a trie over a hashmap?**
A: Use a trie when you need prefix-based operations: autocomplete, "does any stored string start with X?", collecting all strings with a given prefix. A hashmap only supports exact key lookup in O(1) — it cannot answer prefix queries efficiently without scanning all keys.

**Q2: Implement autocomplete. Given a list of words and a prefix, return all words starting with that prefix.**
A: Build a trie from all words (O(total chars)). For a query prefix, traverse the trie to the end of the prefix node — O(prefix length). Then DFS collect all words in the subtree — O(subtree size). Total: O(p + results).

**Q3: What is the space complexity of a trie?**
A: O(ALPHABET_SIZE × N × M) in the worst case where N = number of words and M = average length. In practice, shared prefixes reduce this significantly. Using Maps instead of fixed arrays avoids allocating unused slots.

**Q4: How do you delete a word from a trie without corrupting other words?**
A: Recursively traverse to the word end, mark `isEnd = false`. On the way back up, if a node has no children and is not an end of another word, it can be safely deleted. This requires checking conditions before removing nodes.

**Q5: What is a compressed trie / Patricia tree?**
A: A compressed trie merges single-child chains into single edges labeled with the entire substring (not just one character). Reduces node count from O(total chars) to O(number of words). Used in suffix trees and radix trees.

---

## 9. Exercises

**Exercise 1:** Implement the full Trie class with `insert`, `search`, `startsWith`, and `delete`. Test with edge cases: empty string, prefix of another word, deleting a prefix that is also a word.

**Exercise 2:** Count words with a given prefix. Given a list of words and queries, for each query prefix, count how many words start with it.
*Hint: store prefix counts during insert.*

**Exercise 3:** Implement the Replace Words problem: given a dictionary of root words, replace any word in a sentence with its shortest matching root.

**Exercise 4:** Design a "Magic Dictionary" — a data structure that checks if a given word exists in the dictionary with exactly one character changed.
*Hint: trie + DFS with a tolerance counter.*

**Exercise 5:** Implement Boggle — find all valid words in a grid of characters where words can be formed by adjacent (horizontally/vertically/diagonally) cells, not reusing any cell.
*Hint: trie for word lookup + DFS + backtracking with visited array.*

---

## 10. Further Reading

- [Trie data structure — Wikipedia](https://en.wikipedia.org/wiki/Trie)
- [Aho-Corasick algorithm](https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm) — multi-pattern string search using a trie with failure links
- [Suffix Tree / Suffix Array](https://cp-algorithms.com/string/suffix-array.html)
- LeetCode problems: #208 Implement Trie, #212 Word Search II, #648 Replace Words, #745 Prefix and Suffix Search, #1268 Search Suggestions System
