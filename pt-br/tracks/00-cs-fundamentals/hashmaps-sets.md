# Hashmaps e Sets

## 1. O Que É e Por Que Importa

Hashmaps (tabelas hash) e Sets são os canivetes suíços da resolução de problemas de algoritmos. Eles trocam espaço por velocidade, convertendo problemas de busca O(n) em lookups O(1). No momento em que você se pega varrendo um array repetidamente em busca de um valor, um hashmap ou set geralmente é a resposta.

Em JavaScript/TypeScript, `Map` e `Set` são implementações de tabela hash integradas. Objetos simples (`{}`) também se comportam como hashmaps, mas com diferenças importantes. Entender quando usar qual — e como funcionam internamente — é crítico para escrever código correto e performático.

---

## 2. Conceitos Fundamentais

### Função Hash

Uma função hash converte uma chave (qualquer tipo) em um índice inteiro dentro de um array de suporte de tamanho fixo. Propriedades de uma boa função hash:
- **Determinística:** a mesma chave sempre produz o mesmo índice
- **Distribuição uniforme:** espalha chaves uniformemente para minimizar colisões
- **Rápida de calcular:** O(1)

```
chave "maçã" → hashCode → 7289 % capacidade → índice 17
chave "banana" → hashCode → 9014 % capacidade → índice 3
```

### Resolução de Colisões

Duas chaves podem produzir o mesmo índice — isso é uma **colisão**. Duas estratégias principais:

**Encadeamento:** Cada bucket mantém uma lista encadeada de pares (chave, valor). Lookup: hash para o bucket, depois varre a lista. Média O(1), pior caso O(n) se todas as chaves colidirem.

**Endereçamento Aberto:** Quando ocorre uma colisão, procura o próximo slot vazio:
- **Sondagem linear:** tenta índice+1, índice+2, ...
- **Sondagem quadrática:** tenta índice+1², índice+2², ...
- **Double hashing:** usa uma segunda função hash para o tamanho do passo

### Fator de Carga e Rehashing

Fator de carga = `número_de_entradas / capacidade`. Quando o fator de carga excede um limiar (tipicamente 0,75 para HashMap do Java, similar para V8), a tabela é rehashed:
1. Aloca um novo array (tipicamente 2× a capacidade)
2. Re-insere todas as entradas existentes
3. Descarta o array antigo

Isso mantém o comprimento médio da cadeia (e portanto o tempo de lookup) limitado a O(1) amortizado.

### Map vs Objeto Simples em JavaScript

| Característica | `Map` | `{}` (Object) |
|---|---|---|
| Tipos de chave | Qualquer tipo | Strings e Symbols apenas |
| Ordem de inserção | Preservada | Preservada (JS moderno) |
| Tamanho | `.size` O(1) | `Object.keys().length` O(n) |
| Chaves de protótipo | Nenhuma | Herdadas (pode causar bugs) |
| Performance (add/delete) | Melhor para mutações frequentes | Levemente mais rápido para uso estático |
| Serialização | Manual | JSON.stringify funciona diretamente |

> Use `Map` quando as chaves são não-string, quando você precisa de `.size` com frequência, ou quando a performance para inserções/remoções frequentes importa. Use objetos simples para dados estáticos tipo config ou quando interoperabilidade com JSON é necessária.

### WeakMap e WeakSet

`WeakMap` e `WeakSet` mantêm **referências fracas** para suas chaves. Se o objeto chave é coletado pelo garbage collector, a entrada é automaticamente removida. Usados para:
- Cache de resultados computados por nó DOM sem vazamentos de memória
- Armazenamento de dados privados por instância de objeto (padrão histórico, superado por `#private`)
- Rastrear metadados de objetos sem impedir o GC

---

## 3. Como Funciona

