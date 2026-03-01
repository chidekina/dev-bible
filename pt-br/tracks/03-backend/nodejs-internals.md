# Internals do Node.js

## 1. O Que É e Por Que Importa

Node.js é um runtime JavaScript construído sobre o motor V8 do Chrome com um modelo de I/O orientado a eventos e não-bloqueante, impulsionado pela libuv. Entender seus internals não é exercício acadêmico — é a diferença entre escrever um servidor que lida com 50.000 conexões simultâneas graciosamente e um que trava sob carga.

O modelo mental que a maioria dos desenvolvedores carrega ("Node.js é single-threaded") é simultaneamente correto e perigosamente incompleto. Node.js executa JavaScript em uma única thread, mas delega I/O ao sistema operacional e trabalho intensivo de CPU a um pool de threads. O event loop é o orquestrador que faz a ponte entre o mundo JavaScript e o mundo externo.

Este arquivo cobre cada camada: V8 compila e otimiza seu código, libuv dirige I/O assíncrono, o event loop despacha callbacks em uma ordem específica, e APIs como streams, buffers e worker threads permitem trabalhar com dados binários e paralelismo eficientemente.

---

## 2. Conceitos Fundamentais

### Motor V8

V8 é o motor JavaScript open-source do Google, escrito em C++. Ele compila JavaScript diretamente para código de máquina nativo em vez de interpretá-lo.

**Pipeline de compilação JIT:**

1. **Parser** — tokeniza o source, constrói a AST (Abstract Syntax Tree).
2. **Ignition (interpretador)** — percorre a AST e emite bytecode. A execução começa aqui imediatamente.
3. **Sparkplug (JIT baseline)** — compila bytecode para código de máquina não otimizado sem muita análise; muito rápido para compilar.
4. **Maglev (JIT mid-tier)** — compilador com especialização de tipos (adicionado no Node 20+).
5. **TurboFan (JIT otimizador)** — código nativo fortemente otimizado usando feedback de tipos coletado em runtime. Apenas funções "quentes" são promovidas aqui.

Se as suposições de tipo do V8 são violadas ("desotimização"), a função recua para bytecode. Isso se chama **bailout**.

**Hidden Classes (Shapes)**

V8 rastreia formatos de objetos para evitar lookup dinâmico de propriedades:

```typescript
// Bom — V8 cria uma hidden class para ambos os objetos
function Point(x: number, y: number) {
  this.x = x;
  this.y = y;
}
const p1 = new Point(1, 2);
const p2 = new Point(3, 4); // mesma hidden class — acesso rápido às propriedades

// Ruim — adicionar propriedades em ordem diferente cria hidden classes diferentes
const a: any = {};
a.x = 1; a.y = 2; // Shape: {x, y}

const b: any = {};
b.y = 2; b.x = 1; // Shape: {y, x} — classe diferente, desotimiza inline caches
```

> Nunca adicione propriedades a objetos após a construção em código crítico de performance. Sempre inicialize todas as propriedades no construtor na mesma ordem.

**Garbage Collection**

V8 usa um **garbage collector geracional**:

- **Geração jovem (Nursery):** Novos objetos são alocados aqui. GC é frequente mas rápido (minor GC / Scavenge). A maioria dos objetos morre jovem — é a hipótese de "mortalidade infantil".
- **Geração velha:** Objetos que sobrevivem a dois minor GCs são promovidos. GC é menos frequente mas mais caro (major GC).

O major GC usa **mark-and-sweep tricolor + marcação incremental**:
1. Marca todos os objetos vivos (alcançáveis a partir de GC roots — variáveis globais, stack frames).
2. Varre objetos não alcançáveis, recuperando memória.
3. Compacta opcionalmente para reduzir fragmentação e melhorar localidade de cache.

V8 executa fases do GC em paralelo e concorrentemente (em threads de background) para reduzir pausas stop-the-world.

---

### libuv

libuv é uma biblioteca C que fornece ao Node.js seu event loop e I/O assíncrono. Ela abstrai diferenças do nível do SO: epoll no Linux, kqueue no macOS/BSD, IOCP no Windows.

