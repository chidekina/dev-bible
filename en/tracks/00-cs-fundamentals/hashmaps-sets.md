# Hashmaps & Sets

## 1. What & Why

Hashmaps (hash tables) and Sets are the Swiss Army knives of algorithm problem-solving. They trade space for speed, converting O(n) search problems into O(1) lookup. The moment you find yourself scanning an array repeatedly looking for a value, a hashmap or set is usually the answer.

In JavaScript/TypeScript, `Map` and `Set` are built-in hash table implementations. Plain objects (`{}`) also behave like hashmaps but with important differences. Understanding when to use which — and how they work internally — is critical for writing correct, performant code.

---

## 2. Core Concepts

### Hash Function

A hash function converts a key (any type) into an integer index within a fixed-size backing array. Properties of a good hash function:
- **Deterministic:** same key always produces the same index
- **Uniform distribution:** spreads keys evenly to minimize collisions
- **Fast to compute:** O(1)

```
key "apple" → hashCode → 7289 % capacity → index 17
key "banana" → hashCode → 9014 % capacity → index 3
```

### Collision Resolution

Two keys can hash to the same index — this is a **collision**. Two main strategies:

**Chaining:** Each bucket holds a linked list of (key, value) pairs. Lookup: hash to bucket, then scan the list. Average O(1), worst case O(n) if all keys collide.

**Open Addressing:** When a collision occurs, probe for the next empty slot:
- **Linear probing:** try index+1, index+2, ...
- **Quadratic probing:** try index+1², index+2², ...
- **Double hashing:** use a second hash function for the step size

### Load Factor and Rehashing

