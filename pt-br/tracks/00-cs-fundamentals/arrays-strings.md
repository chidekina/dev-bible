# Arrays e Strings

## 1. O Que É e Por Que Importa

Arrays e strings são as estruturas de dados mais fundamentais da programação. Quase todo problema de algoritmo os envolve, e quase todo sistema em produção os manipula constantemente. Entender seu layout de memória, características de performance e padrões idiomáticos é essencial antes de abordar estruturas de dados mais complexas.

Arrays oferecem acesso indexado em O(1) e iteração amigável ao cache — são a estrutura mais rápida para processamento sequencial. Strings, sendo sequências imutáveis de caracteres na maioria das linguagens (incluindo JavaScript/TypeScript), têm armadilhas de performance que pegam engenheiros experientes de surpresa.

---

## 2. Conceitos Fundamentais

### Arrays Estáticos vs Dinâmicos

Um **array estático** tem tamanho fixo definido na criação. A memória é alocada uma vez — um bloco contíguo. Acessar o elemento `i` calcula `endereço_base + i × tamanho_do_elemento` em O(1).

Um **array dinâmico** (`Array` do JavaScript, `ArrayList` do Java, `vector` do C++) cresce automaticamente. Internamente mantém um array estático de suporte. Quando a capacidade é excedida:
1. Um novo array é alocado — tipicamente com 2× a capacidade antiga
2. Todos os elementos são copiados — O(n)
3. O array antigo é coletado pelo garbage collector

Essa estratégia de duplicação mantém o **custo amortizado do push em O(1)**. Ao longo de n pushes, o trabalho total de cópia é 1 + 2 + 4 + ... + n = 2n = O(n). Dividido por n operações: O(1) cada.

### Imutabilidade de Strings

Em JavaScript/TypeScript, strings são **imutáveis**. Toda operação que parece "modificar" uma string na verdade cria uma nova. Isso tem uma implicação crítica: concatenar em loop é O(n²).

```
"a" + "b" = cria nova string "ab"          (copia 2 chars)
"ab" + "c" = cria nova string "abc"        (copia 3 chars)
"abc" + "d" = cria nova string "abcd"      (copia 4 chars)
...
Total de cópias após n concatenações: 1+2+3+...+n = n(n+1)/2 = O(n²)
```

Solução: colete as partes em um array e faça join uma única vez no final.

### Layout de Memória

Arrays armazenam elementos em **memória contígua**. Isso significa:
- Acesso por índice é O(1) — aritmética sobre o ponteiro base
- Iteração sequencial é amigável ao cache — a CPU faz prefetch dos próximos elementos
- Inserção/remoção no meio é O(n) — todos os elementos subsequentes precisam ser deslocados

A amigabilidade ao cache dos arrays é a razão pela qual um loop simples sobre array frequentemente supera a travessia de uma lista encadeada na prática, mesmo para operações teoricamente de mesma complexidade.

---

## 3. Como Funciona

```
Array na memória (cada elemento ocupa 8 bytes em sistema 64-bit):

Índice:  0       1       2       3       4
       [100]   [200]   [300]   [400]   [500]
End.:  1000    1008    1016    1024    1032

Acesso arr[3]: endereço = 1000 + 3 × 8 = 1024 → O(1)
```

**Redimensionamento de array dinâmico:**
```
Inicial: capacidade=4, tamanho=4  →  [a, b, c, d]
push(e): capacidade cheia → aloca tamanho 8, copia [a,b,c,d], adiciona e
Resultado: capacidade=8, tamanho=5  →  [a, b, c, d, e, _, _, _]
```