**O que a libuv fornece:**
- Sockets TCP/UDP não-bloqueantes (usando I/O assíncrono do SO — sem pool de threads necessário)
- Operações de sistema de arquivos (pool de threads — I/O de arquivo NÃO é verdadeiramente assíncrono no nível do SO na maioria dos sistemas)
- Resolução de DNS (pool de threads para `dns.lookup`; `dns.resolve` usa a biblioteca assíncrona c-ares)
- Processos filhos
- Timers (agendamento de setTimeout/setInterval)
- Pool de threads para trabalho CPU-bound

**Pool de Threads**

O pool de threads da libuv tem **4 threads por padrão**. Ele cuida de:
- I/O de sistema de arquivos (`fs.readFile`, `fs.writeFile`, `fs.stat`, etc.)
- Lookups DNS (`dns.lookup`)
- Operações crypto (`crypto.pbkdf2`, `crypto.scrypt`, `crypto.randomBytes` em algumas plataformas)
- Compressão `zlib`

Aumente com a variável de ambiente `UV_THREADPOOL_SIZE` (máximo 1024):

```bash
UV_THREADPOOL_SIZE=16 node server.js
```

> Se todas as 4 threads do pool estiverem ocupadas com operações longas (ex: 4 operações `bcrypt` hash simultâneas), uma 5ª operação entra em fila e aguarda. Durante um pico de login, seu servidor pode parecer congelado mesmo que o event loop esteja ocioso. Solução: aumente `UV_THREADPOOL_SIZE` ou use alternativas nativas assíncronas.

---

## 3. Como Funciona — O Event Loop

O event loop é o coração do Node.js. Ele roda continuamente enquanto houver tarefas pendentes (timers, I/O, etc.).

### Fases (em ordem)

```
   ┌────────────────────────────────────────────────────┐
   │                     timers                          │
   │   Roda callbacks de setTimeout() e setInterval()   │
   │   cujos limiares de delay expiraram.               │
   ├────────────────────────────────────────────────────┤
   │                pending callbacks                    │
   │   Roda callbacks de I/O adiados da iteração        │
   │   anterior (ex: erros TCP do SO).                  │
   ├────────────────────────────────────────────────────┤
   │                  idle, prepare                      │
   │   Uso interno apenas.                              │
   ├────────────────────────────────────────────────────┤
   │                      poll                           │
   │   Recupera novos eventos de I/O; executa           │
   │   callbacks de I/O. Bloqueia quando fila vazia.    │
   ├────────────────────────────────────────────────────┤
   │                      check                          │
   │   Callbacks de setImmediate() rodam aqui.          │
   ├────────────────────────────────────────────────────┤
   │                  close callbacks                    │
   │   socket.on('close', ...) e similares.             │
   └────────────────────────────────────────────────────┘

   Entre TODA transição de fase:
     1. Drena a fila process.nextTick (completamente)
     2. Drena a fila de microtasks Promise (completamente)
```

**Fase timers:** Roda callbacks para `setTimeout` e `setInterval` cujos limiares expiraram. Node.js impõe um mínimo de 1ms — `setTimeout(fn, 0)` é na verdade `setTimeout(fn, 1)`.

**Fase poll:** O event loop passa a maior parte do tempo ocioso aqui. Ela:
1. Calcula quanto tempo bloquear aguardando eventos de I/O (limitado pelo próximo timer).
2. Processa callbacks para I/O que completou (leituras de rede, leituras de arquivo, etc.).

Se `setImmediate` está agendado e a fila poll esvazia, o event loop move imediatamente para a fase check em vez de bloquear.

**Fase check:** Callbacks de `setImmediate` rodam aqui — sempre após a fase poll atual, antes dos timers na próxima iteração do loop.

### process.nextTick vs setImmediate vs setTimeout(fn, 0)

```typescript
setTimeout(() => console.log('1: setTimeout'), 0);
setImmediate(() => console.log('2: setImmediate'));
Promise.resolve().then(() => console.log('3: Promise .then'));
process.nextTick(() => console.log('4: nextTick'));

console.log('5: síncrono');

// Output (sempre):
// 5: síncrono              ← roda síncronamente primeiro
// 4: nextTick              ← fila nextTick drenada antes de qualquer transição de fase
// 3: Promise .then         ← fila de microtasks drenada após nextTick
// 1: setTimeout            ← fase timers (pode trocar com setImmediate fora de I/O)
// 2: setImmediate          ← fase check
```

