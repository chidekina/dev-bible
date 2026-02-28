# Node.js Internals

## 1. What & Why

Node.js is a JavaScript runtime built on Chrome's V8 engine with an event-driven, non-blocking I/O model powered by libuv. Understanding its internals is not academic exercise — it is the difference between writing a server that handles 50,000 concurrent connections gracefully and one that grinds to a halt under load.

The mental model most developers carry ("Node.js is single-threaded") is simultaneously correct and dangerously incomplete. Node.js executes JavaScript on a single thread, but it offloads I/O to the OS and CPU-intensive work to a thread pool. The event loop is the orchestrator that bridges JavaScript land and the outside world.

This file covers every layer: V8 compiles and optimizes your code, libuv drives async I/O, the event loop dispatches callbacks in a specific order, and APIs like streams, buffers, and worker threads let you work with binary data and parallelism efficiently.

---

## 2. Core Concepts

### V8 Engine

V8 is Google's open-source JavaScript engine, written in C++. It compiles JavaScript directly to native machine code rather than interpreting it.

**JIT Compilation pipeline:**

1. **Parser** — tokenizes source, builds AST (Abstract Syntax Tree).
2. **Ignition (interpreter)** — walks the AST and emits bytecode. Execution starts here immediately.
3. **Sparkplug (baseline JIT)** — compiles bytecode to unoptimized machine code without much analysis; very fast to compile.
4. **Maglev (mid-tier JIT)** — type-specializing compiler (added in Node 20+).
5. **TurboFan (optimizing JIT)** — heavily optimized native code using type feedback gathered at runtime. Only hot functions are promoted here.

If V8's type assumptions are violated (a "deoptimization"), the function falls back to bytecode. This is called a **bailout**.

**Hidden Classes (Shapes)**

V8 tracks object shapes to avoid dynamic property lookup:

```typescript
// Good — V8 creates one hidden class for both objects
function Point(x: number, y: number) {
  this.x = x;
  this.y = y;
}
const p1 = new Point(1, 2);
const p2 = new Point(3, 4); // same hidden class — fast property access

// Bad — adding properties in different order creates different hidden classes
const a: any = {};
a.x = 1; a.y = 2; // Shape: {x, y}

const b: any = {};
b.y = 2; b.x = 1; // Shape: {y, x} — different class, deoptimizes inline caches
```

> ⚠️ Never add properties to objects after construction in performance-critical code. Always initialize all properties in the constructor in the same order.

**Garbage Collection**

V8 uses a **generational garbage collector**:

- **Young generation (Nursery):** New objects are allocated here. GC is frequent but fast (minor GC / Scavenge). Most objects die young — this is the "infant mortality" hypothesis.
- **Old generation:** Objects that survive two minor GCs are promoted. GC is less frequent but more expensive (major GC).

Major GC uses **tri-color mark-and-sweep + incremental marking**:
1. Mark all live objects (reachable from GC roots — global variables, stack frames).
2. Sweep unreachable objects, reclaiming memory.
3. Compact optionally to reduce fragmentation and improve cache locality.

V8 runs GC phases in parallel and concurrently (on background threads) to reduce stop-the-world pauses.

---

### libuv

libuv is a C library that provides Node.js its event loop and async I/O. It abstracts OS-level differences: epoll on Linux, kqueue on macOS/BSD, IOCP on Windows.

**What libuv provides:**
- Non-blocking TCP/UDP sockets (using OS async I/O — no thread pool needed)
- File system operations (thread pool — file I/O is NOT truly async at OS level on most systems)
- DNS resolution (thread pool for `dns.lookup`; `dns.resolve` uses async c-ares library)
- Child processes
- Timers (setTimeout/setInterval scheduling)
- Thread pool for CPU-bound work

**Thread Pool**

libuv's thread pool has **4 threads by default**. It handles:
- File system I/O (`fs.readFile`, `fs.writeFile`, `fs.stat`, etc.)
- DNS lookups (`dns.lookup`)
- Crypto operations (`crypto.pbkdf2`, `crypto.scrypt`, `crypto.randomBytes` on some platforms)
- `zlib` compression

