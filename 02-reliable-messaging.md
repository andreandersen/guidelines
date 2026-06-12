# Reliable Messaging

### MSG-1 · Commit the state change and its outgoing messages in one transaction (transactional outbox) [MUST]

**Rule.** Never write a row and then separately publish a message about it — a crash between the two loses or invents facts. Compose the outbound messages from the **post-write entity state**, never by echoing the raw inbound request — the request hasn't had invariants applied and lacks server-assigned ids. Enlist the messages in the same transaction via a **transactional outbox** and let a relay ship them after commit; a writer that loses the concurrency race rolls back and therefore emits nothing. The relay is now load-bearing infrastructure: export its unrelayed depth and oldest-age (OPS-5). This is non-negotiable for **business transactions**.

**Why.** This is the core reliability guarantee, and the aggregate never touches the broker. *Example: the order row commits but a crash eats the publish — fulfillment never hears of the order; or the publish escapes and the insert rolls back — a shipment for an order that was never placed.*

**When not to apply.** Requires an outbox-capable framework or a hand-rolled outbox table + relay, and adds a relay hop of latency plus a measurable per-operation commit cost (~20% in one measured system). Pay that cost wherever losing the message is unrecoverable; relax it only where the publish is provably idempotent and can be re-derived from committed state.

**Sources.** [microservices.io — Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html).

*Sketch: Appendix A2.*

### MSG-2 · Cross-boundary failures travel as stable enumerated codes, never free text [MUST]

**Rule.** When a failure outcome crosses a service boundary, carry a stable enumerated reason code serialized **by name**, never an internal diagnostic sentence. The consumer branches on the code; the producer's wording and any internal data stay private. Name each member as the **business fact the consumer reacts to**, in the contract's vocabulary (`InsufficientFunds`), never the producer's internal mechanism (`DbTimeout`, `ERR_43`). Reserve at most one infrastructure-shaped member for converted transient failures (see MSG-4). Evolve **append-only**: never rename or remove a member; positional ordering matters only under numeric serialization.

**⚠ Trap:** By-name serialization is rarely the default. Most serializers write enums as integers unless a string converter is explicitly registered on the type or in the options; omit it and the wire silently carries numbers. Pin the chosen representation in the drift gate (TEST-1).

**⚠ Trap:** Adding a code is a deploy-ordered, two-context change. Deploy the consumer first, or an unknown name fails to deserialize.

**Why.** Free text pins the contract to a sentence (any reword breaks consumers) and risks leaking internal state; a code is a deliberate, versioned part of the contract the consumer can switch on.

**When not to apply.** Failures that stay inside one service may remain rich exceptions in private vocabulary — the coded-enum discipline is for outcomes that cross an ownership line.

*Sketch: Appendix A4.*

