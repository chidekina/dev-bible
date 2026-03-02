# Desafios de CS

> Desafios de algoritmos e estruturas de dados cobrindo os padrões mais comuns em entrevistas e programação competitiva. Mistura de problemas fáceis, médios e difíceis. Resolva cada um antes de ler as dicas. Busque a complexidade indicada — uma solução de força bruta que funciona é um ponto de partida, não de chegada. Tópicos: arrays, strings, árvores, grafos, programação dinâmica, sliding window e dois ponteiros.

---

## Exercício 1 — Two Sum (Fácil)

**Problema:** Dado um array de inteiros `nums` e um inteiro `target`, retorne os índices dos dois números que somam `target`. Assuma que existe exatamente uma solução. Você não pode usar o mesmo elemento duas vezes.

**Restrições:**
- `2 <= nums.length <= 10^4`
- `-10^9 <= nums[i] <= 10^9`
- Existe exatamente uma resposta válida

**Exemplo:**
```
Input:  nums = [2, 7, 11, 15], target = 9
Output: [0, 1]   // nums[0] + nums[1] = 9

Input:  nums = [3, 2, 4], target = 6
Output: [1, 2]
```

**Dicas:**
1. Um loop aninhado O(n²) funciona, mas não é aceitável para entradas grandes.
2. Para cada elemento, você precisa do "complemento": `complement = target - nums[i]`. Você já o viu antes?
3. Um hash map armazenando `{ valor → índice }` permite responder isso em O(1).
4. Percorra uma vez: para cada elemento, verifique o map pelo complemento primeiro, depois insira o elemento.

**Complexidade alvo:** Tempo O(n) | Espaço O(n)

---

## Exercício 2 — Parênteses Válidos (Fácil)

**Problema:** Dada uma string `s` contendo apenas `(`, `)`, `{`, `}`, `[`, `]`, determine se a string é válida. Uma string válida tem cada abre-chave fechado pelo mesmo tipo na ordem correta.

**Restrições:**
- `1 <= s.length <= 10^4`
- `s` contém apenas caracteres de colchetes

**Exemplo:**
```
Input:  "()[]{}"  → true
Input:  "([)]"    → false
Input:  "{[]}"    → true
Input:  "("       → false
```

**Dicas:**
1. Quando você encontra um abre-chave, precisa se lembrar dele para depois.
2. Uma pilha é perfeita para isso: empilhe os abre-chaves, desempilhe e verifique a cada fecha-chave.
3. Se a pilha estiver vazia quando você encontrar um fecha-chave, é inválido.
4. Após processar a string inteira, a pilha deve estar vazia.

**Complexidade alvo:** Tempo O(n) | Espaço O(n)

---

## Exercício 3 — Melhor Momento para Comprar e Vender Ação (Fácil)

**Problema:** Dado um array `prices` onde `prices[i]` é o preço da ação no dia `i`, encontre o lucro máximo de uma única transação de compra e venda. O dia de venda deve vir depois do dia de compra. Retorne `0` se não existir negócio lucrativo.

**Restrições:**
- `1 <= prices.length <= 10^5`
- `0 <= prices[i] <= 10^4`

**Exemplo:**
```
Input:  [7, 1, 5, 3, 6, 4]  → Output: 5   // compra no dia 2 (1), vende no dia 5 (6)
Input:  [7, 6, 4, 3, 1]     → Output: 0   // preços só caem
```

**Dicas:**
1. Você quer a maior diferença onde o elemento maior vem depois do menor.
2. Percorra da esquerda para a direita. Mantenha duas variáveis: `minPrice` e `maxProfit`.
3. A cada passo: `maxProfit = max(maxProfit, currentPrice - minPrice)`, depois atualize `minPrice` se necessário.
4. Não é preciso olhar para trás — `minPrice` já guarda o melhor ponto de compra visto até agora.

**Complexidade alvo:** Tempo O(n) | Espaço O(1)

---

## Exercício 4 — Maior Substring Sem Caracteres Repetidos (Médio)

**Problema:** Dada uma string `s`, encontre o comprimento da maior substring que não contém caracteres repetidos.

**Restrições:**
- `0 <= s.length <= 5 * 10^4`
- `s` pode conter letras em inglês, dígitos, símbolos e espaços

**Exemplo:**
```
Input:  "abcabcbb"  → Output: 3   // "abc"
Input:  "bbbbb"     → Output: 1   // "b"
Input:  "pwwkew"    → Output: 3   // "wke"
```

**Dicas:**
1. Este é um problema de sliding window. Mantenha uma janela `[left, right]` sem duplicatas.
2. Use um map para guardar o último índice onde cada caractere foi visto.
3. Quando `s[right]` já está na janela, avance `left` para `max(left, lastSeen[s[right]] + 1)`.
4. A resposta é `max(answer, right - left + 1)` após cada passo.

