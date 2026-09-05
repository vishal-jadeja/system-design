# Resilience & Operations

At scale, failure is a statistical certainty. Spend less effort preventing the inevitable and more on **graceful recovery**. If you handle failure well, in-place upgrades and planned outages become non-events.

**Human error is the leading cause of outages** — configuration errors dominate; hardware is 10–25%. So the highest-leverage investments are fast rollback, staged rollout, realistic sandboxes, and detailed telemetry — not another layer of redundancy.

## 1. The three architectural safety measures

The canonical outage: one slow, low-value downstream (serving <5% of customers) saturated a **shared HTTP connection pool** whose worker-wait timeout was disabled by default. 40 → 800 connections in five minutes. Whole site down.

- **Timeouts** on *every* out-of-process call. Pick a sensible default everywhere, log every timeout, tune from the data. Too long = system-wide slowdown; none = a downstream can hang you indefinitely.
- **Circuit breakers** on all synchronous downstream calls: after N failures (timeout or 5xx) trip → fail fast → probe periodically → reset. While blown: queue and retry for async work, fail fast in a synchronous chain. Also useful *manually*, to make a service safe to take down.
- **Bulkheads** — the most important of the three. A **separate connection pool per downstream**, so one sick dependency cannot exhaust a shared resource. Timeouts and breakers free resources once constrained; bulkheads prevent the constraint. Separate services are themselves bulkheads. **Load shedding** (rejecting requests) is sometimes the correct protection for an overwhelmed service.

Don't write your own — use a maintained library.

**Slow is worse than dead.** A dying downstream that responds slowly cascades far more damage than one that fails fast.

**Timeouts have no correct value.** Too short = false positives that shift work onto already-loaded nodes → cascading failure. Measure the RTT distribution over time and machines, or use an adaptive failure detector. Queueing delay explodes near max capacity — **spare capacity is what drains queues**.

## 2. Degradation is a product decision

For every UI composed of multiple services, and every service with downstream collaborators, ask: **"what happens if this is down?"** — and answer it as a *business* decision. Hide the cart. Show a phone number. Serve the catalog read-only. Serve stale rather than nothing.

Monolith health is binary; once you have services, someone has to decide what "partially working" means for each surface. If nobody decides, the default is "500".

## 3. Deployment and release

- **Build the artifact once**, reuse it in every environment. One artifact + externalized per-environment config. Never build per-environment artifacts — you didn't test what you shipped.
- Minimize what varies per environment; the more config changes behavior, the more "works only in env X" bugs.
- Prefer immutable images and one service per host/container. Multiple services per host muddles monitoring, creates noisy-neighbor effects, and blocks independent deployment.
- One repo + one CI build per service. A single giant build means slow cycle time, unclear deployables, and drift into lock-step releases.
- **Separate deployment from release.** Blue/green (deploy alongside, smoke test in situ, switch traffic, keep the old version briefly for instant fallback) and canary (route a portion of real traffic, score it on latency, error rate, *and* business outcome).
- **MTTR over MTBF.** Effort on fast rollback + good monitoring usually beats more pre-production tests. Most teams over-invest in test suites and under-invest in recovery.
- **Automation is the answer to host sprawl** — overhead grows linearly with hosts only if you're doing it by hand. Give developers the same deployment tooling as production.

## 4. Monitoring

**Monitor the small things; aggregate to see the big picture.**

Per service: inbound response time (bare minimum) → error rates → application/business metrics. Track the health (response time + error rate) of **every downstream dependency**. Expose domain metrics from the service itself — you cannot know in advance which data you'll want.

System-wide:
- One store that supports both roll-up (whole system) and drill-down (single instance) via metadata. Keep data long enough for trends; downsample old data.
- One queryable log aggregation tool, **standardized log format and metric names** (`ResponseTime` vs `RspTimeSecs` is a real, recurring cost).
- **Correlation IDs**: generate at the first call, propagate through every downstream call *and every event*, log structurally. **Retrofitting is very painful — add them early**, especially with event-driven flows.
- **Semantic / synthetic monitoring**: inject fake transactions and assert the expected result. A far better indicator of real problems than low-level metrics (which you still need for root cause). Use known fake users and watch for real side effects.
- **Monitor the integration points** — two services can both look healthy while the link between them is down.
- Percentiles: rolling window, t-digest/HdrHistogram. **Never average percentiles** — sum the histograms.
- Measure response time **client-side**. Head-of-line blocking means a few slow requests delay everything queued behind them, and a server-side view hides it. If one user request fans out to N backends, the user waits for the **slowest** — a small percentage of slow backend calls produces a large percentage of slow user requests.
- Queue depth / consumer lag is a primary health signal: growth means either the downstream is unavailable (back off) or there are too few consumers (scale out).
- Metrics workloads are **constant heavy write, spiky read** — that asymmetry is why a general-purpose DB is the wrong store for them. Tier retention (raw 7 d → 1-min for 30 d → 1-h for a year). Keep every label low-cardinality or the index explodes.

## 5. Testing

