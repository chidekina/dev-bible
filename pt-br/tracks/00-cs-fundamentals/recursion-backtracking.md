# Recursão e Backtracking

## 1. O Que É e Por Que Importa

Recursão é uma técnica onde uma função chama a si mesma para resolver uma instância menor do mesmo problema. É a forma natural de expressar algoritmos em estruturas de dados recursivas (árvores, grafos) e implementar estratégias de divisão e conquista (merge sort, quicksort).

Backtracking é uma forma refinada de recursão para **problemas de satisfação de restrições e enumeração**: explore todos os candidatos possíveis e, quando um candidato parcial não puder levar a uma solução válida, "desfaça" a última decisão e tente a próxima opção. É a base dos algoritmos para combinatória (permutações, subsets), resolução de puzzles (N-rainhas, Sudoku) e busca de caminhos.

---

## 2. Conceitos Fundamentais

### Anatomia da Recursão

Toda função recursiva tem duas partes:
1. **Caso base:** a condição sob a qual a função para de chamar a si mesma (terminação)
2. **Caso recursivo:** a chamada que reduz o problema em direção ao caso base

Uma função recursiva sem caso base roda para sempre (stack overflow). Um caso recursivo que não reduz o problema também roda para sempre.

### Comportamento da Pilha de Chamadas

Cada chamada recursiva cria um novo **frame na pilha** contendo as variáveis locais da função e o endereço de retorno. Esses frames se acumulam na call stack. Quando um caso base é atingido, os frames começam a ser removidos. A profundidade máxima de recursão determina quantos frames estão na pilha simultaneamente — a complexidade de espaço.

### Recursão em Cauda

Uma chamada recursiva é **em cauda** (tail recursive) se é a última operação na função — o chamador não precisa fazer nada depois que a chamada recursiva retornar. Chamadas em cauda podem teoricamente ser otimizadas pelo runtime (tail call optimization / TCO): o frame atual é reutilizado em vez de criar um novo. JavaScript suporta TCO em modo estrito conforme a especificação ES2015, mas na prática a maioria dos engines JS (incluindo o V8) não implementa isso. Não conte com TCO em TypeScript/JavaScript.

### Stack Overflow

A call stack do JavaScript tem um limite (~10.000–15.000 frames no V8). Recursão profunda com entradas grandes vai causar overflow. Soluções:
- Converter recursão em iteração com uma pilha explícita
- Usar memoização para reduzir chamadas
- Usar trampolining (avançado: converter chamadas recursivas em continuações)

### Memoização

Armazene em cache os resultados de chamadas recursivas custosas. Quando os mesmos argumentos aparecerem novamente, retorne o resultado em cache em vez de recomputar. Isso transforma recursão de tempo exponencial (como Fibonacci ingênuo) em tempo linear. Isso é **programação dinâmica top-down**.

### Template de Backtracking

```
function backtrack(estado, candidatos):
  if estado é uma solução completa:
    registra/retorna estado
    return

  for each escolha in candidatos:
    if escolha é válida:
      faz a escolha (modifica estado)
      backtrack(estado atualizado, candidatos restantes)
      desfaz a escolha (restaura estado)  // o passo de "backtrack"
```

O passo de "desfazer" é o que diferencia o backtracking de um DFS simples — você restaura o estado para que iterações futuras comecem do zero.

---

## 3. Como Funciona