**Sources.** [Stripe — error codes](https://docs.stripe.com/api/errors) (a stable-code error contract in the wild).

### MSG-3 · Evolve contracts by adding fields; version only a true break, through the framework's forwarders [MUST]

**Rule.** Adding an optional field is not breaking — keep the same logical version; old consumers ignore it, new consumers default it when absent. Reserve a version bump for breaking changes: removal, rename, type change, or **a semantic shift in an existing field's meaning**, which is breaking even when the shape looks additive. For an *intentional* breaking change, use the framework's idiomatic versioning rather than a bespoke scheme — typically a stable message identity carrying a version tag, plus a registered forwarder/upcaster that transforms old messages into the new type on deserialize. The HTTP door's request and response shapes are wire contracts under this same regime: evolve additively, reserve a versioned route for a breaking change, and pin the shape with endpoint tests (TEST-4) the way golden samples pin messages — deprecating the old route is a contract step (MSG-9), never a silent removal.

**⚠ Trap:** Additive is only safe if the serializer tolerates unknown inbound fields *and* defaults missing ones. Verify both directions before relying on it.

**Why.** Lets you enrich messages for audit and downstream readers without coordinating a two-service version bump per change; registered forwarders handle the intentional upgrades cleanly.

**When not to apply.** If producer and consumer genuinely deploy in lockstep from one repository *and* no serialized message survives the deploy (MSG-8), a coordinated breaking change without a version bump is defensible — say so in the PR.

**Sources.** [Fowler, TolerantReader](https://martinfowler.com/bliki/TolerantReader.html); [Robinson, "Consumer-Driven Contracts"](https://martinfowler.com/articles/consumerDrivenContracts.html); [Hyrum's Law](https://www.hyrumslaw.com/).

### MSG-4 · When retries run out, emit the coded failure — never leave the caller waiting forever [SHOULD]

**Rule.** A transient operation that exhausts its retries must not vanish into the dead-letter queue while an upstream waiter blocks forever. Attach a *best-effort* terminal continuation that emits the same coded failure outcome the business path would. That continuation runs after the handler's transaction rolled back, outside the outbox, so it can itself be lost — which is why you pair it with the durable backstop: a saga timeout (IDEM-8), which actually guarantees the caller resolves. The common framework idiom is a terminal continuation chained onto the retry policy (a compensating action) — and it should *publish a message*, not do work inline.

**Why.** Turns infra failure into a business-shaped outcome so the caller's state machine always resolves.

**When not to apply.** Where no upstream waiter exists — fire-and-forget notifications, self-contained background work — there is nothing to strand, and the continuation isn't worth wiring.

**Sources.** [microservices.io — Saga](https://microservices.io/patterns/data/saga.html) (compensation); Nygard, *Release It!*; [Microsoft — Compensating Transaction pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction).

### MSG-5 · Put the retry counter on the entity; keep the max-attempts ceiling in policy [CONSIDER]

**Rule.** Let the durable entity own the retry *count* and guard only the legality of a retry (which states it may begin from); keep the maximum-attempts *ceiling* in a separate, swappable policy layer. Count completed failures honestly — never increment early. Order exception-handling rules **specific-to-general** under a first-match-wins matcher (register the narrowest exception first; a derived exception placed after its base's catch-all gets the wrong policy).

**Why.** Separates the durable fact (how many times this failed) from the operational decision (how many to keep trying), so the ceiling changes without a data migration.

**When not to apply.** Counting attempts on the entity is only worth it when you need per-attempt provenance; a stateless retry keeps everything in the policy and leaves the entity untouched. The ordering point is specific to first-match-wins matchers — a priority-based matcher changes the required order; verify the derivation chain rather than assuming it.

### MSG-6 · Every handler tunes its own retry and error policy — one global policy fits nobody [SHOULD]

**Rule.** Different handlers have different business deadlines and costs-of-loss; let each declare its own retry ladder and dead-letter policy. A background projection with no deadline can retry long and patiently; a user-facing leg cannot. Space retries with **exponential backoff plus jitter** — synchronized retry storms re-saturate the resource that just failed.

**⚠ Trap:** A per-handler policy must cover **all** of that handler's fault modes. A ladder matching only one exception type silently drops every other failure to the global policy — the handler looks tuned and isn't.

**Why.** One global policy can't be right for both — a background projection failing under a short user-facing policy can permanently lose a terminal-state view that nothing re-emits.

**When not to apply.** More knobs to get right — diverge from the global default only when a handler's needs genuinely differ.

**Sources.** [Brooker (AWS Builders' Library), "Timeouts, retries, and backoff with jitter"](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/); [AWS Architecture Blog, "Exponential Backoff and Jitter"](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/).

### MSG-7 · A command is addressed to one owner; an event is a broadcast fact — don't blur them [SHOULD]

**Rule.** A **command** is directed at exactly one logical handler, owned by the receiver, named imperatively (`ReserveStock`), and may be rejected. An **event** is a past-tense broadcast fact owned by the publisher (`StockReserved`); any number of consumers may subscribe, and the publisher must not care who they are. Past-tense intent naming (ARCH-8) applies to events; never name a command as if it had already happened, and never let an event's consumers instruct the publisher back through the same channel.

**Why.** Ownership and cardinality differ: adding a second consumer to an event is free; a second handler of a command is a bug — the instruction executes twice. Blurring the two hides that hazard until it fires.

**When not to apply.** This is a *semantic* distinction, not a transport prescription — a command may legitimately ride whatever topology the broker offers (consumer-side dedup makes accidental double-execution survivable); don't re-route working infrastructure just to honor the taxonomy.

**Sources.** [Hohpe & Woolf — Command Message](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CommandMessage.html) and [Event Message](https://www.enterpriseintegrationpatterns.com/patterns/messaging/EventMessage.html); [Microsoft — Publisher-Subscriber pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/publisher-subscriber).

### MSG-8 · Pin a stable message identity the moment envelopes outlive the process [MUST]

**Rule.** Some messages are **durably stored and read back by differently-versioned code**: outboxed past a deploy, parked on a durable queue, dead-lettered, or scheduled for the future. Give every such message an explicit stable identity (e.g. a declared `"name.v1"`). This holds even if the message never crosses a service boundary. An identity derived implicitly from the class name orphans every already-stored envelope on a rename or namespace move.

**Why.** Renames are routine refactoring; stored envelopes are forever. The class name is an accident of code layout, not a contract — the moment bytes persist, identity is part of the wire shape (ARCH-5) whether you declared one or not.

**When not to apply.** Genuinely ephemeral, in-memory-only messages can ride the default identity. The trigger is **persistence that outlives a deploy, read by independently-versioned code** — not "uses messaging" (under a transactional outbox every message is briefly stored; the hazard is the envelope still sitting there when the new binary boots).

**Sources.** Young, *Versioning in an Event Sourced System*.

### MSG-9 · Evolve durable shapes expand → migrate → contract; every migration runs against the currently-deployed binary [MUST]

**Rule.** Rows, stored envelopes, and wire contracts all outlive the binary that wrote them; evolve them in three independently-deployed steps. **Expand**: add the new shape additively, so old and new binaries both work. (MSG-2's consumer-first deploy and MSG-3's additive field are this same step, applied to contracts.) **Migrate**: move the data and switch readers and writers, within the already-deployed binary's tolerance. **Contract**: remove the old shape only when nothing deployed reads or writes it. The invariant across all three: **every migration must be compatible with the currently-deployed binary.** During a roll, old binary and new schema coexist; a migration that breaks the running version turns a deploy into an outage. Keep migrations **forward-only** — fix forward with the next migration; down-scripts are rarely exercised and can't be trusted when finally needed — and run the full chain in CI **against the real engine** (TEST-2): an in-memory provider validates neither the DDL, nor lock behavior, nor the data transform.

**⚠ Trap:** A logically-additive `ALTER` can still rewrite or lock a hot table for minutes — verify lock behavior at realistic volume before it meets production.

**⚠ Trap:** A migration interrupted mid-run will be re-applied. Where the engine lacks transactional DDL, write each step guarded (create-if-absent) and each backfill batched and resumable — a migration, like a message, can be redelivered, and must be safe to run twice.

**Why.** A one-step "change the column and the code together" assumes an instant when no old binary exists; rolling deploys, replicas, and queued messages guarantee that instant never exists. Three steps make each deploy independently safe, and revertible by redeploy alone. *Example: renaming `amount` to `amount_minor` in one step breaks every still-running replica mid-roll; expand (add the column, dual-write), migrate (backfill, read the new column), contract (drop the old) never does.*

**When not to apply.** Pre-production, with disposable data and a single instance, drop-and-recreate is honest and cheaper — adopt the discipline at the first deploy whose data must survive. Legal erasure is a deliberate, documented migration, not a routine contract step (ARCH-4). And the contract step is not optional hygiene: skip it long enough and every concept carries two names — schedule the contract when you schedule the expand.

**Sources.** [Fowler, ParallelChange](https://martinfowler.com/bliki/ParallelChange.html) (expand–contract); [Fowler & Sadalage, "Evolutionary Database Design"](https://martinfowler.com/articles/evodb.html). Pairs with build-once-promote (RUN-3; 12-Factor V).

### MSG-10 · Every synchronous call gets a timeout from its caller's deadline, bounded retries, and a circuit breaker [MUST]

**Rule.** Every synchronous out-of-process call — HTTP, gRPC, database, cache — carries an **explicit timeout derived from the caller's own deadline**, never from the dependency's average latency: an unbounded call borrows the caller's entire budget. Retries on synchronous calls are **bounded, spaced with backoff plus jitter (MSG-6), and reserved for idempotent operations** (IDEM-1, IDEM-3). A dependency failing repeatedly gets a **circuit breaker**, so callers fail fast instead of queuing behind a dead socket — and the open-circuit response is a deliberate decision (a degraded answer, a default, or a coded failure per MSG-2), never a raw exception.

**⚠ Trap:** A handler blocked on an un-timeboxed call stalls its entire queue when consumer concurrency is 1 (CON-6) — and the retry ladder (MSG-6) never fires, because nothing throws.

**Why.** The asynchronous legs get retry ladders, dead-letter routes, and timeout backstops; the synchronous call *inside* a handler gets none of that unless it is bounded — it is the one blind spot in everything the other chapters build. *Example: a payment provider hangs instead of failing; every checkout handler waits out a multi-minute socket default, the queue backs up, and the outage spreads to order placement — a two-second timeout and an open circuit would have shed it to a coded failure in milliseconds.*

**When not to apply.** In-process calls need no timeout ceremony. A genuinely long operation (a report, a bulk export) is a *job with progress*, not a long synchronous call — model it as messaging. And circuit breakers earn their keep on dependencies with independent failure modes; don't reflexively wrap the database your transaction already depends on.

**Sources.** [Fowler, CircuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html); Nygard, *Release It!* (Timeouts, Circuit Breaker, Bulkheads); [Microsoft — Retry pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry), [Circuit Breaker pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker).
