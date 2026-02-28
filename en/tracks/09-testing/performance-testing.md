# Performance Testing

## Overview

Performance testing measures how a system behaves under load. It answers questions that functional tests cannot: "Can the API handle 500 requests per second?" "Does response time degrade under sustained traffic?" "At what point does the system start throwing errors?" This chapter covers the types of performance tests, writing load tests with k6, analyzing results, and defining SLOs that guide when to act.

---

## Prerequisites

- Track 09: Testing Fundamentals
- Basic understanding of HTTP APIs
- Familiarity with async patterns

---

## Core Concepts

### Types of performance tests

| Type | What it measures | Typical duration |
|------|-----------------|-----------------|
| **Load test** | Normal expected traffic | 5–30 minutes |
| **Stress test** | Breaking point (ramp up until failure) | 10–60 minutes |
| **Soak test** | Degradation over time (memory leaks, connection pool exhaustion) | Hours |
| **Spike test** | Sudden traffic burst recovery | 5–15 minutes |
| **Smoke test** | Basic sanity: can the API handle minimal load? | 1–2 minutes |

### SLOs (Service Level Objectives)

Define what "good" looks like before running tests:

```
p95 response time < 200ms under 500 RPS
Error rate < 0.1% under normal load
p99 response time < 1s during spike (2x normal traffic)
System recovers to baseline within 5 minutes after spike
```

Performance tests validate SLOs; without SLOs, test results are meaningless numbers.

---

## Hands-On Examples

### k6 setup and basic load test

```bash
# Install k6 (Linux/macOS)
brew install k6          # macOS
# or download from https://k6.io/docs/get-started/installation/
```

```javascript
// tests/load/api.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('error_rate');
const responseTime = new Trend('response_time_ms');

export const options = {
  // Ramp up to 100 VUs (virtual users) over 30 seconds
  // Stay at 100 VUs for 1 minute
  // Ramp down over 30 seconds
  stages: [
    { duration: '30s', target: 100 },
    { duration: '1m', target: 100 },
    { duration: '30s', target: 0 },
  ],

  // Test fails if these thresholds are exceeded
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],  // 95% under 200ms, 99% under 500ms
    error_rate: ['rate<0.01'],                        // less than 1% error rate
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('http://api.example.com/api/products', {
    headers: { Authorization: `Bearer ${__ENV.TEST_TOKEN}` },
  });

  // Record metrics
  responseTime.add(res.timings.duration);
  errorRate.add(res.status !== 200);

  // Assertions
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'has products array': (r) => JSON.parse(r.body).products !== undefined,
  });

  sleep(1); // think time between requests
}
```

```bash
# Run the load test
k6 run tests/load/api.js --env TEST_TOKEN=your-token

# Output
✓ status is 200          1000/1000
✓ response time < 500ms  990/1000
✗ has products array     995/1000

checks.........................: 99.83% ✓ 2985 ✗ 5
data_received..................: 12 MB 120 kB/s
data_sent......................: 3.5 MB 35 kB/s
http_req_blocked...............: avg=1.2ms  p(95)=3.4ms
http_req_duration..............: avg=87ms   p(95)=180ms   p(99)=420ms
http_reqs......................: 10000  100/s
```

### Stress test — finding the breaking point

```javascript
// tests/load/stress.js
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // normal load
    { duration: '5m', target: 100 },   // stay at normal
    { duration: '2m', target: 500 },   // ramp up to 5x
    { duration: '5m', target: 500 },   // stay at 5x
    { duration: '2m', target: 1000 },  // ramp to 10x
    { duration: '5m', target: 1000 },  // stay at 10x
    { duration: '5m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_failed: ['rate<0.10'],    // more lenient threshold for stress
    http_req_duration: ['p(99)<2000'], // 99% under 2s
  },
};
```

### Soak test — finding memory leaks

```javascript
// tests/load/soak.js
export const options = {
  stages: [
    { duration: '5m', target: 100 },  // ramp up
    { duration: '4h', target: 100 },  // hold for 4 hours
    { duration: '5m', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<200'],
    http_req_failed: ['rate<0.01'],
  },
};
```

During a soak test, watch for:
- Gradually increasing memory (leak in Node.js process)
- Gradually increasing response time (connection pool exhaustion)
- Intermittent 500s (uncaught exceptions accumulating)

### Spike test

```javascript
// tests/load/spike.js
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // baseline
    { duration: '10s', target: 1400 }, // sudden spike
    { duration: '3m', target: 1400 },  // hold spike
    { duration: '10s', target: 100 },  // back to baseline
    { duration: '3m', target: 100 },   // recovery period
  ],
};
```

