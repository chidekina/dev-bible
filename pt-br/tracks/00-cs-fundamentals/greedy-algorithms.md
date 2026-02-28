# Algoritmos Gulosos

## 1. O Que É e Por Que Importa

Um algoritmo guloso faz a escolha localmente ótima a cada passo, esperando alcançar uma solução globalmente ótima. Sem backtracking, sem busca exaustiva — apenas pegue a melhor opção disponível agora e siga em frente.

Quando o greedy funciona, ele é elegante e rápido — frequentemente O(n log n) para problemas onde DP seria O(n²). O desafio é saber quando o greedy funciona e quando não funciona. Greedy falha no problema da mochila 0/1, em muitas instâncias de troco de moedas e em outros problemas onde uma escolha localmente subótima agora permite um resultado globalmente melhor depois.

---

## 2. Conceitos Fundamentais

### Propriedade de Escolha Gulosa

Um problema tem a propriedade de escolha gulosa se fazer a escolha localmente ótima a cada passo leva a uma solução globalmente ótima. Isso deve ser **provado** — não assumido.

### Subestrutura Ótima

Após fazer uma escolha gulosa, o problema restante é uma instância do mesmo problema. A escolha gulosa reduz o tamanho do problema.

### Provando Correção Gulosa: Argumento de Troca

A técnica de prova padrão:
1. Assuma que uma solução ótima existe que difere da solução gulosa
2. Mostre que você pode "trocar" uma escolha não-gulosa pela escolha gulosa sem piorar a solução
3. Conclua que a solução gulosa é pelo menos tão boa quanto qualquer solução ótima — portanto é ótima

### Quando o Greedy Falha

- **Mochila 0/1:** pegar o item com maior razão valor/peso primeiro não leva ao empacotamento ótimo (você não pode usar frações)
- **Troco de moedas com denominações arbitrárias:** moedas = [1, 3, 4], valor = 6. Greedy dá 4+1+1 = 3 moedas. Ótimo: 3+3 = 2 moedas.
- **Caminho mais curto com pesos negativos:** Dijkstra (guloso) falha; precisa de Bellman-Ford
- **Multiplicação em cadeia de matrizes:** ordem gulosa não minimiza as multiplicações

---

## 3. Como Funciona

```
Seleção de Atividades (escalonamento de intervalos):
Atividades: [(1,4), (3,5), (0,6), (5,7), (3,9), (5,9), (6,10), (8,11), (8,12), (2,14), (12,16)]
Objetivo: selecionar o número máximo de atividades não sobrepostas.

Guloso: ordena por tempo de término, sempre pega a próxima atividade que começa após a última selecionada terminar.

Ordenadas por término: [(1,4),(3,5),(0,6),(5,7),(3,9),(5,9),(6,10),(8,11),(8,12),(2,14),(12,16)]
Seleciona (1,4): fim=4
Próxima começa >= 4: (3,5)? Não. (5,7)? início=5 >=4. Seleciona (5,7): fim=7
Próxima começa >= 7: (8,11)? início=8 >=7. Seleciona (8,11): fim=11
Próxima começa >= 11: (12,16)? início=12 >=11. Seleciona (12,16): fim=16

Resultado: 4 atividades. Ótimo.
```

---

## 4. Exemplos de Código (TypeScript)