```
Internos de HashMap (encadeamento, capacidade = 8):

Buckets:
[0]: null
[1]: null
[2]: ("cachorro", 1) → null
[3]: null
[4]: ("gato", 2) → ("toga", 3) → null  ← colisão! ambos fazem hash para 4
[5]: null
[6]: ("peixe", 4) → null
[7]: null

get("toga"): hash("toga") = 4 → varre bucket 4 → encontrou ("toga", 3) → retorna 3
Fator de carga = 3/8 = 0,375 → sem rehash ainda necessário
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Two Sum: O(n) usando Map ---
// Encontra dois índices onde arr[i] + arr[j] === target
function twoSum(nums: number[], target: number): [number, number] | null {
  const seen = new Map<number, number>(); // valor → índice

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (seen.has(complement)) {
      return [seen.get(complement)!, i];
    }
    seen.set(nums[i], i);
  }
  return null;
}

// --- Agrupar Anagramas: O(n × m) onde m = comprimento médio da palavra ---
function groupAnagrams(strs: string[]): string[][] {
  const groups = new Map<string, string[]>();

  for (const str of strs) {
    const key = str.split("").sort().join(""); // chars ordenados = forma canônica
    if (!groups.has(key)) groups.set(key, []);
    groups.get(key)!.push(str);
  }

  return [...groups.values()];
}
// Exemplo: ["eat","tea","tan","ate","nat","bat"]
// → [["eat","tea","ate"],["tan","nat"],["bat"]]

// --- Sequência Consecutiva Mais Longa: O(n) ---
// Encontra o comprimento da sequência mais longa de inteiros consecutivos
function longestConsecutive(nums: number[]): number {
  const numSet = new Set(nums);
  let maxLen = 0;

  for (const num of numSet) {
    // Conta somente a partir do início de uma sequência
    if (!numSet.has(num - 1)) {
      let current = num;
      let length = 1;
      while (numSet.has(current + 1)) {
        current++;
        length++;
      }
      maxLen = Math.max(maxLen, length);
    }
  }
  return maxLen;
}

// --- Operações de Set ---
function setUnion<T>(a: Set<T>, b: Set<T>): Set<T> {
  return new Set([...a, ...b]);
}

function setIntersection<T>(a: Set<T>, b: Set<T>): Set<T> {
  return new Set([...a].filter(x => b.has(x)));
}

function setDifference<T>(a: Set<T>, b: Set<T>): Set<T> {
  return new Set([...a].filter(x => !b.has(x)));
}

function isSubset<T>(sub: Set<T>, sup: Set<T>): boolean {
  return [...sub].every(x => sup.has(x));
}

// --- Cache LRU: HashMap + Lista Duplamente Encadeada ---
// Operações get e put em O(1)
class LRUNode {
  constructor(
    public key: number,
    public value: number,
    public prev: LRUNode | null = null,
    public next: LRUNode | null = null
  ) {}
}

class LRUCache {
  private capacity: number;
  private map = new Map<number, LRUNode>();
  // Sentinelas dummy na cabeça e cauda — evitam verificações de null
  private head = new LRUNode(0, 0); // extremidade menos recentemente usada
  private tail = new LRUNode(0, 0); // extremidade mais recentemente usada

  constructor(capacity: number) {
    this.capacity = capacity;
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  get(key: number): number {
    if (!this.map.has(key)) return -1;
    const node = this.map.get(key)!;
    this.remove(node);
    this.insertAtTail(node);
    return node.value;
  }

  put(key: number, value: number): void {
    if (this.map.has(key)) {
      this.remove(this.map.get(key)!);
    }
    const node = new LRUNode(key, value);
    this.insertAtTail(node);
    this.map.set(key, node);

    if (this.map.size > this.capacity) {
      const lru = this.head.next!; // menos recentemente usado fica após dummy head
      this.remove(lru);
      this.map.delete(lru.key);
    }
  }

  private remove(node: LRUNode): void {
    node.prev!.next = node.next;
    node.next!.prev = node.prev;
  }

  private insertAtTail(node: LRUNode): void {
    node.prev = this.tail.prev;
    node.next = this.tail;
    this.tail.prev!.next = node;
    this.tail.prev = node;
  }
}

// --- HashMap do zero: abordagem de encadeamento ---
class HashMap<K, V> {
  private capacity: number;
  private buckets: Array<Array<[K, V]>>;
  private _size = 0;

  constructor(capacity = 16) {
    this.capacity = capacity;
    this.buckets = Array.from({ length: capacity }, () => []);
  }

  private hash(key: K): number {
    const str = String(key);
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = (hash * 31 + str.charCodeAt(i)) % this.capacity;
    }
    return hash;
  }

  set(key: K, value: V): void {
    const idx = this.hash(key);
    const bucket = this.buckets[idx];
    const existing = bucket.find(([k]) => k === key);
    if (existing) {
      existing[1] = value;
    } else {
      bucket.push([key, value]);
      this._size++;
      if (this._size / this.capacity > 0.75) this.rehash();
    }
  }

  get(key: K): V | undefined {
    const bucket = this.buckets[this.hash(key)];
    return bucket.find(([k]) => k === key)?.[1];
  }

  has(key: K): boolean {
    return this.get(key) !== undefined;
  }

  delete(key: K): boolean {
    const idx = this.hash(key);
    const bucket = this.buckets[idx];
    const i = bucket.findIndex(([k]) => k === key);
    if (i === -1) return false;
    bucket.splice(i, 1);
    this._size--;
    return true;
  }

  get size(): number { return this._size; }

  private rehash(): void {
    const old = this.buckets;
    this.capacity *= 2;
    this.buckets = Array.from({ length: this.capacity }, () => []);
    this._size = 0;
    for (const bucket of old) {
      for (const [k, v] of bucket) this.set(k, v);
    }
  }
}

// --- Contador de frequência de palavras ---
function wordFrequency(text: string): Map<string, number> {
  const freq = new Map<string, number>();
  const words = text.toLowerCase().split(/\W+/).filter(Boolean);
  for (const word of words) {
    freq.set(word, (freq.get(word) ?? 0) + 1);
  }
  return freq;
}

// --- Encontrar todos os pares que somam k: O(n) ---
function findPairsWithSum(nums: number[], k: number): [number, number][] {
  const seen = new Set<number>();
  const pairs: [number, number][] = [];
  const used = new Set<number>(); // evita pares duplicados

  for (const num of nums) {
    const complement = k - num;
    if (seen.has(complement) && !used.has(complement)) {
      pairs.push([complement, num]);
      used.add(num);
      used.add(complement);
    }
    seen.add(num);
  }
  return pairs;
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Usar um objeto simples como hashmap com chaves numéricas.** Objetos JavaScript coagem chaves numéricas para strings. `obj[1]` e `obj["1"]` acessam a mesma propriedade. Use `Map` para evitar isso.

```typescript
const obj: Record<string, number> = {};
obj[1] = 100;
console.log(obj["1"]); // 100 — chave foi coagida para string "1"

