# Architecture Katas

> System design exercises to develop your architectural thinking. Each kata presents a real-world system to design under realistic constraints. There are no single correct answers — the goal is to reason through trade-offs, communicate decisions, and understand the implications of your choices. Skill level: intermediate to advanced.
>
> For each kata: spend 30–45 minutes designing before reading the discussion questions. Diagram your solution before writing any text.

---

## Kata 1 — URL Shortener (Beginner)

**System to design:** A URL shortening service similar to bit.ly. Users submit a long URL and receive a short code (e.g., `sht.ly/xK3pQ`). Visiting the short URL redirects to the original.

**Constraints:**
- No user accounts required (anonymous use).
- Short codes are 7 alphanumeric characters.
- A given long URL always maps to the same short code.
- Short URLs do not expire (for now).

**Scale requirements:**
- 100 million URLs created per day (write-heavy is a trick — reads vastly dominate).
- 10:1 read-to-write ratio → 1 billion redirects per day.
- Read latency target: < 10ms p99.
- Data retention: indefinite.

**Discussion questions:**
1. How do you generate short codes? Options: random, hash (MD5/SHA1 prefix), counter-based with base62 encoding. What are the collision risks and trade-offs?
2. Where do you store the mappings? A single SQL table with an index on `short_code` is simple. At what scale do you shard it, and how?
3. The redirect endpoint must be extremely fast. How does a CDN + cache layer change the architecture? What is your cache invalidation strategy?
4. How do you handle duplicate long URLs (same long URL → same short code)? A hash of the long URL as the primary key? An upsert?
5. What analytics data would you capture (clicks, referrer, geography) and how do you avoid slowing down the redirect path?
6. Design the data model. Estimate storage: 500 bytes per record * 100M/day * 365 days * 5 years.

**Solution outline:**
- API service (stateless, horizontally scalable) → writes to a primary DB (PostgreSQL), reads from a replica.
- Cache layer (Redis): `short_code → long_url` with LRU eviction. Cache hit ratio target: 99%+ (Zipf distribution means a small % of URLs handle most traffic).
- Short code generation: base62 encode an auto-incrementing ID from a distributed counter (or use a UUID-based approach with collision retry).
- CDN caches redirects at the edge — HTTP 301 (permanent, browser caches) vs 302 (temporary, server controls). Choose 302 to preserve analytics accuracy.

---

## Kata 2 — Real-Time Chat Application (Intermediate)

**System to design:** A chat application supporting direct messages (DMs) and group channels, with message history and online presence indicators.

**Constraints:**
- Messages must be delivered in order within a conversation.
- Messages must not be lost (at-least-once delivery).
- Offline users receive missed messages when they reconnect.
- Each group channel can have up to 1,000 members.

**Scale requirements:**
- 50 million daily active users.
- Average 30 messages per user per day → 1.5 billion messages per day.
- Peak: 50,000 messages per second.
- P99 message delivery latency: < 500ms.

**Discussion questions:**
1. How do clients maintain persistent connections? WebSockets, Server-Sent Events, or long polling? What are the trade-offs at this scale?
2. With 50M connected clients across multiple server instances, how does a message sent on server A reach a client connected to server B?
3. How do you store messages? Relational DB, NoSQL (Cassandra, DynamoDB), or a combination? Model the schema. How do you paginate message history?
4. How do you implement presence (online/offline/typing indicators)? What TTL would you use on presence keys?
5. Push notifications for offline users: how do you integrate APNs and FCM without coupling them to the core messaging flow?
6. How do you handle message ordering across distributed nodes? Vector clocks? Server-assigned sequence numbers?

**Solution outline:**
- WebSocket servers (stateless) behind a load balancer with sticky sessions.
- Redis Pub/Sub as the message bus: each WebSocket server subscribes to channels. Publishing a message to `channel:<id>` fans out to all subscribed servers, each of which delivers to their local connected clients.
- Message store: Cassandra with partition key `channel_id` and clustering key `timestamp DESC` — optimized for time-range queries.
- Presence: Redis keys `presence:<userId>` with 60s TTL, refreshed by client heartbeat.
- Offline delivery: messages are stored in the DB. On reconnect, the client sends its `lastSeenTimestamp`; the server queries the DB for messages since that point.

---

## Kata 3 — Notification Delivery System (Intermediate)