```typescript
// --- Seleção de Atividades / Escalonamento de Intervalos ---
// Maximizar o número de intervalos não sobrepostos
function activitySelection(intervals: [number, number][]): [number, number][] {
  // Ordena por tempo de término — guloso: sempre pega a atividade que termina mais cedo
  const sorted = [...intervals].sort((a, b) => a[1] - b[1]);
  const selected: [number, number][] = [];
  let lastEnd = -Infinity;

  for (const [start, end] of sorted) {
    if (start >= lastEnd) {
      selected.push([start, end]);
      lastEnd = end;
    }
  }
  return selected;
}

// --- Número Mínimo de Salas de Reunião ---
// Dados intervalos de reuniões, encontrar o número mínimo de salas necessárias
function minMeetingRooms(intervals: [number, number][]): number {
  const starts = intervals.map(i => i[0]).sort((a, b) => a - b);
  const ends = intervals.map(i => i[1]).sort((a, b) => a - b);

  let rooms = 0, maxRooms = 0;
  let s = 0, e = 0;

  while (s < starts.length) {
    if (starts[s] < ends[e]) {
      rooms++;      // uma reunião começa antes de outra terminar → precisa de nova sala
      s++;
    } else {
      rooms--;      // uma reunião terminou → sala liberada
      e++;
    }
    maxRooms = Math.max(maxRooms, rooms);
  }
  return maxRooms;
}

// --- Jogo do Salto: você consegue chegar ao último índice? ---
// Guloso: rastreia o índice máximo alcançável da posição atual
function canJump(nums: number[]): boolean {
  let maxReach = 0;
  for (let i = 0; i < nums.length; i++) {
    if (i > maxReach) return false; // não consegue chegar a esta posição
    maxReach = Math.max(maxReach, i + nums[i]);
  }
  return true;
}

// --- Jogo do Salto II: mínimo de saltos para chegar ao último índice ---
function jump(nums: number[]): number {
  let jumps = 0, currentEnd = 0, farthest = 0;

  for (let i = 0; i < nums.length - 1; i++) {
    farthest = Math.max(farthest, i + nums[i]);
    if (i === currentEnd) {
      // Deve fazer um salto — pega o que chega mais longe
      jumps++;
      currentEnd = farthest;
    }
  }
  return jumps;
}

// --- Posto de Gasolina: rota circular ---
// Encontrar o posto de gasolina a partir do qual você pode completar o circuito
function canCompleteCircuit(gas: number[], cost: number[]): number {
  let totalGas = 0, currentGas = 0, start = 0;

  for (let i = 0; i < gas.length; i++) {
    const net = gas[i] - cost[i];
    totalGas += net;
    currentGas += net;

    if (currentGas < 0) {
      // Não consegue chegar ao posto i+1 a partir do início atual — tenta começar após i
      start = i + 1;
      currentGas = 0;
    }
  }

  return totalGas >= 0 ? start : -1; // se total >= 0, existe solução
}

// --- Mochila Fracionária: maximizar valor, pode pegar frações ---
// Guloso pela razão valor/peso — funciona para fracionária, NÃO para mochila 0/1
function fractionalKnapsack(items: { weight: number; value: number }[], capacity: number): number {
  const sorted = [...items].sort((a, b) => b.value / b.weight - a.value / a.weight);
  let totalValue = 0, remaining = capacity;

  for (const item of sorted) {
    if (remaining === 0) break;
    const take = Math.min(item.weight, remaining);
    totalValue += take * (item.value / item.weight);
    remaining -= take;
  }
  return totalValue;
}

// --- Escalonador de Tarefas: mínimo de intervalos de CPU (com tempo de resfriamento n) ---
// Deve esperar n intervalos antes de executar a mesma tarefa novamente
function leastInterval(tasks: string[], n: number): number {
  const freq: Record<string, number> = {};
  for (const t of tasks) freq[t] = (freq[t] ?? 0) + 1;

  const maxFreq = Math.max(...Object.values(freq));
  const maxCount = Object.values(freq).filter(f => f === maxFreq).length;

  // Fórmula: max((maxFreq - 1) * (n + 1) + maxCount, tasks.length)
  // A fórmula considera os slots ociosos necessários entre as tarefas mais frequentes
  return Math.max((maxFreq - 1) * (n + 1) + maxCount, tasks.length);
}

// --- Distribuir Biscoitos: maximizar o número de crianças satisfeitas ---
// Cada criança i tem fator de ganância g[i], cada biscoito tem tamanho s[j]
// Uma criança fica satisfeita se s[j] >= g[i]. Cada biscoito é atribuído a uma criança.
function findContentChildren(g: number[], s: number[]): number {
  g.sort((a, b) => a - b); // ordena por ganância crescente
  s.sort((a, b) => a - b); // ordena por tamanho crescente

  let child = 0, cookie = 0;
  while (child < g.length && cookie < s.length) {
    if (s[cookie] >= g[child]) child++; // satisfaz esta criança
    cookie++; // usa este biscoito independente
  }
  return child;
}

// --- Rótulos de Partição: particionar string em máx. de partes onde cada letra aparece em só uma parte ---
function partitionLabels(s: string): number[] {
  // Para cada caractere, encontra sua última ocorrência
  const lastOccurrence = new Map<string, number>();
  for (let i = 0; i < s.length; i++) lastOccurrence.set(s[i], i);

  const partitions: number[] = [];
  let start = 0, end = 0;

  for (let i = 0; i < s.length; i++) {
    end = Math.max(end, lastOccurrence.get(s[i])!); // estende partição para incluir todo este char
    if (i === end) {
      // Partição atual está completa
      partitions.push(end - start + 1);
      start = i + 1;
    }
  }
  return partitions;
}
// "ababcbacadefegdehijhklij" → [9, 7, 8]

// --- Número Mínimo de Flechas para Estourar Balões ---
// Balões: [xstart, xend]. Flecha em x estoura todos os balões com xstart <= x <= xend
function findMinArrowShots(points: [number, number][]): number {
  if (points.length === 0) return 0;
  const sorted = [...points].sort((a, b) => a[1] - b[1]); // ordena por término
  let arrows = 1;
  let arrowPos = sorted[0][1]; // primeira flecha no término do primeiro balão

  for (let i = 1; i < sorted.length; i++) {
    if (sorted[i][0] > arrowPos) {
      // Este balão começa depois da flecha atual — precisa de nova flecha
      arrows++;
      arrowPos = sorted[i][1];
    }
  }
  return arrows;
}

// --- Codificação de Huffman (conceito) ---
// Construir código binário livre de prefixo ótimo: caracteres mais frequentes têm códigos menores
// Implementação usa um min-heap (veja heaps.md)
interface HuffmanNode {
  char?: string;
  freq: number;
  left?: HuffmanNode;
  right?: HuffmanNode;
}

function buildHuffmanCodes(freq: Map<string, number>): Map<string, string> {
  // Min-heap por frequência
  const heap: HuffmanNode[] = [...freq.entries()]
    .map(([char, f]) => ({ char, freq: f }))
    .sort((a, b) => a.freq - b.freq);

  while (heap.length > 1) {
    const left = heap.shift()!;  // menor freq
    const right = heap.shift()!; // segunda menor freq
    const merged: HuffmanNode = { freq: left.freq + right.freq, left, right };
    // Insere nó mesclado em posição ordenada
    const insertPos = heap.findIndex(n => n.freq >= merged.freq);
    if (insertPos === -1) heap.push(merged);
    else heap.splice(insertPos, 0, merged);
  }

  const codes = new Map<string, string>();
  function traverse(node: HuffmanNode, code: string): void {
    if (node.char !== undefined) { codes.set(node.char, code); return; }
    if (node.left) traverse(node.left, code + "0");
    if (node.right) traverse(node.right, code + "1");
  }
  traverse(heap[0], "");
  return codes;
}
```