Dentro de um callback de I/O, `setImmediate` **sempre** dispara antes de `setTimeout`:

```typescript
import fs from 'fs';

fs.readFile(__filename, () => {
  // Estamos agora dentro da fase poll
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
  // Output: sempre "immediate" depois "timeout"
  // Porque: poll → check (setImmediate) → close → timers (setTimeout)
});
```

> Use `process.nextTick` para adiar trabalho dentro da operação síncrona atual (ex: emitir um evento após um construtor retornar para que os listeners tenham tempo de se anexar). Use `setImmediate` quando quiser executar após o ciclo de I/O atual. Evite `setTimeout(fn, 0)` para adiamento — tem overhead e ordenação inconsistente em relação ao setImmediate.

> Um `process.nextTick` recursivo (nextTick que agenda outro nextTick) vai privar o I/O indefinidamente — o event loop não pode avançar até a fila nextTick estar vazia.

---

## 4. Exemplos de Código

### Worker Threads para Trabalho Intensivo de CPU

```typescript
// workers/fibonacci.worker.ts
import { workerData, parentPort } from 'worker_threads';

function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

const result = fibonacci(workerData.n as number);
parentPort?.postMessage({ result });
```

```typescript
// main.ts
import { Worker } from 'worker_threads';
import path from 'path';
import { fileURLToPath } from 'url';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

function runFibonacciInWorker(n: number): Promise<number> {
  return new Promise((resolve, reject) => {
    const worker = new Worker(
      path.resolve(__dirname, 'workers/fibonacci.worker.js'),
      { workerData: { n } }
    );

    worker.on('message', ({ result }: { result: number }) => resolve(result));
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker encerrou com código ${code}`));
      }
    });
  });
}

// O event loop permanece livre durante ambas as computações
const [r1, r2] = await Promise.all([
  runFibonacciInWorker(44),
  runFibonacciInWorker(45),
]);
console.log(r1, r2); // roda em 2 núcleos de CPU simultaneamente
```

### SharedArrayBuffer e Atomics

```typescript
// Memória compartilhada entre threads — comunicação sem cópia
const sharedBuffer = new SharedArrayBuffer(4 * Int32Array.BYTES_PER_ELEMENT);
const counter = new Int32Array(sharedBuffer);

// Na thread principal: passa o buffer para o worker
new Worker('./counter-worker.js', {
  workerData: { sharedBuffer }
});

// Na thread worker: incrementa atomicamente (thread-safe)
const counter = new Int32Array(workerData.sharedBuffer);
Atomics.add(counter, 0, 1); // index 0 += 1, atomicamente

// Na thread principal: lê o valor atual
const currentCount = Atomics.load(counter, 0);

// Aguarda mudança de valor (apenas em workers — thread principal não pode bloquear)
Atomics.wait(counter, 0, 0); // bloqueia até counter[0] !== 0
```

### Streams com Backpressure Automático

```typescript
import fs from 'fs';
import { pipeline, Transform, TransformCallback } from 'stream';
import { promisify } from 'util';
import zlib from 'zlib';

const pipelineAsync = promisify(pipeline);

class UpperCaseTransform extends Transform {
  _transform(
    chunk: Buffer,
    _encoding: BufferEncoding,
    callback: TransformCallback
  ): void {
    // Transforma o chunk — backpressure é tratado automaticamente por pipeline()
    callback(null, Buffer.from(chunk.toString().toUpperCase()));
  }
}

async function compressAndUpperCase(src: string, dest: string): Promise<void> {
  await pipelineAsync(
    fs.createReadStream(src),
    new UpperCaseTransform(),    // passo 1: maiúsculas
    zlib.createGzip(),           // passo 2: comprime
    fs.createWriteStream(dest)   // passo 3: escreve
  );
  // pipeline() trata backpressure entre cada par de streams
  // e propaga erros corretamente + limpa todos os streams em caso de falha
}
```

**Tratamento manual de backpressure (para fins didáticos):**

```typescript
const readable = fs.createReadStream('large-file.csv');
const writable = fs.createWriteStream('output.csv');

