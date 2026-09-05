# Service Boundaries

## The one test that matters

**"Can I change this service and deploy it by itself, without changing anything else?"** If no, you have distribution costs without the benefits. Autonomy is the load-bearing property — not smallness.

## When NOT to split

- **Greenfield is the worst place to start.** A new domain means guessed boundaries, and it is far easier to chunk up something you have than something you don't. **Start monolithic (or coarse), model bounded contexts as modules, promote to services once the boundaries stop moving.**
- **Premature decomposition is expensive.** A documented case: boundaries guessed before the domain was understood → constant cross-service changes → the team deliberately **merged back into a monolith**, learned the domain for a year, then split successfully. Re-monolithing is a legitimate move, not an admission of defeat.
- **Fix deployment, testing, and monitoring first.** Manual processes and per-machine monitoring survive 1–2 services, not 10. Teams that scaled to dozens/hundreds of services built the tooling *before* going wide.
- **Sizing tradeoff curve:** smaller services give more independence *and* more moving-part complexity, simultaneously. Only shrink further as your operational capability grows.
- If the org heavily restricts developer autonomy, the payoff of the split largely evaporates.

Cheaper decompositions exist — modules and shared libraries — but they cost you independent deploy, independent scaling, and the *seams* needed for resilience measures.

## Finding the boundary

- Two virtues: **loose coupling** (change one, deploy one) and **high cohesion** (a change hits one place).
- **Bounded context**: a specific responsibility with explicit boundaries, an internal hidden model and a deliberately smaller shared model it exposes. Never expose the internal representation as the shared one.
- The same word means different things in different contexts (a "return" is a shipping label + refund to one context and a restock request to another). That divergence is fine and internal.
- **Model on business capabilities, not data.** Ask "what does this context *do*?" before "what data does it need?" — data-first thinking produces anemic CRUD services with a "god" service orchestrating them.
- **Technical boundaries are a secondary driver, never primary.** Splitting into frontend / data-access layers (a horizontal slice) produces components that always change together and talk chattily. **Slice vertically by business capability.**
- Start with coarse contexts and subdivide later. Choose nested vs top-level by team structure (separate teams → separate services) and by testing cost (a coarse boundary is a cheaper isolation unit).
- Look for **circular references and chatty pairs** on the whiteboard — those are probably one service. Make the mistakes where they're cheapest: moving code is cheap; splitting a database or rewriting a widely-consumed API is expensive.

## Splitting an existing system