### k6 with multiple endpoints and realistic scenarios

```javascript
// tests/load/realistic.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';

const BASE_URL = __ENV.BASE_URL ?? 'http://localhost:3000';

export const options = {
  stages: [
    { duration: '1m', target: 50 },
    { duration: '3m', target: 50 },
    { duration: '1m', target: 0 },
  ],
  thresholds: {
    'group_duration{group:::browse products}': ['p(95)<300'],
    'group_duration{group:::checkout}': ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const token = __ENV.TEST_TOKEN;
  const headers = { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' };

  group('browse products', () => {
    const list = http.get(`${BASE_URL}/api/products?limit=20`, { headers });
    check(list, { 'list status 200': (r) => r.status === 200 });
    sleep(1);

    const products = JSON.parse(list.body).products;
    const product = products[Math.floor(Math.random() * products.length)];

    const detail = http.get(`${BASE_URL}/api/products/${product.id}`, { headers });
    check(detail, { 'detail status 200': (r) => r.status === 200 });
    sleep(2);
  });

  group('checkout', () => {
    const order = http.post(
      `${BASE_URL}/api/orders`,
      JSON.stringify({ productId: 'prod-1', quantity: 1 }),
      { headers }
    );
    check(order, { 'order created': (r) => r.status === 201 });
    sleep(1);
  });
}
```

### Interpreting results

```bash
# k6 output key metrics
http_req_duration................: avg=87ms min=12ms med=72ms max=1200ms p(90)=145ms p(95)=180ms p(99)=420ms

# p50 (median): 50% of requests complete in 72ms or less
# p90: 90% complete in 145ms or less
# p95: 95% complete in 180ms or less  ← matches your SLO target
# p99: 99% complete in 420ms or less
# max: 1200ms — one request was very slow (investigate outliers)
```

Monitor these in a dashboard during the test. A spike in p99 while p50 remains stable usually indicates a queue or lock contention issue.

### Profiling Node.js during load tests

While k6 is running, profile your Node.js process to find bottlenecks:

```bash
# Start the API with profiling enabled
node --prof src/index.js

# After the load test, process the profile
node --prof-process isolate-*.log > profile.txt

# Or use clinic.js for a friendlier interface
npm install -g clinic
clinic doctor -- node src/index.js
# Then run k6 against it
clinic flame -- node src/index.js
```

---

## Common Patterns & Best Practices

- **Define SLOs before running tests** — p95 < 200ms, error rate < 0.1%
- **Use realistic scenarios** — mix of endpoints in realistic proportions, with think time
- **Test in a production-like environment** — staging with the same resources as production
- **Monitor infrastructure during tests** — CPU, memory, DB connections, not just HTTP metrics
- **Run smoke tests in CI** (1-2 minutes, low VUs) and full load tests before major releases

---

## Anti-Patterns to Avoid

- Testing against localhost — your laptop is not a server; results don't transfer
- No think time between requests (`sleep`) — real users don't hammer APIs 1000 RPS each
- Ignoring error rates and focusing only on response time
- Running performance tests against production — you will impact real users

---

## Debugging & Troubleshooting

**"Response times spike at specific intervals"**
Check for garbage collection pauses (Node.js GC) or scheduled jobs (cron, queue processing) running during the test. Node.js GC pauses can be profiled with `--expose-gc` and monitoring.

**"DB connections are exhausted under load"**
The connection pool is too small or queries are taking too long. Add PgBouncer, increase pool size, or optimize slow queries (see Track 08: Database Scaling).

**"k6 metrics look fine but users report slowness"**
k6 tests the server — network latency between user and server is not included. Check CDN and DNS resolution times. Use Lighthouse or WebPageTest for real-user performance.

---

## Further Reading

- [k6 documentation](https://k6.io/docs/)
- [k6 thresholds and SLOs](https://k6.io/docs/using-k6/thresholds/)
- [clinic.js — Node.js profiling](https://clinicjs.org/)
- [Google SRE Book: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- Track 10: [Profiling Tools](../10-performance/profiling-tools.md)

---

## Summary

Performance testing is the practice of defining SLOs and then verifying the system meets them under realistic load. k6 is the right tool for Node.js backend performance testing — script-based, CI-friendly, and produces the percentile metrics (p95, p99) that matter for SLO compliance. Run smoke tests in CI, full load tests before releases, and soak tests when investigating memory or connection pool issues. Always test in a production-like environment with realistic traffic patterns, and monitor infrastructure metrics alongside HTTP metrics during the test.