readable.on('data', (chunk: Buffer) => {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    // Buffer interno do writable está cheio — pausa o produtor
    readable.pause();
    writable.once('drain', () => {
      // Buffer drenado — retoma o produtor
      readable.resume();
    });
  }
});

readable.on('end', () => writable.end());
readable.on('error', (err) => writable.destroy(err));
writable.on('error', (err) => console.error('Erro de escrita:', err));
```

> Sempre prefira `pipeline()` ao `.pipe()`. O método `.pipe()` NÃO propaga erros corretamente — se o readable falha, o writable não é destruído, causando vazamentos de file handle. `pipeline()` cuida de tudo isso corretamente.

### Operações de Buffer

```typescript
// Criando buffers
const buf1 = Buffer.from('Olá, Mundo!', 'utf8');
const buf2 = Buffer.from([0x48, 0x65, 0x6c, 0x6c, 0x6f]); // "Hello" como bytes
const buf3 = Buffer.alloc(16);           // preenchido com zeros, seguro para ler imediatamente
const buf4 = Buffer.allocUnsafe(16);     // memória não inicializada — mais rápido mas preencha antes de ler!

// Conversões de encoding
console.log(buf1.toString('hex'));       // hex da string
console.log(buf1.toString('base64'));    // base64 da string
console.log(buf1.toString('utf8'));      // Olá, Mundo!

// Parsing de protocolo binário
function parseBinaryHeader(buf: Buffer): {
  version: number;
  messageType: number;
  payloadLength: number;
} {
  if (buf.length < 7) throw new Error('Header muito curto');
  return {
    version: buf.readUInt8(0),           // 1 byte
    messageType: buf.readUInt16BE(1),    // 2 bytes, big-endian
    payloadLength: buf.readUInt32LE(3),  // 4 bytes, little-endian
  };
}

// Encode/decode base64 (comum para imagens, tokens de auth)
const encoded = Buffer.from('dados secretos').toString('base64');
const decoded = Buffer.from(encoded, 'base64').toString('utf8');
```

> `Buffer.allocUnsafe()` aloca memória que pode conter dados sensíveis de uma alocação anterior do processo. Sempre inicialize o buffer antes de expô-lo a usuários.

### Sistemas de Módulo: CommonJS vs ESM

```typescript
// === CommonJS (padrão no Node.js sem "type": "module") ===

// require() é síncrono — o código do módulo roda imediatamente
const path = require('path');
const { add, multiply } = require('./math'); // desestrutura de module.exports

// require dinâmico — válido no CJS, útil para carregamento condicional
const config = process.env.NODE_ENV === 'test'
  ? require('./config.test')
  : require('./config.prod');

// Padrões de export
module.exports = { add: (a, b) => a + b };  // exports nomeados via objeto
module.exports = function() {};             // export padrão
exports.add = (a, b) => a + b;              // atalho (mesmo efeito para nomeados)

// __dirname e __filename estão disponíveis
console.log(__dirname, __filename);
```

```typescript
// === ESM (requer "type": "module" no package.json ou extensão .mjs) ===

// Imports estáticos — analisados em parse time (habilita tree-shaking)
import path from 'path';
import { add, multiply } from './math.js'; // extensão .js OBRIGATÓRIA no ESM

// Exports nomeados e padrão
export const add = (a: number, b: number) => a + b;
export default function main() {}

// __dirname e __filename NÃO estão disponíveis — use:
import { fileURLToPath } from 'url';
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// Import dinâmico — funciona em CJS e ESM, retorna uma Promise
const heavyModule = await import('./heavy-processor.js');
```

> Regras de interop ESM e CJS: CJS pode `await import()` um módulo ESM. ESM pode `import` um módulo CJS (obtém seu `module.exports` como export padrão). Porém, um módulo CJS não pode `require()` síncronamente um módulo ESM — você receberá `ERR_REQUIRE_ESM`.

### Processos Filhos: spawn vs exec vs fork

```typescript
import { spawn, exec, fork, execFile } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

