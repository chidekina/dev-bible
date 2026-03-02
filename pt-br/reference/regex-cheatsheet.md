# Regex Cheatsheet

## Classes de Caracteres

| Padrão | Corresponde |
|--------|-------------|
| `.` | Qualquer caractere exceto nova linha (use `[\s\S]` para todos) |
| `\d` | Dígito `[0-9]` |
| `\D` | Não-dígito `[^0-9]` |
| `\w` | Caractere de palavra `[a-zA-Z0-9_]` |
| `\W` | Caractere não-palavra |
| `\s` | Espaço em branco `[ \t\r\n\f\v]` |
| `\S` | Não-espaço |
| `\b` | Limite de palavra (entre `\w` e `\W`) |
| `\B` | Não-limite de palavra |
| `[abc]` | Qualquer um de a, b, c |
| `[^abc]` | Qualquer exceto a, b, c |
| `[a-z]` | Intervalo de a até z |
| `[a-zA-Z0-9]` | Alfanumérico |

---

## Quantificadores

| Padrão | Significado | Ganancioso | Exemplo |
|--------|-------------|------------|---------|
| `*` | 0 ou mais | Sim | `a*` corresponde a `""`, `"a"`, `"aaa"` |
| `+` | 1 ou mais | Sim | `a+` corresponde a `"a"`, `"aaa"` |
| `?` | 0 ou 1 | Sim | `colou?r` corresponde a `color` / `colour` |
| `{n}` | Exatamente n | — | `\d{4}` corresponde a `2024` |
| `{n,}` | n ou mais | Sim | `\d{2,}` corresponde a `12`, `123`... |
| `{n,m}` | n a m | Sim | `\d{2,4}` corresponde a `12`, `123`, `1234` |
| `*?` | 0 ou mais | **Preguiçoso** | `a.*?b` corresponde ao menor trecho entre a e b |
| `+?` | 1 ou mais | **Preguiçoso** | |
| `??` | 0 ou 1 | **Preguiçoso** | |
| `{n,m}?` | n a m | **Preguiçoso** | |

**Ganancioso vs Preguiçoso:** Ganancioso corresponde ao máximo possível; preguiçoso ao mínimo possível.

```
Entrada: "<b>bold</b> and <i>italic</i>"
<.+>   (ganancioso) → "<b>bold</b> and <i>italic</i>"  ← string inteira
<.+?>  (preguiçoso) → "<b>"  ← correspondência mais curta
```

---

## Âncoras

| Padrão | Significado |
|--------|-------------|
| `^` | Início da string (ou linha com flag `m`) |
| `$` | Fim da string (ou linha com flag `m`) |
| `\A` | Início da string (ignora flag `m`) — apenas PCRE/Python |
| `\Z` | Fim da string — apenas PCRE/Python |
| `\b` | Limite de palavra |
| `\B` | Não-limite de palavra |

```
/^\d{5}$/   corresponde a "12345" mas não a "12345-6789" ou " 12345"
/^/m        corresponde ao início de cada linha no modo multiline
```

---

## Grupos e Retrorreferências

```
(abc)          grupo de captura — capturado em $1 / \1
(?:abc)        grupo não-capturante — agrupa sem capturar
(?<name>abc)   grupo nomeado de captura — acessado como $<name> ou \k<name>
\1             retrorreferência ao grupo 1 (mesmo texto capturado)
\k<name>       retrorreferência nomeada

(?:foo|bar)    alternância dentro de grupo não-capturante
(foo|bar)      alternância com captura
```

```javascript
// Grupos de captura
const match = '2024-03-15'.match(/(\d{4})-(\d{2})-(\d{2})/);
// match[1] = '2024', match[2] = '03', match[3] = '15'

// Grupos nomeados
const { year, month, day } = '2024-03-15'.match(
  /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/
).groups;

// Retrorreferência — encontra palavra repetida
/\b(\w+)\s+\1\b/.test('the the')  // true — encontra palavras duplicadas

// Substituição com referência de grupo
'John Smith'.replace(/(\w+)\s(\w+)/, '$2, $1') // → "Smith, John"
'2024-03-15'.replace(/(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})/, '$<d>/$<m>/$<y>')
// → "15/03/2024"
```