const map = new Map<number, number>();
map.set(1, 100);
map.get(1);   // 100
map.get("1" as any); // undefined — tipo de chave diferente
```

> ⚠️ **Verificar `map[key]` em vez de `map.get(key)`.** Usar notação de colchetes em um Map acessa as propriedades do objeto Map, não suas entradas. Sempre use `.get()`, `.set()`, `.has()`, `.delete()`.

> ⚠️ **Esquecer que `Map` e `Set` são iteráveis mas não serializáveis em JSON.** `JSON.stringify(new Map())` retorna `{}`. Converta para array ou objeto simples antes de serializar.

> ⚠️ **Usar object spread `{ ...obj }` para clonar Maps.** Isso converte um Map em um objeto vazio. Use `new Map(existingMap)` para copiar um Map.

> ⚠️ **Confundir WeakMap com Map para cache.** `WeakMap` aceita apenas objetos como chaves, não primitivos. Além disso, você não pode iterar sobre um WeakMap ou obter seu tamanho — isso é intencional.

---

## 6. Quando Usar / Não Usar

**Use um Map/hashmap quando:**
- Teste de pertencimento em coleções grandes (O(1) vs O(n) para arrays)
- Contagem de frequências
- Agrupar dados por uma chave
- Implementar caches, índices ou tabelas de lookup
- Problemas de two-sum, anagrama e a maioria de "encontrar duplicatas"

**Use um Set quando:**
- Precisa de deduplicação
- Teste de pertencimento sem valores associados
- Calcular operações matemáticas de conjunto (união, interseção, diferença)

**Prefira objeto simples `{}` quando:**
- As chaves são sempre strings ou identificadores estáticos conhecidos
- Precisa de serialização JSON direta
- São dados simples de config/registro sem manipulação dinâmica de chave

**Evite hashmaps quando:**
- Precisa de travessia ordenada (use array ordenado + busca binária, ou BST)
- A memória é extremamente limitada (overhead de hashmap por entrada é significativo)
- As chaves são objetos complexos com semântica de igualdade customizada (Map do JS usa igualdade referencial para objetos)

---

## 7. Cenário do Mundo Real

Um encurtador de URL precisa mapear códigos curtos para URLs longas (e vice-versa) com encode/decode em O(1):

```typescript
class URLShortener {
  private shortToLong = new Map<string, string>();
  private longToShort = new Map<string, string>();
  private counter = 0;
  private readonly BASE = "https://curto.ly/";

  encode(longUrl: string): string {
    if (this.longToShort.has(longUrl)) {
      return this.longToShort.get(longUrl)!;
    }
    const short = this.BASE + (this.counter++).toString(36); // base36 = compacto
    this.shortToLong.set(short, longUrl);
    this.longToShort.set(longUrl, short);
    return short;
  }