// --- spawn: output em streaming ---
// Use quando: processos de longa duração, output grande (ex: tail de log, encoding de vídeo)
const ffmpeg = spawn('ffmpeg', ['-i', 'input.mp4', '-vf', 'scale=720:-1', 'output.mp4']);
ffmpeg.stdout.on('data', (data: Buffer) => process.stdout.write(data));
ffmpeg.stderr.on('data', (data: Buffer) => process.stderr.write(data));
ffmpeg.on('close', (code) => console.log(`FFmpeg encerrou: ${code}`));

// --- exec: output bufferizado ---
// Use quando: comandos shell com output moderado (< ~200KB buffer padrão)
const { stdout } = await execAsync('git log --oneline -10');
console.log(stdout);

// --- fork: processo filho Node.js com IPC ---
// Use quando: spawnar workers Node.js com troca de mensagens, isolar crashes
const worker = fork('./workers/image-processor.js');
worker.send({ type: 'resize', path: '/uploads/photo.jpg', width: 800 });
worker.on('message', (result: { success: boolean; outputPath: string }) => {
  console.log('Imagem processada:', result.outputPath);
});
worker.on('exit', (code) => {
  if (code !== 0) console.error('Worker travou com código', code);
});
```

### Detecção de Memory Leak

```typescript
import { EventEmitter } from 'events';

// Padrão 1: Event listeners nunca removidos (vazamento mais comum)
class DataStream extends EventEmitter {}
const stream = new DataStream();

// VAZAMENTO: cada requisição cria um handler que nunca é limpo
app.get('/subscribe/:id', (req, res) => {
  const handler = (data: unknown) => res.write(JSON.stringify(data));
  stream.on('data', handler); // adiciona listener, mas quando é removido?
});

// CORREÇÃO: remove na desconexão do cliente
app.get('/subscribe/:id', (req, res) => {
  const handler = (data: unknown) => res.write(JSON.stringify(data));
  stream.on('data', handler);
  req.on('close', () => stream.removeListener('data', handler)); // limpeza!
});

// Padrão 2: Cache global sem eviction
const cache = new Map<string, Buffer>();

// VAZAMENTO: cache cresce sem limite
async function processFile(path: string): Promise<Buffer> {
  if (!cache.has(path)) {
    cache.set(path, await fs.promises.readFile(path));
  }
  return cache.get(path)!;
}

// CORREÇÃO: use cache LRU com limite de tamanho
import LRU from 'lru-cache';
const lruCache = new LRU<string, Buffer>({ max: 100 });

// Monitora uso de memória
setInterval(() => {
  const mem = process.memoryUsage();
  if (mem.heapUsed > 500 * 1024 * 1024) { // limiar de 500MB
    console.warn('Alto uso de memória:', {
      heapUsed: `${Math.round(mem.heapUsed / 1024 / 1024)}MB`,
      heapTotal: `${Math.round(mem.heapTotal / 1024 / 1024)}MB`,
      rss: `${Math.round(mem.rss / 1024 / 1024)}MB`,
      external: `${Math.round(mem.external / 1024 / 1024)}MB`,
    });
  }
}, 30_000);
```

**Workflow de heap snapshot:**

```bash
# Inicia com inspetor
node --inspect server.js

# Em outro terminal, dispara um heap snapshot programaticamente
node -e "
const v8 = require('v8');
const fs = require('fs');
const snapshot = v8.writeHeapSnapshot();
console.log('Heap snapshot escrito em', snapshot);
"

# Ou use Chrome DevTools:
# 1. Abra chrome://inspect
# 2. Clique "Open dedicated DevTools for Node"
# 3. Aba Memory → Take snapshot
# 4. Execute a operação que vaza
# 5. Tire outro snapshot
# 6. Use a visualização "Comparison" para encontrar objetos vazados
```

---

## 5. Erros Comuns e Armadilhas

**Bloquear o event loop com operações síncronas:**

```typescript
// NUNCA em servidores em produção
app.get('/data', (req, res) => {
  const data = fs.readFileSync('/var/data/huge-file.json'); // bloqueia TODAS as requisições!
  res.json(JSON.parse(data.toString()));
});