---

## 5. Erros Comuns e Armadilhas

> ⚠️ **Aplicar greedy quando apenas DP está correto.** O erro mais perigoso. Greedy para mochila 0/1 ou troco de moedas com denominações arbitrárias dá respostas erradas. Sempre verifique se a propriedade de escolha gulosa vale antes de usar greedy.

> ⚠️ **Ordenar pela métrica errada.** No escalonamento de intervalos, ordenar por tempo de início dá resultados errados (intervalos mais longos bloqueiam mais). Ordene por tempo de término. No escalonamento por lucro máximo, ordene por prazo. Errar a chave de ordenação invalida o greedy.

> ⚠️ **Confundir mochila fracionária (greedy funciona) com mochila 0/1 (greedy falha).** Fracionária: você pode pegar 0.5 de um item. 0/1: você pega ou não pega. A razão valor/peso greedy funciona para fracionária, falha para 0/1.

> ⚠️ **Posto de gasolina: não verificar o combustível total.** Se combustível total < custo total, não existe solução independente do ponto de partida. Verifique isso primeiro antes de rodar a busca gulosa.

---

## 6. Quando Usar / Não Usar

**Greedy funciona bem para:**
- Escalonamento de intervalos (ordena por tempo de término)
- Codificação de Huffman (códigos de prefixo ótimos)
- Algoritmo de Dijkstra (caminho mais curto, pesos não-negativos)
- MST de Prim / Kruskal
- Mochila fracionária
- Problemas onde localmente ótimo = globalmente ótimo (provável via argumento de troca)

**Greedy falha para:**
- Mochila 0/1 (precisa de DP)
- Troco de moedas com denominações não-canônicas (precisa de DP)
- Caminho mais curto com arestas negativas (precisa de Bellman-Ford)
- Qualquer problema onde uma escolha subótima agora permite um resultado global melhor depois

**Comparação com DP:**
- Greedy: O(n log n) típico, faz escolhas irrevogáveis, funciona quando a propriedade gulosa vale
- DP: O(n²) ou O(n³) típico, explora todas as opções, sempre correto para problemas aplicáveis
- Sempre prefira greedy quando comprovadamente correto — é mais simples e mais rápido

---

## 7. Cenário do Mundo Real

Um agendador de reuniões precisa encontrar o número mínimo de salas de conferência:

