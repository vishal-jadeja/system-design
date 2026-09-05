# Distributed Correctness

The defining property of a distributed system is **partial failure**: any message may be lost or delayed, any clock may be wrong, any process may pause. A missing response is indistinguishable between "request lost", "node down", "node paused", "response lost", "response delayed". **Only a positive application-level response proves success** — a TCP ack does not.

Design for the **partially-synchronous, crash-recovery** model: mostly bounded, occasionally arbitrarily bad; stable storage survives a restart, memory does not.

## 1. Idempotency — the workhorse

**Exactly-once = at-least-once (retry) + at-most-once (idempotency key).** Split the problem; each half has a standard mechanism. No framework gives you exactly-once once an effect crosses an external boundary (a DB in another service, an email, a push, a charge).

Recipe:
1. Generate an idempotency key **before** the operation — a UUID, or reuse a natural one (order ID, cart ID, message offset, `reservation_id`).
2. **Back it with a database unique constraint** — make it the primary key and attempt the insert. Insert succeeds = first time; duplicate-key error = already seen. This is the cheapest correct implementation.
3. Repeat with a seen key → return the **status of the original request**, don't reprocess. Concurrent duplicates → process one, return **429** to the rest.
4. **Reuse the same key on retry.** A fresh key on retry turns retries into new charges.
5. Prefer making the *business operation* idempotent (a credit carries the originating order ID) over bolting on a dedup table.

Corollaries: attach the source message offset to the write and skip if already applied. HTTP `GET`/`PUT` are only idempotent if *your* implementation makes them so.

## 2. Retries

- **Exponential backoff by default** (1s → 2s → 4s) with a **cap** and a **retry limit**. Over-aggressive retry is a self-inflicted DDoS; retrying on overload makes overload worse.
- Classify failures: non-retryable (invalid input) → store, don't retry. Retryable → retry queue. Over the threshold → **dead-letter queue** for investigation, not silent drop.
- A poison message + a transacted queue + no retry limit = workers dying in a loop, forever. This is a real outage shape.
- Return `Retry-After` so clients back off correctly.
- Persist a **definitive state per operation in an append-only table** so at any failure point you can decide retry vs refund vs abort.

## 3. Ordering, clocks, and IDs

- **Two clock types.** Monotonic (`process.hrtime`, `System.nanoTime`) for durations and timeouts — the absolute value is meaningless and never comparable across machines. Time-of-day for display only; it **can jump backwards**.
- Real error magnitudes: quartz drift ~200 ppm (≈17 s/day if resynced daily), best-case NTP over the internet ≈35 ms with spikes to ~1 s, VM clocks jumping tens of ms when descheduled, leap seconds producing 61-second minutes. **Clock resolution ≠ accuracy** — microsecond digits are noise when error is ±100 ms.
- **Never order events by wall-clock time.** Last-write-wins by timestamp silently drops acknowledged writes and cannot distinguish sequential from concurrent. NTP can never fix this: sync accuracy is bounded by the network delay being measured.
- Use instead: logical counters, **sequence IDs from a single sequencer**, Lamport timestamps (total order consistent with causality), version vectors (can *detect* concurrency; Lamport cannot).
- **Snowflake-style IDs** (41 bits ms timestamp + datacenter + machine + sequence) give roughly-sortable 64-bit numeric IDs at scale, but still assume synchronized clocks and fixed machine IDs — changing a machine ID risks collisions. A per-channel local sequence is simpler when ordering only has to hold within a scope.
- **Untrusted device clocks:** log three timestamps — event time (device), send time (device), receive time (server) — and use (receive − send) to estimate the device's offset.
- **Any process can pause for minutes** (GC, VM migration, page faults, sync I/O). A node holding a lease may resume after it expired and still believe it is the leader. Checking `lease.isValid()` then acting is a TOCTOU race no matter how large the safety margin.
- **Fencing tokens** are the fix: the lock service returns a monotonically increasing token with every grant; every write carries it; **the resource rejects any token lower than one it has already seen.** The check must live in the resource, not the client — never assume clients are well-behaved.

## 4. Transactions across boundaries

**First: try very hard not to split the state.** Co-locating two tables in one database so a plain ACID transaction works beats every distributed mechanism below, until scale forces the split. Give one service ownership of both APIs.