**Complexidade alvo:** Tempo O(n) | Espaço O(min(n, |alfabeto|))

---

## Exercício 5 — Produto do Array Exceto Si Mesmo (Médio)

**Problema:** Dado um array de inteiros `nums`, retorne um array `output` onde `output[i]` é igual ao produto de todos os elementos de `nums` exceto `nums[i]`. Você não pode usar divisão, e deve executar em O(n).

**Restrições:**
- `2 <= nums.length <= 10^5`
- `-30 <= nums[i] <= 30`
- O produto de qualquer prefixo ou sufixo cabe em um inteiro de 32 bits

**Exemplo:**
```
Input:  [1, 2, 3, 4]     → Output: [24, 12, 8, 6]
Input:  [-1, 1, 0, -3, 3] → Output: [0, 0, 9, 0, 0]
```

**Dicas:**
1. `output[i] = (produto de todos os elementos à esquerda de i) * (produto de todos os elementos à direita de i)`.
2. Primeira passagem da esquerda para a direita: preencha `output[i]` com o produto do prefixo.
3. Segunda passagem da direita para a esquerda: mantenha um produto acumulado do sufixo e multiplique em `output[i]`.
4. Nenhum array extra é necessário — o array de saída serve como o array de prefixo.

**Complexidade alvo:** Tempo O(n) | Espaço O(1) extra (array de saída excluído)

---

## Exercício 6 — Profundidade Máxima de Árvore Binária (Fácil)

**Problema:** Dado o nó raiz de uma árvore binária, retorne sua profundidade máxima — o número de nós ao longo do caminho mais longo da raiz até o nó folha mais distante.

**Restrições:**
- Número de nós: `[0, 10^4]`
- `-100 <= Node.val <= 100`

**Exemplo:**
```
Árvore:    3
          / \
         9  20
            / \
           15   7

Output: 3
```

**Dicas:**
1. Abordagem recursiva: profundidade de um nó é `1 + max(depth(left), depth(right))`.
2. Caso base: nó `null` tem profundidade `0`.
3. Alternativa iterativa: BFS por nível — conte o número de níveis.
4. DFS iterativo com uma pilha de pares `(nó, profundidadeAtual)` também funciona.

**Complexidade alvo:** Tempo O(n) | Espaço O(h) onde h = altura (O(n) no pior caso, O(log n) balanceado)

---

## Exercício 7 — Ancestral Comum Mais Baixo em uma BST (Médio)

**Problema:** Dado um binary search tree e dois nós `p` e `q`, encontre seu ancestral comum mais baixo (LCA). O LCA é o nó mais profundo que tem `p` e `q` como descendentes (um nó é descendente de si mesmo).

**Restrições:**
- Todos os valores dos nós são únicos
- `p` e `q` existem garantidamente na BST
- `2 <= número de nós <= 10^5`

**Exemplo:**
```
BST:          6
             / \
            2   8
           / \ / \
          0  4 7  9
            / \
           3   5

LCA(2, 8) → 6
LCA(2, 4) → 2
LCA(0, 5) → 2
```

**Dicas:**
1. Explore a propriedade da BST: se `p` e `q` são menores que `node.val`, desça à esquerda. Se ambos são maiores, desça à direita. Caso contrário, `node` é o LCA (os caminhos divergem aqui).
2. Sem necessidade de recursão — itere com um loop `while`.
3. Isso funciona porque qualquer nó onde `p.val <= node.val <= q.val` é necessariamente o LCA.

**Complexidade alvo:** Tempo O(h) | Espaço O(1)

---

## Exercício 8 — Número de Ilhas (Médio)

**Problema:** Dado um grid `m x n` de `'1'` (terra) e `'0'` (água), retorne o número de ilhas. Uma ilha é formada conectando células de terra adjacentes horizontal ou verticalmente, rodeadas por água em todos os lados.

**Restrições:**
- `1 <= m, n <= 300`
- `grid[i][j]` é `'0'` ou `'1'`

**Exemplo:**
```
Grid 1:           Grid 2:
1 1 1 1 0         1 1 0 0 0
1 1 0 1 0         1 1 0 0 0
1 1 0 0 0         0 0 1 0 0
0 0 0 0 0         0 0 0 1 1

Output: 1         Output: 3
```

**Dicas:**
1. Itere por cada célula. Quando encontrar um `'1'`, incremente o contador de ilhas e faça flood-fill para marcar toda a ilha como visitada.
2. Flood-fill: DFS ou BFS a partir da célula inicial. Marque cada `'1'` visitado como `'0'` (afunde-o) para evitar revisitá-lo.
3. Direções: cima, baixo, esquerda, direita. Proteja contra índices fora dos limites.
4. A modificação do grid de entrada é aceitável; caso contrário, use um grid booleano `visited` separado.

