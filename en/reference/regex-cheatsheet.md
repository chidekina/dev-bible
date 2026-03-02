# Regex Cheatsheet

## Character Classes

| Pattern | Matches |
|---------|---------|
| `.` | Any character except newline (use `[\s\S]` for all) |
| `\d` | Digit `[0-9]` |
| `\D` | Non-digit `[^0-9]` |
| `\w` | Word char `[a-zA-Z0-9_]` |
| `\W` | Non-word char |
| `\s` | Whitespace `[ \t\r\n\f\v]` |
| `\S` | Non-whitespace |
| `\b` | Word boundary (between `\w` and `\W`) |
| `\B` | Non-word boundary |
| `[abc]` | Any of a, b, c |
| `[^abc]` | Any except a, b, c |
| `[a-z]` | Range a through z |
| `[a-zA-Z0-9]` | Alphanumeric |

---

## Quantifiers

| Pattern | Meaning | Greedy | Example |
|---------|---------|--------|---------|
| `*` | 0 or more | Yes | `a*` matches `""`, `"a"`, `"aaa"` |
| `+` | 1 or more | Yes | `a+` matches `"a"`, `"aaa"` |
| `?` | 0 or 1 | Yes | `colou?r` matches `color` / `colour` |
| `{n}` | Exactly n | — | `\d{4}` matches `2024` |
| `{n,}` | n or more | Yes | `\d{2,}` matches `12`, `123`... |
| `{n,m}` | n to m | Yes | `\d{2,4}` matches `12`, `123`, `1234` |
| `*?` | 0 or more | **Lazy** | `a.*?b` matches shortest between a and b |
| `+?` | 1 or more | **Lazy** | |
| `??` | 0 or 1 | **Lazy** | |
| `{n,m}?` | n to m | **Lazy** | |

**Greedy vs Lazy:** Greedy matches as much as possible; lazy matches as little as possible.

```
Input: "<b>bold</b> and <i>italic</i>"
<.+>   (greedy) → "<b>bold</b> and <i>italic</i>"  ← whole string
<.+?>  (lazy)   → "<b>"  ← shortest match
```

---

## Anchors

| Pattern | Meaning |
|---------|---------|
| `^` | Start of string (or line with `m` flag) |
| `$` | End of string (or line with `m` flag) |
| `\A` | Start of string (ignores `m` flag) — PCRE/Python only |
| `\Z` | End of string — PCRE/Python only |
| `\b` | Word boundary |
| `\B` | Non-word boundary |

```
/^\d{5}$/   matches "12345" but not "12345-6789" or " 12345"
/^/m        matches start of each line in multiline mode
```

---

## Groups and Backreferences

```
(abc)          capturing group — captured in $1 / \1
(?:abc)        non-capturing group — groups without capturing
(?<name>abc)   named capturing group — accessed as $<name> or \k<name>
\1             backreference to group 1 (same text captured)
\k<name>       named backreference

(?:foo|bar)    alternation inside non-capturing group
(foo|bar)      alternation with capture
```

```javascript
// Capturing groups
const match = '2024-03-15'.match(/(\d{4})-(\d{2})-(\d{2})/);
// match[1] = '2024', match[2] = '03', match[3] = '15'

// Named groups
const { year, month, day } = '2024-03-15'.match(
  /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/
).groups;

// Backreference — match repeated word
/\b(\w+)\s+\1\b/.test('the the')  // true — finds doubled words

// Replace with group reference
'John Smith'.replace(/(\w+)\s(\w+)/, '$2, $1') // → "Smith, John"
'2024-03-15'.replace(/(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})/, '$<d>/$<m>/$<y>')
// → "15/03/2024"
```

---

## Lookaheads and Lookbehinds

```
(?=...)    positive lookahead  — must be followed by ...
(?!...)    negative lookahead  — must NOT be followed by ...
(?<=...)   positive lookbehind — must be preceded by ...
(?<!...)   negative lookbehind — must NOT be preceded by ...
```

```javascript
// Lookahead: match word only if followed by something
/\d+(?= dollars)/        // matches "100" in "100 dollars" but not "100 euros"
/\d+(?! dollars)/        // matches "100" in "100 euros" but not "100 dollars"

// Lookbehind: match only if preceded by something
/(?<=\$)\d+/             // matches "100" in "$100" (not "100 items")
/(?<!\$)\d+/             // matches "100" in "100 items" (not "$100")

// Password validation — all conditions using lookaheads
/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^a-zA-Z\d]).{8,}$/
// At least: 1 lowercase, 1 uppercase, 1 digit, 1 special, 8+ chars

// Extract price without currency symbol
'Price: $19.99'.match(/(?<=\$)[\d.]+/)  // → ["19.99"]
```

---

## Flags

| Flag | JS | Meaning |
|------|----|---------|
| `g` | global | Find all matches (not just first) |
| `i` | ignoreCase | Case-insensitive |
| `m` | multiline | `^`/`$` match line starts/ends |
| `s` | dotAll | `.` matches newline |
| `u` | unicode | Enable full Unicode support |
| `v` | unicodeSets | Enhanced Unicode (ES2024, superset of u) |
| `d` | hasIndices | Include match indices in result |
| `y` | sticky | Match only at `lastIndex` position |

