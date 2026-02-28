# Programação Dinâmica

## 1. O Que É e Por Que Importa

Programação dinâmica (DP) é uma técnica de otimização para problemas com duas propriedades:
1. **Subestrutura ótima:** a solução ótima pode ser construída a partir de soluções ótimas para subproblemas
2. **Subproblemas sobrepostos:** os mesmos subproblemas são resolvidos várias vezes em uma abordagem recursiva ingênua

DP elimina trabalho redundante resolvendo cada subproblema uma única vez e armazenando o resultado em cache. É a diferença entre tempo exponencial e polinomial em uma enorme classe de problemas: caminhos mais curtos, alinhamento de sequências, alocação de recursos e escalonamento.

O "programação" em programação dinâmica se refere a tabulação (preencher uma tabela), não a escrever código. O termo foi cunhado por Richard Bellman na década de 1950.

---

## 2. Conceitos Fundamentais

### Top-Down (Memoização)

Escreva a solução recursiva e adicione um cache. Quando um subproblema é resolvido pela primeira vez, armazene o resultado. Quando aparecer novamente, retorne do cache. Natural de escrever — apenas augmente uma solução recursiva existente.

**Vantagens:** Computa apenas os subproblemas que são realmente necessários. Mais fácil de implementar quando o espaço de estados é grande ou esparsamente populado.
**Desvantagens:** Sobrecarga de chamadas de função. A profundidade recursiva pode causar stack overflow.

### Bottom-Up (Tabulação)

Preencha uma tabela começando pelos subproblemas menores, construindo até a resposta final. Iterativo — sem sobrecarga de recursão. Frequentemente permite otimização de espaço (manter apenas a última linha/camada).

**Vantagens:** Sem sobrecarga de recursão. Frequentemente mais eficiente em espaço. Ordem de execução clara.
**Desvantagens:** Deve resolver todos os subproblemas mesmo que alguns sejam desnecessários. Menos intuitivo para transições de estado complexas.

### Como Abordar um Problema de DP

1. **Identifique o estado:** que informação define um subproblema? (ex: índice i, capacidade restante, número de itens usados)
2. **Defina a recorrência:** expresse dp[estado] em termos de estados menores
3. **Estabeleça casos base:** qual é o valor quando não há mais trabalho a fazer?
4. **Determine a ordem de computação:** bottom-up requer conhecer a direção de dependência
5. **Identifique a otimização de espaço:** você pode reduzir de O(n²) para O(n)?

---

## 3. Como Funciona