Increase it with the `UV_THREADPOOL_SIZE` environment variable (max 1024):

```bash
UV_THREADPOOL_SIZE=16 node server.js
```

> ⚠️ If all 4 thread pool threads are busy with long-running operations (e.g., 4 concurrent `bcrypt` hash operations), a 5th concurrent operation queues and waits. Under a login spike, your server can appear to freeze even though the event loop is idle. Solution: increase `UV_THREADPOOL_SIZE` or use async-native alternatives.

---

## 3. How It Works — The Event Loop

The event loop is the heart of Node.js. It runs continuously as long as there are pending tasks (timers, I/O, etc.).

### Phases (in order)

```
   ┌────────────────────────────────────────────────────┐
   │                     timers                          │
   │   Runs setTimeout() and setInterval() callbacks     │
   │   whose delay threshold has elapsed.                │
   ├────────────────────────────────────────────────────┤
   │                pending callbacks                    │
   │   Runs I/O callbacks deferred from previous         │
   │   iteration (e.g., TCP errors from the OS).         │
   ├────────────────────────────────────────────────────┤
   │                  idle, prepare                      │
   │   Internal use only.                               │
   ├────────────────────────────────────────────────────┤
   │                      poll                           │
   │   Retrieve new I/O events; execute I/O callbacks.  │
   │   Blocks here when queue is empty (up to a limit). │
   ├────────────────────────────────────────────────────┤
   │                      check                          │
   │   setImmediate() callbacks run here.               │
   ├────────────────────────────────────────────────────┤
   │                  close callbacks                    │
   │   socket.on('close', ...) and similar.             │
   └────────────────────────────────────────────────────┘

   Between EVERY phase transition:
     1. Drain process.nextTick queue (completely)
     2. Drain Promise microtask queue (completely)
```

**Timers phase:** Runs callbacks for `setTimeout` and `setInterval` whose thresholds have elapsed. Note: Node.js enforces a minimum of 1ms — `setTimeout(fn, 0)` is actually `setTimeout(fn, 1)`.

**Poll phase:** The event loop spends most of its idle time here. It:
1. Calculates how long to block waiting for I/O events (bounded by the next timer).
2. Processes callbacks for I/O that has completed (network reads, file reads, etc.).

If `setImmediate` is scheduled and the poll queue empties, the event loop moves immediately to the check phase rather than blocking.

**Check phase:** `setImmediate` callbacks run here — always after the current poll phase, before timers on the next loop iteration.

### process.nextTick vs setImmediate vs setTimeout(fn, 0)

```typescript
setTimeout(() => console.log('1: setTimeout'), 0);
setImmediate(() => console.log('2: setImmediate'));
Promise.resolve().then(() => console.log('3: Promise .then'));
process.nextTick(() => console.log('4: nextTick'));

console.log('5: synchronous');

// Output (always):
// 5: synchronous           ← runs synchronously first
// 4: nextTick              ← nextTick queue drained before any phase transition
// 3: Promise .then         ← microtask queue drained after nextTick
// 1: setTimeout            ← timers phase (may swap with setImmediate when outside I/O)
// 2: setImmediate          ← check phase
```

Inside an I/O callback, `setImmediate` **always** fires before `setTimeout`:

```typescript
import fs from 'fs';

fs.readFile(__filename, () => {
  // We are now inside the poll phase
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
  // Output: always "immediate" then "timeout"
  // Because: poll → check (setImmediate) → close → timers (setTimeout)
});
```

> 💡 Use `process.nextTick` for deferring work within the current synchronous operation (e.g., emitting an event after a constructor returns so listeners have time to attach). Use `setImmediate` when you want to execute after the current I/O cycle. Avoid `setTimeout(fn, 0)` for deferral — it has overhead and inconsistent ordering relative to setImmediate.

> ⚠️ A recursive `process.nextTick` (nextTick that schedules another nextTick) will starve I/O indefinitely — the event loop cannot advance until the nextTick queue is empty.

