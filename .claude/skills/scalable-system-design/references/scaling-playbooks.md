# Scaling Playbooks

Recurring problem shapes and the designs that solve them. Each starts with the trigger condition — if you don't meet it, don't build it.

## 0. The scaling ladder

Add a rung only when the previous one is the measured bottleneck:

1. Single server (app + DB together)
2. Split app tier / data tier so they scale independently
3. Load balancer + ≥2 app servers (public IP on the LB; app servers private)
4. Read replicas (reads ≫ writes, so replicas ≫ primaries)
5. Cache tier
6. CDN for static assets
7. **Stateless app tier** (session state moved out) → autoscaling becomes possible
8. Multiple datacenters / regions (geo-routing, failover)
9. Message queue → decouple producers from consumers, scale workers separately
10. Logging, metrics, automation, CI/CD
11. Shard the data tier; move the parts that don't need relational guarantees off it

Most products never need rungs 8–11. Say which rung you're on and which bottleneck justifies the next.

## 1. Fan-out: feeds, timelines, notifications

The volume is rarely the problem — the **fan-out** is. Worked example: 4.6K tweets/s becomes 345K timeline writes/s at 75 followers average; a 30M-follower account makes one post cost 30M writes.

| Strategy | Good | Bad |
|---|---|---|
| **Fan-out on write** (push, precompute inboxes) | Fast reads, real-time | Hot-key explosion for high-follower accounts; wasted work for inactive users |
| **Fan-out on read** (pull, merge at query) | No waste, no hot key | Slow reads |
| **Hybrid** ← the answer | Push for normal users, pull for celebrities | Two paths to maintain |

Rules:
- Decision rule: **precompute unless the fan-out for that entity is extreme.**
- Fan-out-on-write works below a small-group threshold (WeChat caps groups at 500); above it, switch to pull/lazy fetch.
- Store **`<post_id, user_id>` pairs in the feed cache, not full objects**; hydrate on read from separate caches. Bound the per-user list.
- Filter at the edge, not at the source: publish once, let each subscriber's handler decide whether it's relevant.
- Bidirectional, capped relationships (friends, max 5,000) have no celebrity problem at all — check whether your graph is follower-shaped or friend-shaped before designing for it.
- Presence: heartbeat every ~5 s, mark offline after ~30 s of silence — a raw disconnect event produces flapping on flaky mobile networks. Above small-group scale, **fetch presence lazily** (on entry / manual refresh) instead of pushing every change; one status flip in a 100K-member group is 100K events.

## 2. Queues and streams

**Only add a broker when you can name the reason:** multiple independent consumers, buffering a spike, decoupling failure domains, or independent scaling of producer and consumer. A single consumer with no fan-out requirement does not need one. Don't add a broker speculatively.

**Traditional queue vs log-based stream:**
- Queue (SQS, RabbitMQ): message deleted on ack, load-balanced across consumers, redelivery reorders messages. Good for **expensive, slow, independent tasks** needing per-message parallelism. Assume short queues; long backlogs degrade throughput.
- Log (Kafka, Kinesis): append-only partitioned log, monotonic offsets, **total order within a partition only**, replayable, retained for days/weeks. Good for **high throughput, ordering, replay, and many independent consumers**.

Design rules:
- **Partition by the aggregation key** so one consumer sees all events for that key. **Over-allocate partitions up front** — a partition can be consumed by only one consumer per group, so consumers beyond the partition count sit idle. Increasing partition count is cheap; decreasing is not, and it remaps keys.
- **Batch everywhere** — producer, broker, consumer. Small I/O is the enemy of throughput. The tradeoff is explicit: bigger batch = more throughput, more latency.
- **Pull beats push** for consumers: the consumer controls its rate, mixes real-time and batch readers off the same log, and batches naturally. Use long polling so idle pulls don't spin.
- **Delivery semantics, chosen per use case:** at-most-once (commit offset before processing) for metrics; **at-least-once + idempotent consumer** as the default; exactly-once only for money.
- One queue per notification type / failure class, so one provider outage doesn't block the others and each class gets its own retry policy.
- **Invalid work goes to an error/dead-letter queue**, never back onto the main one. Monitor consumer lag as the primary health signal.
- If a payload is too big for the queue, put it in object storage and enqueue the **reference**.
- **Push volatile high-volume writes into a log so new consumers never touch the write path** — one ingest stream, many independent consumers (live updates, analytics, ML, reindexing) each owning its own store.
- Consumer-group rebalance with hundreds of consumers takes **minutes** — add consumers off-peak.