**Internos da concatenação de strings:**
```
s = ""
s += "hello"   // aloca "hello"               — 5 chars
s += " mundo"  // aloca "hello mundo"         — 11 chars (copia 5+6)
s += "!"       // aloca "hello mundo!"        — 12 chars (copia 11+1)
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Concatenação de strings: armadilha O(n²) vs solução O(n) ---

// Ruim: O(n²) — cria n strings intermediárias
function buildStringBad(parts: string[]): string {
  let result = "";
  for (const part of parts) {
    result += part; // aloca uma nova string a cada iteração
  }
  return result;
}

// Bom: O(n) — uma única alocação no final
function buildStringGood(parts: string[]): string {
  return parts.join(""); // internamente: aloca uma vez, preenche no lugar
}

// --- Técnica de dois ponteiros ---

// Verificar se uma string é palíndromo: O(n) de tempo, O(1) de espaço
function isPalindrome(s: string): boolean {
  let left = 0;
  let right = s.length - 1;
  while (left < right) {
    if (s[left] !== s[right]) return false;
    left++;
    right--;
  }
  return true;
}

// Two-sum em array ordenado: O(n) de tempo, O(1) de espaço
function twoSumSorted(arr: number[], target: number): [number, number] | null {
  let left = 0;
  let right = arr.length - 1;
  while (left < right) {
    const sum = arr[left] + arr[right];
    if (sum === target) return [left, right];
    if (sum < target) left++;   // precisa mais, avança esquerda
    else right--;               // demais, recua direita
  }
  return null;
}

// Inverter string in-place (sobre array de chars): O(n) de tempo, O(1) de espaço
function reverseString(chars: string[]): void {
  let left = 0, right = chars.length - 1;
  while (left < right) {
    [chars[left], chars[right]] = [chars[right], chars[left]];
    left++;
    right--;
  }
}

// --- Janela deslizante (sliding window) ---

// Janela fixa: soma máxima de subarray de tamanho k — O(n)
function maxSumSubarray(arr: number[], k: number): number {
  if (arr.length < k) throw new Error("Array menor que k");

  // Constrói janela inicial
  let windowSum = arr.slice(0, k).reduce((a, b) => a + b, 0);
  let maxSum = windowSum;

  // Desliza: subtrai elemento que sai, adiciona elemento que entra
  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k];
    maxSum = Math.max(maxSum, windowSum);
  }
  return maxSum;
}

// Janela variável: maior substring sem caracteres repetidos — O(n)
function lengthOfLongestSubstring(s: string): number {
  const charIndex = new Map<string, number>(); // char -> último índice visto
  let maxLen = 0;
  let left = 0;

  for (let right = 0; right < s.length; right++) {
    const char = s[right];
    // Se o char foi visto e está dentro da janela atual, encolhe left além dele
    if (charIndex.has(char) && charIndex.get(char)! >= left) {
      left = charIndex.get(char)! + 1;
    }
    charIndex.set(char, right);
    maxLen = Math.max(maxLen, right - left + 1);
  }
  return maxLen;
}

// --- Verificação de anagrama: O(n) usando mapa de frequência ---
function isAnagram(s: string, t: string): boolean {
  if (s.length !== t.length) return false;
  const freq = new Map<string, number>();
  for (const c of s) freq.set(c, (freq.get(c) ?? 0) + 1);
  for (const c of t) {
    if (!freq.has(c) || freq.get(c)! === 0) return false;
    freq.set(c, freq.get(c)! - 1);
  }
  return true;
}

// --- indexOf do zero: O(n × m) ingênuo ---
// Versão ingênua (força bruta): O(n × m) onde n = comprimento do haystack, m = comprimento do needle
function indexOfNaive(haystack: string, needle: string): number {
  if (needle.length === 0) return 0;
  for (let i = 0; i <= haystack.length - needle.length; i++) {
    let match = true;
    for (let j = 0; j < needle.length; j++) {
      if (haystack[i + j] !== needle[j]) { match = false; break; }
    }
    if (match) return i;
  }
  return -1;
}

// --- Primeiro caractere não repetido: O(n) ---
function firstNonRepeating(s: string): string | null {
  const freq = new Map<string, number>();
  for (const c of s) freq.set(c, (freq.get(c) ?? 0) + 1);
  for (const c of s) {
    if (freq.get(c) === 1) return c;
  }
  return null;
}

// --- Inverter palavras (sobre array de chars): O(n) ---
// "the sky is blue" → "blue is sky the"
function reverseWords(s: string): string {
  const chars = s.trim().split(/\s+/); // divide em qualquer espaço em branco
  chars.reverse();                      // inverte array de palavras
  return chars.join(" ");
}

// --- Encontrar número duplicado em array [1..n]: O(n) de tempo, O(1) de espaço ---
// Usa a propriedade matemática: soma de 1..n = n(n+1)/2
function findDuplicate(nums: number[]): number {
  const n = nums.length - 1;
  const expectedSum = (n * (n + 1)) / 2;
  const actualSum = nums.reduce((a, b) => a + b, 0);
  return actualSum - expectedSum;
}
// Nota: funciona apenas quando existe exatamente um duplicado.
// Para abordagem com detecção de ciclo, veja linked-lists.md

// --- Array de soma prefixada: O(n) para construir, O(1) para consulta de intervalo ---
function buildPrefixSum(arr: number[]): number[] {
  const prefix = new Array(arr.length + 1).fill(0);
  for (let i = 0; i < arr.length; i++) {
    prefix[i + 1] = prefix[i] + arr[i];
  }
  return prefix;
}

// Soma do intervalo [left, right] inclusive: O(1)
function rangeSum(prefix: number[], left: number, right: number): number {
  return prefix[right + 1] - prefix[left];
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Erros de off-by-one.** O bug mais comum em código com arrays. Ao iterar com índices, sempre verifique as condições de fronteira. Um loop `for (let i = 0; i <= arr.length; i++)` lê um elemento além do final.

> ⚠️ **Mutar um array enquanto itera sobre ele.** Remover elementos com `splice` dentro de um loop `for` desloca os elementos subsequentes e causa itens pulados ou processados duas vezes.

```typescript
// Errado: splice desloca elementos, causando pulos
const arr = [1, 2, 3, 4, 5];
for (let i = 0; i < arr.length; i++) {
  if (arr[i] % 2 === 0) arr.splice(i, 1); // desloca tudo para esquerda — pula próximo elemento
}

