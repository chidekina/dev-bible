# Ferramentas de Profiling

## Visão Geral

Profiling é a prática de medir onde o tempo e a memória são consumidos em uma aplicação em execução. Você não consegue otimizar com eficiência sem profiling — otimizar sem medir é chute. Este capítulo cobre as principais ferramentas de profiling para Node.js (clinic.js, --prof, flame graphs com 0x) e para JavaScript no navegador (painel Performance do Chrome DevTools), com fluxos de trabalho práticos para identificar e resolver gargalos reais.

---

## Pré-requisitos

- Fundamentos de Node.js
- Entendimento básico de call stacks
- Track 10: Métricas de Performance Web (profiling do lado do navegador)

---

## Exemplos Práticos

### Chrome DevTools — profiling no navegador

**Gravando um trace de performance:**

1. Abra o DevTools → aba Performance
2. Clique no botão de gravação
3. Interaja com a página (role, clique, navegue)
4. Pare a gravação
5. Analise o flame chart

**Lendo o flame chart:**

```
Atividade da thread principal:
┌─────────────────────────────────────────────────────┐
│ Task (38ms — long task)                             │
│  ├─ Script evaluation                               │
│  │   ├─ handleClick (10ms)                          │
│  │   │   ├─ processData (8ms) ← FUNÇÃO QUENTE       │
│  │   │   │   ├─ sortArray (6ms)                     │
│  │   │   │   └─ filterItems (2ms)                   │
└─────────────────────────────────────────────────────┘
```

Caixas largas no flame chart = tempo gasto ali. Clique para ver o nome da função, arquivo e número de linha.

**Identificando long tasks:**

Long tasks (> 50ms) são destacadas em vermelho na timeline. Elas bloqueiam a entrada do usuário. Cada long task é candidata à otimização.

```typescript
// Use a User Timing API para marcar seções customizadas no trace do DevTools
performance.mark('processData:start');
const result = processData(largeDataset);
performance.mark('processData:end');
performance.measure('processData', 'processData:start', 'processData:end');

// Agora "processData" aparece como uma seção rotulada no painel Performance
```

### Node.js — profiler integrado

```bash
# Executa com o profiler V8
node --prof src/index.js

# Carrega o servidor e gera algum tráfego
k6 run tests/load/api.js --duration 30s

# Para o servidor (Ctrl+C)
# Processa o arquivo isolate-*.log gerado
node --prof-process isolate-0x*.log > profile.txt

# Lê o perfil
cat profile.txt | head -100
```

Exemplo de saída:

```
[Summary]:
   ticks  total  nonlib   name
   1234   45.3%   62.1%  JavaScript
    456   16.7%   22.9%  C++
    ...

[JavaScript]:
   ticks  total  nonlib   name
    890   32.7%   44.8%  LazyCompile: *processQuery /src/lib/db.js:45
    234    8.6%   11.8%  LazyCompile: *serialize /src/lib/json.js:12
```

A função no topo consumindo mais ticks é o hot path a otimizar.

### clinic.js — profiling amigável para Node.js

O Clinic.js fornece três ferramentas: `doctor` (diagnostica problemas), `flame` (flame graph), `bubbleprof` (operações assíncronas):

```bash
npm install -g clinic autocannon

# Doctor — diagnostica problemas comuns (CPU, atraso do event loop, memória)
clinic doctor -- node src/index.js
# Execute tráfego em outro terminal:
autocannon -c 100 -d 20 http://localhost:3000/api/products
# Ctrl+C no servidor, clinic abre o relatório automaticamente

# Flame graph — identifica funções quentes
clinic flame -- node src/index.js
autocannon -c 100 -d 20 http://localhost:3000/api/products
# Ctrl+C — abre o flame graph interativo no navegador
```

**Lendo um flame graph do clinic:**

- Barras horizontais largas = mais tempo gasto
- Cores vermelhas/laranja = hot paths (candidatos à otimização)
- Clique em uma barra para ampliar a call stack daquela função

### 0x — flame graphs para produção

O 0x é mais leve que o clinic e adequado para capturar perfis em produção:

```bash
npm install -g 0x

# Inicia o servidor com profiling
0x -o flame.html -- node src/index.js

# Gera carga
autocannon http://localhost:3000/api/products -d 30

# Ctrl+C — 0x escreve flame.html
open flame.html
```

### Identificando memory leaks

```bash
# Heap snapshot — mostra o que está na memória em um dado momento
# No Chrome DevTools → Memory → Take heap snapshot

# Ou de forma programática
import v8 from 'v8';
import fs from 'fs';

function takeHeapSnapshot(label: string) {
  const snapshot = v8.writeHeapSnapshot();
  console.log(`Heap snapshot salvo: ${snapshot}`);
}

// Tire um snapshot na inicialização e após carga sustentada
// Compare com Chrome DevTools → Memory → Load snapshot
```

**Detectando leaks com `--expose-gc` e monitoramento:**