**Complexidade alvo:** Tempo O(m * n) | Espaço O(m * n) pilha DFS no pior caso

---

## Exercício 9 — Troco (Médio)

**Problema:** Dado um array `coins` de denominações de moedas e um inteiro `amount`, retorne o número mínimo de moedas necessárias para atingir exatamente esse valor. Se for impossível, retorne `-1`. Você tem moedas ilimitadas de cada denominação.

**Restrições:**
- `1 <= coins.length <= 12`
- `1 <= coins[i] <= 2^31 - 1`
- `0 <= amount <= 10^4`

**Exemplo:**
```
Input:  coins = [1, 5, 11], amount = 15
Output: 3   // 5 + 5 + 5

Input:  coins = [2], amount = 3
Output: -1

Input:  coins = [1], amount = 0
Output: 0
```

**Dicas:**
1. DP bottom-up. Defina `dp[i]` = número mínimo de moedas para fazer o valor `i`.
2. Inicialize: `dp[0] = 0`, todas as outras entradas como `Infinity`.
3. Para cada `i` de 1 até `amount`, e para cada moeda `c` onde `c <= i`: `dp[i] = min(dp[i], dp[i - c] + 1)`.
4. Retorne `dp[amount] === Infinity ? -1 : dp[amount]`.

**Complexidade alvo:** Tempo O(amount * n) | Espaço O(amount)

---

## Exercício 10 — Merge de K Listas Ordenadas (Difícil)

**Problema:** Você recebe um array de `k` listas encadeadas, cada uma ordenada em ordem crescente. Faça o merge de todas as listas em uma única lista encadeada ordenada e retorne-a.

**Restrições:**
- `0 <= k <= 10^4`
- `0 <= lists[i].length <= 500`
- Total de nós: no máximo `10^4`
- Valores dos nós: `-10^4 <= val <= 10^4`

**Exemplo:**
```
Input:  [[1,4,5], [1,3,4], [2,6]]
Output: [1, 1, 2, 3, 4, 4, 5, 6]
```

**Dicas:**
1. Merge ingênuo um por um: O(kN) — aceitável para k pequeno, mas não é ótimo.
2. Dividir e conquistar: agrupe as listas em pares e faça o merge de cada par, depois repita. Tempo O(N log k).
3. Min-heap: inicialize o heap com o nó cabeça de cada lista. Retire o mínimo, insira seu `next` no heap. Repita até o heap estar vazio.
4. Em JS/TS não há heap nativo — implemente um binary min-heap simples ou use um array ordenado com inserção binária para k pequeno.

**Complexidade alvo:** Tempo O(N log k) | Espaço O(k)

---

## Exercício 11 — Armadilha de Água da Chuva (Difícil)

**Problema:** Dados `n` inteiros não negativos representando um mapa de elevação (cada barra tem largura 1), calcule quanta água pode ser retida após a chuva.

**Restrições:**
- `n == height.length`
- `1 <= n <= 2 * 10^4`
- `0 <= height[i] <= 10^5`

**Exemplo:**
```
Input:  [0,1,0,2,1,0,1,3,2,1,2,1]
Output: 6

Input:  [4,2,0,3,2,5]
Output: 9
```

**Dicas:**
1. Água no índice `i` = `min(maxLeft[i], maxRight[i]) - height[i]`, com mínimo de 0.
2. Pré-compute os arrays `maxLeft` (varredura à esquerda) e `maxRight` (varredura à direita) — tempo O(n), espaço O(n).
3. Otimize para espaço O(1) com dois ponteiros. Aproxime-se de ambos os extremos. Mova o lado com o menor máximo — a contribuição daquele lado está completamente determinada.
4. O invariante: `water[i] = min(leftMax, rightMax) - height[i]`.

**Complexidade alvo:** Tempo O(n) | Espaço O(1) (abordagem de dois ponteiros)

---

## Exercício 12 — Quebra de Palavras (Médio)

**Problema:** Dada uma string `s` e um dicionário `wordDict`, retorne `true` se `s` puder ser segmentada em uma sequência de palavras do dicionário separadas por espaços.

**Restrições:**
- `1 <= s.length <= 300`
- `1 <= wordDict.length <= 1000`
- Todas as strings usam letras minúsculas do inglês
- As palavras em `wordDict` são únicas

**Exemplo:**
```
Input:  s = "leetcode",      wordDict = ["leet","code"]          → true
Input:  s = "applepenapple", wordDict = ["apple","pen"]          → true
Input:  s = "catsandog",     wordDict = ["cats","dog","sand","and","cat"] → false
```