**System to design:** A multi-channel notification service that sends notifications via email, SMS, push, and in-app channels. Used by multiple internal product teams.

**Constraints:**
- Teams submit notification requests via a REST API or message queue.
- Each notification can target multiple channels simultaneously.
- Users have per-channel preferences (opted-in, opted-out, quiet hours).
- Duplicate notification protection: the same logical event must not send twice.

**Scale requirements:**
- 10 million notifications per day across all channels.
- Email: up to 100ms send latency acceptable.
- SMS: up to 500ms acceptable.
- Push: < 1s acceptable.
- 99.9% delivery success rate target (with retry).

**Discussion questions:**
1. How do you model user notification preferences? A per-user preference table? How do you handle inheritance (org defaults → user overrides)?
2. Each channel has a different provider (SendGrid for email, Twilio for SMS, FCM for push). How do you abstract provider logic so that switching providers requires minimal code changes?
3. How do you implement idempotency? What is your deduplication key and where do you store it?
4. A critical notification (password reset) needs guaranteed delivery. A marketing notification can be dropped on failure. How do you handle different priority tiers?
5. How do you observe delivery failures and trigger retries? Distinguish between transient failures (retry) and permanent failures (invalid phone number — do not retry).
6. Rate limiting: Twilio has per-second limits. How do you smooth out bursts without losing notifications?

**Solution outline:**
- Intake API → event bus (Kafka or SQS). Decoupled from delivery.
- Notification router service: reads events, checks user preferences, fans out to per-channel queues (email queue, SMS queue, push queue).
- Channel workers: one worker type per channel. Each worker is independently scalable.
- Idempotency: `SET notif:<eventId>:<channel>:<userId> 1 NX EX 86400` in Redis before sending.
- Retry: exponential backoff with dead-letter queues after 5 attempts. Dead-letter triggers an alert.
- Priority tiers: separate high-priority queues processed first. Marketing goes to standard queues.

---

## Kata 4 — E-Commerce Order Processing (Intermediate)

**System to design:** The backend system that handles order placement, payment, inventory reservation, and fulfillment for an online retailer.

**Constraints:**
- Inventory must never be oversold (two users buying the last item must not both succeed).
- Payment failure must rollback the inventory reservation.
- The system must handle order processing even if the payment provider is temporarily unavailable (retry later).
- Orders must be idempotent — if the client retries a request, it should not create duplicate orders.

**Scale requirements:**
- 500 orders per second at peak (e.g., flash sale).
- Inventory check + reservation + payment confirmation in < 3 seconds end-to-end.
- Order history must be queryable for up to 7 years.

**Discussion questions:**
1. Inventory reservation: optimistic locking (`UPDATE inventory SET qty = qty - N WHERE qty >= N AND version = ?`) vs. pessimistic locking vs. a dedicated reservation service. What are the trade-offs under high concurrency?
2. The saga pattern vs. a two-phase commit for the distributed transaction (inventory + payment + fulfillment). Why is 2PC rarely used in microservices?
3. If payment succeeds but the fulfillment service is down, what happens? How do you ensure eventual consistency?
4. How do you handle the idempotency of the `POST /orders` endpoint? Where is the idempotency key stored and for how long?
5. Flash sale scenario: 10,000 users simultaneously trying to buy the last 100 units of a product. How does your design prevent the "thundering herd" from overwhelming the DB?
6. Design the event sourcing model for an order (events: `OrderCreated`, `PaymentPending`, `PaymentConfirmed`, `InventoryReserved`, `Shipped`, `Delivered`, `Cancelled`).

**Solution outline:**
- Order service: accepts request, writes `OrderCreated` event, publishes to event bus.
- Inventory service: listens for `OrderCreated`, reserves with optimistic locking. Publishes `InventoryReserved` or `InventoryUnavailable`.
- Payment service: listens for `InventoryReserved`, initiates payment. Publishes `PaymentConfirmed` or `PaymentFailed`.
- Compensating transactions: `PaymentFailed` triggers `InventoryReleased`.
- Idempotency: client-provided `Idempotency-Key` header stored in Redis (or DB) with response body cached for 24h.

---

## Kata 5 — Distributed Rate Limiter (Advanced)

**System to design:** A rate limiter service used by an API gateway to enforce per-client rate limits across a distributed cluster of gateway nodes.

