# Databases: NoSQL

## 1. What & Why

"NoSQL" does not mean "no SQL" — it means "not only SQL." It is an umbrella term for databases that diverge from the traditional relational model in exchange for different trade-offs: flexible schemas, horizontal scalability, specific access pattern optimization, or specialized data models (documents, key-value, wide-column, graph).

Understanding NoSQL is not about knowing which is "better than SQL" — it is about choosing the right tool. PostgreSQL handles 90% of most applications perfectly. The remaining 10% often have specific requirements: ultra-low latency lookups (Redis), massive write throughput with simple queries (Cassandra), flexible evolving schemas (MongoDB), or deeply connected relationship data (Neo4j).

This file covers the four main NoSQL families, when to use each, the CAP theorem that governs their fundamental trade-offs, and the practical patterns that matter in production.

---

## 2. Core Concepts

### CAP Theorem

The CAP theorem states that a distributed data store can only provide **two** of the following three guarantees simultaneously — especially during a network partition:

- **Consistency (C):** Every read receives the most recent write or an error. All nodes see the same data at the same time.
- **Availability (A):** Every request receives a response (not necessarily the most recent data). The system keeps working even if some nodes are down.
- **Partition Tolerance (P):** The system continues operating even when network partitions (message loss between nodes) occur.

In practice, network partitions are unavoidable in distributed systems. So the real choice is: **during a partition, do you sacrifice Consistency (AP) or Availability (CP)?**

```
CP systems (consistency over availability):
  - MongoDB (default — primary refuses reads/writes if cannot confirm quorum)
  - HBase, Zookeeper
  - Choose when: data correctness is non-negotiable (banking, inventory)

AP systems (availability over consistency):
  - Cassandra, CouchDB, DynamoDB
  - Choose when: availability matters more than seeing the absolute latest data
    (shopping carts, user preferences, metrics)
  - Eventually consistent: all nodes will converge to the same state given enough time

CA systems (no partition tolerance):
  - Traditional single-node RDBMS (PostgreSQL, MySQL without replication)
  - Only valid in single-datacenter, controlled-network environments
```

> ⚠️ CAP is often oversimplified. Real systems make more nuanced choices: MongoDB allows configuring read/write concerns per operation, Cassandra allows tuning consistency level per query. PACELC extends CAP to also model latency vs consistency trade-offs.

### BASE vs ACID

| Property | ACID | BASE |
|----------|------|------|
| Model | Strong consistency | Eventual consistency |
| State | Always consistent | Soft state (may be temporarily inconsistent) |
| Availability | May sacrifice for consistency | Prioritizes availability |
| Transactions | Multi-record ACID | Often single-record operations |
| Use case | Financial, transactional | Social feeds, metrics, user activity |

**BASE:** Basically Available, Soft state, Eventually consistent.

---

## 3. Document Stores (MongoDB)

MongoDB stores data as BSON (Binary JSON) documents grouped in collections. Documents in the same collection can have different schemas.

### Schema Design: Embed vs Reference

This is the most important design decision in MongoDB.

