---
name: scalable-system-design
description: Use BEFORE building or reviewing any backend feature, data model, service, pipeline, endpoint, socket flow, or job that will carry real traffic. Triggers on "design", "architecture", "will this scale", "scalable", "high traffic", "capacity", "QPS", "load", "sharding", "partition key", "replication", "caching", "cache invalidation", "message queue", "worker", "fan-out", "real-time", "websocket", "rate limit", "idempotency", "exactly-once", "distributed transaction", "eventual consistency", "microservice", "split this service", "bottleneck", "slow endpoint", "p99", "tail latency", "should we use Redis/Kafka/a queue", or any request to plan a system before coding it. Distilled from DDIA (Kleppmann), System Design Interview Vol 1+2 (Alex Xu), Building Microservices (Newman), Fundamentals of Software Architecture (Richards & Ford).
---

# Scalable System Design

Design so today's build survives 10x without a rewrite — **and without paying for scale that never arrives.** Both failures are real. Over-engineering kills more early systems than load does.

## The gate: estimate before you architect

**Do the arithmetic first. It decides the design.** Never pick a technology, a shard key, or a queue before you have numbers.

```
1. Daily volume  → QPS      = events/day ÷ 10^5   (86,400 s ≈ 10^5)
2. Peak QPS      = 2× avg (steady product) … 5× avg (spiky/market/event-driven)
3. Storage       = bytes/record × records/day × retention (× replication factor)
4. Read:write ratio, and the 2-4 queries that MUST be fast
5. Working-set size — does the hot data fit in one machine's RAM?
```

Then apply the sizing verdict:

| Verdict | Response |
|---|---|
| Fits one node, one DB | **Build it that way.** Replicas for HA, not capacity. Say so explicitly. |
| Read-bound, data fits | Cache or read replicas. **Not** sharding. |
| Write-bound or data exceeds one node | Now shard — pick the key from the dominant query. |
| Fan-out is the bottleneck (not volume) | Fan-out-on-write vs on-read vs hybrid — see `references/scaling-playbooks.md`. |

Real examples from the books: 73M rows + 3 TPS ⇒ single DB. 200M businesses ⇒ 1.71 GB index ⇒ one server. Stack Overflow ran 10M monthly visitors on **one** master DB. A geo index that fits in RAM does not need sharding, and adding a cache in front of a DB whose working set already fits in RAM buys ~nothing.

→ Numbers, tables, and worked templates: `references/estimation.md`

## Order of work

1. **Scope** — functional vs non-functional requirements, separately and explicitly. Non-functional means *numbers*: p99 latency budget, availability target, durability, correctness class, data-residency. "Fast" is not a requirement.
2. **Estimate** (the gate above).
3. **Data before code** — write the access patterns down, then design the schema/keys/indexes to serve them. The partition key is chosen by the dominant query, never by what looks tidy.
4. **Draw the boxes** — clients, API, stores, cache, queue, workers. Walk two real use cases through it and find the edge cases.
5. **Deep-dive the 2-3 genuinely hard parts** only. Everything else is boring on purpose.
6. **Failure pass** — for every out-of-process call: what if it's slow? dead? duplicated? For every UI surface: what do users see when this dependency is down? That answer is a *product* decision.
7. **Evolution pass** — what breaks at 10x, what you'd rewrite at 100x, and which metric tells you you're getting close.

Output a short design record: requirements + numbers, chosen design, **rejected alternatives and why**, known failure modes, and the metric that says "revisit this."

## Laws (violate only with a written reason)

1. **Everything is a tradeoff.** If a proposal looks free, you haven't found the cost yet. Name it.
2. **Estimate before choosing.** No technology decision without the arithmetic behind it.
3. **If it fits on one machine, keep it there.** Distribute for fault tolerance and geography — not reflexively for scale. A single box does an enormous amount today.
4. **Design for ~10x growth; plan to rewrite before 100x.** Front-loading massive-scale work for load that may never come, while the product is still unproven, is the classic fatal move. Needing to rearchitect is a sign of success.
5. **Latency is a distribution.** Specify and measure **p95/p99/p99.9**, never averages. Measure client-side. Never average percentiles — sum histograms.
6. **Slow is worse than dead.** A dying-but-responding dependency cascades further than one that fails fast. Timeouts on every out-of-process call, bulkhead the pools, break the circuit.
7. **The database is the source of truth; the cache is a hint.** Always re-validate the invariant at the DB. Cache↔DB inconsistency is acceptable when the DB does final validation — reason about it as UX, not purity.
8. **Enforce invariants in the database**, not in the client or app layer. Unique constraint, CHECK constraint, atomic operator. Check-then-update across a transaction boundary is always a race.
9. **Retry + idempotency key = exactly-once.** Never assume a framework gives it to you once effects cross an external boundary.
10. **Never dual-write.** One store leads; everything else derives from its change log. Concurrent dual writes diverge permanently and raise no error.
11. **Immutable log + derived views.** Immutable inputs make retries safe, rollback possible, and replay a debugging tool.
12. **Never trust wall-clock time for ordering.** Use logical/monotonic counters, version vectors, or sequence IDs. Last-write-wins silently deletes acknowledged writes.
13. **Human error causes most outages.** Fast rollback, staged rollout, good telemetry, and a realistic sandbox beat another layer of redundancy.
14. **Coupling costs more than duplication.** Across service boundaries, relax DRY. A shared database between services is the top anti-pattern.
15. **Don't shard, cache, queue, or split a service reflexively.** Each has an explicit "only if"; state which one you met.

