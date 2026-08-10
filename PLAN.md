# System Design — Concept Plan

Start after backend + DBMS refresh (needs that vocabulary). Method for every HLD case: draw from memory → find bottlenecks → argue trade-offs aloud — never just watch/read. Priorities: **P0** interview-critical · **P1** strongly expected · **P2** differentiator.

Notes land in `concepts/` and `designs/` using the templates.

## Stage 1 — Vocabulary `P0`

- [ ] Scalability: vertical vs horizontal
- [ ] Latency vs throughput; percentiles (p50/p95/p99)
- [ ] CAP theorem — what it actually says (partition tolerance isn't optional)
- [ ] Strong vs eventual consistency; read-your-writes
- [ ] Fault tolerance: redundancy, failover, graceful degradation
- [ ] Back-of-envelope estimation (QPS, storage, bandwidth)

**Checkpoint →** vocabulary Qs in [INTERVIEW-QUESTIONS.md](INTERVIEW-QUESTIONS.md)

## Stage 2 — Building blocks `P0`

- [ ] Load balancing: L4 vs L7, algorithms, health checks
- [ ] Caching: layers (browser→CDN→gateway→app→DB), distributed caching, invalidation, stampede
- [ ] Replication & leader election basics
- [ ] Partitioning/sharding; consistent hashing
- [ ] Message queues & pub/sub: delivery guarantees, ordering, DLQs; Kafka model (partitions, consumer groups, offsets)
- [ ] API gateway; proxy vs reverse proxy
- [ ] WebSockets at scale (sticky sessions vs pub/sub backplane)
- [ ] Event-driven architecture; sync vs async communication
- [ ] Batch vs stream processing

**Checkpoint →** building-block Qs

## Stage 3 — HLD cases `P0`

*One per session. Draw → bottleneck → trade-offs → write the design note.*

- [ ] 1. URL shortener (hashing, ID generation, redirects at scale, analytics)
- [ ] 2. Unique ID generator (Snowflake — why not UUID/auto-increment)
- [ ] 3. News feed (fan-out on write vs read, ranking, celebrity problem)
- [ ] 4. WhatsApp/chat (connection servers, delivery guarantees, presence, offline queue)
- [ ] 5. Uber nearby drivers (geospatial indexing: geohash/quadtree/H3, location update firehose)
- [ ] 6. **LLM serving system** (the AI-engineer differentiator: gateway → router → vLLM pool; KV cache, continuous batching, streaming, prompt caching, cost/latency SLOs) — pairs with applied-ai stage 8

**Checkpoint →** can present any of the 6 in 35 minutes cold

## Stage 4 — LLD problems `P1`

*Python. Zepto-style bar: entities → patterns → DB schema → APIs, extensibility follow-ups.*

- [ ] Chess game (Piece hierarchy, move validation Strategy, Factory, game state, schema for games/moves, APIs)
- [ ] Parking lot (spot allocation, pricing strategy, concurrency on entry)
- [ ] Splitwise (debt graph, simplification, precision)
- [ ] Kafka-lite distributed queue (partitions, consumer offsets, ack semantics)

## Stage 5 — Learn from the best engineers `P1`

*Rotation: one post per session → short note in `concepts/` or `designs/` with the trade-off it teaches.*

- [ ] Figma — how multiplayer technology works (CRDTs, sync)
- [ ] Notion — the data model behind Notion's flexibility (blocks)
- [ ] Linear — scaling the sync engine
- [ ] Discord — scaling to 11M concurrent (Elixir/Rust, message store on Cassandra→ScyllaDB)
- [ ] Stripe — idempotency and payments reliability
- [ ] Netflix tech blog — resilience/chaos engineering pick
- [ ] Cloudflare — how a CDN/edge actually works pick
- [ ] Uber — real-time geospatial pick (pairs with HLD case 5)
- [ ] Instagram/Meta — sharding or feed infrastructure pick
- [ ] DoorDash — Kafka migration or reliability postmortem pick
- Blog homes: netflixtechblog.com, blog.cloudflare.com, stripe.com/blog/engineering, discord.com/blog, careersatdoordash.com/engineering-blog, eng.uber.com, instagram-engineering.com, engineering.fb.com, engineering.linkedin.com, github.blog/engineering, spotify.engineering, airbnb.io

## Stage 6 — Depth extras `P2`

- [ ] CQRS + event sourcing; Saga pattern
- [ ] Service mesh; serverless trade-offs
- [ ] CRDTs beyond the Figma post; Google Docs OT vs CRDT

## Out of scope (deliberate)

Memorizing 30 case designs — 6 deep beats 30 shallow (real loops probe depth, not coverage).
