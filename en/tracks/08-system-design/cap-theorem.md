# CAP Theorem

## Overview

The CAP theorem states that a distributed system can provide at most two of three guarantees simultaneously: Consistency, Availability, and Partition tolerance. Formulated by Eric Brewer in 2000 and proven by Gilbert and Lynch in 2002, it is the foundational constraint that shapes every distributed database and distributed service design decision. Understanding CAP means understanding *why* your database behaves the way it does under failures, and why "just make it consistent and available" is not always an option.

---

## Prerequisites

- Basic database concepts (transactions, reads, writes)
- Understanding of networks and what "network partition" means
- Familiarity with terms like replication and distributed systems at a high level

---

## Core Concepts

### The three properties

**Consistency (C):** Every read receives the most recent write, or an error. All nodes see the same data at the same time.

**Availability (A):** Every request receives a response (not an error), but it may not contain the most recent write.

**Partition Tolerance (P):** The system continues to operate even when network partitions (dropped messages between nodes) occur.

### Why you can only have two

In a distributed system communicating over a real network, network partitions are not theoretical — they happen. Hardware fails, network cables break, data centers lose connectivity. This means **Partition Tolerance is not optional** for any system running across more than one machine. The real choice is between **CP** and **AP**.

```
      ┌─────────────┐
      │    Partition │
      │   Tolerance  │
      │      (P)     │
      └──────┬───────┘
             │
    ┌─────────┴─────────┐
    │                   │
┌───▼───┐           ┌───▼───┐
│  CP   │           │  AP   │
│       │           │       │
│Consist│           │ Avail │
│ency + │           │ability│
│Parttn │           │+ Parttn│
└───────┘           └───────┘
```

### CP systems

When a partition occurs, the system refuses to serve stale data. It returns an error rather than an inconsistent response. Users experience unavailability; data is always correct.

Examples: HBase, ZooKeeper, etcd, most relational databases in single-primary mode.

### AP systems

When a partition occurs, the system continues to serve requests, potentially returning stale data. All nodes remain available; data may be temporarily inconsistent.

Examples: DynamoDB (with eventual consistency), Cassandra, CouchDB, DNS.

### The PACELC extension

CAP is about failures. PACELC extends it to normal operation: even when there is no partition (P is not in play), there is a latency-consistency trade-off. Do you want low Latency (L) or strong Consistency (C)?

---

## Hands-On Examples

### Demonstrating eventual consistency in Redis

Redis Cluster uses AP semantics. During a failover, reads from a replica may be stale:

```typescript
import { createClient } from 'redis';

const primary = createClient({ url: 'redis://primary:6379' });
const replica = createClient({ url: 'redis://replica:6380' });

await primary.connect();
await replica.connect();

// Write to primary
await primary.set('user:42:name', 'Alice');

// Immediately read from replica — may return old value or null
// (replication is asynchronous, ~1-5ms lag in normal operation)
const name = await replica.get('user:42:name');
console.log(name); // could be null or 'Alice' depending on replication lag
```

**Mitigation for reads that must be fresh:** read from the primary. Accept the latency cost.

```typescript
// For critical reads, always go to primary
const freshName = await primary.get('user:42:name'); // always consistent
```

### Read-your-writes consistency

A common pattern that bridges AP and CP: after a write, the same client reads from the primary for a short window.

```typescript
// src/lib/db-routing.ts
import { PrismaClient } from '@prisma/client';

const primaryClient = new PrismaClient({ datasources: { db: { url: process.env.DATABASE_PRIMARY_URL } } });
const replicaClient = new PrismaClient({ datasources: { db: { url: process.env.DATABASE_REPLICA_URL } } });

// After a write, store a timestamp. Route reads to primary within the replication lag window.
const POST_WRITE_PRIMARY_WINDOW_MS = 1000;
const lastWriteByUser = new Map<string, number>();

export function markWrite(userId: string): void {
  lastWriteByUser.set(userId, Date.now());
}

export function getDbClient(userId?: string): PrismaClient {
  if (!userId) return replicaClient;

  const lastWrite = lastWriteByUser.get(userId) ?? 0;
  const timeSinceWrite = Date.now() - lastWrite;

  return timeSinceWrite < POST_WRITE_PRIMARY_WINDOW_MS
    ? primaryClient   // still within the post-write window — use primary
    : replicaClient;  // safe to use replica
}
```

### Strong consistency with PostgreSQL transactions

Postgres (single-primary) is a CP system. Within a transaction, all reads are consistent:

```typescript
// All reads and writes in this transaction see a consistent snapshot
await db.$transaction(async (tx) => {
  const account = await tx.account.findUnique({
    where: { id: accountId },
    // SELECT ... FOR UPDATE — prevents concurrent modifications
    // In Prisma: use raw query or optimistic locking
  });

  if (!account || account.balance < amount) {
    throw new Error('Insufficient funds');
  }

  await tx.account.update({
    where: { id: accountId },
    data: { balance: { decrement: amount } },
  });

  await tx.account.update({
    where: { id: toAccountId },
    data: { balance: { increment: amount } },
  });
});
// If any step fails, the entire transaction rolls back — no partial transfers
```