```
Traço de recursão para fatorial(4):
fatorial(4)
  → 4 × fatorial(3)
        → 3 × fatorial(2)
              → 2 × fatorial(1)
                    → 1 × fatorial(0) = 1  ← caso base
                  = 1 × 1 = 1
            = 2 × 1 = 2
        = 3 × 2 = 6
  = 4 × 6 = 24

Profundidade da call stack = 5 frames (0 a 4)

Traço de backtracking para permutações([1,2,3]):
Início: atual=[], restante=[1,2,3]
  Escolhe 1: atual=[1], restante=[2,3]
    Escolhe 2: atual=[1,2], restante=[3]
      Escolhe 3: atual=[1,2,3] → REGISTRA [1,2,3]
      Desfaz 3: atual=[1,2]
    Desfaz 2: atual=[1]
    Escolhe 3: atual=[1,3], restante=[2]
      Escolhe 2: atual=[1,3,2] → REGISTRA [1,3,2]
      ...
  Desfaz 1: atual=[]
  Escolhe 2: ...
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Recursão Básica ---

// Fatorial: O(n) de tempo, O(n) de espaço (call stack)
function factorial(n: number): number {
  if (n === 0) return 1; // caso base
  return n * factorial(n - 1);
}

// Fibonacci: ingênuo O(2ⁿ) — árvore de chamadas tem 2^n nós
function fibNaive(n: number): number {
  if (n <= 1) return n;
  return fibNaive(n - 1) + fibNaive(n - 2);
}

// Fibonacci: memoizado — O(n) de tempo, O(n) de espaço
function fibMemo(n: number, memo = new Map<number, number>()): number {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n)!;
  const result = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  memo.set(n, result);
  return result;
}

// Fibonacci: iterativo — O(n) de tempo, O(1) de espaço (sem recursão)
function fibIterative(n: number): number {
  if (n <= 1) return n;
  let a = 0, b = 1;
  for (let i = 2; i <= n; i++) {
    [a, b] = [b, a + b];
  }
  return b;
}

// Potenciação rápida: O(log n) usando exponenciação rápida
function power(base: number, exp: number): number {
  if (exp === 0) return 1;
  if (exp % 2 === 0) {
    const half = power(base, exp / 2);
    return half * half; // evita recomputar
  }
  return base * power(base, exp - 1);
}

// Achatar um objeto profundamente aninhado de forma recursiva
function flattenObject(obj: Record<string, unknown>, prefix = ""): Record<string, unknown> {
  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(obj)) {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === "object" && value !== null && !Array.isArray(value)) {
      Object.assign(result, flattenObject(value as Record<string, unknown>, fullKey));
    } else {
      result[fullKey] = value;
    }
  }
  return result;
}
// { a: { b: { c: 1 } } } → { "a.b.c": 1 }

// --- Converter recursão em iteração com pilha explícita ---
// Percurso inorder de árvore: recursivo vs iterativo
class TreeNode {
  constructor(public val: number, public left: TreeNode | null = null, public right: TreeNode | null = null) {}
}

// Recursivo: risco de stack overflow em árvores degeneradas
function inorderRecursive(root: TreeNode | null): number[] {
  if (!root) return [];
  return [...inorderRecursive(root.left), root.val, ...inorderRecursive(root.right)];
}

// Iterativo: seguro para qualquer profundidade
function inorderIterative(root: TreeNode | null): number[] {
  const result: number[] = [];
  const stack: TreeNode[] = [];
  let current = root;
  while (current !== null || stack.length > 0) {
    while (current !== null) { stack.push(current); current = current.left; }
    current = stack.pop()!;
    result.push(current.val);
    current = current.right;
  }
  return result;
}

// --- Template de Backtracking ---

// Gerar todas as permutações de um array: O(n! × n) de tempo
function permutations<T>(arr: T[]): T[][] {
  const result: T[][] = [];

  function backtrack(current: T[], remaining: T[]): void {
    if (remaining.length === 0) {
      result.push([...current]); // snapshot — current é mutado
      return;
    }
    for (let i = 0; i < remaining.length; i++) {
      current.push(remaining[i]);                           // escolha
      backtrack(current, [...remaining.slice(0, i), ...remaining.slice(i + 1)]);
      current.pop();                                        // desfaz
    }
  }

  backtrack([], arr);
  return result;
}

// Gerar todos os subsets (conjunto potência): O(2ⁿ × n) de tempo
function subsets<T>(arr: T[]): T[][] {
  const result: T[][] = [];

  function backtrack(start: number, current: T[]): void {
    result.push([...current]); // todo estado é um subset válido
    for (let i = start; i < arr.length; i++) {
      current.push(arr[i]);       // inclui arr[i]
      backtrack(i + 1, current);  // recursa nos elementos restantes
      current.pop();              // desfaz
    }
  }

  backtrack(0, []);
  return result;
}

// Gerar todas as combinações válidas de parênteses: O(4ⁿ / √n) — número de Catalan
function generateParentheses(n: number): string[] {
  const result: string[] = [];

  function backtrack(current: string, open: number, close: number): void {
    if (current.length === 2 * n) {
      result.push(current);
      return;
    }
    if (open < n) backtrack(current + "(", open + 1, close);
    if (close < open) backtrack(current + ")", open, close + 1);
  }

  backtrack("", 0, 0);
  return result;
}

// N-Rainhas: posicionar N rainhas em tabuleiro N×N sem que nenhuma ataque outra
function solveNQueens(n: number): string[][] {
  const result: string[][] = [];
  const board = Array.from({ length: n }, () => new Array(n).fill("."));

  // Rastrear colunas e diagonais atacadas
  const cols = new Set<number>();
  const diag1 = new Set<number>(); // linha - coluna (diagonal principal)
  const diag2 = new Set<number>(); // linha + coluna (diagonal secundária)

  function backtrack(row: number): void {
    if (row === n) {
      result.push(board.map(r => r.join("")));
      return;
    }
    for (let col = 0; col < n; col++) {
      if (cols.has(col) || diag1.has(row - col) || diag2.has(row + col)) continue;

      // Coloca a rainha
      board[row][col] = "Q";
      cols.add(col); diag1.add(row - col); diag2.add(row + col);

      backtrack(row + 1);

      // Remove a rainha (backtrack)
      board[row][col] = ".";
      cols.delete(col); diag1.delete(row - col); diag2.delete(row + col);
    }
  }

  backtrack(0);
  return result;
}

// Solucionador de Sudoku
function solveSudoku(board: string[][]): boolean {
  const [row, col] = findEmpty(board);
  if (row === -1) return true; // nenhuma célula vazia — resolvido!

  for (let num = 1; num <= 9; num++) {
    const c = num.toString();
    if (isValid(board, row, col, c)) {
      board[row][col] = c;         // coloca número
      if (solveSudoku(board)) return true;
      board[row][col] = ".";       // backtrack
    }
  }
  return false; // nenhum número válido para esta célula
}

function findEmpty(board: string[][]): [number, number] {
  for (let r = 0; r < 9; r++) {
    for (let c = 0; c < 9; c++) {
      if (board[r][c] === ".") return [r, c];
    }
  }
  return [-1, -1];
}

function isValid(board: string[][], row: number, col: number, num: string): boolean {
  // Verifica linha
  if (board[row].includes(num)) return false;
  // Verifica coluna
  for (let r = 0; r < 9; r++) { if (board[r][col] === num) return false; }
  // Verifica caixa 3×3
  const boxRow = Math.floor(row / 3) * 3;
  const boxCol = Math.floor(col / 3) * 3;
  for (let r = boxRow; r < boxRow + 3; r++) {
    for (let c = boxCol; c < boxCol + 3; c++) {
      if (board[r][c] === num) return false;
    }
  }
  return true;
}

// Busca de Palavra em Grade: a palavra existe como caminho de células adjacentes?
function wordSearch(board: string[][], word: string): boolean {
  const rows = board.length, cols = board[0].length;

  function dfs(r: number, c: number, index: number): boolean {
    if (index === word.length) return true; // todos os caracteres encontrados
    if (r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] !== word[index]) return false;

    const temp = board[r][c];
    board[r][c] = "#"; // marca como visitado

    const found = dfs(r+1,c,index+1) || dfs(r-1,c,index+1) ||
                  dfs(r,c+1,index+1) || dfs(r,c-1,index+1);

    board[r][c] = temp; // restaura (backtrack)
    return found;
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (dfs(r, c, 0)) return true;
    }
  }
  return false;
}

// Soma de Combinação: todas as combinações que somam ao alvo (pode reutilizar elementos)
function combinationSum(candidates: number[], target: number): number[][] {
  const result: number[][] = [];

  function backtrack(start: number, current: number[], remaining: number): void {
    if (remaining === 0) { result.push([...current]); return; }
    if (remaining < 0) return;

    for (let i = start; i < candidates.length; i++) {
      current.push(candidates[i]);
      backtrack(i, current, remaining - candidates[i]); // i, não i+1, permite reutilização
      current.pop();
    }
  }

  candidates.sort((a, b) => a - b); // ordena para poda antecipada
  backtrack(0, [], target);
  return result;
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Caso base ausente ou errado.** O bug de recursão mais comum. Um caso base que nunca é atingido causa recursão infinita e stack overflow. Um caso base que dispara cedo demais retorna resultados errados.

> ⚠️ **Não "desfazer" a escolha no backtracking.** Se você modifica o estado (push num array, marca como visitado, etc.) antes de recursar, você deve reverter essa modificação após retornar. Esquecer isso corrompe o estado para os galhos irmãos.

> ⚠️ **Fazer push de `current` diretamente em vez de uma cópia.** `result.push(current)` adiciona uma referência ao mesmo array. Quando `current` for modificado depois (elementos removidos), todas as entradas anteriormente registradas em `result` mudam. Sempre faça push de uma cópia: `result.push([...current])`.

> ⚠️ **Stack overflow em entradas grandes.** DFS recursivo em um grafo com 100.000 nós, ou permutações de um array grande, vai estourar a call stack. Converta para iterativo com uma pilha explícita, ou aumente o tamanho da pilha do Node.js com `--stack-size`.

> ⚠️ **Tempo exponencial sem memoização.** Se os mesmos subproblemas aparecem várias vezes (subproblemas sobrepostos), recursão pura é exponencial. Adicione memoização ou mude para DP bottom-up.

---

## 6. Quando Usar / Não Usar

**Use recursão quando:**
- O problema é definido recursivamente (árvores, grafos, divisão e conquista)
- O código fica significativamente mais limpo de forma recursiva do que iterativa
- A profundidade é limitada e não é grande (< 1.000 níveis)

**Use backtracking quando:**
- Enumerando todas as soluções que satisfazem restrições
- Encontrando qualquer solução que satisfaça restrições
- Problemas envolvendo combinações, permutações ou arranjos com verificações de validade

**Converta para iteração quando:**
- A profundidade de recursão pode ser grande (grafos, grandes conjuntos de dados)
- É recursão em cauda e você precisa de desempenho
- Você teve stack overflow

**Adicione memoização quando:**
- Subproblemas se repetem (os mesmos argumentos aparecem várias vezes na árvore de chamadas)
- Se a maioria dos subproblemas se repete, considere converter para DP bottom-up

---

## 7. Cenário do Mundo Real

Uma travessia de sistema de arquivos processa recursivamente todos os arquivos em diretórios aninhados:

```typescript
import * as fs from "fs";
import * as path from "path";

