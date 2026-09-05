# Data & Storage

Data outlives code. A deploy replaces code in minutes; five-year-old rows keep their original shape. Get the model and the keys right first — moving code is cheap, splitting a database or rewriting a widely-consumed API is expensive.

## 1. Choose the model from the relationship shape

| Shape | Model |
|---|---|
| Self-contained tree loaded whole (one-to-many) | **Document** — storage locality, one read instead of a join |
| Many-to-one / many-to-many, joins | **Relational** — an ID never has to change; duplicated human-readable data costs write amplification and drift |
| "Anything may relate to anything", variable-length traversal | **Graph** |
| Aggregate over millions of rows, few columns | **Columnar / warehouse**, not the OLTP store |

Data tends to get **more** interconnected as features are added. If you denormalize to avoid a join, the application now owns consistency of the copies — emulating joins in app code is usually slower and more complex than a DB join.

**Document sizing pitfall:** a document is stored as one contiguous string. The DB loads the *whole* document for a small field, and an update rewrites it unless the encoded size is unchanged. **Keep documents small; avoid writes that grow a document unboundedly** (append-forever arrays are the classic bug).

**Schema-on-read is still a schema** — an implicit one. Choose it for heterogeneous or externally-controlled records; choose schema-on-write when uniformity matters. A backfilling `UPDATE` on a big table is slow everywhere; prefer nullable-with-default and fill at read time.

## 2. Indexes

**Well-chosen indexes speed reads; every index slows writes.** Don't index by default — index for known query patterns.

- **Concatenated / compound index** is usable left-to-right only (phone-book rule): `(lastname, firstname)` cannot answer "find by firstname".
- **Covering index** answers the query from the index alone — costs storage and write speed.
- **Multi-dimensional** queries (lat/lng bounding box, `date × value`) cannot be served efficiently by a 1-D index; you scan one dimension and filter. Map 2-D → 1-D (geohash / Hilbert curve) so one index works, or use a spatial index type.
- **Fuzzy/typo search** needs different structures entirely (term dictionary + automaton). Don't bolt it onto a B-tree.

### Storage engines, when you get to pick
| | LSM (append + compact) | B-tree (update in place) |
|---|---|---|
| Writes | faster, sequential | ≥2 physical writes (WAL + page) |
| Reads | slower (check levels; Bloom filters help) | faster |
| Tail latency | **spiky** — compaction competes with queries | **predictable** |
| Transactions | multiple copies of a key | one copy → locks attach cleanly |

Rule: latency SLO ⇒ favor predictability. Write-once-read-many ⇒ B-tree/relational. Write-heavy ⇒ LSM (RocksDB, Cassandra). **Benchmark with your workload; the rules of thumb are not decisive.**

## 3. Separate OLTP from OLAP early

| | OLTP | OLAP |
|---|---|---|
| Read | few records by key | aggregate over millions |
| Bottleneck | **disk seek** | **disk bandwidth** |

Do not let analysts run ad-hoc scans against the transactional store — those queries scan large parts of the dataset and degrade concurrent transactions. Extract to a warehouse / read model. Star schema (one fact table + dimension tables) and columnar storage exist because analytic queries touch 4–5 of 100+ columns.

**Store raw AND aggregated.** Raw enables debugging and recomputation after a bug; aggregated is the query-tuned active set. Aggregate-only is lossy and unrecoverable; raw-only is too slow to query.

## 4. Partitioning / sharding

Only shard when the data or write volume genuinely exceeds one node. Then:

- **Shard key = the field your dominant queries already carry.** Not what looks tidy. Object storage sharded by `hash(bucket_name, object_name)` because every access is URI-driven; hotel inventory by `hotel_id`; mailboxes by `user_id`.
- Do the arithmetic: 30,000 QPS ÷ 16 shards = 1,875 QPS/shard — inside one node's capacity. Choose the count from that, not from a round number.
- **Key-range** partitioning gives range scans but timestamp-prefixed keys create a write hot spot on "today". Fix by prefixing with a higher-cardinality field.
- **Hash** partitioning distributes evenly but destroys range queries.
- **Hybrid (usually right):** compound key — hash the first column for placement, sort by the rest within the partition. `(user_id, timestamp)` gives even spread *and* cheap per-user time ranges.
- **Never `hash mod N`** for anything whose N will change — it remaps almost every key. Use consistent hashing (ring + **virtual nodes**: ~10% load stddev at 100 vnodes, ~5% at 200) or a fixed large partition count (e.g. 1,000 partitions on 10 nodes; the count then caps your future node count).
- **Hot key / celebrity:** hashing does not help. Append a small random suffix to split the few known-hot keys across N sub-keys (reads fan out and merge), or route them to a dedicated shard. Hot keys are **structural**, not accidental — big advertisers, huge creators, popular products.
- Rebalancing should keep serving traffic and move no more data than necessary. **Keep a human in the loop** — fully automatic rebalance + automatic failure detection cascades: an overloaded node is declared dead, its load moves, more nodes overload.

