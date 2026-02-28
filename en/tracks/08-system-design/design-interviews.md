# System Design Interviews

## Overview

System design interviews assess your ability to architect a large-scale system from a vague problem statement. Unlike coding interviews, there is no single right answer — the goal is to demonstrate structured thinking, knowledge of trade-offs, and the ability to have a productive technical conversation. This chapter provides a repeatable framework, common estimation techniques, and walkthroughs of the most frequently asked system design questions.

---

## Prerequisites

- All previous chapters in Track 08
- Track 05 — DevOps: containers, networking, CDN basics
- Track 07 — Security: auth, rate limiting basics

---

## Core Concepts

### The interview framework (RADISH)

A structured approach for any system design problem:

| Step | Action | Time |
|------|--------|------|
| **R** — Requirements | Clarify functional and non-functional requirements | 5 min |
| **A** — API | Define the API surface | 5 min |
| **D** — Data model | Schema, storage choices | 5 min |
| **I** — Infrastructure | High-level components | 5 min |
| **S** — Scale | Handle the stated traffic/data volume | 10 min |
| **H** — Harden | Bottlenecks, failure modes, monitoring | 5 min |

### Estimation cheat sheet

Memorize these numbers for back-of-envelope calculations:

```
Latency numbers:
  L1 cache:          ~1 ns
  L2 cache:          ~4 ns
  RAM access:        ~100 ns
  SSD read:          ~100 µs
  HDD read:          ~10 ms
  Network RTT:       ~20 ms (same region)
  Network RTT:       ~150 ms (cross-continent)

Throughput:
  1 Gbps network:    ~125 MB/s
  SSD:               ~500 MB/s read
  Postgres:          ~1,000–10,000 QPS (depending on query complexity)
  Redis:             ~100,000 ops/s

Storage:
  1 million users × 1KB profile = 1 GB
  1 million photos × 3 MB = 3 TB
  1 char = 1 byte | 1 int = 4 bytes | 1 UUID = 16 bytes

Time conversion:
  1 day = 86,400 seconds ≈ 100,000 seconds (handy approximation)
  1 year = 31.5 million seconds
```

---

## Interview Walkthroughs

### Design a URL shortener (bit.ly)

**1. Requirements**

Functional:
- Given a long URL, generate a short URL
- Redirect short URL to original URL
- (Optional) Analytics: click count, geographic distribution

Non-functional:
- 100M URLs created per day
- Read:Write ratio = 100:1 (reads are dominant)
- URLs should be available for 10 years

**2. Estimation**

```
Writes: 100M / day = 1,000 writes/second
Reads:  100:1 ratio = 100,000 reads/second

Storage per URL: long_url (500 bytes) + short_code (7 bytes) + metadata (100 bytes) ≈ 700 bytes
Total storage: 100M URLs/day × 10 years × 700 bytes ≈ 255 TB
```

**3. API design**

```typescript
// Create short URL
POST /api/urls
Body: { longUrl: string, customAlias?: string, expiresAt?: string }
Response: { shortCode: string, shortUrl: string }

// Redirect
GET /:code → 301/302 to original URL

// Analytics
GET /api/urls/:code/stats → { clicks: number, ... }
```

**4. Data model**

```sql
-- URLs table
CREATE TABLE urls (
  id          BIGSERIAL PRIMARY KEY,
  short_code  VARCHAR(8)   UNIQUE NOT NULL,
  long_url    TEXT         NOT NULL,
  user_id     UUID,
  created_at  TIMESTAMPTZ  DEFAULT now(),
  expires_at  TIMESTAMPTZ,
  click_count BIGINT       DEFAULT 0
);

CREATE INDEX ON urls (short_code);  -- primary lookup path
```

**5. Short code generation**

```typescript
// Option A: Random base62 code (7 chars = 62^7 = ~3.5 trillion combinations)
function generateCode(length = 7): string {
  const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  let result = '';
  const bytes = crypto.randomBytes(length);
  for (const byte of bytes) {
    result += chars[byte % chars.length];
  }
  return result;
}

// Option B: Base62 of auto-incrementing ID (no collision check needed)
function encodeId(id: number): string {
  const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  let result = '';
  while (id > 0) {
    result = chars[id % 62] + result;
    id = Math.floor(id / 62);
  }
  return result.padStart(7, 'a');
}
```

**6. Architecture**

```
Client → CDN (cache redirects)
       → Load Balancer
       → API servers (stateless, horizontal scale)
       → Redis (hot URL cache, TTL=1h)
       → Postgres (primary source of truth, read replica for reads)
       → Kafka (click events for async analytics processing)
```

**7. Scale bottlenecks**

- **Read throughput:** 100K reads/s. Solution: Redis cache with 99%+ hit rate. Fallback to read replica.
- **Write throughput:** 1K writes/s. Solution: Single Postgres primary handles this comfortably.
- **Analytics writes:** 100K click events/s. Solution: Kafka → batch insert, not synchronous DB update.
- **Hot URLs:** top 1% of URLs drive 99% of traffic. Redis LRU cache handles this automatically.

---

### Design a real-time chat application (Slack/WhatsApp)

**1. Requirements**

Functional:
- Send and receive messages in real-time
- Group chats (up to 500 members)
- Online/offline presence
- Message history
- Push notifications for offline users

Non-functional:
- 100M daily active users
- Each user sends ~10 messages/day
- Messages delivered in < 100ms

**2. Estimation**

```
Messages/second: 100M users × 10 msgs / 86,400 seconds ≈ 12,000 msgs/second
Message storage: 12,000 msgs/s × 100 bytes × 86,400s × 365 days ≈ 37 TB/year
```