**Constraints:**
- Limits must be enforced globally (not per-node) — a client cannot bypass by round-robining across nodes.
- Latency budget for the rate limit check: < 5ms p99.
- At least 99.9% availability — rate limiting infrastructure must not become a single point of failure.
- Support multiple limit windows: per-second, per-minute, per-hour, per-day (hierarchical).

**Scale requirements:**
- 1 million API requests per second to rate-limit.
- 100,000 unique clients.
- Each check touches Redis.

**Discussion questions:**
1. Fixed window vs. sliding window vs. token bucket vs. leaky bucket — compare the algorithms for this use case.
2. Redis is the central counter store. How do you avoid Redis becoming a bottleneck at 1M RPS? (Cluster, pipelining, local counters with periodic sync?)
3. What happens when Redis is unavailable? Fail open (allow all) or fail closed (block all)? What are the business implications of each?
4. Hierarchical limits: a client has both a per-second limit (100 req) and a per-day limit (1M req). How do you check both atomically?
5. How do you handle Redis clock drift across nodes in a Redis Cluster? Does the sliding window algorithm break?
6. A client complains their requests are being rate-limited incorrectly. How do you build an audit trail?

**Solution outline:**
- Token bucket in Redis: Lua script (atomic) checks and decrements the bucket. Refills happen lazily on each request (compute tokens added since last check using timestamp delta).
- Redis Cluster with 6 nodes (3 primary, 3 replica) for horizontal scaling and HA.
- Local cache layer on each gateway node: batch counters locally for 100ms windows, flush to Redis. Trades accuracy for lower Redis pressure.
- Fail open: on Redis unavailability, allow requests but log every bypass for audit.
- Hierarchical check: single Lua script that checks all windows in one round trip.

---

## Kata 6 — Video Streaming Platform (Advanced)

**System to design:** A video-on-demand (VoD) platform. Users upload videos; other users watch them with adaptive bitrate streaming.

**Constraints:**
- Uploaded videos are transcoded into multiple resolutions (360p, 720p, 1080p).
- Streaming uses HLS (HTTP Live Streaming) with 6-second segments.
- The platform serves 50 countries — latency must be minimized globally.
- Videos must not be hotlinked from external sites (signed URLs).

**Scale requirements:**
- 50,000 video uploads per day (avg 500MB each).
- 10 million concurrent viewers at peak.
- Global p99 stream start time: < 3 seconds.

**Discussion questions:**
1. Upload pipeline: direct browser-to-S3 upload (presigned URL) vs. upload through your backend. What are the trade-offs for a 500MB file?
2. Transcoding: how do you trigger and monitor transcoding jobs? AWS MediaConvert vs. a self-hosted FFmpeg worker pool. When would you choose each?
3. HLS delivery: why serve `.m3u8` manifests and `.ts` segments via a CDN rather than your origin servers?
4. Adaptive bitrate: the player switches quality based on bandwidth. What does the `.m3u8` master playlist contain and how does the player choose the initial quality?
5. Signed URL security: how do you prevent a user from sharing the signed URL with others to bypass paywall? What parameters do you include in the signature?
6. How do you design the view count system? A naive `UPDATE SET views = views + 1` on every play event would be a write bottleneck.

**Solution outline:**
- Upload: presigned S3 URL → raw video in S3 → S3 event triggers transcoding job → MediaConvert produces HLS segments → segments stored in a separate S3 bucket → CloudFront CDN in front.
- Signed URLs: include `userId`, `videoId`, expiry, IP hash in the signature. Verify at the CDN origin (Lambda@Edge).
- View counts: write to a Kafka topic on each play event. A consumer aggregates counts every 60 seconds and writes the rollup to the DB.
- Global delivery: CloudFront with edge locations in all target regions. Origin shield to protect the S3 origin from cache misses.

---

## Kata 7 — Multi-Region Database Replication (Advanced)

**System to design:** A SaaS application that must keep data available and consistent across three regions (US, EU, APAC) with a 99.99% uptime SLA.

**Constraints:**
- Writes are accepted in any region.
- Users should always read their own writes.
- EU data must never physically leave EU data centers (GDPR).
- A full regional outage must be survivable without data loss.