function findFilesRecursive(dir: string, extension: string): string[] {
  const results: string[] = [];

  function traverse(currentDir: string): void {
    const entries = fs.readdirSync(currentDir, { withFileTypes: true });
    for (const entry of entries) {
      const fullPath = path.join(currentDir, entry.name);
      if (entry.isDirectory()) {
        traverse(fullPath); // recursa no subdiretório
      } else if (entry.name.endsWith(extension)) {
        results.push(fullPath);
      }
    }
  }

  traverse(dir);
  return results;
}
```

Um parser de configuração lida com JSON arbitrariamente aninhado:

```typescript
function validateConfig(config: unknown, schema: Record<string, unknown>): boolean {
  if (typeof config !== "object" || config === null) return false;
  const cfg = config as Record<string, unknown>;

  for (const [key, expectedType] of Object.entries(schema)) {
    if (!(key in cfg)) return false;
    if (typeof expectedType === "object") {
      // Recursa no schema aninhado
      if (!validateConfig(cfg[key], expectedType as Record<string, unknown>)) return false;
    } else {
      if (typeof cfg[key] !== expectedType) return false;
    }
  }
  return true;
}
```

---

## 8. Perguntas de Entrevista

**P1: Qual é a diferença entre recursão e iteração?**
R: Recursão usa a call stack para rastrear estado entre chamadas repetidas da função. Iteração usa variáveis de loop explícitas. Recursão é mais expressiva para estruturas recursivas (árvores, grafos), mas usa O(profundidade) de espaço na pilha. Iteração é mais eficiente em memória. Qualquer algoritmo recursivo pode ser convertido para iterativo usando uma pilha explícita.

**P2: Como você achataria um objeto profundamente aninhado de forma recursiva?**
R: Percorra as entradas do objeto. Se um valor for um objeto não-nulo (e não um array), recurse nele com o caminho atual como prefixo. Caso contrário, adicione o par chave-valor ao resultado. Veja a função `flattenObject` acima.

**P3: Qual é a complexidade de tempo de gerar todas as permutações?**
R: O(n! × n). Existem n! permutações, e cada uma leva O(n) para copiar para o resultado. A árvore de recursão tem n! folhas e O(n × n!) nós no total.

**P4: Qual é a diferença fundamental entre backtracking e DFS simples?**
R: Ambos exploram uma árvore de decisões. DFS geralmente marca nós como visitados permanentemente. Backtracking desfaz mudanças após explorar um galho, restaurando o estado para galhos irmãos. Isso é essencial quando o mesmo nó pode aparecer em múltiplas soluções ou quando você precisa explorar todas as combinações válidas.

**P5: Quando recursão leva a tempo exponencial? Como corrigir?**
R: Quando subproblemas se sobrepõem — as mesmas entradas são computadas várias vezes (ex: Fibonacci ingênuo: fib(n-1) e fib(n-2) ambos computam fib(n-2)). Correção: memoize (DP top-down) para armazenar em cache resultados, ou reescreva como DP bottom-up.

---

## 9. Exercícios

**Exercício 1:** Gere todas as combinações válidas de n pares de parênteses.
Entrada: n = 3 → Saída: ["((()))", "(()())", "(())()", "()(())", "()()()"]
*Dica: rastreie a contagem de parênteses abertos e fechados. Só adicione "(" se open < n, só adicione ")" se close < open.*

**Exercício 2:** Dado um número de telefone como string, retorne todas as combinações de letras possíveis (como teclados T9 antigos).
Entrada: "23" → Saída: ["ad", "ae", "af", "bd", "be", "bf", "cd", "ce", "cf"]
*Dica: backtracking sobre o mapa de dígito para letras.*

**Exercício 3:** Encontre todas as combinações de números em um array que somam a um alvo (elementos podem ser reutilizados).
*Dica: candidatos ordenados + backtrack com índice de início. Permita que o mesmo índice seja usado novamente.*

**Exercício 4:** Resolva o problema das N-Rainhas — posicione N rainhas em um tabuleiro N×N de xadrez de modo que nenhuma rainha ataque outra. Retorne todas as soluções.

**Exercício 5:** Implemente um clone profundo recursivo de um objeto JavaScript (lidando com objetos aninhados, arrays e valores primitivos). Trate referências circulares.

---

## 10. Leituras Complementares

- CLRS Capítulo 4 — Divisão e Conquista
- [Backtracking — padrões e templates](https://labuladong.online/algo/essential-technique/backtrack-framework/)
- [The Recursive Book of Recursion](https://inventwithpython.com/recursion/) — por Al Sweigart (gratuito online)
- LeetCode: #46 Permutations, #78 Subsets, #22 Generate Parentheses, #51 N-Queens, #37 Sudoku Solver, #79 Word Search