1. **Find seams** — code that can be worked on in isolation. Reorganize the monolith's *packages* around contexts **before** extracting anything; the leftovers that fit nowhere reveal contexts you missed. Dependencies that don't exist in the real org are bugs in the model.
2. **Choose which seam first** by: pace of change (extract what's about to change a lot), team/geography, security isolation, and fewest tangled dependencies (extract leaf-most first).
3. **Split the schema first, keep the code together.** Verify, then split the code. Reversible, invisible to consumers.
4. Database recipes:
   - FK across contexts → replace the join with an API call. You lose referential integrity; whether dangling references are acceptable is a **business** decision. Ask "how fast does it need to be, and how fast is it now?" — slower is fine if slower is still acceptable.
   - Shared static data → config file or enum; a duplicate table is acceptable; a whole service is overkill.
   - Shared *mutable* data written by two contexts → usually a **missing domain concept** implicitly modeled in the database. Make it concrete and give it an owner.
   - One table conflating two concepts → split the table.
5. Losing the transaction is the real cost — see `distributed-correctness.md` §4. In preference order: retry later (eventual consistency) → compensating transaction → distributed transaction (never write your own). **If state must stay consistent, try very hard not to split it.**
6. Reporting: don't let the reporting query shape leak back into the operational schema. An **event pump** (subscribe to state-change events) decouples the reporting store from internal schemas; a DB-level pump is acceptable only if the same team owns and deploys it with the service.

**It is fine for a service to grow until it needs a split.** Services get too big because splitting is hard — so invest in making splitting cheap (templates, self-service provisioning), not in preventing growth.

## Integration

Criteria for any integration technology: avoid breaking changes, stay technology-agnostic, be simple for consumers, and **hide internal implementation detail**.

- **A shared database between services is the top anti-pattern.** The internal schema becomes a huge brittle public API (kills loose coupling) and the logic for changing an entity is duplicated across every consumer (kills cohesion). It also pins consumers to your database technology.
- **REST over HTTP is the sensible default** for service-to-service. Binary RPC is faster but couples you to stubs and lock-step releases; REST wins on debuggability, ecosystem, and evolvability. Very low latency justifies something else.
- **RPC's location transparency is a fallacy.** A network call is not a local call: a timeout leaves you not knowing whether it executed, retries duplicate effects unless the protocol is idempotent, and latency varies by orders of magnitude.
- **Choreography over orchestration** as the default: emit `ThingHappened`, let subscribers react. New subscribers need no change to the publisher. The cost is that the business process is only implicit — so **model the intended process explicitly in monitoring** so you can spot steps that silently didn't happen. Heavily orchestrated systems become brittle god-services surrounded by anemic CRUD.
- **Keep the middleware dumb and the endpoints smart.** Business logic in the bus is the ESB mistake.
- **Services are state machines, not CRUD wrappers.** The owning service decides whether a state change is allowed. If that decision leaks out, cohesion is gone.
- Beware frameworks that serialize database objects straight onto the wire — that coupling costs more than the effort of avoiding it.
- **Access by reference**: a fetched entity is a memory that ages. Events should say what happened *and* carry a reference so consumers can re-fetch current state.

## Versioning without lock-step

- **Defer breaking changes as long as possible.**
- **Tolerant Reader**: extract only the fields you use and ignore the rest. Be conservative in what you send, liberal in what you accept.
- **Expand and contract**: coexist old and new endpoints inside one running service, migrate consumers, then delete the old. Internally translate V1→V2 so dead code is obvious. Three concurrent versions is a mess.
- Catch breaks early with **consumer-driven contracts** run in the producer's CI.
- Running multiple concurrent *deployed* versions is only for short windows (blue/green, canary) or rare legacy-client cases. Otherwise it means branching, routing smarts in middleware, and shared persistent state across versions.

## Coupling hazards

- **Don't violate DRY within a service; be relaxed about violating it across services.** Too much coupling is far worse than duplicated code. Shared domain-object libraries force lock-step upgrades.
- Shared code is fine only when it never leaks past the boundary (logging utilities, HTTP client wrappers that propagate correlation IDs). Service templates should be copyable or optional, not a mandated central framework.
- Client libraries: separate transport concerns from destination-service concerns, and **the client must control when it upgrades**.

## Org shape (Conway's Law)

- System structure mirrors the organization's communication structure. In one large study, organizational metrics were the *strongest* predictor of component defect-proneness — beating code complexity.
- **Coordination cost rule**: as the cost of coordinating a change rises, people either lower it or stop making changes. The latter is how large unmaintainable codebases happen. Geography and time zones are therefore a good seam.
- **One service, one owning team**, owning the full lifecycle: requirements → build → deploy → operate. Teams that deploy their own service make it easy to deploy.
- **Shared services are a smell.** Understand the driver: too hard to split (invest in decomposition), a layer-aligned org (realign to business domains), or a delivery bottleneck (hand ownership to the team driving the change — justified by expected *future* volume of change, not one feature).
- Conway in reverse: structure ossifies into org structure. Changing the architecture may require changing the org first.

## Standards worth fixing globally

Be liberal inside a service, strict about what happens between services:
- Uniform health, metrics, and log emission, aggregated centrally.
- One or two integration styles, with the details settled (pagination, versioning, error semantics).
- Every downstream call gets its own connection pool and a circuit breaker; honor response-code semantics or your safety measures silently break.
- **Governance through code, not documents**: a real exemplar service plus an optional service template, so the easy path is the correct path.

## The closing principle

Make each decision **small in scope**, so being wrong damages little. Evolutionary architecture — a series of changes — never a big-bang rewrite.
</content>