  decode(shortUrl: string): string | undefined {
    return this.shortToLong.get(shortUrl);
  }
}
```

Um sistema de feature flags usa um Set para verificar se um usuário está em um rollout:

```typescript
class FeatureFlags {
  private flags = new Map<string, Set<string>>(); // nomeDaFlag → Set de userIds

  enable(flag: string, userId: string): void {
    if (!this.flags.has(flag)) this.flags.set(flag, new Set());
    this.flags.get(flag)!.add(userId);
  }

  isEnabled(flag: string, userId: string): boolean {
    return this.flags.get(flag)?.has(userId) ?? false;
  }

  getUsersForFlag(flag: string): string[] {
    return [...(this.flags.get(flag) ?? [])];
  }
}
```

---

## 8. Perguntas de Entrevista

**Q1: Como um hashmap funciona internamente?**
R: Um hashmap usa uma função hash para converter uma chave em um índice de array. Em colisão, encadeamento armazena múltiplas entradas em uma lista encadeada naquele bucket, ou endereçamento aberto procura o próximo slot vazio. Lookup médio O(1); pior caso O(n) com todas as colisões.

**Q2: O que é o fator de carga e por que importa?**
R: Fator de carga = entradas / capacidade. Quando excede ~0,75, o comprimento médio da cadeia cresce, degradando o lookup para O(n). Rehashing — dobrando a capacidade e re-inserindo todas as entradas — restaura performance O(1) amortizada.

**Q3: Projete um cache LRU.**
R: Combine um HashMap (chave → nó, para lookup O(1)) com uma lista duplamente encadeada (para movimentações de ordem em O(1)). Get: busca o nó, move para extremidade MRU. Put: insere na extremidade MRU; se acima da capacidade, despeja da extremidade LRU. Todas as operações O(1).

**Q4: Encontre todos os pares em um array que somam k.**
R: Uma passagem com um Set. Para cada elemento x, verifique se `k - x` está no Set. Se sim, registre o par. Caso contrário, adicione x ao Set. O(n) de tempo, O(n) de espaço.

**Q5: Qual é a diferença entre Map e WeakMap?**
R: `Map` mantém referências fortes — entradas impedem o GC das chaves. `WeakMap` mantém referências fracas — se não há outra referência para um objeto chave, ele pode ser coletado e a entrada desaparece. `WeakMap` não é iterável e não tem `.size`. Usado para anexar metadados a objetos sem impedir seu GC.

**Q6: Qual é a complexidade de tempo de `Map.get()` em JavaScript?**
R: O(1) em média. O V8 usa uma tabela hash especializada com sondagem. O hash é calculado, o slot é lido — tipicamente um acesso à memória. Pior caso O(n) com todas as colisões, mas isso é praticamente impossível com uma boa função hash.

---

## 9. Exercícios

**Exercício 1:** Implemente uma classe HashMap do zero usando encadeamento. Suporte `set`, `get`, `has`, `delete` e rehashing automático a 0,75 de fator de carga.

**Exercício 2:** Construa um contador de frequência de palavras. Dado um texto, retorne um Map de palavras para sua frequência, ordenado por frequência decrescente.

**Exercício 3:** Encontre o primeiro caractere duplicado em uma string. Retorne o caractere e seu segundo índice. Alcance O(n) de tempo.
*Dica: Set para caracteres visitados.*

**Exercício 4:** Implemente o algoritmo `longestConsecutive` do zero sem ordenação. Alcance O(n).
*Dica: para cada número x, comece a contar apenas se `x - 1` NÃO está no Set (ou seja, x é o início de uma sequência).*

**Exercício 5:** Dado um array de inteiros e uma soma alvo k, conte o número de subarrays contíguos que somam k.
*Dica: soma prefixada + Map. `count[soma_prefixada]` — se `somaPrefix - k` foi visto antes, esses subarrays somam k.*

---

## 10. Leituras Complementares

- CLRS Capítulo 11 — Hash Tables
- [Internos do HashMap no V8](https://v8.dev/blog/hash-code) — como o V8 implementa Map/Set
- [Hash Aberto vs Fechado — Wikipedia](https://en.wikipedia.org/wiki/Hash_table)
- Problemas LeetCode: #1 Two Sum, #49 Group Anagrams, #128 Longest Consecutive Sequence, #146 LRU Cache, #560 Subarray Sum Equals K
