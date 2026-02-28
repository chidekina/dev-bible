# Algoritmos de Busca

## 1. O Que É e Por Que Importa

Busca — encontrar um valor alvo ou responder uma pergunta booleana sobre dados — é uma das operações mais comuns em software. A escolha do algoritmo de busca pode ser a diferença entre uma funcionalidade que parece instantânea e uma que derruba o servidor.

A busca linear funciona em qualquer estrutura, mas é O(n). A busca binária é O(log n), mas exige dados ordenados. O verdadeiro poder da busca binária vai além de pesquisar arrays: a **busca binária na resposta** aplica a busca binária a uma função monótona, transformando "encontre o menor valor x tal que condição(x) seja verdadeira" em um problema O(log(espaço_de_busca)). Esse padrão resolve uma ampla variedade de problemas que, à primeira vista, não parecem ser problemas de busca em arrays.

---

## 2. Conceitos Fundamentais

### Busca Linear

Percorre cada elemento até encontrar o alvo ou chegar ao fim. O(n) de tempo, O(1) de espaço. Funciona em dados não ordenados, não indexados e em qualquer container. É o ponto de partida.

### Busca Binária

Requer um **array ordenado**. Divide o espaço de busca pela metade a cada iteração, comparando o alvo com o elemento do meio. O(log n) de tempo, O(1) de espaço (versão iterativa).

**Invariante:** A resposta, se existir, está sempre dentro de [esquerda, direita]. Estreite essa janela eliminando metade a cada passo.

**Três templates:**
1. Correspondência exata: verificar se o alvo existe
2. Encontrar a primeira/mais à esquerda posição que satisfaz uma condição
3. Encontrar a última/mais à direita posição que satisfaz uma condição

Os templates 2 e 3 são mais versáteis — eles lidam com "encontrar o elemento mais à esquerda ≥ alvo" e consultas similares.

### Busca Binária na Resposta

Quando você tem uma **condição monótona** — um predicado que é falso para valores pequenos e verdadeiro para valores grandes (ou vice-versa) — você pode fazer busca binária no espaço de respostas.

Padrão:
```
lo = menor resposta possível
hi = maior resposta possível
while lo < hi:
  mid = (lo + hi) / 2
  if condição(mid):
    hi = mid   (ou lo = mid + 1, dependendo da direção)
  else:
    lo = mid + 1
return lo
```

### Busca Ternária

Usada para **funções unimodais** (funções que aumentam depois diminuem, ou vice-versa) para encontrar o máximo ou mínimo. Reduz o espaço de busca a 2/3 a cada passo. O(log n).

---

## 3. Como Funciona