### Secondary indexes on partitioned data
- **Local (document-partitioned):** writes touch one partition; reads scatter/gather across all → tail-latency amplification.
- **Global (term-partitioned):** reads hit one partition; writes touch many → updated asynchronously in practice.
- Read-heavy filtering across many partitions ⇒ global, accept staleness. Write-heavy or few partitions ⇒ local.

### When a query fights your shard key
Give it its own **denormalized, differently-sharded table** rather than degrading the primary path. Cross-shard pagination is the hidden cost: each shard returns a different partial count, so the cursor must carry a per-shard offset. Hundreds of shards ⇒ hundreds of offsets.

## 5. Replication

- **Leader-based**: writes to leader, reads from replicas. Fully synchronous is impractical (one slow follower halts writes); **semi-synchronous** (one sync follower) guarantees an up-to-date second copy. Fully async keeps writes flowing but **acknowledged writes can be lost on failover**.
- Typical lag is sub-second with **no upper bound**. "Pretending replication is synchronous when it is asynchronous is a recipe for problems." Ask: what does the user see if lag hits minutes?
- Three lag anomalies and their fixes:
  1. **Read-after-write** (user doesn't see their own submission) → read user-owned data from the leader, or from the leader for ~1 min after a write, or track a write position client-side and require a replica at least that fresh.
  2. **Monotonic reads** (time appears to go backwards) → pin a user to one replica (hash of user ID).
  3. **Consistent prefix** (answer before question) → route causally related writes to the same partition.
- **Failover hazards:** detection is only a timeout (too long = outage, too short = spurious failover on an already-loaded system); async replication loses writes; split brain; and the classic cross-store corruption — a promoted lagging replica **reused auto-increment primary keys** already referenced elsewhere, exposing private data to the wrong users. **Never use DB-generated sequential IDs as cross-store keys.** Many teams prefer manual failover for exactly these reasons.
- **Separate availability of service from durability of data.** A safe replica with no failover mechanism means your data survives but your service does not.
- Place replicas in distinct failure domains (node → rack → AZ). Correlated failure is what kills you: power, cooling, a shared switch, a shared SAN.

## 6. Caching

Add a cache when data is **read frequently and modified infrequently** *and* the working set does not already fit in DB RAM. If it does, add replicas instead — a cache in front of an already-memory-resident dataset buys almost nothing.

- **Always set a TTL.** Too short → reload storms; too long → staleness.
- **Cache IDs/references, not fat objects**; hydrate on read. Bound per-user lists (users rarely scroll deep).
- **Key design matters:** raw lat/long is a terrible key (GPS jitter ⇒ near-zero hit rate); the grid cell is a good one. Design the key so small input variation maps to the same entry.
- **The DB is truth. Re-validate the invariant at the DB even when the cache says yes.** Worst case the user sees "someone just took the last one" — acceptable.
- Write path: **update the DB first, propagate to cache asynchronously** (from app code after save, or via CDC).
- **Stampede/thundering herd:** mass simultaneous expiry (nightly bulk invalidation) collapses the cache tier. Jitter TTLs. On miss, fail fast and repopulate **asynchronously** rather than letting every request hit an origin sized for a fraction of traffic.
- **Serve stale rather than nothing** where the product allows it — caching is a resilience tool, not just a speed tool.
- **Keep it simple:** every extra cache layer makes staleness harder to reason about. Prefer one. Understand the *whole* path — a bad `Expires` header once stuck pages permanently in CDN/ISP/browser caches, unfixable except by changing URLs.
- Layers: client-side (fewest calls, hardest to invalidate) / CDN or proxy (easiest to retrofit, generic) / server-side (easiest to reason about invalidation). HTTP gives you `cache-control`, `Expires`, and ETag + `If-None-Match` → 304 for free; don't mix all three carelessly.
- **CDN:** static assets from the nearest edge (~10 ms vs ~300 ms to origin). Charged per transfer — don't CDN rarely-accessed assets. Invalidate by object versioning (`image.png?v=2`). Plan the CDN-outage fallback. Exploit long-tail popularity: CDN the hot content, serve the cold tail from origin.
- **Precompute + CDN beats dynamic generation whenever the key space is enumerable** — dynamic generation costs server load *and* destroys cacheability.

## 7. Consistency and isolation — pick deliberately

**CAP, stated properly:** when partitioned, choose consistency *or* availability. CA does not exist in a distributed system. Choose **per capability, not per system**: a catalog can be AP while inventory is CP; a balance can be stale on read but consistent on decrement.

**Isolation ladder — what each level actually stops:**

| Anomaly | Prevented by |
|---|---|
| Dirty read / dirty write | Read committed |
| Read skew (non-repeatable read) | Snapshot isolation (MVCC) |
| Lost update | Atomic op, auto-detection, explicit lock, or CAS |
| **Write skew** (read a premise, write elsewhere, premise now false) | **Only serializable** |
| Phantoms | Index-range locks / SSI |

"Repeatable read" means nothing reliable — the naming is inconsistent across vendors. Check what your engine actually does.

**Lost-update fixes, in order of preference:** (1) a database atomic operator (`SET x = x + 1`) — best when expressible; (2) automatic detection; (3) explicit `SELECT … FOR UPDATE`; (4) compare-and-set on a version column. ORMs make unsafe read-modify-write loops easy to write by accident.

**Concurrency control — match the mechanism to actual contention, not imagined contention:**
- Low contention → **DB constraint** (`CHECK(total - reserved >= 0)`) or **optimistic locking** (version column; prefer a version number over a timestamp — clocks drift).
- High contention → pessimistic locking is correct but holds locks across the request, risks deadlock, and does not scale. Prefer redesigning to a commutative/atomic operation.
- Constraints aren't version-controlled like app code, and not every engine supports them — a real cost, still usually worth it.

**Conflict resolution when writes are concurrent** (multi-leader/leaderless): last-write-wins converges by **silently discarding acknowledged writes** — only safe when each key is written once and thereafter immutable. Otherwise use version vectors and merge siblings, or prefer commutative operations (counter increment, set-add) that merge without conflict. Deletions need tombstones or removed items resurrect.

## 8. Encoding & schema evolution

- Old and new code, old and new data coexist during every rolling deploy. You need **backward compatibility** (new code reads old data) *and* **forward compatibility** (old code ignores new fields).
- Never use language-native serialization for anything persisted or sent (lock-in, RCE vector, no versioning story).
- Schema formats (Protobuf/Thrift/Avro) beat JSON on size and give you a compatibility check before deploy. Rules: field tags are permanent, never reuse a tag, new fields must be optional or defaulted, and type widening breaks old readers.
- **Silent data-loss pitfall:** old code reads a record containing an unknown field, decodes, re-encodes, writes back — the field is gone. Preserve unknown fields explicitly.
- Prefer schema evolution (add nullable column) over migrating large datasets.

## 9. Object / blob storage

Store bytes you never query in **object storage, not a database** — it's the wrong (expensive) way to hold bytes and you use none of the DB's features.

- Separate **metadata store** from **data store** (inode analogy): data keyed by immutable UUID; metadata maps name → ID and is mutable. Scale and optimize each independently.
- Objects are immutable — replace or delete whole; that's what makes aggressive caching and replication safe.
- **Versioning by insert, delete by marker.** Never overwrite the metadata row: insert a new row with a new TIMEUUID version; current = largest. A delete is a new "delete marker" version. Soft-delete by construction; garbage-collect later by compaction.
- **Multipart upload** for large files: initiate → upload parts independently with ETags → complete. Avoids restarting a multi-GB transfer and enables parallelism.
- **Verify checksums at every process boundary.** Silent in-memory/in-transit corruption is routine at scale; whole-disk failure is the *easy* case.
- **Delta sync**: split files into blocks, hash each, transfer only changed blocks. De-dup by hash.
- **Pre-signed upload URLs**: client asks your API for a URL and uploads directly to storage — authorization without proxying the bytes.
</content>
