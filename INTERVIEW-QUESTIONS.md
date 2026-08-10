# System Design — Interview Q&A Bank

Trade-off answers, not definitions — interviewers probe "why" three levels deep.

## Vocabulary

**Q: What does CAP actually force you to choose?**
A: Only during a partition: consistency (refuse/delay) or availability (serve possibly-stale). Partition tolerance isn't optional in distributed systems. Outside partitions the real trade-off is latency vs consistency (PACELC). Wrong framing to avoid: "pick 2 of 3."

**Q: Why do we care about p99 latency and not average?**
A: Averages hide tail pain. One page = many backend calls, so a user hits the tail with high probability (1 - 0.99^N). Tail latency compounds through fan-out; SLOs are set on percentiles. Also: p99 regressions surface capacity/GC/lock problems that averages absorb.

**Q: Estimate storage for a URL shortener at 100M new URLs/month.**
A: ~500 bytes/record × 100M ≈ 50 GB/month → ~3 TB over 5 years — fits one Postgres box; the interesting scaling is read QPS on redirects, not storage. The skill on display: round aggressively, state assumptions, and let the number redirect the design.

## Building blocks

**Q: L4 vs L7 load balancing?**
A: L4 routes on IP/port — fast, no payload inspection, TCP passthrough. L7 terminates HTTP — routes on path/headers/cookies, TLS termination, per-route policies, sticky sessions. Typical stack: L4 (or anycast) in front of L7 fleet.

**Q: How do you handle a cache stampede?**
A: Hot key expires → thousands of concurrent DB hits. Fixes: per-key mutex (one rebuilds, rest wait/serve stale), probabilistic early refresh, stale-while-revalidate, jittered TTLs so keys don't expire together. Know one concrete mechanism, not just the term.

**Q: Consistent hashing — what problem, what mechanism?**
A: Naive `hash(key) % N` remaps nearly everything when N changes. Consistent hashing puts nodes and keys on a ring — adding/removing a node moves only ~1/N of keys. Virtual nodes smooth load. Used by caches, Cassandra/Dynamo-style stores.

**Q: Kafka in 60 seconds.**
A: Distributed append-only log. Topics → partitions (ordering only within a partition); producers pick partition by key; consumer groups split partitions among members; offsets = consumer-managed positions enabling replay; replication with leader per partition. Delivery is at-least-once by default → consumers must be idempotent; "exactly-once" = transactions + idempotent producer within Kafka's boundary.

**Q: When is a message queue the wrong answer?**
A: When the caller needs the result to respond (sync path), when strict global ordering across entities is required, when p99 latency budget can't absorb queueing delay, or when volume is trivial — a table + polling worker is simpler to operate and debug.

## HLD talking points

**Q: Fan-out on write vs read for a news feed?**
A: Write: precompute followers' feeds on post — fast reads, storage-heavy, celebrity posts explode (10M writes). Read: query followees at read time — no write amplification, slow reads. Production answer: hybrid — fan-out on write for normal users, merge celebrity posts at read time. Name the celebrity problem unprompted.

**Q: How does WhatsApp deliver a message to an offline user?**
A: Persistent connection servers (WebSocket/custom) with a user→server registry. Online: route directly, ack tiers (sent/delivered/read). Offline: persist to per-user queue, push notification, drain on reconnect. Key guarantees: at-least-once + client-side dedupe by message ID; ordering per conversation, not global.

**Q: Sketch an LLM serving system for 10k concurrent users.**
A: Gateway (auth, rate limits, streaming passthrough) → router (model selection by task/cost, fallback chains) → inference pool (vLLM: continuous batching + paged attention), separate pools per model class; prompt caching for shared prefixes; KV-cache-aware routing for multi-turn (session affinity); SSE token streaming; observability per request (TTFT, tokens/sec, cost per tenant); degraded mode: smaller model or queued batch when the pool saturates. This case = applied-ai stage 8 made concrete.

## LLD talking points

**Q: Where do design patterns land in the chess LLD?**
A: Factory — piece creation; Strategy — per-piece move validation (open/closed for new pieces: swap in validators without touching Board); Observer — game events (clock, spectators); Memento or move-log — undo/replay. Schema: `games`, `moves` (game_id, ply, from, to, san, fen_after) — moves as event log, board state derivable; index (game_id, ply). APIs: `POST /games`, `POST /games/{id}/moves` (validate server-side, 409 on illegal), `GET /games/{id}`. The reasoning matters more than the pattern names.