// Correto
app.get('/data', async (req, res) => {
  const data = await fs.promises.readFile('/var/data/huge-file.json');
  res.json(JSON.parse(data.toString()));
});

// Também bloqueante: computação síncrona complexa
app.get('/sort', (req, res) => {
  const arr = new Array(10_000_000).fill(0).map(Math.random);
  arr.sort(); // ~500ms bloqueando — use Worker Thread
  res.json({ sorted: arr.length });
});
```

**Não tratar erros em pipelines:**

```typescript
// Isso trava o processo quando o readable falha
readable.pipe(transform).pipe(writable);

// Isso trata erros em todos os streams da cadeia
import { pipeline } from 'stream/promises';
await pipeline(readable, transform, writable); // lança em qualquer erro
```

**Esquecer que operações de fs competem por slots do pool de threads:**

```typescript
// Cenário: 4 chamadas bcrypt.hash() em andamento preenchem o pool de threads
// Um 5º fs.readFile() agora aguarda — parece travar sem motivo aparente
import bcrypt from 'bcrypt';

// Se você tem requisições de login + operações de arquivo simultâneas, aumente:
// UV_THREADPOOL_SIZE=8 node server.js (ou mais, até CPU count * 2)
```

**JSON.parse em payloads enormes:**

```typescript
// JSON.parse é síncrono e pode bloquear o event loop para payloads grandes
// Um JSON de 10MB pode levar 100ms+ para parsear
app.post('/import', async (req, res) => {
  const body = req.body; // se o body tem 50MB, o parse já bloqueou

  // Correção para JSON grande: use parser em streaming
  // npm install stream-json
  import { parser } from 'stream-json';
  import { streamArray } from 'stream-json/streamers/StreamArray';
  // faz streaming do body pelo parser
});
```

---

## 6. Quando Usar / Não Usar

| Feature | Use Quando | Evite Quando |
|---------|-----------|-------------|
| Worker Threads | CPU-intensivo: resize de imagem, crypto, ML | Trabalho I/O-bound |
| Child Process (fork) | Precisa de processo Node.js isolado + IPC | Precisa de memória compartilhada |
| Child Process (spawn/exec) | Programas não-Node (ffmpeg, python) | Código Node que deve compartilhar memória |
| Streams | Arquivos grandes, bodies HTTP, pipelines em tempo real | Dados pequenos em memória |
| SharedArrayBuffer | Compartilhamento de contador de alta frequência entre workers | Estruturas de dados complexas (use message passing) |
| process.nextTick | Emissão de evento adiado dentro de operação síncrona | Adiamento async geral (use setImmediate) |

---

## 7. Cenário Real

**Problema:** Uma API de relatórios Node.js tem latência p99 de 50ms normalmente mas sobe para 8 segundos aleatoriamente sob carga moderada.

**Passos de investigação:**

```typescript
// Passo 1: Adiciona timing básico para identificar endpoints lentos
app.use((req, res, next) => {
  const start = process.hrtime.bigint();
  res.on('finish', () => {
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    if (ms > 1000) console.warn(`Requisição lenta: ${req.method} ${req.path} ${ms.toFixed(0)}ms`);
  });
  next();
});
// Resultado: /api/reports/export é consistentemente lento

// Passo 2: Profile com --inspect
// Perfil de CPU mostra 95% do tempo em csv-stringify (síncrono)

// Passo 3: A correção — move geração de CSV para um Worker Thread
import { Worker } from 'worker_threads';

async function generateCsv(rows: ReportRow[]): Promise<Buffer> {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./workers/csv-generator.js', {
      workerData: { rows }
    });
    const chunks: Buffer[] = [];
    worker.on('message', (chunk: Buffer) => chunks.push(chunk));
    worker.on('error', reject);
    worker.on('exit', () => resolve(Buffer.concat(chunks)));
  });
}