**3. Components**

```
Client (WebSocket)
  ↓
WebSocket Gateway (maintains persistent connections)
  ↓
Message Service
  ↓
Kafka (message events) → Fan-out Service → Recipient WebSocket connections
                       → Push Notification Service (Firebase/APNs)
                       → Message Store (Cassandra — write-heavy, time-series)

Presence Service (Redis — TTL-based online status)
Media Storage (S3 + CDN for images, videos)
```

**4. WebSocket architecture**

```typescript
// WebSocket gateway — maintains user connections
const connections = new Map<string, WebSocket>();

wss.on('connection', (ws, request) => {
  const userId = extractUserIdFromRequest(request);
  connections.set(userId, ws);

  // Update presence
  await redis.setEx(`presence:${userId}`, 30, 'online');

  ws.on('message', async (data) => {
    const message = JSON.parse(data.toString());
    // Validate, persist, and fan-out
    await kafka.producer.send({
      topic: 'messages',
      messages: [{ value: JSON.stringify(message) }],
    });
  });

  ws.on('close', () => {
    connections.delete(userId);
    redis.del(`presence:${userId}`);
  });
});

// Fan-out worker: deliver to connected recipients
kafka.consumer.subscribe({ topic: 'messages' });
await kafka.consumer.run({
  eachMessage: async ({ message }) => {
    const msg = JSON.parse(message.value.toString());
    for (const recipientId of msg.recipients) {
      const ws = connections.get(recipientId);
      if (ws?.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify(msg));
      } else {
        await sendPushNotification(recipientId, msg);
      }
    }
  },
});
```

**5. Scaling WebSocket servers**

The challenge: WebSocket connections are stateful. User A is connected to Server 1, User B to Server 2. A message from A to B must route from Server 1 to Server 2.

Solution: Redis Pub/Sub as a message bus between WebSocket servers:

```typescript
// Server 1: publish message when A sends to B
await redis.publish(`user:${recipientId}`, JSON.stringify(message));

// Server 2: subscribed to B's channel
redis.subscribe(`user:${userId}`, (message) => {
  const ws = localConnections.get(userId);
  ws?.send(message);
});
```

---

### Design Twitter's timeline

**1. The core problem: fan-out**

When a user with 10M followers tweets, how do you show that tweet in 10M timelines?

**Fan-out on write (push model):**
- On tweet creation, write to every follower's timeline cache
- Pro: reads are O(1) — just read the timeline cache
- Con: 10M follower accounts × 1 write = very slow for celebrities

**Fan-out on read (pull model):**
- On timeline read, fetch tweets from all followees and merge
- Pro: no write amplification
- Con: reads are slow (merge N followee timelines)

**Hybrid approach (Twitter's actual solution):**
- Small accounts (<100K followers): fan-out on write
- Celebrity accounts (>100K followers): fan-out on read, inject into timelines at read time

```typescript
async function getTimeline(userId: string): Promise<Tweet[]> {
  // 1. Get pre-computed timeline from cache (regular users' tweets already injected)
  const cachedTimeline = await redis.zRange(`timeline:${userId}`, 0, 20, { REV: true });

  // 2. Fetch celebrity tweets that were not fanned out (merge on read)
  const followedCelebrities = await db.follow.findMany({
    where: { followerId: userId, followee: { followerCount: { gte: 100000 } } },
    select: { followeeId: true },
  });

  const celebrityTweets = await Promise.all(
    followedCelebrities.map((f) =>
      redis.zRange(`user-tweets:${f.followeeId}`, 0, 20, { REV: true })
    )
  );

  // 3. Merge and sort by timestamp
  return mergeAndSort([...cachedTimeline, ...celebrityTweets.flat()]).slice(0, 20);
}
```

---

## Common Interview Mistakes

- **Not clarifying requirements** — jumping into the solution before understanding the scale
- **Designing for the wrong scale** — asking "how many users?" changes everything
- **Ignoring failure modes** — what happens when Redis goes down? When Kafka is slow?
- **Not discussing trade-offs** — every choice has pros and cons; articulate them
- **Over-engineering** — a cron job may be simpler than Kafka for your scale
- **Under-specifying the data model** — the schema often reveals hidden complexity

---

## Real-World Scenarios

**How to handle "design X in 45 minutes" pressure:**

1. (5 min) Ask 4–5 clarifying questions to establish scale: DAU, read/write ratio, data size, latency requirements, consistency requirements
2. (5 min) Define the API: 2–3 core endpoints
3. (5 min) Draw the data model: 3–4 tables/collections with key indexes
4. (10 min) High-level component diagram: client, load balancer, API, cache, DB, queue, CDN
5. (15 min) Deep dive on the hardest part (usually the scaling problem)
6. (5 min) Discuss failure modes and monitoring

Always drive the conversation. The interviewer is evaluating whether you can architect a system, not whether you have a perfect answer.

---

## Further Reading

- [System Design Primer — GitHub](https://github.com/donnemartin/system-design-primer)
- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [High Scalability blog](http://highscalability.com/)
- [ByteByteGo newsletter](https://blog.bytebytego.com/)
- All previous chapters in Track 08

---

## Summary

System design interviews reward structured thinking over encyclopedic knowledge. Follow the RADISH framework: clarify requirements, define the API, model the data, draw the infrastructure, handle the scale, and harden against failures. Estimation skills are critical — they determine whether you need Redis or a simple database column, whether you need Kafka or a cron job. The most important skill is articulating trade-offs: why this approach and not that one, what you gain, and what you give up. Every component you add has a cost; the best architects minimize components while meeting requirements.