**Embed when:**
- Data is accessed together (no need to join)
- Relationship is 1:1 or 1:few (not 1:many-thousands)
- Embedded data does not change independently at high frequency
- Total document size stays < 16MB (MongoDB's limit)

```javascript
// Embedded: post + comments (accessed together, comment count manageable)
{
  "_id": ObjectId("..."),
  "title": "Getting Started with MongoDB",
  "author": { "id": "user_42", "name": "Alice" },  // embedded snapshot
  "tags": ["mongodb", "nosql"],
  "comments": [
    { "author": "Bob", "text": "Great post!", "createdAt": ISODate("...") },
    { "author": "Carol", "text": "Very helpful", "createdAt": ISODate("...") }
  ]
}
```

**Reference when:**
- Many:many relationships
- Referenced documents are large or change independently
- The relationship count is unbounded (1:millions)
- Data needs to be queried independently

```javascript
// Referenced: user profile → orders (orders can be millions)
// Users collection:
{ "_id": ObjectId("user_42"), "name": "Alice", "email": "alice@example.com" }

// Orders collection:
{ "_id": ObjectId("order_1"), "userId": ObjectId("user_42"), "total": 149.99 }
{ "_id": ObjectId("order_2"), "userId": ObjectId("user_42"), "total": 89.00 }
```

> 💡 Unlike SQL, MongoDB does NOT enforce referential integrity. If you delete a user, their orders still reference the deleted user ID. Your application code must manage this consistency.

### Aggregation Pipeline

```javascript
// Complex analytics: revenue by product category for the last 30 days
db.orders.aggregate([
  // Stage 1: Filter
  { $match: {
    status: 'completed',
    createdAt: { $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) }
  }},

  // Stage 2: Unwind array (one doc per item)
  { $unwind: '$items' },

  // Stage 3: Join with products collection
  { $lookup: {
    from: 'products',
    localField: 'items.productId',
    foreignField: '_id',
    as: 'product'
  }},
  { $unwind: '$product' },

  // Stage 4: Group by category
  { $group: {
    _id: '$product.category',
    totalRevenue: { $sum: { $multiply: ['$items.price', '$items.quantity'] } },
    orderCount: { $sum: 1 },
    avgOrderValue: { $avg: { $multiply: ['$items.price', '$items.quantity'] } }
  }},

  // Stage 5: Shape the output
  { $project: {
    category: '$_id',
    totalRevenue: { $round: ['$totalRevenue', 2] },
    orderCount: 1,
    avgOrderValue: { $round: ['$avgOrderValue', 2] },
    _id: 0
  }},

  // Stage 6: Sort by revenue descending
  { $sort: { totalRevenue: -1 } }
])
```

### Indexes in MongoDB

```javascript
// Single field index
db.users.createIndex({ email: 1 });  // 1 = ascending, -1 = descending

// Compound index (left-prefix rule applies)
db.orders.createIndex({ userId: 1, createdAt: -1 });

// Unique index
db.users.createIndex({ email: 1 }, { unique: true });

// Partial index (MongoDB 3.2+)
db.orders.createIndex(
  { userId: 1, createdAt: -1 },
  { partialFilterExpression: { status: 'active' } }
);

// Text index (full-text search)
db.articles.createIndex({ title: 'text', body: 'text' });
db.articles.find({ $text: { $search: 'mongodb performance' } });

// Geospatial index
db.stores.createIndex({ location: '2dsphere' });
db.stores.find({
  location: { $near: {
    $geometry: { type: 'Point', coordinates: [-73.9857, 40.7484] },
    $maxDistance: 5000  // meters
  }}
});
```

### Multi-Document Transactions (MongoDB 4.0+)

```javascript
const session = await client.startSession();
try {
  await session.withTransaction(async () => {
    await db.accounts.updateOne(
      { _id: fromId },
      { $inc: { balance: -amount } },
      { session }
    );
    await db.accounts.updateOne(
      { _id: toId },
      { $inc: { balance: amount } },
      { session }
    );
    await db.transactions.insertOne(
      { fromId, toId, amount, createdAt: new Date() },
      { session }
    );
  });
} finally {
  await session.endSession();
}
```

> ⚠️ Multi-document transactions in MongoDB carry significant performance overhead. Design your schema to avoid needing them (use embedded documents for atomic updates when possible). Use transactions only when the invariant genuinely spans multiple documents/collections.

---

## 4. Key-Value Stores (Redis)

Redis stores data in memory with optional persistence. Operations are O(1) to O(log n) for most data types.

### Data Types and Use Cases

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

// --- Strings: simple cache, counters ---
await redis.set('user:42:name', 'Alice');
await redis.setex('session:abc123', 3600, JSON.stringify(sessionData)); // TTL 1hr
await redis.incr('page:views:/home'); // atomic counter
await redis.incrby('user:42:credits', 50);

// --- Hashes: objects, user sessions ---
await redis.hset('user:42', 'name', 'Alice', 'email', 'alice@example.com');
await redis.hset('user:42', { plan: 'pro', logins: '0' });
const user = await redis.hgetall('user:42');
await redis.hincrby('user:42', 'logins', 1);

// --- Lists: queues, activity feeds (FIFO/LIFO) ---
await redis.lpush('notifications:42', JSON.stringify({ type: 'like', postId: '1' }));
const notifications = await redis.lrange('notifications:42', 0, 9); // last 10
await redis.ltrim('notifications:42', 0, 99); // keep only 100 most recent

// --- Sets: unique visitors, tags, relationships ---
await redis.sadd('post:1:viewers', userId);
const viewCount = await redis.scard('post:1:viewers');
const commonFriends = await redis.sinter('user:1:friends', 'user:2:friends');

// --- Sorted Sets: leaderboards, rate limiting ---
await redis.zadd('leaderboard:season1', score, userId);
const top10 = await redis.zrevrange('leaderboard:season1', 0, 9, 'WITHSCORES');
const userRank = await redis.zrevrank('leaderboard:season1', userId);

// --- Streams: event log, pub/sub with persistence ---
const id = await redis.xadd('events', '*', 'type', 'page_view', 'path', '/home', 'userId', '42');
const events = await redis.xrange('events', '-', '+', 'COUNT', 100);
```

### Persistence

```
RDB (Redis Database): Point-in-time snapshots
  save 900 1    # snapshot if 1+ keys changed in 15 minutes
  save 300 10   # snapshot if 10+ keys changed in 5 minutes
  save 60 10000 # snapshot if 10000+ keys changed in 1 minute
  Pros: compact, fast restarts, minimal I/O impact
  Cons: potential data loss between snapshots

AOF (Append-Only File): logs every write operation
  appendonly yes
  appendfsync everysec  # fsync every second (compromise)
  # appendfsync always  # safest but slowest
  # appendfsync no      # fastest, OS decides when to flush
  Pros: minimal data loss (at most 1 second with everysec)
  Cons: larger files, slower restarts

Combined: use both for best durability + fast restart
```

### Eviction Policies

```
noeviction:      Return error when memory full. Good for primary data store.
allkeys-lru:     Evict least recently used keys. Good for pure cache.
volatile-lru:    LRU but only keys with TTL set. Evict cache, keep persistent data.
allkeys-lfu:     Evict least frequently used. Better than LRU for skewed access.
volatile-ttl:    Evict keys with shortest remaining TTL first.
allkeys-random:  Random eviction. Rarely useful.
```

> 💡 For a pure cache (all keys have TTL), use `allkeys-lru` or `allkeys-lfu`. For a mixed store (some cached, some persistent), use `volatile-lru` to protect persistent keys from eviction.

---

## 5. Wide-Column Stores (Cassandra)

Cassandra is optimized for massive write throughput across multiple datacenters with no single point of failure.

### Data Model

The Cassandra data model is driven by query patterns, not by domain relationships. Design tables for the queries you will run.

```cql
-- Partition key: determines which node stores the data (consistent hashing)
-- Clustering key: determines sort order within a partition
-- Rule: queries must always specify the partition key

CREATE TABLE events_by_user (
  user_id     UUID,
  occurred_at TIMESTAMP,
  event_id    UUID,
  event_type  TEXT,
  payload     TEXT,
  PRIMARY KEY (user_id, occurred_at, event_id)
)
WITH CLUSTERING ORDER BY (occurred_at DESC);

-- Efficient: partition key specified
SELECT * FROM events_by_user
WHERE user_id = ? AND occurred_at >= ? AND occurred_at <= ?;

-- Inefficient / disallowed: no partition key — full cluster scan
-- SELECT * FROM events_by_user WHERE event_type = 'click'; ← ALLOW FILTERING needed
```

### Consistency Levels

```cql
-- Write with quorum (majority of replicas must acknowledge)
INSERT INTO events_by_user ... USING CONSISTENCY QUORUM;

-- Read with quorum
SELECT ... FROM events_by_user WHERE ... USING CONSISTENCY QUORUM;

-- Eventual consistency (fastest, may read stale data)
SELECT ... USING CONSISTENCY ONE;

-- Strong consistency: write QUORUM + read QUORUM ensures latest data
-- (requires: reads + writes > replication factor)
```

Cassandra is **write-optimized**: writes are appended to a commit log and a memtable (in-memory structure), making writes O(1). Reads are more expensive — data may be in memtable, SSTables, or a bloom filter check.

---

## 6. Graph Databases (Neo4j)

Graph databases store data as nodes (entities) and relationships (edges between entities), each with properties.

```cypher
// Create nodes and relationship
CREATE (alice:User {id: '1', name: 'Alice', city: 'NYC'})
CREATE (bob:User {id: '2', name: 'Bob', city: 'LA'})
CREATE (alice)-[:FOLLOWS {since: date('2024-01-01')}]->(bob)

// Find friends of friends (2-hop traversal)
MATCH (me:User {id: $userId})-[:FOLLOWS]->(friend)-[:FOLLOWS]->(fof)
WHERE fof <> me AND NOT (me)-[:FOLLOWS]->(fof)
RETURN fof.name, COUNT(*) as mutual_friends
ORDER BY mutual_friends DESC
LIMIT 10

// Shortest path between two users
MATCH path = shortestPath(
  (alice:User {id: '1'})-[:FOLLOWS*]-(target:User {id: '99'})
)
RETURN path

// Recommendation: products bought by users who bought what I bought
MATCH (me:User {id: $userId})-[:PURCHASED]->(p:Product)<-[:PURCHASED]-(similar:User)
      -[:PURCHASED]->(recommended:Product)
WHERE NOT (me)-[:PURCHASED]->(recommended)
RETURN recommended.name, COUNT(*) as score
ORDER BY score DESC LIMIT 10
```

**When graph databases shine:**
- Social networks (friend recommendations, connection paths)
- Fraud detection (detecting unusual patterns in transaction graphs)
- Knowledge graphs
- Access control with complex role hierarchies
- Route planning and logistics

> 💡 If relationship traversal is the core query pattern — not just an occasional JOIN — a graph database eliminates the exponential cost of multi-hop SQL JOINs. A 6-hop traversal in SQL would require 6 JOINs; Cypher handles it naturally.

---

## 7. MongoDB Schema Design Deep Dive

```typescript
// Pattern: Bucket pattern for time-series data
// Instead of one document per event (millions of documents), group events by time bucket

// Bad: one document per measurement (10,000 documents/day per sensor)
{
  "_id": ObjectId("..."),
  "sensorId": "sensor_42",
  "temperature": 22.5,
  "recordedAt": ISODate("2024-01-15T14:30:00Z")
}

// Good: bucket pattern — one document per hour per sensor (288 documents/day)
{
  "_id": ObjectId("..."),
  "sensorId": "sensor_42",
  "date": ISODate("2024-01-15T14:00:00Z"),
  "measurements": [
    { "minute": 0, "temperature": 22.5 },
    { "minute": 1, "temperature": 22.6 },
    // ... up to 60 measurements
  ],
  "stats": {
    "min": 22.1,
    "max": 23.4,
    "avg": 22.7,
    "count": 60
  }
}

// Pattern: Computed pattern — pre-aggregate frequently-read stats
{
  "_id": ObjectId("post_123"),
  "title": "Getting Started",
  "commentCount": 47,    // ← denormalized, updated on comment create/delete
  "likeCount": 312,
  "viewCount": 9841
}
// Avoids COUNT(*) aggregation on every page load
```

---

## 8. Common Mistakes & Pitfalls

**Unbounded embedded arrays in MongoDB:**

```javascript
// WRONG: comments grow unbounded — document hits 16MB limit
{ _id: '...', title: 'Viral Post', comments: [/* millions of comments */] }

// CORRECT: reference comments separately when count is unbounded
// Post document: commentCount: 15420 (pre-computed)
// Comments collection: { postId: '...', text: '...', createdAt: ... }
```

**Ignoring Cassandra's partition key requirement:**

```cql
-- WRONG: will either fail or require ALLOW FILTERING (full cluster scan)
SELECT * FROM events WHERE event_type = 'purchase';

-- CORRECT: always design tables around query patterns
-- Create a separate table for this query pattern
CREATE TABLE events_by_type (
  event_type TEXT,
  occurred_at TIMESTAMP,
  event_id UUID,
  user_id UUID,
  PRIMARY KEY (event_type, occurred_at, event_id)
);
```

**Using Redis as a primary database without persistence:**

```
If Redis is your only data store and you use default settings (no AOF, no RDB),
a restart loses ALL data.

Minimum for primary store:
  appendonly yes
  appendfsync everysec
  requirepass <strong-password>

Better: use Redis as a cache + a durable DB as the source of truth
```

**MongoDB without indexing:**

```javascript
// WRONG: querying a 10M document collection without an index
db.orders.find({ userId: 'user_42', status: 'pending' }).sort({ createdAt: -1 });
// → Seq scan, potentially seconds

// CORRECT: index on query fields
db.orders.createIndex({ userId: 1, status: 1, createdAt: -1 });
// → Index scan, milliseconds

// Always run explain() to check
db.orders.find({ userId: 'user_42', status: 'pending' })
  .explain('executionStats');
// Look for: COLLSCAN (bad) vs IXSCAN (good)
```

---

## 9. When NoSQL vs SQL: Decision Matrix

| Scenario | Recommendation | Reasoning |
|----------|---------------|-----------|
| User accounts + orders + products | PostgreSQL | Complex relationships, ACID transactions |
| Product catalog with variable attributes | MongoDB | Flexible schema, schema varies by product type |
| User sessions and caching | Redis | In-memory, O(1) ops, TTL support |
| IoT sensor data (billions of events) | Cassandra or TimescaleDB | Write-heavy, time-series, horizontal scale |
| Social graph, fraud detection | Neo4j | Traversal is the primary query pattern |
| Real-time leaderboards | Redis sorted sets | O(log n) ranking, atomic updates |
| Full-text search | Elasticsearch or PostgreSQL GIN | Inverted index, relevance scoring |
| E-commerce with inventory | PostgreSQL | ACID for stock decrement, complex queries |
| Content management, CMS | MongoDB or PostgreSQL | Document model OR JSONB |
| Analytics/reporting | PostgreSQL or ClickHouse | Window functions, aggregations |

---

## 10. Real-World Scenario

**Problem:** A social media platform stores user activity events (likes, comments, shares, views) in a single PostgreSQL table. With 500 million daily events, the table has grown to 2 billion rows. Writes are slow, reads timeout, and VACUUM cannot keep up.

**Analysis:**
- Events are append-only — never updated.
- Primary query: "give me all events for user X in the last 7 days" (partition key = userId + date).
- Retention: 90 days then archive to cold storage.
- No complex joins needed.

**Migration to Cassandra:**

```cql
CREATE TABLE user_activity (
  user_id     UUID,
  bucket_date DATE,
  occurred_at TIMESTAMP,
  event_id    UUID,
  event_type  TEXT,   -- 'like', 'comment', 'share', 'view'
  target_id   TEXT,
  metadata    TEXT,   -- JSON blob
  PRIMARY KEY ((user_id, bucket_date), occurred_at, event_id)
)
WITH CLUSTERING ORDER BY (occurred_at DESC)
AND default_time_to_live = 7776000;  -- 90 days TTL, auto-delete

-- Write (from application)
INSERT INTO user_activity (user_id, bucket_date, occurred_at, event_id, event_type, target_id)
VALUES (?, ?, toTimestamp(now()), uuid(), ?, ?)
USING TTL 7776000;

-- Read: last 7 days of activity for a user
SELECT * FROM user_activity
WHERE user_id = ? AND bucket_date IN (?, ?, ?, ?, ?, ?, ?)
ORDER BY occurred_at DESC
LIMIT 100;
```

**Result:** Writes go from 800ms average to 2ms. Read latency drops from 30 seconds (with timeouts) to 8ms. Auto-TTL handles data expiration without maintenance.

---

## 11. Interview Questions

**Q1: Explain the CAP theorem in practical terms.**

A distributed system during a network partition must choose: either refuse requests (to stay consistent) or serve potentially stale data (to stay available). MongoDB defaults to CP — the primary stops accepting writes if it cannot confirm a quorum of replicas, ensuring all reads get the latest data. Cassandra is AP — it keeps accepting writes and reads even if some nodes are unreachable, with the trade-off that some reads may return stale data until the nodes reconcile. In practice, modern systems (like MongoDB with configurable read/write concerns) let you tune this per-operation.

**Q2: When should you embed vs reference documents in MongoDB?**

Embed when: data is always accessed together (post + its 5 most recent comments), the relationship count is bounded and small (< a few hundred), and the embedded data does not change independently at high frequency. Reference when: the relationship is many-to-many, the array is unbounded (could grow to thousands), the referenced document is large and often queried independently, or you need to query the referenced documents without going through the parent. The key question: "Do I always need both pieces of data together, or do I sometimes need them separately?"

**Q3: When would you choose Cassandra over PostgreSQL?**

Cassandra is ideal when: write throughput is extreme (millions of writes per second), data is append-only with simple access patterns (time-series, activity logs, IoT sensor data), multi-datacenter geographic distribution is required with no single point of failure, and you can design your tables around specific query patterns. PostgreSQL is better when you need complex queries, joins, ACID multi-record transactions, or ad-hoc analytics. The key Cassandra constraint: every query must specify the partition key, so you pre-design tables for known queries.

**Q4: What is eventual consistency and what problems does it solve vs create?**

Eventual consistency means all nodes will eventually converge to the same data if writes stop, but at any point in time different nodes may return different values. It solves: availability (the system keeps working even when some nodes are unreachable), global distribution (nodes can accept writes locally without coordinating with distant datacenters in real-time). Problems it creates: reading your own writes may fail (you write on node A, then read from node B before replication), and concurrent writes to the same record can conflict (requires conflict resolution strategy: last-write-wins, vector clocks, or application-level merge).

**Q5: For an e-commerce platform (users, products, orders, inventory), would you use MongoDB or PostgreSQL?**

PostgreSQL. E-commerce requires: ACID transactions for inventory decrements (preventing oversell), complex JOINs (order history with product details, customer analytics), referential integrity (foreign keys between orders and users/products), and ad-hoc reporting. MongoDB can work but requires application-level transaction management and loses the benefit of SQL for analytics. The exception: product catalog attributes (variable per product type) are well-suited for MongoDB's flexible schema — many e-commerce platforms use PostgreSQL for transactions and MongoDB (or JSONB in Postgres) for product catalog.

**Q6: When would you choose a graph database?**

When the primary value of your data lies in the relationships between entities, not in the entities themselves. Social networks (friend-of-friend recommendations, shortest connection path), fraud detection (detect rings of suspicious transactions), access control with complex inheritance hierarchies, and knowledge graphs. The deciding question: "Is my core query pattern multi-hop relationship traversal?" If a SQL query for your use case would require 5+ JOINs or recursive CTEs, a graph database is worth considering.

**Q7: Compare BASE and ACID.**

ACID (typical of SQL databases): Atomicity (all-or-nothing), Consistency (constraints always maintained), Isolation (transactions appear serial), Durability (committed data survives crashes). Provides strong guarantees at the cost of reduced availability and throughput in distributed settings. BASE (typical of NoSQL): Basically Available (system stays operational), Soft state (data can be temporarily inconsistent), Eventually consistent (all nodes converge given time). Trades correctness guarantees for higher availability and performance. Use ACID for financial, medical, inventory systems; BASE for social feeds, activity logs, user preferences.

---

## 12. Exercises

**Exercise 1: MongoDB schema design for a blog**

Design a MongoDB schema for a blog with:
- Posts (title, body, tags, author info)
- Comments (text, author, reactions)
- Tags with post counts

Requirements: viewing a post shows its content + author + first 10 comments; tag browsing shows posts sorted by popularity; author profile shows their recent posts. Design documents, embedding decisions, and indexes. Justify each embedding vs reference decision.

*Hint:* Posts should embed author snapshot (not full user object) and comment count as a pre-computed field. Comments should be a separate collection (unbounded). Tags should be a separate collection with a `postCount` field.

**Exercise 2: Redis rate limiter**

Implement a rate limiter using Redis sorted sets that:
- Allows 100 requests per user per minute
- Uses a sliding window (not fixed window)
- Returns remaining quota and reset time in headers
- Handles Redis connection failures gracefully (fail open)

*Hint:* Use `ZADD` with timestamp as score, `ZREMRANGEBYSCORE` to remove old entries, `ZCARD` to count current window entries. Wrap in a Lua script for atomicity.

**Exercise 3: Cassandra time-series table**

Design a Cassandra table for storing website analytics events (page views, clicks, conversions) with requirements:
- Query by website + date range (e.g., last 30 days)
- Query by website + event type + date
- Store at most 1 year of data (use TTL)
- Support 100,000 writes per second across 10 websites

Design the table schema, partition key, clustering key, and access pattern queries. Explain why you chose this partition strategy over alternatives.

---

## 13. Further Reading

- [MongoDB data modeling guide](https://www.mongodb.com/docs/manual/data-modeling/)
- [Cassandra data modeling — DataStax](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/dml/dmlAboutDataModels.html)
- [Redis data types documentation](https://redis.io/docs/data-types/)
- [Neo4j Cypher manual](https://neo4j.com/docs/cypher-manual/)
- [PACELC theorem (extends CAP)](https://dl.acm.org/doi/10.1145/2360276.2360337)
- [Martin Fowler — NoSQL Distilled (book)](https://martinfowler.com/books/nosql.html)
- [Designing Data-Intensive Applications by Martin Kleppmann](https://dataintensive.net/) — chapters on replication and consistency
- [MongoDB University (free courses)](https://learn.mongodb.com/)
- [The Little Redis Book (free)](http://openmymind.net/redis.pdf)