If you must span services:

| Mechanism | Shape | Cost |
|---|---|---|
| **2PC / XA** | prepare-all, then commit-all | **Blocking**: coordinator crash leaves participants in doubt holding locks; coordinator is a SPOF; reported >10× slower than single-node; amplifies failure (needs *all* participants). |
| **TC/C** (Try-Confirm/Cancel) | try reserves; second phase confirms or reverses; each phase is its own local transaction | DB-agnostic, allows parallelism; you own the complexity in business logic; intermediate state is visibly unbalanced. |
| **Saga** | linear sequence of local transactions; on failure compensate in reverse | n operations means writing **2n**; strictly sequential (no parallelism); eventual consistency only. |
| **Try again later** (queue + retry) | eventual consistency | Simplest; best for long-lived business operations. Often the right answer. |

Rules that are easy to get wrong:
- **Always deduct before adding.** Crediting first lets a third party spend money you may have to claw back.
- Keep a **phase status table** (transaction ID, per-participant try status, which second phase, second-phase status, out-of-order flag) so a restarted coordinator can resume.
- **A Cancel can arrive before its Try.** Cancel-without-Try writes an out-of-order flag; every Try checks the flag first and fails if set.
- Compensating transactions can themselves fail; past 2–3 participants they become unmanageable.
- Model the distributed operation as a **concrete domain entity** (an "in-progress order") so you have something to monitor, retry, and inspect.

**Compensating business processes** are often better than technical consistency: email the loser of a username race, backorder the stock, charge an overdraft fee with a daily cap. Businesses already have apology workflows — a strong technical constraint may be unnecessary.

## 5. Reconciliation — the last line of defense