---

## 4. Code Examples

### Worker Threads for CPU-Intensive Work

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
        reject(new Error(`Worker stopped with exit code ${code}`));
      }
    });
  });
}

// Event loop remains free during both computations
const [r1, r2] = await Promise.all([
  runFibonacciInWorker(44),
  runFibonacciInWorker(45),
]);
console.log(r1, r2); // runs on 2 CPU cores simultaneously
```

### SharedArrayBuffer and Atomics

```typescript
// Shared memory between threads — zero-copy communication
const sharedBuffer = new SharedArrayBuffer(4 * Int32Array.BYTES_PER_ELEMENT);
const counter = new Int32Array(sharedBuffer);

// In main thread: pass buffer to worker
new Worker('./counter-worker.js', {
  workerData: { sharedBuffer }
});

// In worker thread: atomically increment (thread-safe)
const counter = new Int32Array(workerData.sharedBuffer);
Atomics.add(counter, 0, 1); // index 0 += 1, atomically

// In main thread: read current value
const currentCount = Atomics.load(counter, 0);

// Wait for a value change (only in workers — main thread cannot block)
Atomics.wait(counter, 0, 0); // block until counter[0] !== 0
```

### Streams with Automatic Backpressure

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
    // Transform the chunk — backpressure is handled automatically by pipeline()
    callback(null, Buffer.from(chunk.toString().toUpperCase()));
  }
}

async function compressAndUpperCase(src: string, dest: string): Promise<void> {
  await pipelineAsync(
    fs.createReadStream(src),
    new UpperCaseTransform(),    // step 1: uppercase
    zlib.createGzip(),           // step 2: compress
    fs.createWriteStream(dest)   // step 3: write
  );
  // pipeline() handles backpressure between every pair of streams
  // and properly propagates errors + cleans up all streams on failure
}
```

**Manual backpressure handling (for educational purposes):**

```typescript
const readable = fs.createReadStream('large-file.csv');
const writable = fs.createWriteStream('output.csv');

readable.on('data', (chunk: Buffer) => {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    // Writable internal buffer is full — pause the producer
    readable.pause();
    writable.once('drain', () => {
      // Buffer has drained — resume producer
      readable.resume();
    });
  }
});

readable.on('end', () => writable.end());
readable.on('error', (err) => writable.destroy(err));
writable.on('error', (err) => console.error('Write error:', err));
```

> 💡 Always prefer `pipeline()` over `.pipe()`. The `.pipe()` method does NOT properly propagate errors — if the readable errors, the writable is not destroyed, causing file handle leaks. `pipeline()` handles all of this correctly.

### Buffer Operations

```typescript
// Creating buffers
const buf1 = Buffer.from('Hello, World!', 'utf8');
const buf2 = Buffer.from([0x48, 0x65, 0x6c, 0x6c, 0x6f]); // "Hello" as bytes
const buf3 = Buffer.alloc(16);           // zero-filled, safe to read immediately
const buf4 = Buffer.allocUnsafe(16);     // uninitialized memory — faster but fill before reading!

// Encoding conversions
console.log(buf1.toString('hex'));       // 48656c6c6f2c20576f726c6421
console.log(buf1.toString('base64'));    // SGVsbG8sIFdvcmxkIQ==
console.log(buf1.toString('utf8'));      // Hello, World!

// Binary protocol parsing
function parseBinaryHeader(buf: Buffer): {
  version: number;
  messageType: number;
  payloadLength: number;
} {
  if (buf.length < 7) throw new Error('Header too short');
  return {
    version: buf.readUInt8(0),           // 1 byte
    messageType: buf.readUInt16BE(1),    // 2 bytes, big-endian
    payloadLength: buf.readUInt32LE(3),  // 4 bytes, little-endian
  };
}

// Base64 encode/decode (common for images, auth tokens)
const encoded = Buffer.from('secret data').toString('base64');
const decoded = Buffer.from(encoded, 'base64').toString('utf8');
```