```javascript
/hello/gi.test('Hello World')    // true — global + case-insensitive
/^line/gm.test('first\nline')    // true — multiline
/.+/s.test('line1\nline2')       // true — dotAll makes . match \n
```

---

## JavaScript Regex Methods

```javascript
// RegExp methods
const re = /pattern/flags;
re.test('string')              // boolean — does pattern match?
re.exec('string')              // array | null — first match with groups
re.lastIndex                   // used with sticky/global for position

// String methods
'str'.match(/pattern/)         // array | null (first match, with groups)
'str'.match(/pattern/g)        // array of all matches (no groups)
'str'.matchAll(/pattern/g)     // iterator of all matches WITH groups
'str'.search(/pattern/)        // index of first match, or -1
'str'.replace(/pattern/, '$1') // replace first match
'str'.replaceAll(/pattern/g, '') // replace all (must use /g)
'str'.split(/,\s*/)            // split on pattern

// matchAll — best for extracting all matches with groups
const text = 'color: red; background: blue';
const matches = [...text.matchAll(/(\w+):\s*(\w+)/g)];
// matches[0] = ['color: red', 'color', 'red', ...]
// matches[1] = ['background: blue', 'background', 'blue', ...]

// replaceAll with function
'hello world'.replace(/\b\w/g, (char) => char.toUpperCase());
// → 'Hello World'

'2024-03-15'.replace(
  /(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})/,
  (_, y, m, d) => `${d}/${m}/${y}`
);
// → '15/03/2024'
```

---

## Common Patterns

### Email

```
/^[^\s@]+@[^\s@]+\.[^\s@]+$/
```
Pragmatic — accepts most valid emails. Use a proper validation library for critical paths.

### URL

```
/https?:\/\/[\w\-.]+(:\d+)?(\/[^\s]*)?/
```
For full RFC 3986 compliance, use the `URL` constructor and catch exceptions instead.

### Phone (flexible)

```
/^[+]?[(]?[0-9]{1,4}[)]?[-\s./0-9]{6,14}[0-9]$/
```

### US Phone

```
/^(\+1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}$/
```

### Date YYYY-MM-DD

```
/^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$/
```

### IPv4

```
/^((25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(25[0-5]|2[0-4]\d|[01]?\d\d?)$/
```

### IPv6 (simplified)

```
/^([0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}$/
```

### Hex color

```
/^#([0-9a-fA-F]{3}|[0-9a-fA-F]{6})$/i
```

### Slug

```
/^[a-z0-9]+(?:-[a-z0-9]+)*$/
```

### Strong password

```
/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*]).{8,}$/
```

### JWT

```
/^[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+\.[A-Za-z0-9-_]*$/
```

### Semver

```
/^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(?:-((?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*)(?:\.(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?(?:\+([0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?$/
```

### Credit card (basic format)

```
/^(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13}|6(?:011|5[0-9]{2})[0-9]{12})$/
```

---

## Named Capture Groups

```javascript
// Definition
const re = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;

// Access
const m = '2024-03-15'.match(re);
m.groups.year   // '2024'
m.groups.month  // '03'
m.groups.day    // '15'

// Destructuring
const { year, month, day } = '2024-03-15'.match(re).groups;

// Replace with named reference
'2024-03-15'.replace(re, '$<day>/$<month>/$<year>') // → '15/03/2024'

// Named backreference in pattern
/(?<word>\w+)\s+\k<word>/ // matches doubled words: "the the"
```

---

## Useful Techniques

```javascript
// Escape user input for use in regex
function escapeRegex(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

// Build regex from string
const pattern = 'hello';
const re = new RegExp(escapeRegex(pattern), 'gi');

// Test with multiple patterns (OR)
const keywords = ['error', 'fail', 'exception'];
const re = new RegExp(keywords.map(escapeRegex).join('|'), 'i');

// Trim whitespace (equivalent to .trim() but as regex)
'  hello  '.replace(/^\s+|\s+$/g, '')

// Normalize whitespace
'hello   world\ttab'.replace(/\s+/g, ' ')

// Remove HTML tags
html.replace(/<[^>]*>/g, '')

// Extract all URLs from text
const urls = [...text.matchAll(/https?:\/\/[\w\-./?=%&#+:@!~,*()[\]']+/g)]
  .map(m => m[0]);

// Camel to snake case
'camelCaseString'.replace(/([a-z])([A-Z])/g, '$1_$2').toLowerCase()
// → 'camel_case_string'

// Snake to camel case
'snake_case_string'.replace(/_([a-z])/g, (_, c) => c.toUpperCase())
// → 'snakeCaseString'
```

---

## Performance Tips

- **Avoid catastrophic backtracking:** nested quantifiers like `(a+)+` on non-matching input causes exponential time. Use atomic groups or possessive quantifiers if your engine supports them.
- **Prefer character classes over alternation for single chars:** `[aeiou]` is faster than `(a|e|i|o|u)`.
- **Anchor when possible:** `/^\d+$/` is faster than `/\d+/` because it fails early.
- **Compile once, use many times:** Store `/pattern/` in a variable rather than recreating it in a loop.
- **Use `String.includes()` for literal substring checks** — regex has overhead for simple cases.
- **Non-capturing groups `(?:)` are marginally faster** than capturing groups when you don't need the capture.
