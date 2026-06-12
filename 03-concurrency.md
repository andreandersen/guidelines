# Concurrency

### CON-1 · Protect an aggregate with optimistic concurrency; choose optimistic vs pessimistic by conflict frequency [MUST]

**Rule.** Use optimistic concurrency via a version token, modifying **one aggregate per transaction**, as the default for protecting an aggregate's invariants (Vernon, *Effective Aggregate Design*). When N writers race for a single resource, one row + one token elects exactly one winner; the rest get a stale-token rejection — no locks, queue, or coordinator. Choose optimistic vs pessimistic locking by **conflict frequency and the cost of wasted work** (Fowler, PoEAA *Optimistic* / *Pessimistic Offline Lock*): optimistic when conflict is rare, pessimistic when conflict is likely or the bounced work is costly. For genuinely collaborative, high-contention state, first ask whether the race is a *requirements* problem you can design away — by introducing time, or a long-running process — before reaching for any lock (Udi Dahan, "Race Conditions Don't Exist").

**⚠ Trap:** The version token arbitrates *different* writers; it does not deduplicate the *same* one. A retried command that re-reads fresh state and re-applies its delta passes every version check and lands twice — suppressing repeats is a dedup guard's job (IDEM-1, IDEM-3). The chapter-04 opener maps which guard answers which question.

**Why.** Optimistic concurrency prevents double-commit without locks and keeps throughput high when conflicts are rare; the right tool is a function of how often writers actually collide. *Example: two buyers claim the last unit in stock; the version token elects exactly one, and the loser gets a clean conflict instead of a double-sale.*

**When not to apply.** *False sharing.* One row / one token means two operations on different sub-parts of one aggregate still collide on its single version, capping per-key throughput. If a key is structurally hot, split the aggregate, use finer-grained tokens, or model it differently.

*The multi-aggregate carve-out.* Sometimes a true invariant genuinely spans multiple aggregates in one context and one store (e.g. conservation across an atomic exchange). The first option is to redraw the boundary so the invariant fits inside one aggregate. The second is a deliberate, documented multi-aggregate transaction, under three disciplines:

