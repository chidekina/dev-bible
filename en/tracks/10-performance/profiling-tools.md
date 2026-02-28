# Profiling Tools

## Overview

Profiling is the practice of measuring where time and memory are spent in a running application. You cannot optimize effectively without profiling — optimization without measurement is guesswork. This chapter covers the primary profiling tools for Node.js (clinic.js, --prof, 0x flame graphs) and browser JavaScript (Chrome DevTools Performance panel), with practical workflows for identifying and resolving real bottlenecks.

---

## Prerequisites

- Node.js fundamentals
- Basic understanding of call stacks
- Track 10: Web Performance Metrics (browser-side profiling)

---

## Hands-On Examples

### Chrome DevTools — browser profiling

**Recording a performance trace:**

1. Open DevTools → Performance tab
2. Click the record button
3. Interact with the page (scroll, click, navigate)
4. Stop recording
5. Analyze the flame chart

**Reading the flame chart:**

```
Main thread activity:
┌─────────────────────────────────────────────────────┐
│ Task (38ms — long task)                             │
│  ├─ Script evaluation                               │
│  │   ├─ handleClick (10ms)                          │
│  │   │   ├─ processData (8ms) ← HOT FUNCTION        │
│  │   │   │   ├─ sortArray (6ms)                     │
│  │   │   │   └─ filterItems (2ms)                   │
└─────────────────────────────────────────────────────┘
```

Wide boxes in the flame chart = time spent there. Click to see the function name, file, and line number.

**Identifying long tasks:**

Long tasks (> 50ms) are highlighted in red in the timeline. They block user input. Each long task is a candidate for optimization.

```typescript
// Use the User Timing API to mark custom sections in the DevTools trace
performance.mark('processData:start');
const result = processData(largeDataset);
performance.mark('processData:end');
performance.measure('processData', 'processData:start', 'processData:end');

// Now "processData" appears as a labeled section in the Performance panel
```

### Node.js — built-in profiler

```bash
# Run with V8 profiler
node --prof src/index.js

# Load the server and generate some traffic
k6 run tests/load/api.js --duration 30s

# Stop the server (Ctrl+C)
# Process the generated isolate-*.log file
node --prof-process isolate-0x*.log > profile.txt

# Read the profile
cat profile.txt | head -100
```

Sample output:

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

The function at the top consuming the most ticks is the hot path to optimize.

### clinic.js — friendly Node.js profiling

Clinic.js provides three tools: `doctor` (diagnoses issues), `flame` (flame graph), `bubbleprof` (async operations):

```bash
npm install -g clinic autocannon

# Doctor — diagnoses common issues (CPU, event loop delay, memory)
clinic doctor -- node src/index.js
# Run traffic in another terminal:
autocannon -c 100 -d 20 http://localhost:3000/api/products
# Ctrl+C the server, clinic opens the report automatically

# Flame graph — identifies hot functions
clinic flame -- node src/index.js
autocannon -c 100 -d 20 http://localhost:3000/api/products
# Ctrl+C — opens interactive flame graph in browser
```

**Reading a clinic flame graph:**

- Wide horizontal bars = more time spent
- Red/orange colors = hot paths (optimization candidates)
- Click a bar to zoom in on that function's call stack

### 0x — production flame graphs

0x is lighter-weight than clinic and suitable for capturing profiles in production:

```bash
npm install -g 0x

# Start the server with profiling
0x -o flame.html -- node src/index.js

# Generate load
autocannon http://localhost:3000/api/products -d 30

# Ctrl+C — 0x writes flame.html
open flame.html
```

### Identifying memory leaks

```bash
# Heap snapshot — shows what's in memory at a point in time
# In Chrome DevTools → Memory → Take heap snapshot

# Or programmatically
import v8 from 'v8';
import fs from 'fs';

function takeHeapSnapshot(label: string) {
  const snapshot = v8.writeHeapSnapshot();
  console.log(`Heap snapshot saved: ${snapshot}`);
}

// Take snapshot at startup and after sustained load
// Compare with Chrome DevTools → Memory → Load snapshot
```

**Detecting leaks with `--expose-gc` and monitoring:**