## Red flags — stop and re-derive

- A technology chosen before any number was computed.
- Sharding/microservices/Kafka proposed for a dataset that fits one node.
- No p99 target, or targets stated as averages.
- No stated read:write ratio.
- Partition key that the dominant query does not carry.
- A hot key / celebrity account assumed away.
- Cache with no TTL, no eviction policy, no stampede plan, and no answer for "what if it's empty?"
- `mod N` hashing for anything that will change N.
- Timestamp/UUID-prefixed keys creating a write hot spot on "today".
- Windowing or ordering by processing time when event time is available.
- Client-supplied score, price, ID, or timestamp trusted as authoritative.
- Any out-of-process call with no timeout.
- New service that cannot be deployed without deploying another one.
- Distributed transaction reached for before co-locating the tables was considered.
- Retries with no cap, no backoff, and no dead-letter path.
- "We'll add monitoring later." Correlation IDs are agony to retrofit.

## Quick tables

**Latency (order of magnitude):** memory ref 100 ns · 1 MB seq from RAM 3 µs · SSD random read 16 µs · **same-DC round trip 500 µs** · disk seek 2 ms (HDD 10 ms) · 1 MB from disk ~1 ms (HDD 30 ms) · **cross-continent RTT 150 ms**. Consequence: avoid seeks, compress before the network, and count your network hops — several hops is already milliseconds.

**Availability:** 99% = 3.65 d/yr · 99.9% = 8.8 h/yr · 99.99% = 52.6 min/yr (**8.6 s/day**) · 99.999% = 5.3 min/yr. Every nine changes what "recovery" has to mean.

**Quorum:** `w + r > n` for overlap. n=3, w=r=2 tolerates 1 loss; n=5, w=r=3 tolerates 2. Quorums are probabilistic, not absolute — sloppy quorums, clock skew, and partial writes still yield stale reads.

**Durability:** 3× replication ≈ 6 nines, 200% storage overhead. Erasure coding (8+4) ≈ 11 nines, 50% overhead, but parity math on every write and fan-out reads. Replication for latency-sensitive; EC when storage cost dominates.

## Reference files

Load the one that matches the problem — don't read them all.

| File | Use when |
|---|---|
| `references/estimation.md` | Sizing, BOTE templates, latency/availability/capacity numbers |
| `references/data-and-storage.md` | Schema, indexes, storage engines, partitioning, replication, caching, consistency & isolation levels |
| `references/distributed-correctness.md` | Idempotency, exactly-once, transactions/Saga/TCC, clocks, ordering, consensus, event sourcing/CQRS |
| `references/resilience-and-operations.md` | Timeouts, circuit breakers, bulkheads, degradation, rollout, monitoring, testing |
| `references/scaling-playbooks.md` | Fan-out/feeds, queues & streams, real-time push, geo/spatial, rate limiting, leaderboards, media & object storage, search |
| `references/service-boundaries.md` | Splitting a monolith, bounded contexts, service integration, versioning, team/ownership shape |
| `references/glitchover-context.md` | **Read this one for any work in this repo** — what the stack actually is and what that constrains |

## Working style

- State assumptions out loud; if two readings of the requirement lead to different designs, ask.
- Give a recommendation, not a survey. Then name the runner-up and why it lost.
- Prefer the boring option until a measured number forbids it. "Use Kafka/Redis/a new service" needs a number behind it.
- Scale the design to the ask: a startup design ≠ a 100M-user design, and pretending otherwise is the most common failure mode in this whole discipline.
</content>
</invoke>