app.get('/api/reports/export', async (req, res) => {
  const rows = await reportService.fetchRows(req.query);
  const csv = await generateCsv(rows); // event loop permanece livre
  res.setHeader('Content-Type', 'text/csv');
  res.setHeader('Content-Disposition', 'attachment; filename="report.csv"');
  res.send(csv);
});
```

**Resultado:** p99 cai de 8 segundos para 60ms. O event loop nunca é bloqueado — worker threads cuidam de toda a serialização CSV concorrentemente em núcleos de CPU.

---

## 8. Perguntas de Entrevista

**P1: Explique as fases do event loop do Node.js em ordem.**

O event loop percorre seis fases: (1) **Timers** — roda callbacks de `setTimeout`/`setInterval` cujos delays expiraram; (2) **Pending callbacks** — callbacks de I/O adiados da iteração anterior; (3) **Idle/Prepare** — uso interno; (4) **Poll** — recupera eventos de I/O e executa seus callbacks, bloqueia se não houver outro trabalho pendente; (5) **Check** — roda callbacks de `setImmediate`; (6) **Close callbacks** — trata `socket.on('close')` etc. Entre cada fase, Node.js drena a fila `process.nextTick` primeiro, depois a fila de microtasks Promise.

**P2: O que é o pool de threads na libuv, o que roda nele e o que NÃO roda?**

A libuv mantém um pool de 4 worker threads (configurável via `UV_THREADPOOL_SIZE`). Ele cuida de: operações de sistema de arquivos, `dns.lookup` e operações crypto pesadas de CPU (`pbkdf2`, `scrypt`). I/O de rede (TCP, UDP, HTTP) NÃO usa o pool de threads — usa os mecanismos nativos de I/O assíncrono do SO (epoll/kqueue/IOCP) que são verdadeiramente orientados a eventos. É por isso que o Node.js pode lidar com milhares de requisições HTTP simultâneas com threads mínimas.

**P3: Qual é a ordem de execução de process.nextTick, setImmediate e setTimeout(fn, 0)?**

Em um script top-level: código síncrono → fila nextTick → microtasks Promise → timers (setTimeout) → check (setImmediate). Dentro de um callback de I/O: setImmediate sempre dispara antes de setTimeout porque entramos na fase check antes de voltar para os timers. nextTick roda entre cada fase incluindo antes de qualquer callback de I/O, tornando-o o mecanismo de adiamento de maior prioridade.

**P4: Quando você usaria Worker Threads vs child_process.fork?**

Use Worker Threads quando precisar de verdadeiro paralelismo dentro do mesmo processo para trabalho CPU-intensivo (processamento de imagens, crypto, computação), especialmente quando quiser memória compartilhada via `SharedArrayBuffer`. Use `child_process.fork` quando precisar de isolamento a nível de processo (um crash no filho não afeta o pai), ao rodar código não confiável, ou quando precisar de um heap V8 completamente separado. Worker threads compartilham o mesmo espaço de memória do processo; processos fork têm memória separada.

**P5: O que é backpressure em streams Node.js e como é implementado?**

Backpressure é o mecanismo que evita que um produtor rápido sobrecarregue um consumidor lento. Quando o buffer interno de um writable stream fica cheio, `.write()` retorna `false`. Um readable corretamente implementado deve responder chamando `.pause()`, parando a produção de dados. Quando o writable drena seu buffer, emite um evento `drain`, e o readable deve chamar `.resume()`. O utilitário `pipeline()` cuida disso automaticamente, tornando-o a forma recomendada de encadear streams.

**P6: Quais são as principais diferenças entre módulos CommonJS e ESM?**

CommonJS: `require()` síncrono, dinâmico (pode estar dentro de if/else), `module.exports`/`exports`, cacheado após primeiro carregamento, `__dirname`/`__filename` disponíveis, sem top-level await. ESM: `import`/`export` estático (analisado em parse time habilitando tree-shaking e detecção de dependência circular), avaliação de módulo assíncrona, suporta top-level `await`, sem `__dirname` (use `import.meta.url`), extensões de arquivo obrigatórias nos imports. Interop: ESM pode importar CJS (como default); CJS não pode fazer require síncrono de ESM.

**P7: Como você encontra e diagnostica um memory leak em Node.js em produção?**

1. Monitore `process.memoryUsage().heapUsed` — se cresce continuamente sem limite, há um vazamento. 2. Use `--inspect` com Chrome DevTools: tire heap snapshots antes/depois da operação suspeita, compare com a visualização "Comparison" para encontrar objetos retidos. 3. Causas comuns: event listeners não removidos (use `emitter.listenerCount()` para auditar), closures mantendo referências a objetos grandes, Maps/Sets globais crescendo sem limite (eviction ausente), timers que nunca são limpos. 4. Em produção: use `clinic heap` ou `heapdump` para capturar snapshots sem reiniciar.

---

## 9. Exercícios

**Exercício 1: Monitor de lag do event loop**

Implemente uma função `lagMonitor` que:
- Agenda um `setTimeout(fn, 0)` a cada 500ms
- Mede o delay real vs esperado usando `hrtime.bigint()`
- Emite um evento `'lag'` se o delay real exceder o esperado em mais de 100ms
- Escreva um teste que confirma que dispara quando você roda um busy loop síncrono de 200ms

*Dica:* Registre `hrtime.bigint()` antes de agendar, verifique dentro do callback do timeout.

**Exercício 2: Pool de Worker Threads com controle de concorrência**

Construa uma classe `WorkerPool` que:
- Aceita `size` (padrão: `os.cpus().length`) worker threads
- Tem um método `run(workerData: unknown): Promise<unknown>`
- Roteia tarefas para workers ociosos; enfileira tarefas quando todos estão ocupados
- Reutiliza workers entre tarefas (não cria um novo worker por tarefa)
- Emite um evento quando o pool está drenado (todos os workers ociosos, fila vazia)

*Dica:* Use uma fila de tarefas e um conjunto de IDs de workers disponíveis. Quando um worker termina, desenfileira a próxima tarefa.

**Exercício 3: Transform stream de JSON delimitado por newline**

Escreva uma classe `NdjsonTransform` (extends Transform) que:
- Aceita chunks arbitrários (linhas podem ser divididas entre chunks)
- Parseia registros JSON completos delimitados por newline
- Emite objetos parseados downstream (em object mode)
- Trata JSON malformado graciosamente (emite `'error'` com contexto da linha)
- Testa com um arquivo NDJSON de 100MB e verifica que `process.memoryUsage().heapUsed` permanece < 50MB

*Dica:* Bufferiza linhas incompletas em uma variável de instância `string`. Divida em `\n` no `_transform`, libera o restante no `_flush`.

**Exercício 4: Caça ao memory leak**

O seguinte servidor Express tem pelo menos dois memory leaks. Encontre e corrija:

```typescript
const cache = new Map<string, Buffer>();
const emitter = new EventEmitter();