> ⚠️ `Buffer.allocUnsafe()` allocates memory that may contain sensitive data from a previous process allocation. Always initialize the buffer before exposing it to users.

### Module Systems: CommonJS vs ESM

```typescript
// === CommonJS (default in Node.js without "type": "module") ===

// require() is synchronous — module code runs immediately
const path = require('path');
const { add, multiply } = require('./math'); // destructure from module.exports

// Dynamic require — valid in CJS, useful for conditional loading
const config = process.env.NODE_ENV === 'test'
  ? require('./config.test')
  : require('./config.prod');

// Export patterns
module.exports = { add: (a, b) => a + b };  // named exports via object
module.exports = function() {};             // default export
exports.add = (a, b) => a + b;              // shorthand (same as above for named)

// __dirname and __filename are available
console.log(__dirname, __filename);
```

```typescript
// === ESM (requires "type": "module" in package.json or .mjs extension) ===

// Static imports — analyzed at parse time (enables tree-shaking)
import path from 'path';
import { add, multiply } from './math.js'; // .js extension REQUIRED in ESM

// Named and default exports
export const add = (a: number, b: number) => a + b;
export default function main() {}

// __dirname and __filename are NOT available — use:
import { fileURLToPath } from 'url';
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// Dynamic import — works in both CJS and ESM, returns a Promise
const heavyModule = await import('./heavy-processor.js');
```

> ⚠️ ESM and CJS interop rules: CJS can `await import()` an ESM module. ESM can `import` a CJS module (gets its `module.exports` as the default export). However, a CJS module cannot synchronously `require()` an ESM module — you'll get `ERR_REQUIRE_ESM`.

### Child Processes: spawn vs exec vs fork

```typescript
import { spawn, exec, fork, execFile } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

// --- spawn: streaming output ---
// Use when: long-running processes, large output (e.g., log tailing, video encoding)
const ffmpeg = spawn('ffmpeg', ['-i', 'input.mp4', '-vf', 'scale=720:-1', 'output.mp4']);
ffmpeg.stdout.on('data', (data: Buffer) => process.stdout.write(data));
ffmpeg.stderr.on('data', (data: Buffer) => process.stderr.write(data));
ffmpeg.on('close', (code) => console.log(`FFmpeg exited: ${code}`));

// --- exec: buffered output ---
// Use when: shell commands with modest output (< ~200KB default buffer)
const { stdout } = await execAsync('git log --oneline -10');
console.log(stdout);

// --- fork: Node.js child process with IPC ---
// Use when: spawning Node.js workers with message passing, isolating crashes
const worker = fork('./workers/image-processor.js');
worker.send({ type: 'resize', path: '/uploads/photo.jpg', width: 800 });
worker.on('message', (result: { success: boolean; outputPath: string }) => {
  console.log('Image processed:', result.outputPath);
});
worker.on('exit', (code) => {
  if (code !== 0) console.error('Worker crashed with code', code);
});
```

### Memory Leak Detection

```typescript
import { EventEmitter } from 'events';

// Pattern 1: Event listeners never removed (most common leak)
class DataStream extends EventEmitter {}
const stream = new DataStream();

// LEAK: each request creates a handler that is never cleaned up
app.get('/subscribe/:id', (req, res) => {
  const handler = (data: unknown) => res.write(JSON.stringify(data));
  stream.on('data', handler); // adds listener, but when does it get removed?
});

// FIX: remove on client disconnect
app.get('/subscribe/:id', (req, res) => {
  const handler = (data: unknown) => res.write(JSON.stringify(data));
  stream.on('data', handler);
  req.on('close', () => stream.removeListener('data', handler)); // cleanup!
});

// Pattern 2: Global cache without eviction
const cache = new Map<string, Buffer>();

// LEAK: cache grows unbounded
async function processFile(path: string): Promise<Buffer> {
  if (!cache.has(path)) {
    cache.set(path, await fs.promises.readFile(path));
  }
  return cache.get(path)!;
}

// FIX: use LRU cache with size limit
import LRU from 'lru-cache';
const lruCache = new LRU<string, Buffer>({ max: 100 });

// Monitor memory usage
setInterval(() => {
  const mem = process.memoryUsage();
  if (mem.heapUsed > 500 * 1024 * 1024) { // 500MB threshold
    console.warn('High memory usage:', {
      heapUsed: `${Math.round(mem.heapUsed / 1024 / 1024)}MB`,
      heapTotal: `${Math.round(mem.heapTotal / 1024 / 1024)}MB`,
      rss: `${Math.round(mem.rss / 1024 / 1024)}MB`,
      external: `${Math.round(mem.external / 1024 / 1024)}MB`,
    });
  }
}, 30_000);
```