```
Fibonacci com DP:
Estado: dp[i] = i-ésimo número de Fibonacci
Recorrência: dp[i] = dp[i-1] + dp[i-2]
Casos base: dp[0] = 0, dp[1] = 1

Preenchendo a tabela bottom-up:
dp: [0, 1, 1, 2, 3, 5, 8, 13, ...]
    i=0 i=1 i=2 i=3 ...

Cada valor computado uma vez: O(n) de tempo, O(n) de espaço.
Otimização de espaço: só precisamos dos dois últimos valores → O(1) de espaço.

Troco de Moedas:
moedas=[1,5,10], valor=11
Estado: dp[i] = mínimo de moedas para fazer o valor i
dp[0] = 0 (caso base)
dp[1] = min(dp[0]+1) = 1 (usa moeda 1)
dp[5] = min(dp[4]+1, dp[0]+1) = 1 (usa moeda 5)
dp[10] = min(dp[9]+1, dp[5]+1, dp[0]+1) = 1 (usa moeda 10)
dp[11] = min(dp[10]+1, dp[6]+1, dp[1]+1) = 2 (moeda 10 + moeda 1)
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Problemas de DP 1D ---

// Subindo Escadas: contar maneiras de subir n degraus (1 ou 2 por vez)
// Estado: dp[i] = número de maneiras de chegar ao degrau i
function climbStairs(n: number): number {
  if (n <= 2) return n;
  let prev2 = 1, prev1 = 2;
  for (let i = 3; i <= n; i++) {
    const current = prev1 + prev2;
    prev2 = prev1;
    prev1 = current;
  }
  return prev1; // O(n) de tempo, O(1) de espaço
}

// Ladrão de Casas: dinheiro máximo de casas não adjacentes
// Estado: dp[i] = dinheiro máximo das primeiras i casas
function rob(nums: number[]): number {
  if (nums.length === 0) return 0;
  if (nums.length === 1) return nums[0];
  let prev2 = nums[0];
  let prev1 = Math.max(nums[0], nums[1]);
  for (let i = 2; i < nums.length; i++) {
    const current = Math.max(prev1, prev2 + nums[i]);
    prev2 = prev1;
    prev1 = current;
  }
  return prev1;
}

// Troco de Moedas: mínimo de moedas para atingir o valor
// Estado: dp[i] = mínimo de moedas para o valor i
function coinChange(coins: number[], amount: number): number {
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;

  for (let i = 1; i <= amount; i++) {
    for (const coin of coins) {
      if (coin <= i && dp[i - coin] !== Infinity) {
        dp[i] = Math.min(dp[i], dp[i - coin] + 1);
      }
    }
  }
  return dp[amount] === Infinity ? -1 : dp[amount];
}

// Quebra de Palavra: string s pode ser segmentada em palavras do dicionário?
// Estado: dp[i] = s[0..i-1] pode ser segmentada?
function wordBreak(s: string, wordDict: string[]): boolean {
  const wordSet = new Set(wordDict);
  const dp = new Array(s.length + 1).fill(false);
  dp[0] = true; // string vazia é sempre válida

  for (let i = 1; i <= s.length; i++) {
    for (let j = 0; j < i; j++) {
      if (dp[j] && wordSet.has(s.slice(j, i))) {
        dp[i] = true;
        break;
      }
    }
  }
  return dp[s.length];
}

// Produto Máximo de Subarray: O(n) de tempo, O(1) de espaço
// Estado: rastreia tanto máximo quanto mínimo (mínimo pode virar máximo ao multiplicar por negativo)
function maxProduct(nums: number[]): number {
  let maxSoFar = nums[0], minSoFar = nums[0], result = nums[0];

  for (let i = 1; i < nums.length; i++) {
    const candidates = [nums[i], maxSoFar * nums[i], minSoFar * nums[i]];
    maxSoFar = Math.max(...candidates);
    minSoFar = Math.min(...candidates);
    result = Math.max(result, maxSoFar);
  }
  return result;
}

// Maior Subsequência Crescente: DP O(n²)
// Estado: dp[i] = comprimento da LIS terminando no índice i
function lengthOfLIS(nums: number[]): number {
  const dp = new Array(nums.length).fill(1);
  let maxLen = 1;

  for (let i = 1; i < nums.length; i++) {
    for (let j = 0; j < i; j++) {
      if (nums[j] < nums[i]) {
        dp[i] = Math.max(dp[i], dp[j] + 1);
      }
    }
    maxLen = Math.max(maxLen, dp[i]);
  }
  return maxLen;
}
// Nota: O(n log n) é possível usando patience sorting / busca binária

// Número de Decodificações: quantidade de formas de decodificar uma string de dígitos em letras
// 'A'=1, 'B'=2, ..., 'Z'=26
function numDecodings(s: string): number {
  const n = s.length;
  const dp = new Array(n + 1).fill(0);
  dp[0] = 1; // string vazia: uma forma
  dp[1] = s[0] === "0" ? 0 : 1;

  for (let i = 2; i <= n; i++) {
    const oneDigit = parseInt(s[i - 1]);
    const twoDigits = parseInt(s.slice(i - 2, i));

    if (oneDigit >= 1) dp[i] += dp[i - 1];
    if (twoDigits >= 10 && twoDigits <= 26) dp[i] += dp[i - 2];
  }
  return dp[n];
}

// --- Problemas de DP 2D ---

// Maior Subsequência Comum: O(m×n) de tempo e espaço
// Estado: dp[i][j] = comprimento da LCS de s1[0..i-1] e s2[0..j-1]
function longestCommonSubsequence(text1: string, text2: string): number {
  const m = text1.length, n = text2.length;
  const dp: number[][] = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (text1[i - 1] === text2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1] + 1; // chars iguais — estende LCS
      } else {
        dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]); // pula um char
      }
    }
  }
  return dp[m][n];
}

// Distância de Edição (Levenshtein): mínimo de operações para transformar word1 em word2
// Operações: inserir, deletar, substituir (cada uma custa 1)
// Estado: dp[i][j] = distância de edição entre word1[0..i-1] e word2[0..j-1]
function minDistance(word1: string, word2: string): number {
  const m = word1.length, n = word2.length;
  const dp: number[][] = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));

  // Casos base: transformar de/para string vazia
  for (let i = 0; i <= m; i++) dp[i][0] = i; // deleta todos os chars de word1
  for (let j = 0; j <= n; j++) dp[0][j] = j; // insere todos os chars de word2

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (word1[i - 1] === word2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1]; // nenhuma operação necessária
      } else {
        dp[i][j] = 1 + Math.min(
          dp[i - 1][j],     // deleta de word1
          dp[i][j - 1],     // insere em word1
          dp[i - 1][j - 1]  // substitui
        );
      }
    }
  }
  return dp[m][n];
}

// Mochila 0/1: valor máximo com capacidade de peso
// Estado: dp[i][w] = valor máximo usando os primeiros i itens com capacidade w
function knapsack(weights: number[], values: number[], capacity: number): number {
  const n = weights.length;
  // Otimização de espaço: DP 1D (array rolante)
  const dp = new Array(capacity + 1).fill(0);

  for (let i = 0; i < n; i++) {
    // Itera da direita para a esquerda para evitar usar o item i duas vezes
    for (let w = capacity; w >= weights[i]; w--) {
      dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
    }
  }
  return dp[capacity];
}

// Caminhos Únicos: robô em grade m×n, só pode mover para direita ou baixo
// Estado: dp[i][j] = número de caminhos do canto superior esquerdo até (i, j)
function uniquePaths(m: number, n: number): number {
  const dp = new Array(n).fill(1); // primeira linha: tudo 1 (só um jeito: sempre para a direita)

  for (let i = 1; i < m; i++) {
    for (let j = 1; j < n; j++) {
      dp[j] += dp[j - 1]; // de cima (dp[j]) + da esquerda (dp[j-1])
    }
  }
  return dp[n - 1];
}

// Substrings Palíndromas: conta todas as substrings palíndromas
// Estado: dp[i][j] = true se s[i..j] é um palíndromo
function countSubstrings(s: string): number {
  const n = s.length;
  let count = 0;

  // Abordagem de expansão pelo centro: O(n²) de tempo, O(1) de espaço
  function expandAroundCenter(left: number, right: number): void {
    while (left >= 0 && right < n && s[left] === s[right]) {
      count++;
      left--;
      right++;
    }
  }

  for (let i = 0; i < n; i++) {
    expandAroundCenter(i, i);     // palíndromos de comprimento ímpar
    expandAroundCenter(i, i + 1); // palíndromos de comprimento par
  }
  return count;
}

// Maior Substring Palíndroma: expansão pelo centro O(n²)
function longestPalindrome(s: string): string {
  let start = 0, maxLen = 1;

  function expand(left: number, right: number): void {
    while (left >= 0 && right < s.length && s[left] === s[right]) {
      if (right - left + 1 > maxLen) {
        maxLen = right - left + 1;
        start = left;
      }
      left--; right++;
    }
  }

  for (let i = 0; i < s.length; i++) {
    expand(i, i);
    expand(i, i + 1);
  }
  return s.slice(start, start + maxLen);
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Direção errada de recorrência na mochila 0/1.** Ao usar um array 1D rolante, itere a capacidade de ALTO para BAIXO. Iterar de baixo para cima permite que o mesmo item seja usado várias vezes (vira mochila ilimitada acidentalmente).

> ⚠️ **Off-by-one na indexação do array dp.** Muitos problemas de DP usam `dp[0]` como caso base para entrada vazia. Então `dp[i]` representa os primeiros i elementos, não o i-ésimo elemento. Misturar esses índices por um causa resultados errados.

> ⚠️ **Mutar o cache de memoização compartilhado entre casos de teste.** Se você define `memo` fora da função como uma variável de módulo, ela persiste entre chamadas. Crie um novo Map por chamada ou use um closure.

> ⚠️ **Inicializar o array dp com 0 quando a resposta deveria ser -Infinity ou Infinity.** Para problemas de "produto máximo" ou "custo mínimo", inicialize com o elemento neutro correto.

> ⚠️ **Confundir LCS com Substring Comum Mais Longa.** LCS (subsequência) pode pular caracteres. Substring Comum Mais Longa (contígua) não pode. Recorrências diferentes.

---

## 6. Quando Usar / Não Usar

**DP se aplica quando:**
- O problema pede valor ótimo (min/max/contagem)
- O problema tem subestrutura ótima (soluções de subproblemas se combinam para formar a solução completa)
- O problema tem subproblemas sobrepostos (os mesmos subproblemas aparecem repetidamente)
- O problema pergunta "de quantas formas" ou "é possível"

**DP NÃO se aplica quando:**
- Greedy funciona (ótimo local = ótimo global) — greedy é mais simples e rápido
- O problema não tem subproblemas sobrepostos — recursão simples é suficiente
- Subproblemas são independentes — divisão e conquista (não DP)

---

## 7. Cenário do Mundo Real

Corretores ortográficos usam distância de edição (Levenshtein) para sugerir correções:

```typescript
function spellSuggest(word: string, dictionary: string[], maxDistance = 2): string[] {
  return dictionary
    .map(dictWord => ({
      word: dictWord,
      dist: minDistance(word, dictWord)
    }))
    .filter(({ dist }) => dist <= maxDistance)
    .sort((a, b) => a.dist - b.dist)
    .map(({ word }) => word);
}
// "helo" → ["hello", "help", "hero"] (dentro da distância de edição 2)
```

Um otimizador de portfólio usa a mochila 0/1 para maximizar retorno dentro de um orçamento:

```typescript
interface Investment {
  name: string;
  cost: number;    // em milhares
  expectedReturn: number;
}