app.get('/process/:id', async (req, res) => {
  const handler = (update: Buffer) => {
    cache.set(req.params.id, update);
  };
  emitter.on('update', handler);
  const result = cache.get(req.params.id) ?? await db.fetch(req.params.id);
  res.json({ size: result.length });
});

emitter.emit('update', Buffer.alloc(1024 * 1024)); // update de 1MB
```

*Dica:* (1) O handler nunca é removido do emitter. (2) O cache não tem eviction. Corrija ambos.

---

## 10. Leituras Complementares

- [Guia oficial do Event Loop do Node.js](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick)
- [Documentação da libuv](https://docs.libuv.org/en/v1.x/)
- [Blog V8 — compilador JIT TurboFan](https://v8.dev/blog/turbofan-jit)
- [Blog V8 — interpretador Ignition](https://v8.dev/blog/ignition-interpreter)
- [API Worker Threads do Node.js](https://nodejs.org/api/worker_threads.html)
- [Guia aprofundado de backpressure em streams](https://nodejs.org/en/docs/guides/backpressuring-in-streams)
- [clinic.js — toolkit de profiling em produção](https://clinicjs.org/)
- [0x — profiler de flame graph](https://github.com/davidmarkclements/0x)
- [Não bloqueie o event loop (docs Node.js)](https://nodejs.org/en/docs/guides/dont-block-the-event-loop)
- Livro: *Node.js Design Patterns* por Mario Casciaro & Luciano Mammino (3ª ed.)