## 3. Real-time push: pick the transport by traffic shape

| Transport | Use when |
|---|---|
| Plain HTTP request/response | Everything that isn't real-time. Signup, profile, settings — don't put these on the socket. |
| **Long polling** | One-directional, infrequent, non-bursty updates (file sync notifications) |
| **SSE** | Server→client only, streaming |
| **WebSocket** | Bidirectional, bursty, real-time (chat, live collaboration, navigation) |
| Mobile push | Offline delivery only — payload capped (4 KB on iOS), no web |

Consequences of persistent connections: the socket tier is **stateful**, so it needs connection management, service discovery to pick a server, sticky routing, draining on deploy, and over-provisioning rather than autoscaling. Keep the stateless REST tier separate from the stateful socket tier even when they sit behind the same load balancer. Mass reconnect after a crash is inherently slow — plan for the thundering herd.

Fairness note: if a publisher fans out in connection order, clients will race to connect first. Multicast or randomized subscriber order removes the incentive.

## 4. Rate limiting

Client-side limits are unenforceable. Enforce server-side or at the gateway.

| Algorithm | Property |
|---|---|
| **Token bucket** | Simple, memory-efficient, **allows bursts** (right when flash-sale bursts are legitimate). Two params to tune. |
| Leaking bucket | Stable outflow; a burst of stale requests can starve recent ones. |
| Fixed window counter | Cheapest; **edge-of-window bursts allow 2× the quota**. |
| Sliding window log | Exact in any rolling window; memory heavy (even rejected requests stored). |
| Sliding window counter | Smooths spikes, memory-efficient, approximate (measured 0.003% misclassified over 400M requests). |

Counters live in an in-memory store, never the primary DB. Read-check-increment is a race — use an atomic script/structure, not a lock. Never use sticky sessions to "solve" cross-instance sync; use a shared counter store. Respond **429** with `X-Ratelimit-Remaining` / `-Limit` / `-Retry-After`. Distinguish hard vs soft limits, and monitor whether the rules are too strict.

## 5. Geo / spatial

- A 2-D range query (`lat BETWEEN … AND lng BETWEEN …`) is a trap: the index helps one dimension and you intersect two huge sets. **Map 2-D → 1-D** (geohash / Hilbert curve) so a single index works.
- **Geohash**: no tree to build, trivial updates, natural radius search — but fixed cell size per precision, so it can't adapt to density. Precision 4 = 39×20 km, 5 = 4.9 km, 6 = 1.2 km.
- **Quadtree**: adapts to density, supports k-nearest — but in-memory, O(log n) updates with locking, and rebuild-on-startup (an availability risk).
- **S2 / Hilbert**: best for geofencing and arbitrary region cover.
- **Boundary bug**: two nearby points can share no prefix (opposite sides of a meridian), so `LIKE 'prefix%'` misses neighbors. **Always query the cell plus its 8 neighbors.** For thin results, drop the last digit and re-query (the cell grows 32×).
- Index table shape: one row per `(cell, entity_id)` compound key — not a JSON array of IDs per cell, which forces read-scan-lock-write on every insert.
- **Tiling is the universal spatial scaling primitive**: precompute static tiles per zoom level, serve from a CDN, and give the graph/data multiple levels of detail with cross-level edges so long-range queries run on coarse tiles.
- **Hierarchical keys as a pre-filter**: store each active entity's current cell *plus its chain of ancestor cells*. "Who is affected by this incident?" becomes a containment test that eliminates most rows instantly, instead of a scan.

## 6. Leaderboards / ranking