function optimizePortfolio(investments: Investment[], budget: number): Investment[] {
  const dp = new Array(budget + 1).fill(0);
  const chosen: boolean[][] = Array.from({ length: investments.length }, () => new Array(budget + 1).fill(false));

  for (let i = 0; i < investments.length; i++) {
    const { cost, expectedReturn } = investments[i];
    for (let w = budget; w >= cost; w--) {
      if (dp[w - cost] + expectedReturn > dp[w]) {
        dp[w] = dp[w - cost] + expectedReturn;
        chosen[i][w] = true;
      }
    }
  }

  // Retrocede para encontrar quais investimentos foram escolhidos
  const result: Investment[] = [];
  let remainingBudget = budget;
  for (let i = investments.length - 1; i >= 0; i--) {
    if (chosen[i][remainingBudget]) {
      result.push(investments[i]);
      remainingBudget -= investments[i].cost;
    }
  }
  return result;
}
```

---

## 8. Perguntas de Entrevista

**P1: Como identificar se um problema precisa de programação dinâmica?**
R: Procure por: (1) pede valor ótimo (min/max), contagem de formas ou possibilidade, (2) o problema pode ser decomposto em subproblemas do mesmo tipo, (3) a solução recursiva ingênua recomputa as mesmas entradas. Sinais clássicos: "número mínimo de...", "soma/produto máximo", "contar formas distintas", "maior/menor subsequência".

**P2: Qual é a diferença entre memoização (top-down) e tabulação (bottom-up)?**
R: Memoização: recursiva, armazena resultados em cache conforme computados. Resolve apenas subproblemas necessários. Sobrecarga de pilha. Tabulação: iterativa, preenche tabela de pequeno para grande. Resolve todos os subproblemas mas permite otimização de espaço. Geralmente mais rápida na prática.

**P3: Explique a transição de estado para o troco de moedas.**
R: Estado: `dp[i]` = mínimo de moedas para o valor i. Caso base: `dp[0] = 0`. Transição: para cada moeda c e cada valor i ≥ c, `dp[i] = min(dp[i], dp[i-c] + 1)`. A ideia: se podemos fazer o valor `i-c` de forma ótima, podemos fazer o valor `i` adicionando uma moeda de valor `c`.

**P4: Como resolver a mochila 0/1 com otimização de espaço?**
R: Use um array 1D em vez de 2D. Para cada item, itere a capacidade de alto para baixo. Isso evita usar o mesmo item duas vezes (o que aconteceria se iterasse de baixo para cima, pois dp[w-custo] já incluiria o item atual).

**P5: Qual é a diferença entre Maior Subsequência Comum e Maior Substring Comum?**
R: LCS subsequência permite lacunas — caracteres não precisam ser adjacentes. Recorrência: se chars iguais, estende por 1; senão pega o max de excluir qualquer char. LCS substring requer caracteres contíguos — se chars diferentes, reseta para 0. Recorrências diferentes produzem resultados diferentes.

---

## 9. Exercícios

**Exercício 1:** Ladrão de Casas II — mesmo que o ladrão de casas, mas as casas estão dispostas em círculo (a primeira e a última são adjacentes, então você não pode roubar as duas).
*Dica: execute o ladrão de casas duas vezes — uma em casas[0..n-2], outra em casas[1..n-1]. Pegue o máximo.*

**Exercício 2:** Maior Subsequência Palíndroma — encontre o comprimento da maior subsequência de s que é um palíndromo.
*Dica: inverta s e encontre a LCS de s e s invertida.*

**Exercício 3:** Partição com Soma Igual — você pode particionar um array em dois subsets com soma igual?
*Dica: reduza para mochila 0/1 — você consegue atingir soma/2 usando elementos do array?*

**Exercício 4:** Soma máxima de um subarray não-vazio e contíguo (algoritmo de Kadane).
*Dica: dp[i] = soma máxima de subarray terminando no índice i. dp[i] = max(nums[i], dp[i-1] + nums[i]).*

**Exercício 5:** Número de maneiras de subir n degraus se você pode saltar 1, 2 ou 3 degraus por vez.
*Dica: generalize subindo escadas. dp[i] = dp[i-1] + dp[i-2] + dp[i-3].*

---

## 10. Leituras Complementares

- CLRS Capítulo 15 — Programação Dinâmica
- [Padrões de DP — NeetCode roadmap](https://neetcode.io/roadmap)
- [Pensando em DP — labuladong](https://labuladong.online/algo/dynamic_programming/)
- LeetCode: #70 Climbing Stairs, #198 House Robber, #322 Coin Change, #1143 LCS, #72 Edit Distance, #416 Partition Equal Subset Sum, #300 LIS