**Heap snapshot workflow:**

```bash
# Start with inspector
node --inspect server.js

# In another terminal, trigger a heap snapshot programmatically
node -e "
const v8 = require('v8');
const fs = require('fs');
const snapshot = v8.writeHeapSnapshot();
console.log('Heap snapshot written to', snapshot);
"

# Or use Chrome DevTools:
# 1. Open chrome://inspect
# 2. Click "Open dedicated DevTools for Node"
# 3. Memory tab → Take snapshot
# 4. Perform operation that leaks
# 5. Take another snapshot
# 6. Use "Comparison" view to find leaked objects
```

---

## 5. Common Mistakes & Pitfalls

**Blocking the event loop with synchronous operations:**

```typescript
// NEVER in production servers
app.get('/data', (req, res) => {
  const data = fs.readFileSync('/var/data/huge-file.json'); // blocks ALL requests!
  res.json(JSON.parse(data.toString()));
});

// Correct
app.get('/data', async (req, res) => {
  const data = await fs.promises.readFile('/var/data/huge-file.json');
  res.json(JSON.parse(data.toString()));
});

// Also blocking: complex synchronous computation
app.get('/sort', (req, res) => {
  const arr = new Array(10_000_000).fill(0).map(Math.random);
  arr.sort(); // ~500ms blocking — use Worker Thread instead
  res.json({ sorted: arr.length });
});
```

**Not handling errors in pipelines:**

```typescript
// This crashes the process when readable errors
readable.pipe(transform).pipe(writable);

// This handles errors on every stream in the chain
import { pipeline } from 'stream/promises';
await pipeline(readable, transform, writable); // throws on any error
```

**Forgetting that fs operations compete for thread pool slots:**

```typescript
// Scenario: 4 ongoing bcrypt.hash() calls fill the thread pool
// A 5th fs.readFile() now waits — appears to hang for no reason
import bcrypt from 'bcrypt';

// If you have concurrent login requests + file operations, increase:
// UV_THREADPOOL_SIZE=8 node server.js (or more, up to CPU count * 2)
```

**JSON.parse on huge payloads:**

```typescript
// JSON.parse is synchronous and can block event loop for large payloads
// A 10MB JSON might take 100ms+ to parse
app.post('/import', async (req, res) => {
  const body = req.body; // if body is 50MB, parse already blocked

  // Fix for large JSON: use streaming parser
  // npm install stream-json
  import { parser } from 'stream-json';
  import { streamArray } from 'stream-json/streamers/StreamArray';
  // stream the body through the parser
});
```

---

## 6. When to Use / Not Use

| Feature | Use When | Avoid When |
|---------|----------|------------|
| Worker Threads | CPU-intensive: image resize, crypto, ML | I/O-bound work |
| Child Process (fork) | Need isolated Node process + IPC | Need shared memory |
| Child Process (spawn/exec) | Non-Node programs (ffmpeg, python) | Node code that should share memory |
| Streams | Large files, HTTP bodies, real-time pipelines | Small in-memory data |
| SharedArrayBuffer | High-frequency counter sharing between workers | Complex data structures (use message passing) |
| process.nextTick | Deferred event emission within sync operation | General async deferral (use setImmediate) |

---

## 7. Real-World Scenario