- every precondition checked before any mutation;
- a concurrency token on every mutated row;
- no partial state left to compensate (see CON-5 on matching an invariant's scope to its enforcement point).

The license is the invariant plus that discipline, never merely "same database."

*Sketch: Appendix A5.*

**Sources.** Vernon, *Effective Aggregate Design* (dddcommunity.org); Fowler, PoEAA [Optimistic](https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html) / [Pessimistic](https://martinfowler.com/eaaCatalog/pessimisticOfflineLock.html) Offline Lock; [Dahan, "Race Conditions Don't Exist"](https://udidahan.com/2010/08/31/race-conditions-dont-exist/).

### CON-2 · Bump the concurrency token in one central save hook, not in each method [SHOULD]

**Rule.** Increment the optimistic-concurrency version for every modified entity in a single override of the persistence layer's save method, not in each domain method. Business logic mutates state and enforces invariants; the version is infrastructure, owned by infrastructure.

**Why.** No transition method can forget to bump it or bump it twice; one provably-correct place to reason about the token.

**When not to apply.** Requires an ORM exposing a change tracker and save hook; with raw SQL or an event store you bump explicitly in the write path. And question whether you need an app-managed counter at all: if no **application-visible monotonic version** is needed, the store's native token (Postgres `xmin`, SQL Server `rowversion`, a document store's built-in versioning) gives the same protection with zero application code. The hand-rolled counter earns its keep when the version is part of the public API (CON-4's `ExpectedVersion` / ETag), feeds ordering logic, or must be pinnable in tests — native tokens are opaque and carry no business meaning.

**Sources.** [Fowler, PoEAA — Optimistic Offline Lock](https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html).

### CON-3 · Reject doomed work at the earliest authoritative step (Fail Fast applied to contention) [CONSIDER]

**Rule.** If a resource can be committed to N competing operations, prefer to make them contend at the *first* step that touches authoritative state — provided the conflict is cheap to detect there. Losers are then rejected before they ever exist as durable state, instead of all N being created and N−1 failing downstream as garbage someone must clean up. This is a **case-by-case heuristic** (an application of Fowler's "Fail Fast"), not a universal law.

**Why.** Resolving late lets doomed work accumulate as failures something must clean up; resolving early eliminates that load.

**When not to apply.** There is **no single named best-practice** for this — it's folklore, not doctrine. Earliest-point contention raises per-key write traffic at the front door, so a hot key serializes sooner; worth it when conflicts per key are rare. For *set-wide* constraints specifically, the alternative is to validate against eventually-consistent data and **compensate** (Greg Young, "Eventual Consistency and Set Validation") rather than enforce up front.

**Sources.** [Fowler, "Fail Fast"](https://martinfowler.com/ieeeSoftware/failFast.pdf); Greg Young, set validation (original post defunct; via community relays).

### CON-4 · Expose a client precondition (ETag / ExpectedVersion), but keep the store's token as the real guard [MUST]

**Rule.** Let callers optionally assert the version they read — an `If-Match` / `ExpectedVersion` precondition that fails fast and cheap before work is done — but never rely on it for correctness. The store's own concurrency token is the real guard and catches both callers who omit the precondition and true races. This client precondition is **especially valuable in collaborative domains**, where multiple actors edit the same state and last-writer-wins would silently lose edits.

**Why.** The client precondition is an optimization, not a guarantee — a race can still open between check and commit; two layers mean a forgetful client cannot cause a lost update.

**When not to apply.** Adding the client-precondition half before any caller needs it is gold-plating — the mandatory engine token alone is correct. Add the precondition when a collaborative UI genuinely needs optimistic-concurrency feedback.

**Sources.** [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) (ETag, If-Match, 412); [Sookocheff, "Optimistic Locking in a REST API"](https://sookocheff.com/post/api/optimistic-locking-in-a-rest-api/).

### CON-5 · An invariant inside one aggregate gets the version token; a rule across many rows gets a database constraint [MUST]

**Rule.** A true invariant needing immediate consistency lives **inside one aggregate's transactional boundary**, protected by its version token (Vernon: *model true invariants in consistency boundaries*; *one aggregate per transaction*; everything else is eventually consistent). **Set-wide uniqueness ("at most one active X across all instances") is NOT an aggregate invariant** — it spans instances, so it cannot be enforced at the aggregate root. Enforce it with a domain-service check, which can see global data, **backed by a database unique constraint — the only race-free enforcement point**. Or treat it as a business decision, per Greg Young: validate against eventually-consistent data and compensate when a rare conflict slips through.

**⚠ Trap:** The constraint is only race-free if the engine actually enforces it. Some stores scope uniqueness per partition; some only deduplicate eventually in the background. Verify the engine's real guarantee before leaning on it — an unenforced "constraint" leaves the check-then-write race open.

**Why.** A service-level uniqueness check alone has a check-then-write race; the DB constraint is the race-free backstop. Trying to enforce a set-wide rule as an aggregate invariant violates the consistency-boundary rule. *Example: "at most one active subscription per customer" spans rows — no single aggregate instance can see its siblings; only the unique index closes the race.*

**When not to apply / store specifics.** You trade off completeness, purity, or performance (Khorikov's "DDD Trilemma"). Mechanisms differ: PostgreSQL **partial unique index** (`WHERE` predicate); SQL Server **filtered unique *index*** (filters can't go on constraints); Cosmos DB unique keys scope per logical partition and are settable only at container creation; ClickHouse's ReplacingMergeTree dedups eventually, in the background. A DB constraint also "doesn't embed the rule in the model" — accept that as the pragmatic price.

**Sources.** Vernon, *Effective Aggregate Design*; [Khorikov, email-uniqueness](https://enterprisecraftsmanship.com/posts/email-uniqueness-as-aggregate-invariant/) & DDD Trilemma; [ardalis/DDD-NoDuplicates](https://github.com/ardalis/DDD-NoDuplicates); Greg Young, set validation; Postgres / SQL Server / Cosmos DB / ClickHouse docs.

### CON-6 · Find your consumer's concurrency knob — defaults are often 1 [CONSIDER]

**Rule.** A throughput ceiling reached while CPU and datastore sit idle usually traces to a single-threaded intake stage, not hardware. Messaging frameworks frequently default consumer/listener concurrency to 1, serializing one round-trip per message; find that knob in the source and raise it as the first lever — scoped to the data path, not shared control-plane infrastructure. (Typical knobs: listener/consumer count, max-concurrent-calls, prefetch depth.)

**⚠ Trap:** Not every knob is safe to raise. A framework's *global dispatch* concurrency can reconfigure control-plane channels and break the runtime — scope the change to the data path's listener, never to shared infrastructure.

**Why.** Settles whether you're hardware-bound (often you're not) and can multiply throughput with a configuration change, no rearchitecting.

**When not to apply.** Highly problem-domain-specific — there is no one-size-fits-all setting. The lever is only worth chasing when the metric matters (PROC-8).

**Sources.** [Hohpe & Woolf — Competing Consumers](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CompetingConsumers.html). Pairs with scale-out via the process model (RUN-2; 12-Factor VIII); [Microsoft — Competing Consumers pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers).

### CON-7 · Isolate the hot key with a bulkhead; don't cap global concurrency to contain it [CONSIDER]

**Rule.** When one hot or hostile key concentrates contention, bound *that key's* blast radius — a per-key admission limit (a bulkhead: contain the damage to the hot key, leave every other key untouched) or atomic conditional updates — without throttling everyone. Reject "fixes" that buy isolation by imposing a **global** concurrency cap, which taxes the common well-distributed case to insure against a rare one. **Measure the actual contention under realistic load before adopting any structural fix.**

**Why.** If a realistic per-key test shows near-zero conflicts, a global cap is pure cost on normal traffic — the wrong trade for a high-volume system.

**When not to apply / mechanism.** The *mechanism* depends on your runtime — a virtual-actor model (e.g. Orleans grains, Dapr actors) gives per-key single-threaded isolation natively (the bulkhead is free); without one you build it. And if hot-key contention is the *common* case, slot-based isolation may be the right, simpler choice — zero conflicts in a synthetic test can hide a real production hotspot.

**Sources.** Nygard, *Release It!* (Bulkheads); [Microsoft — Bulkhead pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead).