**Scale requirements:**
- 5,000 writes per second globally.
- Read-to-write ratio: 20:1.
- RTO (Recovery Time Objective): 30 seconds.
- RPO (Recovery Point Objective): 0 (zero data loss).

**Discussion questions:**
1. The CAP theorem: with the GDPR constraint and the requirement for cross-region writes, what consistency model can you realistically offer?
2. How do you route writes to the correct region, especially given the GDPR constraint that EU user data must stay in the EU?
3. Conflict resolution: two users in different regions simultaneously update the same record. How do you detect and resolve the conflict? Last-write-wins? CRDTs? Application-level merge?
4. "Read your own writes" consistency: how do you guarantee this when reads may be served from a local replica that is 200ms behind the primary?
5. How do you test regional failover in production (chaos engineering)? What metrics tell you the failover succeeded?
6. Database options: CockroachDB (distributed SQL), YugabyteDB, Cassandra (multi-region), Aurora Global Database — compare them for this use case.

**Solution outline:**
- Data partitioning by `tenantRegion`: EU tenants' data lives only in EU. No cross-region replication for that partition.
- US and APAC: async replication with conflict detection. Conflicts are rare (different users) — LWW with timestamps works for most cases; explicit conflict queues for critical data.
- Read routing: session-level sticky routing to primary after any write (30-second window). After that, serve from local replica.
- Aurora Global Database: primary in US-EAST, read replicas in EU-WEST and APAC. Replication lag < 1 second. Failover promotes a replica in < 30 seconds.
- GDPR: separate Aurora cluster in EU-WEST for EU tenants. Not replicated outside EU.

---

## Kata 8 — Search-as-a-Service (Intermediate)

**System to design:** An internal search service used by multiple product teams. Teams send documents (JSON objects) and receive ranked full-text search results.

**Constraints:**
- Teams register "indexes" with a schema. Each index is isolated from others.
- Search must support: full-text, filters (exact match, range), sorting, and faceted counts.
- Near-real-time indexing: a document indexed via API must be searchable within 5 seconds.
- Read/write isolation: heavy write bursts (bulk re-index) must not degrade search latency.

**Scale requirements:**
- 20 product teams, each with 1–50M documents.
- 500 search queries per second at peak.
- Average query latency target: < 100ms.
- Bulk re-index: up to 5M documents per hour.

**Discussion questions:**
1. Elasticsearch vs. OpenSearch vs. Typesense vs. a custom solution — when would you build custom vs. adopt an existing engine?
2. How do you handle schema evolution? A team adds a new field to their index. Does this require a full re-index? How do Elasticsearch dynamic mappings help and hurt?
3. How do you implement multi-tenancy with Elasticsearch? One cluster per team, one index per team (same cluster), or one index with a `tenantId` field? Compare isolation, cost, and operational overhead.
4. Read/write isolation: Elasticsearch re-indexing (index refresh) is expensive. How do alias-based zero-downtime re-indexing work?
5. How do you design the API so that callers are decoupled from Elasticsearch internals (so you can swap the engine later)?
6. Relevance tuning: a product team complains their search results are irrelevant. What tools and workflows do you provide for them to tune relevance without your involvement?

**Solution outline:**
- One Elasticsearch index per team (isolated, simple). Alias-based access: `alias: <team>-search` pointing to the current index version.
- Write path: REST API → Kafka topic per team → indexing workers (fan-out). Workers batch-write to Elasticsearch (bulk API) every 2 seconds — achieves the 5-second NRT target.
- Read path: REST API → query builder → Elasticsearch. The query builder translates team-facing query language to Elasticsearch DSL.
- Re-index without downtime: write to new index `v2` while serving reads on `v1`. When `v2` is ready, atomically swap the alias. Then delete `v1`.
- Multi-tenancy isolation: index-level access with API key per team. Each team's API key only has permissions on their index.

---

## Kata 9 — Financial Transaction Ledger (Advanced)

**System to design:** A ledger system for recording financial transactions with correctness guarantees. Used by a fintech company to track user account balances.

**Constraints:**
- Every debit must have a corresponding credit (double-entry bookkeeping).
- Balances must be consistent at all times — no overdrafts unless the account is explicitly configured for it.
- Every transaction must be auditable: who authorized it, when, from which system.
- The ledger is append-only — no updates or deletes to transaction records.