Async communication guarantees nothing about delivery. The systematic fix is a **periodic job that compares state across systems** (yours vs the provider's settlement file; ledger vs wallet; stream result vs a nightly batch recomputation).

Classify every mismatch into three buckets and design ops around them:
1. classifiable + auto-fixable → write the adjustment program;
2. classifiable but auto-fix not cost-effective → job queue for manual fix;
3. unclassifiable → separate queue, human investigation.

**Even if the external API is idempotent, still reconcile.** Never assume the external system is right. Note what reconciliation *cannot* do: it tells you records differ, never *how* — root cause needs reproducibility (below).

## 6. Third-party integration

- Prefer the provider-hosted page/SDK so sensitive data never touches your servers.
- Canonical flow: register the intent with **your own nonce/order ID** (makes registration exactly-once) → provider returns a token → **persist the token before rendering the provider page** → user completes → redirect back → **the async webhook is the authoritative status**, not the redirect.
- **Design for a `pending` state from day one.** Manual review, step-up auth, and settlement legitimately take hours or days. Show it in the UI, give the user a status page.
- The provider token doubles as the idempotency key for the external call — which is what protects you when their success response is lost in the network.

## 7. Consensus and coordination

- **Don't implement consensus yourself** — the track record is poor. Outsource to ZooKeeper / etcd / Consul / a Raft library.
- Consensus needs a **majority alive**: 3 nodes tolerate 1 failure, 5 tolerate 2. Run coordination services on a fixed 3 or 5 nodes serving many clients — that's how you get coordination without paying majority-vote cost per app node.
- Coordination stores are for **small, slow-changing data that fits in memory** ("node X is leader for partition 7"), not runtime application state at thousands of writes/sec.
- The useful bundle: linearizable CAS (→ lock as an expiring **lease**), total ordering (→ fencing tokens), failure detection via heartbeat sessions + ephemeral nodes, and watches so clients don't poll.
- **Service discovery does NOT need consensus** — it tolerates staleness; prioritize availability. Only leader election does.
- A single leader "kicks the can down the road": you still need consensus for leadership *changes*, just less often. Three honest responses to leader loss — block until it recovers, manual human failover (bounded by human speed, and a legitimate choice for a new system), or a proven consensus algorithm.

**Linearizability** (behave as if there's one copy, every op atomic at a point in time) is genuinely required for: leader election/locks, uniqueness constraints, and **cross-channel races** — two paths between components (write to storage *and* enqueue a job) race unless the shared store is linearizable. That last shape is the most commonly-missed real bug. Its everyday cost is **latency**, not just fault tolerance. **Causal consistency** is the strongest model that doesn't slow down under network delay — many systems that "need linearizability" only need causality.

## 8. Event sourcing & CQRS

Store an **immutable log of state-changing events** as the source of truth; current state is a projection rebuilt by replay.

Vocabulary that keeps this honest:
- **Command** = intent from outside. May be invalid, may involve randomness/IO.
- **Event** = a validated fact, past tense, and **must be deterministic**.
- **State machine** validates commands → events and applies events → state, and **must contain zero randomness or IO**.

What it buys:
- **Reproducibility** — the balance at any point in time; whether current state is correct (recompute); whether a code change is correct (run both versions over the same events and diff).
- **Recovery** = replay. Crash recovery needs no separate state-transfer protocol.
- **Audit trail** — "how did it get into this state", which a current-state schema cannot answer.
- **Determinism makes HA cheap**: a warm replica consumes the same events, computes the same state, and suppresses its outputs until promotion.
- **CQRS**: publish events, not state. One write-side state machine; many read-only projections, each shaped for its query, each freely denormalized. Run a new projection alongside the old one instead of doing a schema migration, then retire the old.

Costs and rules:
- Only the **event log** needs strong durability — state and snapshots are regenerable from events, and **commands are not sufficient** (event generation may be non-deterministic).
- Read-your-writes across an async projection needs care: update synchronously in one unit, or partition log and state identically so a single-threaded consumer needs no concurrency control.
- Snapshots are an optimization so replay resumes from a checkpoint instead of genesis.
- True deletion is hard — copies persist in snapshots and backups.
- Under event sourcing, **absolute timestamps stop mattering; only order matters** — which collapses replay time.

## 9. Streams, time, and windows

- **Never dual-write.** Two concurrent writes land in different orders in two stores and diverge permanently, with no error raised — and one can fail while the other succeeds. Make one store the leader and derive the rest from its change log (**CDC**). Log-based transport preserves order.
- Bootstrapping a derived store: consistent snapshot tied to a known log offset, then apply changes from there. **Log compaction** (keep the latest value per key) lets a new consumer read from offset 0 and rebuild the whole current state.
- **Event time vs processing time.** Windowing by processing time fabricates artifacts — a restarted consumer replaying a backlog shows a false traffic spike while the real rate was flat. Use embedded event timestamps.
- You can never know a window is complete. **Watermark** = a small grace period (e.g. +15 s) to catch slightly-late events; long watermark = more accuracy, more latency. Watermarks deliberately do not handle badly-late events — accept the miss and fix it in end-of-day reconciliation. A complex design for low-probability events is bad ROI, **unless a few percent equals millions**, in which case pay for exactly-once.
- Window types: tumbling (fixed, non-overlapping — "count per minute"), hopping, sliding ("top N in last M minutes"), session (activity-delimited).
- **Never do per-record remote lookups** inside a bulk or stream job — throughput becomes RTT-bound, you can overwhelm the production DB, and the job becomes non-deterministic. Keep a local replica of the lookup table fresh via CDC.
- **The offset-commit ordering trap**, in sequence: commit offset before emitting ⇒ silent data loss; emit then crash before commit ⇒ duplicates; the only correct form wraps emit + offset-save + ack in one transaction, or leans on idempotency instead.
- In-memory aggregation state is lost on crash and replaying from the beginning is too slow → **incremental checkpoints** of (upstream offset + derived state); a replacement node loads the last snapshot and replays only the delta.
- Recompute-based recovery requires **deterministic operators**. Non-determinism creeps in via hash iteration order, RNGs, the system clock, and external lookups.

## 10. Byzantine-ish defenses worth having

Assume nodes are unreliable but honest inside your own infrastructure — full BFT is not economical, and it doesn't protect you from a bug deployed to every node anyway. But do add the cheap weak-lying defenses: application-level checksums (TCP checksums do miss corruption), input range/size limits (a huge allocation is a DoS), and NTP against multiple servers so outliers get excluded.

**Never trust the client.** An ID from the client is not authorization — ownership is checked somewhere authoritative. A score, price, or outcome set by the client is forgeable by a proxy; compute it server-side.
</content>