**Problem:** A Node.js report API has p99 latency of 50ms normally but spikes to 8 seconds randomly under moderate load.

**Investigation steps:**

```typescript
// Step 1: Add basic timing to identify slow endpoints
app.use((req, res, next) => {
  const start = process.hrtime.bigint();
  res.on('finish', () => {
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    if (ms > 1000) console.warn(`Slow request: ${req.method} ${req.path} ${ms.toFixed(0)}ms`);
  });
  next();
});
// Result: /api/reports/export is consistently slow

// Step 2: Profile with --inspect
// CPU profile shows 95% of time in csv-stringify (synchronous)

// Step 3: The fix — move CSV generation to a Worker Thread
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
  const csv = await generateCsv(rows); // event loop stays free
  res.setHeader('Content-Type', 'text/csv');
  res.setHeader('Content-Disposition', 'attachment; filename="report.csv"');
  res.send(csv);
});
```

**Result:** p99 drops from 8 seconds to 60ms. The event loop is never blocked — worker threads handle all CSV serialization concurrently across CPU cores.

---

## 8. Interview Questions

**Q1: Explain the Node.js event loop phases in order.**

The event loop cycles through six phases: (1) **Timers** — runs `setTimeout`/`setInterval` callbacks whose delay has elapsed; (2) **Pending callbacks** — I/O callbacks deferred from the previous iteration; (3) **Idle/Prepare** — internal use; (4) **Poll** — retrieves I/O events and executes their callbacks, blocks if no other work is pending; (5) **Check** — runs `setImmediate` callbacks; (6) **Close callbacks** — handles `socket.on('close')` etc. Between every phase, Node.js drains the `process.nextTick` queue first, then the Promise microtask queue.

**Q2: What is the thread pool in libuv, what runs on it, and what does NOT?**

libuv maintains a pool of 4 worker threads (configurable via `UV_THREADPOOL_SIZE`). It handles: file system operations, `dns.lookup`, and CPU-heavy crypto operations (`pbkdf2`, `scrypt`). Network I/O (TCP, UDP, HTTP) does NOT use the thread pool — it uses the OS's native async I/O mechanisms (epoll/kqueue/IOCP) which are truly event-driven. This is why Node.js can handle thousands of concurrent HTTP requests with minimal threads.

**Q3: What is the execution order of process.nextTick, setImmediate, and setTimeout(fn, 0)?**

In a top-level script: synchronous code → nextTick queue → Promise microtasks → timers (setTimeout) → check (setImmediate). Inside an I/O callback: setImmediate always fires before setTimeout because we enter the check phase before looping back to timers. nextTick runs between every phase including before any I/O callbacks, making it the highest-priority deferral mechanism.

**Q4: When should you use Worker Threads vs child_process.fork?**

Use Worker Threads when you need true parallelism within the same process for CPU-intensive work (image processing, crypto, computation), especially when you want shared memory via `SharedArrayBuffer`. Use `child_process.fork` when you need process-level isolation (a crash in the child does not affect the parent), when running untrusted code, or when you need a completely separate V8 heap. Worker threads share the same process memory space; forked processes have separate memory.

**Q5: What is backpressure in Node.js streams and how is it implemented?**

Backpressure is the mechanism that prevents a fast producer from overwhelming a slow consumer. When a writable stream's internal buffer fills up, `.write()` returns `false`. A correctly implemented readable should respond by calling `.pause()`, stopping data production. When the writable drains its buffer, it emits a `drain` event, and the readable should call `.resume()`. The `pipeline()` utility handles this automatically, making it the recommended way to chain streams.

**Q6: What are the key differences between CommonJS and ESM modules?**

CommonJS: synchronous `require()`, dynamic (can be inside if/else), `module.exports`/`exports`, cached after first load, `__dirname`/`__filename` available, no top-level await. ESM: static `import`/`export` (analyzed at parse time enabling tree-shaking and circular dependency detection), async module evaluation, supports top-level `await`, no `__dirname` (use `import.meta.url`), file extensions required in imports. Interop: ESM can import CJS (as default); CJS cannot synchronously require ESM.

