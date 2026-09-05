# Applying this in the GlitchOver repo

What the stack actually is, and what that constrains. **Verify anything here against the code before relying on it** — this file records the shape as of the last review, not a guarantee.

## Stack facts that change the design

| Area | Reality | Consequence |
|---|---|---|
| Database | **MongoDB / Mongoose**, ~117 models, flat `/models/` | No cross-document transactions by default. Multi-document invariants need either co-location in one document, a Mongo transaction (replica set), or an idempotent + reconciled design. |
| Backend | Express, two layers (legacy `/routes/`, new `/server/routes/` + controllers) | New work goes in the new layer. Don't create a third pattern. |
| Real-time | **Socket.io with the default in-memory adapter** (`startup/socket.js`) | **Rooms and emits do not cross Node processes.** The socket tier is effectively single-instance today. Any design that assumes "emit reaches all users" breaks the moment a second instance exists. If horizontal scaling of sockets is on the table, that's a Redis/Mongo adapter decision, and it must be stated explicitly — not assumed. |
| Queue | `config/queue.js` — **SQS by default**, BullMQ provider available via `QUEUE_PROVIDER`, Noop under test | There is already one queue abstraction. Use it. Do not introduce a second queue technology or a direct SDK call. |
| Scheduled work | `node-cron` in-process (`utils/cronjob.js`, social feed worker) | In-process cron fires **once per running instance** — it is not instance-safe. Prefer deriving state lazily from a stored timestamp on read over adding another timer. |
| Payments | External HTTP microservice (separate process/port), webhook-driven | Treat it as a third party: nonce → token → **webhook is truth, redirect is UX** → reconcile. Never trust the redirect. |
| Frontend data | RTK Query (cache/invalidation) or Redux slice + thunk — not mixed within a feature | Cache invalidation strategy is a design decision, not an afterthought. |
| Search / vectors | MongoDB Atlas `$vectorSearch` (support bot) | No separate search cluster to hide behind. |

## Repo-specific rules that override generic advice

- **New code references `User` with `isInfluencer: true`** — not the legacy `Influencer` model. New schemas name the owner field `creatorId` (ref `users`).
- **Never hard-delete user-generated data.** Status field + reason + timestamp. This is a design constraint on every schema you propose — a soft-delete model changes your indexes and your queries.
- **`$lookup.from` is `"livesessions"`** (lowercase). Wrong casing silently returns empty.
- **One-time migrations are admin API routes**, not standalone scripts — and ask before creating one.
- **Validate every payload before touching the DB**: required fields, types, reject unknown fields with 400.
- **Never trust client-provided IDs** — use `req.user._id`.
- Socket events: emit only **after** a successful DB write, use `domain:action` naming, send minimal payloads (never full Mongoose documents).

## Mongo-specific sizing notes

- **Index for the queries you actually run.** 92 of ~117 models already declare indexes — read the existing ones before adding another; every index slows writes.
- **Documents are contiguous.** An unbounded array field (message list, participant list, event log) is a growing document rewritten on every update. Cap it or move it to its own collection. This is the single most common scaling bug in a document store.
- **Compound key pattern**: `{ ownerId: 1, createdAt: -1 }` gives even distribution across owners *and* cheap per-owner time-range reads. Make the shard/partition key the entity that the dominant query already carries.
- Aggregation pipelines that scan large collections belong on a derived/read model or a scheduled rollup, not on a user-facing request path.
- Before proposing sharding: compute the collection size and the QPS. Most collections here will not come close.

## Default answers for common asks in this repo

| Ask | Default answer |
|---|---|
| "Should we add Redis?" | Not unless the working set genuinely doesn't fit and you've measured. There is no Redis tier today; adding one adds an operational dependency and a consistency problem. Say what it buys in numbers. |
| "Should we add a queue for X?" | Only if X has multiple independent consumers, needs spike buffering, or must not block the request. Otherwise call it inline. If yes, use the existing `getQueueProvider()`. |
| "Should this be a new service?" | Almost certainly no. New module in the existing app, clean boundary. Split only when the deploy-independence test justifies the distributed cost. |
| "Should we shard?" | Compute the size first. Then almost certainly no. |
| "Should this be real-time?" | Only if the user perceives the delay. Remember the single-instance socket constraint. |
| "Exactly-once?" | Retry + idempotency key backed by a **unique index**. Then reconcile if money is involved. |
| "Add a cron for this deadline?" | Prefer computing state on read from a stored timestamp. In-process cron and in-memory `setTimeout` are both instance-unsafe. |

## The output to produce

For anything non-trivial, write a short design record before coding:

```
Requirements   — functional; non-functional as numbers (p99 target, availability, correctness class)
Numbers        — QPS avg/peak, storage, read:write, working-set size
Design         — the boxes, the keys, the indexes, the invariants and where they're enforced
Rejected       — what you didn't do and the number that ruled it out
Instrumentation — conversion events (`conversion-tracking`) + XP events (`xp-event`) fired, or why none apply
Failure modes  — what breaks, what degrades, what the user sees
Revisit when   — the specific metric/threshold that says this design has expired
```
</content>