---

## Lookaheads e Lookbehinds

```
(?=...)    lookahead positivo  — deve ser seguido por ...
(?!...)    lookahead negativo  — NÃO deve ser seguido por ...
(?<=...)   lookbehind positivo — deve ser precedido por ...
(?<!...)   lookbehind negativo — NÃO deve ser precedido por ...
```

```javascript
// Lookahead: corresponde à palavra apenas se seguida por algo
/\d+(?= dollars)/        // corresponde a "100" em "100 dollars" mas não em "100 euros"
/\d+(?! dollars)/        // corresponde a "100" em "100 euros" mas não em "100 dollars"

// Lookbehind: corresponde apenas se precedido por algo
/(?<=\$)\d+/             // corresponde a "100" em "$100" (não em "100 itens")
/(?<!\$)\d+/             // corresponde a "100" em "100 itens" (não em "$100")

// Validação de senha — todas as condições usando lookaheads
/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^a-zA-Z\d]).{8,}$/
// Ao menos: 1 minúscula, 1 maiúscula, 1 dígito, 1 especial, 8+ chars

// Extrair preço sem símbolo de moeda
'Price: $19.99'.match(/(?<=\$)[\d.]+/)  // → ["19.99"]
```

---

## Flags

| Flag | JS | Significado |
|------|----|-------------|
| `g` | global | Encontra todas as correspondências (não apenas a primeira) |
| `i` | ignoreCase | Sem distinção de maiúsculas/minúsculas |
| `m` | multiline | `^`/`$` correspondem a inícios/fins de linha |
| `s` | dotAll | `.` corresponde a nova linha |
| `u` | unicode | Habilita suporte completo a Unicode |
| `v` | unicodeSets | Unicode aprimorado (ES2024, superset de u) |
| `d` | hasIndices | Inclui índices de correspondência no resultado |
| `y` | sticky | Corresponde apenas na posição `lastIndex` |

```javascript
/hello/gi.test('Hello World')    // true — global + sem distinção de maiúsculas
/^line/gm.test('first\nline')    // true — multiline
/.+/s.test('line1\nline2')       // true — dotAll faz . corresponder a \n
```

---

## Métodos Regex do JavaScript

```javascript
// Métodos de RegExp
const re = /pattern/flags;
re.test('string')              // boolean — o padrão corresponde?
re.exec('string')              // array | null — primeira correspondência com grupos
re.lastIndex                   // usado com sticky/global para posição

// Métodos de String
'str'.match(/pattern/)         // array | null (primeira correspondência, com grupos)
'str'.match(/pattern/g)        // array de todas as correspondências (sem grupos)
'str'.matchAll(/pattern/g)     // iterador de todas as correspondências COM grupos
'str'.search(/pattern/)        // índice da primeira correspondência, ou -1
'str'.replace(/pattern/, '$1') // substitui primeira correspondência
'str'.replaceAll(/pattern/g, '') // substitui todas (deve usar /g)
'str'.split(/,\s*/)            // divide no padrão

// matchAll — melhor para extrair todas as correspondências com grupos
const text = 'color: red; background: blue';
const matches = [...text.matchAll(/(\w+):\s*(\w+)/g)];
// matches[0] = ['color: red', 'color', 'red', ...]
// matches[1] = ['background: blue', 'background', 'blue', ...]

// replaceAll com função
'hello world'.replace(/\b\w/g, (char) => char.toUpperCase());
// → 'Hello World'

'2024-03-15'.replace(
  /(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})/,
  (_, y, m, d) => `${d}/${m}/${y}`
);
// → '15/03/2024'
```

---

## Padrões Comuns