**Dicas:**
1. Defina `dp[i]` = `true` se `s[0..i-1]` puder ser segmentado usando o dicionário.
2. Caso base: `dp[0] = true` (prefixo vazio).
3. Para cada posição `i`, tente cada ponto de divisão `j < i`: se `dp[j]` for verdadeiro e `s[j..i-1]` estiver no dicionário, defina `dp[i] = true`.
4. Converta `wordDict` para um `Set` para lookup em O(1). Retorne `dp[s.length]`.

**Complexidade alvo:** Tempo O(n² * m) onde m = comprimento médio das palavras | Espaço O(n)

---

## Exercício 13 — Serializar e Desserializar Árvore Binária (Difícil)

**Problema:** Projete um algoritmo para serializar uma árvore binária em uma string, e desserializar essa string de volta para a árvore original. Não há restrição de formato — você o define.

**Restrições:**
- Número de nós: `[0, 10^4]`
- `-1000 <= Node.val <= 1000`
- O desserializador deve produzir uma árvore estruturalmente idêntica à original

**Exemplo:**
```
Árvore:    1
          / \
         2   3
            / \
           4   5

Serializado (um formato válido): "1,2,X,X,3,4,X,X,5,X,X"
Desserializado: mesma árvore
```

**Dicas:**
1. Use percurso em pré-ordem (raiz → esquerda → direita). Represente filhos nulos com um sentinela como `"X"`.
2. Serialização: pré-ordem recursiva. Adicione `node.val` ou `"X"` a cada passo, unido por vírgulas.
3. Desserialização: divida pela vírgula, use um ponteiro ou fila. Consuma tokens em pré-ordem: leia um token, se for `"X"` retorne null, senão crie um nó e recurse para esquerda depois direita.
4. BFS por nível também funciona e produz um formato mais legível, mas a pré-ordem tem recursão mais simples.

**Complexidade alvo:** Tempo O(n) | Espaço O(n)

---

## Exercício 14 — Encontrar a Mediana de um Fluxo de Dados (Difícil)

**Problema:** Projete uma estrutura de dados suportando `addNum(num)` e `findMedian()`. `findMedian` retorna a mediana de todos os números adicionados até agora (média dos dois valores do meio se a contagem for par).

**Restrições:**
- `-10^5 <= num <= 10^5`
- No máximo `5 * 10^4` chamadas no total
- Pelo menos um número adicionado antes de chamar `findMedian`

**Exemplo:**
```
addNum(1) → addNum(2) → findMedian() = 1.5
addNum(3) → findMedian() = 2.0
```

**Dicas:**
1. Mantenha dois heaps: um **max-heap** para a metade inferior, um **min-heap** para a metade superior.
2. Mantenha os tamanhos balanceados: `|maxHeap.size - minHeap.size| <= 1`.
3. Adicione ao max-heap primeiro. Se o topo dele ultrapassar o topo do min-heap, mova o topo do max-heap para o min-heap. Depois rebalanceie os tamanhos.
4. `findMedian`: se os tamanhos são iguais, faça a média dos dois topos. Caso contrário, retorne o topo do heap maior.
5. JS não tem heap nativo — implemente uma classe `MinHeap` com `push` e `pop` em O(log n).

**Complexidade alvo:** addNum O(log n) | findMedian O(1) | Espaço O(n)

---

## Exercício 15 — Dicionário Alienígena (Difícil)

**Problema:** Você recebe uma lista de palavras ordenadas de acordo com a ordenação de caracteres de uma língua alienígena (usando as mesmas letras do inglês). Derive a ordem dos caracteres. Se nenhuma ordem válida existir (ciclo detectado), retorne `""`. Se múltiplas ordens válidas existirem, retorne qualquer uma.

**Restrições:**
- `1 <= words.length <= 100`
- `1 <= words[i].length <= 100`
- Todos os caracteres são letras minúsculas do inglês

**Exemplo:**
```
Input:  ["wrt","wrf","er","ett","rftt"]   → Output: "wertf"
Input:  ["z","x"]                          → Output: "zx"
Input:  ["z","x","z"]                      → Output: ""   // ciclo
Input:  ["abc","ab"]                       → Output: ""   // ordenação de prefixo inválida
```

**Dicas:**
1. Compare cada par de palavras adjacentes caractere por caractere. O primeiro par de caracteres diferentes fornece uma regra de ordenação: `charA` vem antes de `charB`.
2. Construa um grafo dirigido com arestas `charA → charB`. Rastreie os graus de entrada de cada caractere.
3. Aplique a ordenação topológica de Kahn: comece com todos os nós de grau de entrada 0, processe em ordem BFS.
4. Se o comprimento da saída ordenada não for igual ao número de caracteres únicos, existe um ciclo — retorne `""`.
5. Caso especial: se `words[i]` é prefixo de `words[i-1]`, a entrada em si é inválida.

**Complexidade alvo:** Tempo O(C) onde C = total de caracteres | Espaço O(1) (tamanho do alfabeto é constante em 26)