Load factor = `number_of_entries / capacity`. When load factor exceeds a threshold (typically 0.75 for Java's HashMap, similar for V8), the table is rehashed:
1. Allocate a new array (typically 2× capacity)
2. Re-insert all existing entries
3. Discard old array

This keeps average chain length (and thus lookup time) bounded at O(1) amortized.

### JavaScript Map vs Plain Object

| Feature | `Map` | `{}` (Object) |
|---|---|---|
| Key types | Any type | Strings and Symbols only |
| Insertion order | Preserved | Preserved (modern JS) |
| Size | `.size` O(1) | `Object.keys().length` O(n) |
| Prototype keys | None | Inherited (can cause bugs) |
| Performance (add/delete) | Better for frequent mutations | Slightly faster for static use |
| Serialization | Manual | JSON.stringify works directly |

> 💡 Use `Map` when keys are non-string, when you need `.size` frequently, or when performance for frequent insertions/deletions matters. Use plain objects for simple config-like data or when JSON interop is needed.

### WeakMap and WeakSet

`WeakMap` and `WeakSet` hold **weak references** to their keys. If the key object is garbage collected, the entry is automatically removed. Used for:
- Caching computed results per DOM node without memory leaks
- Private data storage per object instance (historical pattern, superseded by `#private`)
- Tracking object metadata without preventing GC

---

## 3. How It Works

```
HashMap internals (chaining, capacity = 8):

Buckets:
[0]: null
[1]: null
[2]: ("dog", 1) → null
[3]: null
[4]: ("cat", 2) → ("act", 3) → null  ← collision! both hash to 4
[5]: null
[6]: ("fish", 4) → null
[7]: null

get("act"): hash("act") = 4 → scan bucket 4 → found ("act", 3) → return 3
Load factor = 3/8 = 0.375 → no rehash needed yet
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- Two Sum: O(n) using Map ---
// Find two indices where arr[i] + arr[j] === target
function twoSum(nums: number[], target: number): [number, number] | null {
  const seen = new Map<number, number>(); // value → index

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (seen.has(complement)) {
      return [seen.get(complement)!, i];
    }
    seen.set(nums[i], i);
  }
  return null;
}

// --- Group Anagrams: O(n × m) where m = avg word length ---
function groupAnagrams(strs: string[]): string[][] {
  const groups = new Map<string, string[]>();

  for (const str of strs) {
    const key = str.split("").sort().join(""); // sorted chars = canonical form
    if (!groups.has(key)) groups.set(key, []);
    groups.get(key)!.push(str);
  }

  return [...groups.values()];
}
// Example: ["eat","tea","tan","ate","nat","bat"]
// → [["eat","tea","ate"],["tan","nat"],["bat"]]

// --- Longest Consecutive Sequence: O(n) ---
// Find the length of the longest sequence of consecutive integers
function longestConsecutive(nums: number[]): number {
  const numSet = new Set(nums);
  let maxLen = 0;

  for (const num of numSet) {
    // Only start counting from the beginning of a sequence
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

// --- Set Operations ---
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

// --- LRU Cache: HashMap + Doubly Linked List ---
// O(1) get and put operations
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
  // Dummy head and tail sentinels — avoid null checks
  private head = new LRUNode(0, 0); // least recently used end
  private tail = new LRUNode(0, 0); // most recently used end

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
      const lru = this.head.next!; // least recently used is after dummy head
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

// --- HashMap from scratch: chaining approach ---
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

// --- Word frequency counter ---
function wordFrequency(text: string): Map<string, number> {
  const freq = new Map<string, number>();
  const words = text.toLowerCase().split(/\W+/).filter(Boolean);
  for (const word of words) {
    freq.set(word, (freq.get(word) ?? 0) + 1);
  }
  return freq;
}

// --- Find all pairs that sum to k: O(n) ---
function findPairsWithSum(nums: number[], k: number): [number, number][] {
  const seen = new Set<number>();
  const pairs: [number, number][] = [];
  const used = new Set<number>(); // avoid duplicate pairs

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

## 5. Common Mistakes & Pitfalls

> ⚠️ **Using a plain object as a hashmap with numeric keys.** JavaScript objects coerce numeric keys to strings. `obj[1]` and `obj["1"]` access the same property. Use `Map` to avoid this.

```typescript
const obj: Record<string, number> = {};
obj[1] = 100;
console.log(obj["1"]); // 100 — key was coerced to string "1"

const map = new Map<number, number>();
map.set(1, 100);
map.get(1);   // 100
map.get("1" as any); // undefined — different key type
```

> ⚠️ **Checking `map[key]` instead of `map.get(key)`.** Using bracket notation on a Map accesses the Map object's properties, not its entries. Always use `.get()`, `.set()`, `.has()`, `.delete()`.

> ⚠️ **Forgetting that `Map` and `Set` are iterable but not JSON-serializable.** `JSON.stringify(new Map())` returns `{}`. Convert to array or plain object before serializing.

> ⚠️ **Using object spread `{ ...obj }` to clone Maps.** This converts a Map to an empty object. Use `new Map(existingMap)` to copy a Map.

> ⚠️ **Confusing WeakMap with Map for caching.** `WeakMap` only accepts objects as keys, not primitives. Also, you cannot iterate over a WeakMap or get its size — it's intentional.

---

## 6. When to Use / Not Use

**Use a Map/hashmap when:**
- Membership testing for large collections (O(1) vs O(n) for arrays)
- Counting frequencies
- Grouping data by a key
- Implementing caches, indexes, or lookup tables
- Two-sum, anagram, and most "find duplicates" problems

**Use a Set when:**
- You need deduplication
- Membership testing without associated values
- Computing mathematical set operations (union, intersection, difference)

**Prefer plain object `{}` when:**
- Keys are always strings or known static identifiers
- You need JSON serialization directly
- It's simple config/record data with no dynamic key manipulation

**Avoid hashmaps when:**
- You need ordered traversal (use sorted array + binary search, or BST)
- Memory is extremely tight (hashmap overhead per entry is significant)
- Keys are complex objects with custom equality semantics (JS `Map` uses reference equality for objects)

---

## 7. Real-World Scenario

A URL shortener needs to map short codes to long URLs (and vice versa) with O(1) encode/decode:

```typescript
class URLShortener {
  private shortToLong = new Map<string, string>();
  private longToShort = new Map<string, string>();
  private counter = 0;
  private readonly BASE = "https://short.ly/";

  encode(longUrl: string): string {
    if (this.longToShort.has(longUrl)) {
      return this.longToShort.get(longUrl)!;
    }
    const short = this.BASE + (this.counter++).toString(36); // base36 = compact
    this.shortToLong.set(short, longUrl);
    this.longToShort.set(longUrl, short);
    return short;
  }

  decode(shortUrl: string): string | undefined {
    return this.shortToLong.get(shortUrl);
  }
}
```

A feature flag system uses a Set to check if a user is in a feature rollout:

```typescript
class FeatureFlags {
  private flags = new Map<string, Set<string>>(); // flagName → Set of userIds

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

## 8. Interview Questions

**Q1: How does a hashmap work internally?**
A: A hashmap uses a hash function to convert a key into an array index. On collision, chaining stores multiple entries in a linked list at that bucket, or open addressing probes for the next empty slot. Average O(1) lookup; worst case O(n) with all collisions.

**Q2: What is the load factor and why does it matter?**
A: Load factor = entries / capacity. When it exceeds ~0.75, the average chain length grows, degrading lookup toward O(n). Rehashing — doubling capacity and re-inserting all entries — restores O(1) amortized performance.

**Q3: Design an LRU cache.**
A: Combine a HashMap (key → node, for O(1) lookup) with a doubly linked list (for O(1) ordering moves). Get: look up node, move to MRU end. Put: insert at MRU end; if over capacity, evict from LRU end. All operations O(1).

**Q4: Find all pairs in an array that sum to k.**
A: One pass with a Set. For each element x, check if `k - x` is in the Set. If yes, record the pair. Otherwise, add x to the Set. O(n) time, O(n) space.

**Q5: What is the difference between Map and WeakMap?**
A: `Map` holds strong references — entries prevent GC of keys. `WeakMap` holds weak references — if no other reference to a key object exists, it can be GC'd and the entry disappears. `WeakMap` is non-iterable and has no `.size`. Used for attaching metadata to objects without preventing their GC.

**Q6: What is the time complexity of Map.get() in JavaScript?**
A: O(1) average. V8 uses a specialized hash table with probing. The hash is computed, the slot is read — typically one memory access. Worst case O(n) with all collisions, but this is practically impossible with a good hash function.

---

## 9. Exercises

**Exercise 1:** Implement a HashMap class from scratch using chaining. Support `set`, `get`, `has`, `delete`, and automatic rehashing at 0.75 load factor.

**Exercise 2:** Build a word frequency counter. Given a string of text, return a Map of words to their frequency, sorted by frequency descending.

**Exercise 3:** Find the first duplicate character in a string. Return the character and its second index. Achieve O(n) time.
*Hint: Set for visited characters.*

**Exercise 4:** Implement the `longestConsecutive` sequence algorithm from scratch without sorting. Achieve O(n).
*Hint: for each number x, only start counting if `x - 1` is NOT in the Set (i.e., x is the start of a sequence).*

**Exercise 5:** Given an array of integers and a target sum k, count the number of contiguous subarrays that sum to k.
*Hint: prefix sum + Map. `count[prefix_sum]` — if `prefixSum - k` has been seen before, those subarrays sum to k.*

---

## 10. Further Reading

- CLRS Chapter 11 — Hash Tables
- [HashMap internals in V8](https://v8.dev/blog/hash-code) — how V8 implements Map/Set
- [Open vs Closed Hashing — Wikipedia](https://en.wikipedia.org/wiki/Hash_table)
- LeetCode problems: #1 Two Sum, #49 Group Anagrams, #128 Longest Consecutive Sequence, #146 LRU Cache, #560 Subarray Sum Equals K