### E-mail

```
/^[^\s@]+@[^\s@]+\.[^\s@]+$/
```
Pragmático — aceita a maioria dos e-mails válidos. Use uma biblioteca de validação adequada para caminhos críticos.

### URL

```
/https?:\/\/[\w\-.]+(:\d+)?(\/[^\s]*)?/
```
Para conformidade total com RFC 3986, use o construtor `URL` e capture exceções.

### Telefone (flexível)

```
/^[+]?[(]?[0-9]{1,4}[)]?[-\s./0-9]{6,14}[0-9]$/
```

### Telefone EUA

```
/^(\+1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}$/
```

### Data YYYY-MM-DD

```
/^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$/
```

### IPv4

```
/^((25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(25[0-5]|2[0-4]\d|[01]?\d\d?)$/
```

### IPv6 (simplificado)

```
/^([0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}$/
```

### Cor hexadecimal

```
/^#([0-9a-fA-F]{3}|[0-9a-fA-F]{6})$/i
```

### Slug

```
/^[a-z0-9]+(?:-[a-z0-9]+)*$/
```

### Senha forte

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

### Cartão de crédito (formato básico)

```
/^(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13}|6(?:011|5[0-9]{2})[0-9]{12})$/
```

---

## Grupos de Captura Nomeados

```javascript
// Definição
const re = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;

// Acesso
const m = '2024-03-15'.match(re);
m.groups.year   // '2024'
m.groups.month  // '03'
m.groups.day    // '15'

// Desestruturação
const { year, month, day } = '2024-03-15'.match(re).groups;

// Substituição com referência nomeada
'2024-03-15'.replace(re, '$<day>/$<month>/$<year>') // → '15/03/2024'

// Retrorreferência nomeada no padrão
/(?<word>\w+)\s+\k<word>/ // corresponde a palavras duplicadas: "the the"
```

---

## Técnicas Úteis

```javascript
// Escapar entrada do usuário para uso em regex
function escapeRegex(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

// Construir regex a partir de string
const pattern = 'hello';
const re = new RegExp(escapeRegex(pattern), 'gi');

// Testar com múltiplos padrões (OU)
const keywords = ['error', 'fail', 'exception'];
const re = new RegExp(keywords.map(escapeRegex).join('|'), 'i');

// Remover espaços (equivalente ao .trim() mas como regex)
'  hello  '.replace(/^\s+|\s+$/g, '')

// Normalizar espaços em branco
'hello   world\ttab'.replace(/\s+/g, ' ')

// Remover tags HTML
html.replace(/<[^>]*>/g, '')

// Extrair todas as URLs de um texto
const urls = [...text.matchAll(/https?:\/\/[\w\-./?=%&#+:@!~,*()[\]']+/g)]
  .map(m => m[0]);

// camelCase para snake_case
'camelCaseString'.replace(/([a-z])([A-Z])/g, '$1_$2').toLowerCase()
// → 'camel_case_string'

// snake_case para camelCase
'snake_case_string'.replace(/_([a-z])/g, (_, c) => c.toUpperCase())
// → 'snakeCaseString'
```

---

## Dicas de Performance

- **Evite backtracking catastrófico:** quantificadores aninhados como `(a+)+` em entradas sem correspondência causam tempo exponencial. Use grupos atômicos ou quantificadores possessivos se seu motor suportar.
- **Prefira classes de caracteres a alternância para chars únicos:** `[aeiou]` é mais rápido que `(a|e|i|o|u)`.
- **Ancore quando possível:** `/^\d+$/` é mais rápido que `/\d+/` porque falha antecipadamente.
- **Compile uma vez, use muitas vezes:** armazene `/padrão/` em uma variável em vez de recriar em loop.
- **Use `String.includes()` para verificações de substring literal** — regex tem overhead para casos simples.
- **Grupos não-capturantes `(?:)` são marginalmente mais rápidos** que grupos capturantes quando você não precisa da captura.