```typescript
interface Meeting {
  title: string;
  start: number; // minutos desde a meia-noite
  end: number;
}

function scheduleRooms(meetings: Meeting[]): Map<number, Meeting[]> {
  const sorted = [...meetings].sort((a, b) => a.start - b.start);
  const rooms = new Map<number, number>(); // roomId → quando fica livre
  const assignments = new Map<number, Meeting[]>();
  let nextRoomId = 0;

  for (const meeting of sorted) {
    // Encontra uma sala que está livre (a reunião anterior termina antes desta começar)
    let assignedRoom = -1;
    for (const [roomId, freeAt] of rooms.entries()) {
      if (freeAt <= meeting.start) {
        assignedRoom = roomId;
        break;
      }
    }

    if (assignedRoom === -1) {
      // Sem sala livre — aloca uma nova
      assignedRoom = nextRoomId++;
      assignments.set(assignedRoom, []);
    }

    rooms.set(assignedRoom, meeting.end);
    assignments.get(assignedRoom)!.push(meeting);
  }

  return assignments;
}
```

---

## 8. Perguntas de Entrevista

**P1: Quando o greedy falha e você precisa de DP?**
R: Quando uma escolha localmente ótima em um passo impede um resultado global melhor. Exemplo: troco de moedas com moedas [1, 3, 4], valor = 6. Greedy pega 4 primeiro (localmente melhor), restando 2 → precisa de 4+1+1 = 3 moedas. DP encontra 3+3 = 2 moedas. A escolha gulosa (pegar a maior moeda) viola a propriedade de escolha gulosa para esse conjunto de denominações.

**P2: Como você prova que um algoritmo guloso está correto?**
R: Argumento de troca: assuma que qualquer solução ótima O difere da solução gulosa G em algum passo. Mostre que você pode modificar O para corresponder à escolha de G naquele passo sem aumentar o custo. Por indução, G é tão bom quanto O a cada passo — portanto G é ótimo.

**P3: Resolva o problema de maximização de escalonamento de intervalos.**
R: Ordene os intervalos por tempo de término. Selecione gulositamente o próximo intervalo que começa em ou após o término do último intervalo selecionado. Isso sempre deixa mais espaço para intervalos futuros. O(n log n) de tempo.

**P4: Explique o problema do escalonador de tarefas.**
R: Com tempo de resfriamento n, o mínimo de intervalos = max((maxFreq - 1) × (n + 1) + contagem_de_maxFreq, total de tarefas). A fórmula: arrange a tarefa mais frequente com lacunas, conta os tempos ociosos necessários. Se as tarefas preenchem as lacunas, nenhum tempo ocioso é necessário.

**P5: Qual é a abordagem gulosa do posto de gasolina?**
R: Rastreia o saldo acumulado de combustível. Quando fica negativo, o posto de início atual é inválido — qualquer posto até o atual também é inválido (eles teriam o mesmo ou pior déficit). Resete o início para o próximo posto. Se combustível total ≥ custo total, existe um início válido.

---

## 9. Exercícios

**Exercício 1:** Distribua biscoitos. Dados os fatores de ganância das crianças e os tamanhos dos biscoitos, atribua no máximo um biscoito por criança para maximizar o número de crianças satisfeitas.
*Dica: ordene ambos os arrays, use dois ponteiros.*

**Exercício 2:** Particione uma string em tantas partes quanto possível de modo que cada letra apareça em no máximo uma parte.
*Dica: para cada caractere, registre sua última ocorrência. Uma partição termina quando o índice atual é igual à maior última ocorrência de qualquer caractere visto.*

**Exercício 3:** Encontre o número mínimo de flechas para estourar todos os balões (cada balão é um intervalo, uma flecha em x estoura todos os balões que contêm x).
*Dica: ordene pela coordenada de término. Uma flecha pode estourar todos os balões pelos quais passa — coloque-a no término do balão que termina mais cedo.*

**Exercício 4:** Prove que a seleção de atividades por tempo de término é ótima usando um argumento de troca.

**Exercício 5:** Dado um array de inteiros não-negativos, encontre o número mínimo de passos para chegar ao fim se do índice i você pode saltar no máximo nums[i] passos.
*Dica: rastreie o índice mais distante alcançável e o limite do salto atual.*

---

## 10. Leituras Complementares

- CLRS Capítulo 16 — Algoritmos Gulosos
- [Algoritmos gulosos — cp-algorithms.com](https://cp-algorithms.com/greedy/)
- [Técnica de prova por argumento de troca](https://cp-algorithms.com/greedy/exchange_argument.html)
- LeetCode: #455 Assign Cookies, #763 Partition Labels, #452 Minimum Arrows, #55 Jump Game, #134 Gas Station, #621 Task Scheduler