**Scale requirements:**
- 5,000 transactions per second at peak.
- Balance query latency: < 50ms.
- Audit trail retention: 10 years.
- Regulatory requirement: daily reconciliation report must balance to zero.

**Discussion questions:**
1. Double-entry bookkeeping: model the schema. What tables do you need? What invariants must hold at the DB level?
2. Balance calculation: sum all transactions for an account vs. a running balance column. What are the performance implications? How do you handle large accounts with millions of entries?
3. How do you prevent concurrent transactions from overdrafting an account? Pessimistic row-level lock? Optimistic lock with version column? A serializable isolation level?
4. The ledger is append-only. How does this affect backup, archival, and query performance over time (as the table grows to billions of rows)?
5. Event sourcing is a natural fit here. How does the ledger system map to the event sourcing pattern? What is the "aggregate" and what are the "events"?
6. Reconciliation: at end of day, the sum of all debits must equal the sum of all credits. How do you implement this check efficiently without locking the table?

**Solution outline:**
- Schema: `accounts(id, type, currency, created_at)`, `journal_entries(id, account_id, amount, direction ENUM(debit,credit), txn_id, created_at, metadata JSONB)`, `transactions(id, description, initiated_by, created_at)`.
- Balance query: materialized balance in a `account_balances` table, updated via DB trigger or application code inside the same transaction. Raw `SUM()` as a fallback for audit.
- Concurrency: `SELECT ... FOR UPDATE` on the account row before inserting journal entries. This serializes writes to the same account.
- Archival: partition `journal_entries` by year. Cold years (> 2 years old) moved to cheap object storage (S3) but remain queryable via Athena or a data warehouse.
- Reconciliation: a nightly aggregate job (not a live query) that sums by transaction, checks debit = credit, writes a `reconciliation_report` record.

---

## Kata 10 — IoT Data Ingestion Pipeline (Advanced)

**System to design:** A data ingestion platform that receives telemetry data from 5 million IoT devices (sensors, vehicles, industrial equipment) in real time.

**Constraints:**
- Devices send data over MQTT or HTTP.
- Data must be persisted and queryable by device ID and time range.
- Anomaly detection rules run against the incoming stream and trigger alerts within 10 seconds.
- Devices may send duplicate readings (at-least-once from device side) — deduplication is required.

**Scale requirements:**
- 5 million devices, each sending one reading every 30 seconds.
- Peak ingestion: 200,000 messages per second.
- Storage: 50 bytes per reading * 200K/sec * 86,400 sec/day = ~864 GB/day.
- Time-series query p99: < 200ms for a 24-hour range query on a single device.

**Discussion questions:**
1. MQTT vs. HTTP for device communication: when does MQTT's persistent connections and QoS levels make it the better choice?
2. At 200K messages/second, how do you scale the ingestion layer without a single broker becoming a bottleneck?
3. Time-series databases (TimescaleDB, InfluxDB, Apache Cassandra, ClickHouse) — compare their write and read characteristics for this workload.
4. Deduplication: each message has a device-generated sequence number. How do you deduplicate without storing all sequence numbers forever?
5. Anomaly detection: stateful stream processing (Flink, Spark Streaming, Kafka Streams) vs. simple threshold rules applied per message. When is each appropriate?
6. Device firmware sends the wrong timestamp (device clock drift). How do you handle and correct this in the ingestion pipeline?

**Solution outline:**
- Ingestion layer: MQTT broker cluster (EMQX, HiveMQ) for MQTT devices; Nginx + Fastify for HTTP devices. Both publish to Kafka (200 partitions for 200K msg/s at 1KB/msg).
- Kafka consumers: ingestion workers validate, deduplicate (Bloom filter per device for recent 1-hour window), and write to the time-series DB.
- Time-series store: TimescaleDB (PostgreSQL extension) with hypertables partitioned by time. Compression policies for data older than 7 days.
- Anomaly detection: Kafka Streams application reads the raw topic, applies windowed rules (e.g., temperature > 100°C for 5 consecutive readings), emits to an `alerts` topic. Alert consumer sends notifications.
- Deduplication: per-device Bloom filter in Redis (TTL 1 hour). Accept some false positives (re-sending is idempotent — device data is the same).
- Clock drift correction: record `device_timestamp` and `server_timestamp`. Use `server_timestamp` as the primary time axis for queries. Expose `clock_drift_seconds` as a metric.