Relational is the honest starting point and fails predictably: computing an arbitrary user's rank requires sorting every row, and caching doesn't help because the data changes constantly. `ORDER BY score DESC LIMIT 10` fixes top-N only. RDBMS is viable for a **batch** leaderboard.

A sorted-set structure (hash map + skip list) gives O(log n) insert/update/rank with automatic ordering. One key per time segment (`leaderboard_2026_02`) makes monthly resets and expiry trivial.

**The score is set server-side by the service that validated the event, never by the client.**

## 7. Media, uploads, and object storage

- Two parallel flows on upload: bytes → blob storage (via **pre-signed URL**, so your API never proxies the payload) and metadata → API → DB. A completion event/queue joins them.
- Chunk on the client (independently playable/uploadable units) for parallel, resumable uploads.
- Model the processing pipeline as a **DAG of small single-purpose stages** (inspect, transcode per variant, thumbnail, watermark) with queues between stages, driven by config so different content types get different pipelines.
- Exploit the long tail for cost: CDN the popular content, serve the cold tail from origin, generate fewer variants for unpopular content, keep regionally-popular content regional. **Analyze real access patterns before optimizing.**
- Error playbook: recoverable (one segment failed) → bounded retries then error; non-recoverable (malformed input) → stop the pipeline and report.

## 8. Search

Characterize first — scoped to one user or global? sorted by relevance or by attributes? is lag acceptable? Email-style search (per-user, exact, near-real-time, writes vastly outnumbering reads) is a completely different system from web search.

Rule of thumb: small scale → an off-the-shelf search engine partitioned by the natural entity key. Large scale → the same, plus a dedicated team. Extreme scale → search embedded in the primary store. The index is rebuildable from primary storage, so it carries no data-loss risk — but it is a second system with a second copy of the data and a consistency problem.

Precomputed top-k structures (a prefix tree with the top-k cached at every node) turn autocomplete into an O(1) read. **Don't update it per query** — rebuild from append-only logs on a cadence tuned to how fresh the results must be, and swap. Sample the logs at scale.

## 9. Batch and offline processing

- **Immutable inputs, outputs replace wholesale, no side effects.** This is what makes retries safe, rollback possible (repoint at the previous output), and experimentation cheap. Read/write-in-place jobs lack this: rolling back the code does not roll back the corrupted data.
- Build the derived dataset in the job, write immutable files, then **bulk-load and atomically swap**. Do not write record-by-record to the production DB from a job — it's slow, it overwhelms the DB, and partial output becomes externally visible.
- Never do a per-record remote lookup inside a bulk job; take a snapshot and join locally.
- **Skew is the killer**: one hot key stalls the whole stage because the next stage waits for the slowest worker. Fixes: route hot-key records to random workers and replicate the other side, or two-stage (pre-aggregate randomly, then combine).
- Don't reach for a cluster until a single machine actually fails — command-line tools on one box beat a small cluster more often than people expect.
- Prefer **one stream engine with replay** (Kappa) over separate batch and streaming code paths (Lambda). If you replay, do it through a dedicated instance so reprocessing never starves real-time.

## 10. Hot-path / low-latency systems

Only relevant when a measured latency requirement demands it — but the principles generalize:

- **Split flows by SLA** and push everything non-critical off the critical path. In an exchange design even *logging* comes off the path; reporting and analytics subscribe to the event stream asynchronously.
- Count the hops: a same-DC round trip is ~500 µs, so a few hops is already milliseconds, and a disk-backed event store is tens of ms. Collapsing components onto one box with shared-memory IPC took an exchange design from tens of ms to tens of µs.
- Single-writer, pinned-core, busy-poll loops and pre-allocated lock-free ring buffers exist to remove context switches, lock contention, and GC pressure — the things that create tail latency.
- **Measure p99 and p99.99, not averages.** Stable tail latency *is* the product feature. Chase large fluctuations to their cause (GC pauses are the classic).
- "Some large exchanges run almost everything on a single gigantic server, or even one process." **Distribution is a cost, not a virtue.**
</content>