```
Busca binária por 7 em [1, 3, 5, 7, 9, 11, 13]:

Iteração 1:
esquerda=0, direita=6, meio=3
arr[3] = 7 → Encontrado! Retorna 3.

Busca binária por 6 (não presente):
Iteração 1: esq=0, dir=6, meio=3, arr[3]=7 > 6 → dir=2
Iteração 2: esq=0, dir=2, meio=1, arr[1]=3 < 6 → esq=2
Iteração 3: esq=2, dir=2, meio=2, arr[2]=5 < 6 → esq=3
esq > dir → retorna -1

Exemplo de busca binária na resposta:
"Qual é o número mínimo de dias para entregar todos os pacotes?"
Pacotes: [1,2,3,4,5,6,7,8,9,10], limite de capacidade por dia

lo = max(pacotes) = 10 (mínimo possível: enviar um por vez)
hi = soma(pacotes) = 55 (máximo: enviar tudo em um dia)
Busca binária: dá pra fazer em 30 dias? Verifica. E em 15 dias? ...
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Busca Linear: O(n) ---
function linearSearch<T>(arr: T[], target: T): number {
  for (let i = 0; i < arr.length; i++) {
    if (arr[i] === target) return i;
  }
  return -1;
}

// --- Busca Binária (correspondência exata): O(log n) ---
function binarySearch(arr: number[], target: number): number {
  let left = 0, right = arr.length - 1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2); // evita overflow de inteiro
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}

// --- Busca Binária: Encontrar a primeira posição do alvo ---
// Retorna o índice mais à esquerda onde arr[i] === target
function findFirst(arr: number[], target: number): number {
  let left = 0, right = arr.length - 1;
  let result = -1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] === target) {
      result = mid;
      right = mid - 1; // continua buscando à esquerda
    } else if (arr[mid] < target) {
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }
  return result;
}

// --- Busca Binária: Encontrar a última posição do alvo ---
function findLast(arr: number[], target: number): number {
  let left = 0, right = arr.length - 1;
  let result = -1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] === target) {
      result = mid;
      left = mid + 1; // continua buscando à direita
    } else if (arr[mid] < target) {
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }
  return result;
}

// --- Busca Binária: Lower bound (primeira posição >= alvo) ---
// Equivalente à std::lower_bound da biblioteca padrão
function lowerBound(arr: number[], target: number): number {
  let left = 0, right = arr.length;
  while (left < right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] < target) left = mid + 1;
    else right = mid;
  }
  return left; // primeiro índice onde arr[i] >= target
}

// --- Busca Binária: Upper bound (primeira posição > alvo) ---
function upperBound(arr: number[], target: number): number {
  let left = 0, right = arr.length;
  while (left < right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] <= target) left = mid + 1;
    else right = mid;
  }
  return left; // primeiro índice onde arr[i] > target
}

// --- Busca em Array Ordenado Rotacionado: O(log n) ---
// [4, 5, 6, 7, 0, 1, 2], alvo = 0
function searchRotated(nums: number[], target: number): number {
  let left = 0, right = nums.length - 1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (nums[mid] === target) return mid;

    // Determinar qual metade está ordenada
    if (nums[left] <= nums[mid]) {
      // Metade esquerda está ordenada
      if (target >= nums[left] && target < nums[mid]) {
        right = mid - 1; // alvo está na metade esquerda
      } else {
        left = mid + 1;
      }
    } else {
      // Metade direita está ordenada
      if (target > nums[mid] && target <= nums[right]) {
        left = mid + 1; // alvo está na metade direita
      } else {
        right = mid - 1;
      }
    }
  }
  return -1;
}

// --- Encontrar Elemento de Pico: O(log n) ---
// Elemento de pico é qualquer elemento maior que seus vizinhos
function findPeakElement(nums: number[]): number {
  let left = 0, right = nums.length - 1;

  while (left < right) {
    const mid = left + Math.floor((right - left) / 2);
    if (nums[mid] > nums[mid + 1]) {
      right = mid; // pico está à esquerda (incluindo mid)
    } else {
      left = mid + 1; // pico está à direita
    }
  }
  return left;
}

// --- Encontrar Mínimo em Array Rotacionado: O(log n) ---
function findMin(nums: number[]): number {
  let left = 0, right = nums.length - 1;

  while (left < right) {
    const mid = left + Math.floor((right - left) / 2);
    if (nums[mid] > nums[right]) {
      left = mid + 1; // mínimo está na metade direita
    } else {
      right = mid; // mínimo está na metade esquerda (incluindo mid)
    }
  }
  return nums[left];
}

// --- Raiz quadrada inteira: O(log n) busca binária na resposta ---
function mySqrt(x: number): number {
  if (x < 2) return x;
  let left = 1, right = Math.floor(x / 2);
  let result = 1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (mid * mid === x) return mid;
    if (mid * mid < x) {
      result = mid;
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }
  return result;
}

// --- Busca Binária na Resposta: Capacidade para Entregar em D Dias ---
// Encontra a capacidade mínima para entregar todos os pacotes em D dias
function shipWithinDays(weights: number[], days: number): number {
  let lo = Math.max(...weights); // mínimo: precisa carregar o pacote mais pesado
  let hi = weights.reduce((a, b) => a + b, 0); // máximo: entregar tudo em um dia

  while (lo < hi) {
    const mid = lo + Math.floor((hi - lo) / 2);
    if (canShip(weights, days, mid)) {
      hi = mid; // mid funciona, tenta menor
    } else {
      lo = mid + 1; // mid não funciona, precisa de mais capacidade
    }
  }
  return lo;
}

function canShip(weights: number[], days: number, capacity: number): boolean {
  let currentLoad = 0, daysNeeded = 1;
  for (const weight of weights) {
    if (currentLoad + weight > capacity) {
      daysNeeded++;
      currentLoad = 0;
    }
    currentLoad += weight;
  }
  return daysNeeded <= days;
}

// --- Busca Binária na Resposta: Koko Comendo Bananas ---
// Encontra a velocidade mínima k para comer todas as bananas em h horas
function minEatingSpeed(piles: number[], h: number): number {
  let lo = 1, hi = Math.max(...piles);

  while (lo < hi) {
    const mid = lo + Math.floor((hi - lo) / 2);
    const hoursNeeded = piles.reduce((sum, pile) => sum + Math.ceil(pile / mid), 0);
    if (hoursNeeded <= h) hi = mid;
    else lo = mid + 1;
  }
  return lo;
}

// --- Busca em Matriz 2D: O(log(m × n)) ---
// Linhas ordenadas, primeiro elemento de cada linha > último da linha anterior
function searchMatrix(matrix: number[][], target: number): boolean {
  const m = matrix.length, n = matrix[0].length;
  let left = 0, right = m * n - 1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    const val = matrix[Math.floor(mid / n)][mid % n]; // converte índice 1D para 2D
    if (val === target) return true;
    if (val < target) left = mid + 1;
    else right = mid - 1;
  }
  return false;
}

// --- Busca Ternária: encontrar máximo de função unimodal ---
// Funciona em qualquer função f onde f aumenta depois diminui
function ternarySearchMax(f: (x: number) => number, lo: number, hi: number, eps = 1e-9): number {
  while (hi - lo > eps) {
    const m1 = lo + (hi - lo) / 3;
    const m2 = hi - (hi - lo) / 3;
    if (f(m1) < f(m2)) lo = m1;
    else hi = m2;
  }
  return (lo + hi) / 2;
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Overflow de inteiro no cálculo do `mid`.** `(left + right) / 2` pode causar overflow quando left e right são inteiros grandes. Use `left + Math.floor((right - left) / 2)`. Em TypeScript/JavaScript isso raramente é um problema por causa de números de ponto flutuante, mas é um bug crítico em Java/C++.

> ⚠️ **Loops infinitos por atualização incorreta dos limites.** Ao buscar a primeira ocorrência, certifique-se de usar `right = mid` e não `right = mid - 1` quando `arr[mid] === target`. Usar a atualização errada causa tanto a perda de respostas válidas quanto loops infinitos.

> ⚠️ **Esquecer a pré-condição de "ordenado".** A busca binária produz resultados errados silenciosamente em arrays não ordenados. Se os dados não estiverem ordenados, ordene primeiro — ordenação O(n log n) + busca O(log n) ainda é melhor que O(n) por consulta quando muitas consultas são feitas.

> ⚠️ **Busca binária na resposta: limites iniciais errados.** `lo` deve ser a menor resposta possível (limite inferior exclusivo está errado), `hi` deve ser a maior resposta possível (inclusivo). Errar isso faz o código pular a resposta correta.

> ⚠️ **Confundir `while (lo <= right)` com `while (lo < right)`.** Para correspondência exata: use `<=`. Para encontrar a primeira posição que satisfaz uma condição: use `<` e retorne `lo` após o loop. Misturar esses dois é uma fonte comum de bugs de off-by-one.

---

## 6. Quando Usar / Não Usar

**Busca linear quando:**
- Array não está ordenado e não vale a pena ordenar
- n é pequeno
- Dados estão em uma lista encadeada (sem acesso aleatório)
- Você precisa encontrar todas as ocorrências, não só uma

**Busca binária quando:**
- Dados estão ordenados (ou você pode ordenar uma vez para muitas consultas)
- Você precisa de O(log n) por consulta
- O problema tem uma condição monótona (busca binária na resposta)

**Busca binária na resposta quando:**
- Você precisa encontrar o valor mínimo/máximo que satisfaz uma condição
- A condição é monótona (se vale para x, vale para todo x' > x)
- Comum em problemas de otimização: "encontrar capacidade mínima", "encontrar dias mínimos", "encontrar velocidade mínima"

---

## 7. Cenário do Mundo Real

Um pipeline de deploy precisa encontrar qual commit introduziu uma regressão. O `git bisect` usa busca binária:

```typescript
// Modelo do "git bisect" — encontra o primeiro commit que introduziu um bug
function gitBisect(
  commits: string[],
  isBuggy: (commitHash: string) => boolean
): string | null {
  // commits[0] é o mais antigo, commits[n-1] é o mais recente
  // Premissa: todos os commits antes do ruim passam, todos depois falham (monótono)
  let left = 0, right = commits.length - 1;
  let firstBad = -1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (isBuggy(commits[mid])) {
      firstBad = mid;
      right = mid - 1; // busca commit ruim mais antigo
    } else {
      left = mid + 1;
    }
  }
  return firstBad === -1 ? null : commits[firstBad];
}
// Transforma O(n) testar-cada-commit em O(log n) — crítico em repositórios com milhares de commits
```

Um otimizador de queries de banco de dados usa busca binária para verificar páginas de índice:

```typescript
// Busca simplificada em página folha de B-tree
function findInBTreePage(page: number[], key: number): number {
  // Cada página é um array ordenado de chaves
  return lowerBound(page, key); // Encontra posição para inserir/consultar a chave
}
```

---

## 8. Perguntas de Entrevista

**P1: Por que a busca binária é O(log n)?**
R: Cada iteração divide o espaço de busca pela metade. Começando com n elementos: após 1 passo temos n/2, após 2 passos n/4, ..., após k passos n/2^k. Paramos quando n/2^k = 1, portanto k = log₂(n).

**P2: Como fazer busca binária em um array ordenado rotacionado?**
R: A cada passo, determine qual metade está ordenada comparando `nums[left]` com `nums[mid]`. Se a metade esquerda está ordenada (`nums[left] <= nums[mid]`), verifique se o alvo cai nesse intervalo ordenado. Caso contrário, a metade direita está ordenada — verifique esse intervalo. Elimine a metade que não pode conter o alvo. O(log n).

**P3: Como encontrar o mínimo em um array ordenado rotacionado?**
R: Busca binária onde você compara `nums[mid]` com `nums[right]`. Se `nums[mid] > nums[right]`, o mínimo está na metade direita. Caso contrário, está na metade esquerda (incluindo mid). O(log n).

**P4: Explique a busca binária na resposta com um exemplo.**
R: O padrão: em vez de buscar um valor em um array, busque no espaço de respostas o menor valor que satisfaz uma condição. Exemplo: "capacidade mínima de navio para entregar todos os pacotes em D dias." Defina `canShip(capacity)` — verdadeiro se os pacotes podem ser entregues em D dias com a capacidade dada. Essa função é monótona (se a capacidade funciona, capacidades maiores também funcionam). Busca binária em [peso_máximo, soma_pesos] para encontrar a menor capacidade que funciona.

**P5: O que é o problema da raiz quadrada inteira e como resolvê-lo com busca binária?**
R: Encontrar o maior inteiro k tal que k² ≤ x. Busca binária em [1, x/2] (ou [0, x]). Para cada mid, verificar se mid² ≤ x. Estreitar o espaço de busca. O(log x).

---

## 9. Exercícios

**Exercício 1:** Encontre a raiz quadrada inteira (parte inteira do piso) sem usar `Math.sqrt`. Alcance O(log n).

**Exercício 2:** Pesquise em uma matriz 2D onde cada linha está ordenada e o primeiro elemento de cada linha é maior que o último elemento da linha anterior. Verifique se um alvo existe. Alcance O(log(m×n)).

**Exercício 3:** Encontre o k-ésimo menor elemento em uma BST usando busca binária (não percurso em ordem).
*Dica: busca binária no intervalo de valores, contando elementos ≤ mid usando a estrutura da BST.*

**Exercício 4:** Dado um array de inteiros, verifique se existem dois elementos a e b tais que a² + b² = target usando busca binária.
*Dica: ordene o array. Para cada elemento a, faça busca binária por `target - a²` entre os quadrados.*

**Exercício 5:** Encontre o número mínimo de dias para fazer pelo menos m buquês de um jardim de flores que florescem em dias diferentes. Cada buquê precisa de k flores adjacentes.
*Dica: busca binária na resposta (número de dias). Para uma quantidade de dias, verifique se flores adjacentes suficientes já floresceram.*

---

## 10. Leituras Complementares

- CLRS Capítulo 2.3 — Projetando Algoritmos (apresenta busca binária formalmente)
- [Busca Binária — padrões explicados](https://labuladong.online/algo/essential-technique/binary-search-framework/)
- [Busca Binária na Resposta — exemplos](https://cp-algorithms.com/num_methods/binary_search.html)
- LeetCode: #704 Binary Search, #33 Search in Rotated Sorted Array, #153 Find Minimum in Rotated Sorted Array, #875 Koko Eating Bananas, #1011 Capacity to Ship Packages