```typescript
// src/lib/memory-monitor.ts
export function startMemoryMonitor(intervalMs = 30000) {
  setInterval(() => {
    const usage = process.memoryUsage();
    logger.info({
      rss: Math.round(usage.rss / 1024 / 1024) + 'MB',       // resident set size
      heap: Math.round(usage.heapUsed / 1024 / 1024) + 'MB',  // JS heap
      external: Math.round(usage.external / 1024 / 1024) + 'MB', // C++ objects
    }, 'Memory usage');
  }, intervalMs);
}
```

If heap usage grows steadily without returning to baseline, there is a memory leak. Common causes:
- Event listeners added but never removed
- Closures holding references to large objects
- Caches without size limits or TTLs
- Global arrays/maps that grow without bound

### Event loop lag monitoring

```typescript
// Measure how delayed the event loop is (high lag = something is blocking it)
export function monitorEventLoopLag() {
  let lastTick = process.hrtime.bigint();

  const check = () => {
    const now = process.hrtime.bigint();
    const lag = Number(now - lastTick - BigInt(10_000_000)) / 1_000_000; // lag in ms (expected ~10ms)
    lastTick = now;

    if (lag > 50) {
      logger.warn({ lag: `${lag.toFixed(1)}ms` }, 'High event loop lag detected');
    }

    setTimeout(check, 10); // check every 10ms
  };

  setTimeout(check, 10);
}

// Or use the built-in
const { monitorEventLoopDelay } = require('perf_hooks');
const h = monitorEventLoopDelay({ resolution: 10 });
h.enable();
setInterval(() => {
  console.log(`Event loop P99 lag: ${h.percentile(99) / 1_000_000}ms`);
  h.reset();
}, 5000);
```

### Profiling in production safely

```typescript
// Conditional profiling — only when a debug header is present
fastify.addHook('onRequest', async (request, reply) => {
  if (request.headers['x-profile'] === process.env.PROFILING_SECRET) {
    // Start a 10-second profile and dump to file
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

## Common Patterns & Best Practices

- **Profile before optimizing** — identify the actual bottleneck; do not guess
- **Profile under realistic load** — run with `autocannon` or `k6` while profiling to simulate production traffic
- **Use clinic.js doctor first** — it identifies the category of problem (CPU, I/O, event loop) before you dig deeper
- **Monitor event loop lag in production** — expose it as a metric; spikes indicate blocking operations
- **Track heap trend, not absolute size** — a growing heap that never stabilizes is a leak

---

## Anti-Patterns to Avoid

- Optimizing without measuring first — you will optimize the wrong thing
- Profiling with a single request — you need sustained load to reveal bottlenecks
- Ignoring I/O waits — your API may appear CPU-bound when the real issue is a slow database query
- Premature optimization — add profiling only after you observe a problem in production

---

## Debugging & Troubleshooting

**"Flame graph shows time in 'unknown' or native code"**
This is V8 internal operations (GC, compiler). If GC takes a large percentage, look at heap allocation rates — something is creating many short-lived objects.

**"Event loop lag spikes every 30 minutes"**
Something is running on a schedule (cron job, garbage collection pressure from a scheduled batch). Check what's running at that interval.

**"Memory grows during the soak test but the flame graph looks normal"**
The leak is not in CPU-intensive code — it is a slow accumulation. Use heap snapshots 10 minutes apart and compare using Chrome DevTools "comparison" view to find what objects are accumulating.

---

## Further Reading

- [clinic.js documentation](https://clinicjs.org/documentation/)
- [Chrome DevTools: Record runtime performance](https://developer.chrome.com/docs/devtools/performance/)
- [0x — Flamegraphs for Node.js](https://github.com/davidmarkclements/0x)
- [Node.js diagnostics guide](https://nodejs.org/en/docs/guides/diagnostics/)
- Track 10: [Web Performance Metrics](web-performance-metrics.md)

---

## Summary

Profiling is the foundation of performance optimization. For Node.js servers, clinic.js is the entry point — its `doctor` mode diagnoses whether the bottleneck is CPU, event loop delay, or I/O. The `flame` mode then shows which specific functions consume the most time. For browser performance, Chrome DevTools Performance panel shows the full call stack during interactions, making long tasks and their root causes visible. The universal workflow is: reproduce the problem under load, profile to find the hot path, optimize the specific bottleneck, verify the improvement with profiling, and repeat.