```typescript
// src/lib/memory-monitor.ts
export function startMemoryMonitor(intervalMs = 30000) {
  setInterval(() => {
    const usage = process.memoryUsage();
    logger.info({
      rss: Math.round(usage.rss / 1024 / 1024) + 'MB',       // resident set size
      heap: Math.round(usage.heapUsed / 1024 / 1024) + 'MB',  // JS heap
      external: Math.round(usage.external / 1024 / 1024) + 'MB', // objetos C++
    }, 'Uso de memória');
  }, intervalMs);
}
```

Se o uso do heap cresce continuamente sem retornar à linha de base, há um memory leak. Causas comuns:
- Event listeners adicionados mas nunca removidos
- Closures mantendo referências a objetos grandes
- Caches sem limites de tamanho ou TTLs
- Arrays/maps globais que crescem sem limite

### Monitoramento de event loop lag

```typescript
// Mede o atraso do event loop (lag alto = algo está bloqueando)
export function monitorEventLoopLag() {
  let lastTick = process.hrtime.bigint();

  const check = () => {
    const now = process.hrtime.bigint();
    const lag = Number(now - lastTick - BigInt(10_000_000)) / 1_000_000; // lag em ms (esperado ~10ms)
    lastTick = now;

    if (lag > 50) {
      logger.warn({ lag: `${lag.toFixed(1)}ms` }, 'Alto event loop lag detectado');
    }

    setTimeout(check, 10); // verifica a cada 10ms
  };

  setTimeout(check, 10);
}

// Ou use o built-in
const { monitorEventLoopDelay } = require('perf_hooks');
const h = monitorEventLoopDelay({ resolution: 10 });
h.enable();
setInterval(() => {
  console.log(`Event loop P99 lag: ${h.percentile(99) / 1_000_000}ms`);
  h.reset();
}, 5000);
```

### Profiling em produção com segurança

```typescript
// Profiling condicional — apenas quando um header de debug está presente
fastify.addHook('onRequest', async (request, reply) => {
  if (request.headers['x-profile'] === process.env.PROFILING_SECRET) {
    // Inicia um perfil de 10 segundos e salva em arquivo
    const { Session } = await import('inspector');
    const session = new Session();
    session.connect();

    session.post('Profiler.enable', () => {
      session.post('Profiler.start', () => {
        setTimeout(() => {
          session.post('Profiler.stop', (err, { profile }) => {
            fs.writeFileSync(`profile-${Date.now()}.cpuprofile`, JSON.stringify(profile));
            session.disconnect();
          });
        }, 10000);
      });
    });
  }
});
```

---

## Padrões e Boas Práticas

- **Faça profiling antes de otimizar** — identifique o gargalo real; não adivinhe
- **Faça profiling sob carga realista** — execute com `autocannon` ou `k6` durante o profiling para simular tráfego de produção
- **Use clinic.js doctor primeiro** — ele identifica a categoria do problema (CPU, I/O, event loop) antes de você aprofundar a análise
- **Monitore event loop lag em produção** — exponha como métrica; picos indicam operações bloqueantes
- **Acompanhe a tendência do heap, não o tamanho absoluto** — um heap crescente que nunca estabiliza é um leak

---

## Anti-Padrões a Evitar

- Otimizar sem medir primeiro — você vai otimizar a coisa errada
- Fazer profiling com uma única requisição — você precisa de carga sustentada para revelar gargalos
- Ignorar waits de I/O — sua API pode parecer CPU-bound quando o problema real é uma query lenta no banco
- Otimização prematura — adicione profiling apenas após observar um problema em produção

---

## Debugging e Resolução de Problemas

**"O flame graph mostra tempo em 'unknown' ou código nativo"**
São operações internas do V8 (GC, compilador). Se o GC consome uma grande porcentagem, analise as taxas de alocação no heap — algo está criando muitos objetos de curta duração.

**"O event loop lag tem picos a cada 30 minutos"**
Algo está sendo executado em agendamento (cron job, pressão de garbage collection de um batch agendado). Verifique o que roda nesse intervalo.

**"A memória cresce durante o soak test mas o flame graph parece normal"**
O leak não está em código CPU-intensivo — é uma acumulação lenta. Use heap snapshots com 10 minutos de diferença e compare usando a view "comparison" do Chrome DevTools para encontrar quais objetos estão acumulando.

---

## Leituras Complementares

- [Documentação do clinic.js](https://clinicjs.org/documentation/)
- [Chrome DevTools: Record runtime performance](https://developer.chrome.com/docs/devtools/performance/)
- [0x — Flamegraphs para Node.js](https://github.com/davidmarkclements/0x)
- [Guia de diagnósticos do Node.js](https://nodejs.org/en/docs/guides/diagnostics/)
- Track 10: [Métricas de Performance Web](web-performance-metrics.md)

---

## Resumo

Profiling é a base da otimização de performance. Para servidores Node.js, o clinic.js é o ponto de entrada — seu modo `doctor` diagnostica se o gargalo é CPU, atraso do event loop ou I/O. O modo `flame` então mostra quais funções específicas consomem mais tempo. Para performance no navegador, o painel Performance do Chrome DevTools mostra a call stack completa durante interações, tornando visíveis as long tasks e suas causas raiz. O fluxo universal é: reproduza o problema sob carga, faça profiling para encontrar o hot path, otimize o gargalo específico, verifique a melhoria com profiling e repita.