### Quorum reads and writes (Cassandra-style)

In a Cassandra cluster with replication factor 3:
- Write quorum: write to 2 of 3 nodes (W=2)
- Read quorum: read from 2 of 3 nodes (R=2)
- W + R > N (2+2=4 > 3) guarantees reading the latest write

```typescript
// Simulating quorum logic (conceptual — Cassandra drivers handle this)
const REPLICATION_FACTOR = 3;

function quorumCount(n: number): number {
  return Math.floor(n / 2) + 1; // majority
}

const writeQuorum = quorumCount(REPLICATION_FACTOR); // 2
const readQuorum = quorumCount(REPLICATION_FACTOR);  // 2

// If writeQuorum + readQuorum > REPLICATION_FACTOR, reads always see latest writes
console.log(writeQuorum + readQuorum > REPLICATION_FACTOR); // true → strong consistency
```

---

## Common Patterns & Best Practices

**Choose CP when:**
- Financial transactions (banking, payments, inventory)
- User authentication (incorrect login state is a security issue)
- Distributed locks and leader election (ZooKeeper, etcd)

**Choose AP when:**
- Shopping carts (temporary inconsistency is acceptable)
- Social media feeds (showing slightly stale posts is fine)
- DNS (staleness is acceptable for seconds or minutes)
- Analytics counters (eventual accuracy is sufficient)

**Design for the failure you expect:**
- If your nodes are in a single data center, network partitions are rare — prioritize consistency
- If your nodes span multiple data centers, partitions are expected — design for AP

**Mitigate AP inconsistency with:**
- Read-your-writes sessions
- Monotonic reads (never return data older than what you showed the client before)
- Bounded staleness (replica lag guaranteed to be under N seconds)

---

## Anti-Patterns to Avoid

- Building a financial system on an AP database without understanding the implications
- Assuming your SQL database is always consistent — in a primary/replica setup, replicas lag
- Choosing "strong consistency everywhere" without measuring the latency cost
- Ignoring the possibility of network partitions because "it never happens" — it does

---

## Debugging & Troubleshooting

**"Users are seeing stale data after an update"**
You are reading from a replica that has not yet replicated the write. Solutions: route post-write reads to primary, add replica lag monitoring, or increase consistency level at the cost of latency.

**"Distributed lock is not working — two nodes are both leaders"**
This is a split-brain problem from a partition. Use etcd or ZooKeeper for distributed locking — they are CP and use the Raft consensus algorithm to guarantee only one leader.

**"My Cassandra cluster is rejecting writes during a node failure"**
Check your write consistency level. `QUORUM` will fail if too many nodes are down. `ONE` or `ANY` will succeed but with lower consistency guarantees.

---

## Real-World Scenarios

**Scenario: Payment processing system design**

Requirements: never double-charge, never allow a transfer when funds are insufficient.

Decision: **CP database** (PostgreSQL with transactions). Availability can be reduced during a partition — an error message is better than an incorrect charge.

Architecture:
- All financial writes go through a single Postgres primary
- No read replicas for balance checks — always read from primary
- Wrap every multi-step operation in a database transaction
- Use database-level constraints as the last line of defense

**Scenario: Social feed system design**

Requirements: show users recent posts, acceptable to show posts from 1-2 seconds ago.

Decision: **AP database** (Cassandra or DynamoDB with eventual consistency). Showing a slightly stale feed is acceptable; unavailability is not.

Architecture:
- Write posts to Cassandra with `LOCAL_QUORUM` (consistent within a data center)
- Read feeds from any replica
- Accept that a post may take 2-3 seconds to appear for all users

---

## Further Reading

- [Eric Brewer's original CAP conjecture talk](https://people.eecs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf)
- [Gilbert and Lynch: Brewer's Conjecture and Feasibility of Consistent, Available, Partition-Tolerant Web Services](https://dl.acm.org/doi/10.1145/564585.564601)
- [Kleppmann: Designing Data-Intensive Applications](https://dataintensive.net/) — Chapter 9
- [PACELC theorem](https://en.wikipedia.org/wiki/PACELC_theorem)
- Track 08: [Database Scaling](database-scaling.md)
- Track 08: [Distributed Systems](distributed-systems.md)

---

## Summary

The CAP theorem is not a design choice — it is a constraint. Because network partitions happen in any distributed system, you must choose between Consistency (errors during partitions) and Availability (stale data during partitions). For financial, authentication, and coordination systems, choose CP and accept that errors are better than incorrect data. For social, analytics, and content delivery systems, choose AP and design mitigations (read-your-writes, monotonic reads, bounded staleness) to minimize the impact of inconsistency. The important thing is to make the choice deliberately — most incidents attributed to "data corruption" are actually unintended CP → AP transitions under load.