// Correto: itere de trás para frente ao usar splice
for (let i = arr.length - 1; i >= 0; i--) {
  if (arr[i] % 2 === 0) arr.splice(i, 1);
}

// Melhor: use filter (cria novo array)
const evensRemoved = arr.filter(x => x % 2 !== 0);
```

> ⚠️ **Concatenação de strings em loop.** Sempre use `Array.join()` ou `Array.push()` + `join()` para construir strings iterativamente.

> ⚠️ **Esquecer que `typeof arr[i]` pode ser `undefined`.** Acessar um índice fora dos limites retorna `undefined` em JavaScript, não um erro. Isso causa bugs silenciosos.

> ⚠️ **Usar `==` em vez de `===` para comparação de strings.** O `==` do JavaScript faz coerção de tipos. `"5" == 5` é verdadeiro. Sempre use `===`.

> ⚠️ **`Array.slice()` vs `Array.splice()`.** `slice` não muta; `splice` muta no lugar. Confundi-los é um bug clássico.

---

## 6. Quando Usar / Não Usar

**Use arrays quando:**
- Precisa de acesso indexado em O(1)
- Itera sequencialmente (amigável ao cache)
- O tamanho é conhecido ou limitado
- Precisa de memória contígua para performance

**Considere alternativas quando:**
- Inserções/remoções frequentes na frente ou no meio (use lista encadeada, deque)
- Teste de pertencimento em coleções grandes (use Set — O(1) vs O(n))
- Lookup chave-valor (use Map/Object — O(1) vs O(n))
- Os dados estão ordenados e você precisa de lookup eficiente (use BST ou array ordenado + busca binária)

---

## 7. Cenário do Mundo Real

Um sistema de logging constrói mensagens concatenando campos:

```typescript
// Ruim: O(n²) — usado em produção, causou picos de latência em escala
function formatLogBad(fields: string[]): string {
  let log = "";
  for (const field of fields) {
    log += field + "|"; // nova alocação de string a cada iteração
  }
  return log;
}

// Bom: O(n)
function formatLogGood(fields: string[]): string {
  return fields.join("|");
}
```

Com 1.000 entradas de log por segundo e 20 campos por entrada, a versão ruim faz ~200 alocações de string por entrada = 200.000 alocações/segundo, cada uma copiando progressivamente mais dados. O garbage collector fica sobrecarregado. A versão boa: 1 alocação por entrada = 1.000/segundo. Problema resolvido.

Outro cenário comum: encontrar duplicatas em dados enviados por usuários.

```typescript
// Ruim: O(n²) — usado quando os dados são "pequenos o suficiente" até que não são mais
function findDuplicateEmails(emails: string[]): string[] {
  const duplicates: string[] = [];
  for (let i = 0; i < emails.length; i++) {
    for (let j = i + 1; j < emails.length; j++) {
      if (emails[i] === emails[j] && !duplicates.includes(emails[i])) {
        duplicates.push(emails[i]);
      }
    }
  }
  return duplicates;
}