- **Test pyramid**: an order of magnitude more tests at each lower level. The inverted version (mostly end-to-end) produces glacial, permanently-broken builds.
- **End-to-end tests are the trap**: flaky (more moving parts = nondeterminism), unowned, slow, and the moment you version and deploy services together to make them pass, you have conceded independent deployability. Flaky tests cause normalization of deviance — fix or delete them.
- **Test journeys, not stories.** A very small number (low double digits, even for complex systems) of agreed core journeys; everything else covered by isolated service tests.
- **Consumer-driven contracts** are the main replacement: consumer expectations captured as tests, run in the *producer's* CI against the producer in isolation — fast, reliable, and they name the impacted consumer when they break.
- Prefer stubs over mocks; mocks assert that a call happened and get brittle.
- **Non-functional requirements need explicit targets and tests too** — latency, throughput, durability — set **per service** (payment durability ≠ recommendation durability). Set targets so the build actually goes red, and view the results with the same tooling as production monitoring so you're comparing like with like.
- Performance testing matters more once calls cross the network: one DB call becomes 3–4 network hops, and any slow link in a synchronous chain poisons everything.

## 6. Chaos and rehearsal

Deliberately inject failure — most critical bugs are in error-handling paths that never run. Game days, killing instances, killing an availability zone, degrading the network. Pair it with a blameless culture and developers owning their services in production.

Two pitfalls that redundancy does not fix:
- **False alarms cause unnecessary failovers**, and a failover on a loaded system makes things worse.
- **A bug that killed the primary will kill the backups too** — replicas share the code path. Redundancy protects against machines, not logic.

Therefore: **fail over manually while a system is new.** Automate once you've accumulated failure signals and operational confidence.

## 7. Scaling operations

- **Vertical first** is legitimate — quick, and one box goes further than people assume — but the cost is superlinear, software often can't use extra cores, and it buys no resilience.
- **Spread risk**: distinct physical hosts, not one rack, multiple AZs. Beware shared SANs and shared DB infrastructure — saving machines by co-hosting many schemas on one engine reintroduces a catastrophic SPOF.
- **Stateless tiers** scale trivially and fail over for free. Move session state out of app servers. This is the single highest-leverage structural move.
- **Stateful tiers are not stateless tiers.** Anything holding subscriber lists, connections, consumer offsets, or an in-memory index: over-provision for daily peak, **do not autoscale up and down**, resize deliberately at the traffic trough, and make **single-node replacement the routine operation** (it moves far less state than a resize). Mark a node "draining" at the load balancer, stop new connections, wait for existing ones to close, then remove it.
- **Slow startup is an availability risk.** If a node takes minutes to build an in-memory index, it cannot serve while building — roll out to a small subset at a time, and beware a whole new cluster pulling the full dataset at once and hammering the DB.
- **Autoscaling**: predictive (known daily/seasonal shape, scale up *before* the peak) plus reactive. Know your spin-up lead time and keep enough headroom to bridge it. Load-test the scaling rules. Its most valuable use is **failure recovery** ("always ≥ 5 instances"), not load. Be very cautious scaling down.
- **Worker pools**: a reliable shared queue with interchangeable, unreliable workers is ideal for batch, async, and peaky load. The queue must be reliable; the workers need not be.
- Autoscaling a twice-monthly report job, or blue/green for an intranet wiki, is over-engineering. Right-size the operational effort to the thing.

## 8. Security posture that scales

- Coarse-grained authentication at the edge; **fine-grained authorization inside the service that owns the rule**. Encoding service-specific rules into central directory roles puts one service's business logic in a system another team owns.
- **Never write your own crypto or security protocol.** Use well-patched platform libraries, salted password hashing. Badly implemented encryption is worse than none because of the false confidence.
- Keys live outside the data they protect, with rotation and versioning. Encrypt on first sight, decrypt on demand, never persist the decrypted form, encrypt backups too.
- **The confused deputy**: a service acting on a caller's behalf can be tricked into fetching data the caller shouldn't see. An ID from the client is never authorization.
- **Be frugal with data**: don't store what you don't need — truncate IPs, keep an age range rather than a birth date. What you don't store can't be stolen or subpoenaed.
- Defense in depth: perimeter + host firewalls, network segregation per risk level, least-privilege OS users, automated patching, log aggregation for detection — with sensitive fields culled from the logs, or the logs become the target.
- Bake it in: cheap scanners in normal CI, heavier scans at load-test cadence, external penetration testing sized to the release.

**DDoS / public-surface hardening:** isolate public services and data from private ones and serve public reads from read-only copies; cache infrequently-changing data so most queries never reach the DB; **design cacheable URLs** — `/data/recent` is cacheable at the CDN, while `/data?from=123&to=456` lets an attacker mint unlimited unique, uncacheable requests; safelist/blocklist at the gateway; rate limit.

## 9. Governance: keep the "-ilities" from decaying

Architecture is structure **+ characteristics (the -ilities) + decisions (rules) + design principles (guidelines)**. Naming a style describes only the structure. Any design that leaves the characteristics implicit leaves them unmeasured and ungoverned.

- **Fitness functions**: turn each critical characteristic into an objective, automated check that **runs in CI** (a page-load-time test, a dependency rule, a latency budget assertion). Without verification, developers route around decisions for local wins and the required properties quietly stop holding.
- **Guide, don't specify.** "Use a reactive front-end framework" is an architecture decision; "use React" is a technical one — legitimate only when the specific technology is what preserves a characteristic.
- **Architecture vitality**: re-ask every so often whether a design from 3+ years ago is still viable. Structural decay accrues precisely because nobody re-analyzes the existing architecture.
- **Every architecture is a product of its context.** Never copy a pattern without re-checking the constraints that produced it.
- **Test and release environments are part of the architecture.** Fast code changes with a weeks-long release path is not agility.
- Cautionary tale worth remembering: **Pets.com** — traffic arrived, the site slowed, transactions were lost, the company closed. "Too much success can kill the business." Scale *readiness* is a characteristic, not a later optimization — which is not the same as building for 100M users on day one.
</content>