**Q7: How do you find and diagnose a memory leak in production Node.js?**

1. Monitor `process.memoryUsage().heapUsed` — if it grows continuously without bound, there is a leak. 2. Use `--inspect` with Chrome DevTools: take heap snapshots before/after suspected operation, compare with "Comparison" view to find retained objects. 3. Common causes: event listeners not removed (use `emitter.listenerCount()` to audit), closures keeping references to large objects, global Maps/Sets that grow unbounded (missing eviction), timers that are never cleared. 4. In production: use `clinic heap` or `heapdump` to capture snapshots without restarting.

**Q8: What does --inspect do and how would you use it to debug a production issue?**

`--inspect` opens a V8 debugging WebSocket (default port 9229) that Chrome DevTools or VS Code can connect to. With it you can: set breakpoints, step through async code, inspect heap snapshots for memory leaks, record CPU profiles to identify hot functions, view the async call stack. For production: use `--inspect` on a staging replica with production-like load. Use `kill -USR1 <pid>` to enable the inspector on a running process without restarting. `--inspect-brk` pauses execution on the first line, useful for debugging startup issues.

---

## 9. Exercises

**Exercise 1: Event loop lag monitor**

Implement a `lagMonitor` function that:
- Schedules a `setTimeout(fn, 0)` every 500ms
- Measures the actual delay vs expected delay using `hrtime.bigint()`
- Emits a `'lag'` event if actual delay exceeds expected by more than 100ms
- Write a test that confirms it fires when you run a 200ms synchronous busy loop

*Hint:* Record `hrtime.bigint()` before scheduling, check it inside the timeout callback.

**Exercise 2: Worker Thread pool with concurrency control**

Build a `WorkerPool` class that:
- Accepts `size` (default: `os.cpus().length`) worker threads
- Has a `run(workerData: unknown): Promise<unknown>` method
- Routes tasks to idle workers; queues tasks when all workers are busy
- Reuses workers across tasks (do not spawn a new worker per task)
- Emits an event when the pool is drained (all workers idle, queue empty)

*Hint:* Use a task queue and a set of available worker IDs. When a worker finishes, dequeue the next task.

**Exercise 3: Newline-delimited JSON transform stream**

Write a `NdjsonTransform` class (extends Transform) that:
- Accepts arbitrary chunks (lines may be split across chunks)
- Parses complete newline-delimited JSON records
- Emits parsed objects downstream (in object mode)
- Handles malformed JSON gracefully (emit `'error'` with line context)
- Test with a 100MB NDJSON file and verify `process.memoryUsage().heapUsed` stays < 50MB throughout

*Hint:* Buffer incomplete lines in a `string` instance variable. Split on `\n` in `_transform`, flush remainder in `_flush`.

**Exercise 4: Memory leak hunt**

The following Express server has at least two memory leaks. Find and fix them:

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

emitter.emit('update', Buffer.alloc(1024 * 1024)); // 1MB update
```

*Hint:* (1) The handler is never removed from the emitter. (2) The cache has no eviction. Fix both.

---

## 10. Further Reading

- [Node.js Event Loop official guide](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick)
- [libuv documentation](https://docs.libuv.org/en/v1.x/)
- [V8 blog — TurboFan JIT compiler](https://v8.dev/blog/turbofan-jit)
- [V8 blog — Ignition interpreter](https://v8.dev/blog/ignition-interpreter)
- [Node.js Worker Threads API](https://nodejs.org/api/worker_threads.html)
- [Stream backpressure in-depth guide](https://nodejs.org/en/docs/guides/backpressuring-in-streams)
- [clinic.js — production profiling toolkit](https://clinicjs.org/)
- [0x — flame graph profiler](https://github.com/davidmarkclements/0x)
- [Don't block the event loop (Node.js docs)](https://nodejs.org/en/docs/guides/dont-block-the-event-loop)
- Book: *Node.js Design Patterns* by Mario Casciaro & Luciano Mammino (3rd ed.)