// Bom: O(n)
function findDuplicateEmailsFast(emails: string[]): string[] {
  const seen = new Set<string>();
  const duplicates = new Set<string>();
  for (const email of emails) {
    if (seen.has(email)) duplicates.add(email);
    else seen.add(email);
  }
  return [...duplicates];
}
```

---

## 8. Perguntas de Entrevista

**Q1: Por que a concatenação de strings em loop é O(n²)?**
R: Strings são imutáveis em JavaScript. Cada `+=` aloca uma nova string e copia todos os caracteres anteriores mais os novos. Após k concatenações, total de caracteres copiados = 1 + 2 + ... + k = k(k+1)/2 = O(k²). Solução: use `Array.push()` depois `.join("")` — uma alocação no final.

**Q2: Qual é a complexidade de tempo do `Array.push()` em JavaScript?**
R: O(1) amortizado. A maioria dos pushes é O(1). Ocasionalmente o array de suporte dobra — O(n) — mas distribuído ao longo de n pushes isso dá em média O(1) por push.

**Q3: Implemente `indexOf` do zero.**
R: Veja a função `indexOfNaive` acima. A versão ingênua é O(n × m). Mencione o algoritmo KMP para O(n + m) se pedirem a solução ótima.

**Q4: Encontre o primeiro caractere não repetido em uma string.**
R: Duas passagens: a primeira constrói um mapa de frequência (O(n)), a segunda encontra o primeiro caractere com contagem 1 (O(n)). Total: O(n) de tempo, O(1) de espaço (o alfabeto tem tamanho fixo).

**Q5: Qual é a diferença entre `Array.slice()` e `Array.splice()`?**
R: `slice(start, end)` retorna um novo array sem modificar o original — O(k) onde k = comprimento do slice. `splice(start, deleteCount, ...items)` muta o array original no lugar — O(n) porque desloca elementos.

**Q6: Como você encontraria a maior substring sem caracteres repetidos?**
R: Janela deslizante com um Map rastreando o último índice visto. Expande o ponteiro direito a cada passo; quando uma repetição é encontrada dentro da janela, avança o ponteiro esquerdo além da ocorrência anterior. O(n) de tempo, O(min(n, alfabeto)) de espaço.

**Q7: O que é um array de soma prefixada e quando você o usa?**
R: Um array de soma prefixada armazena somas cumulativas: `prefix[i] = soma de arr[0..i-1]`. Uma vez construído em O(n), qualquer soma de intervalo `arr[left..right]` = `prefix[right+1] - prefix[left]` em O(1). Usado para consultas repetidas de soma de intervalo.

**Q8: O que significa "amigável ao cache" para arrays?**
R: CPUs carregam memória em linhas de cache (~64 bytes). Elementos de array ficam contíguos na memória, então carregar um elemento carrega os vizinhos no cache. Acesso sequencial ao array fica muito rápido porque a maioria dos acessos acerta o cache. Listas encadeadas, com nós espalhados na memória, têm localidade de cache ruim.

---

## 9. Exercícios

**Exercício 1:** Inverta as palavras em uma frase. "the sky is blue" → "blue is sky the". Faça em O(n) de tempo.
*Dica: inverta a string inteira, depois inverta cada palavra individualmente.*

**Exercício 2:** Encontre o número duplicado em um array de inteiros `[1..n]` onde um número aparece duas vezes. Alcance O(n) de tempo e O(1) de espaço.
*Dica: use a fórmula da soma OU use a detecção de ciclo de Floyd (veja linked-lists.md).*

**Exercício 3:** Encontre o comprimento da maior substring sem caracteres repetidos.
Entrada: `"abcabcbb"` → Saída: `3` ("abc")
*Dica: janela deslizante com um Map ou Set para rastrear caracteres na janela atual.*

**Exercício 4:** Dado um array e uma soma alvo k, encontre todos os pares de índices `[i, j]` onde `arr[i] + arr[j] === k`. Retorne em O(n) de tempo.
*Dica: abordagem do complemento — para cada elemento x, verifique se `k - x` está em um Set.*

**Exercício 5:** Implemente uma função que verifica se uma string é anagrama de outra em O(n) de tempo.
*Dica: mapa de frequência de caracteres — incremente para uma string, decremente para a outra, verifique se todos são zero.*

---

## 10. Leituras Complementares

- [Two Pointers Pattern — NeetCode](https://neetcode.io/roadmap)
- [Sliding Window — LeetCode Explore](https://leetcode.com/explore/)
- CLRS Capítulo 2 — Insertion Sort e Loop Invariants (raciocínio fundamental sobre arrays)
- [MDN — Métodos de String](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String)
- [Algoritmo KMP explicado](https://cp-algorithms.com/string/prefix-function.html) — busca em string O(n)
