# Architecture & DDD

### ARCH-1 · Code speaks the ubiquitous language [SHOULD]

**Rule.** Name code in the business's own vocabulary: one named mutator per business transition, in the business's verbs (`Claim`, `Cancel`, `Expire` — never `SetStatus`); domain exceptions that name the violated rule; and when the business renames a concept, rename the code as routine model maintenance, not optional cleanup. A transition method is a sentence in the domain language. That is why aggregates get named transitions instead of generic mutators (ARCH-3): the point is meaning, not style.

**Why.** When code and conversation share words, a domain expert can spot a wrong rule in code review. A generic setter hides exactly those wrong moves.

**When not to apply.** The vocabulary is per bounded context — the same word may legitimately mean different things across contexts; don't unify wordings across a boundary. Infrastructure (outboxes, serializers, retry policies) rightly speaks technical vocabulary; the discipline governs the domain model and its tests.

**Sources.** Evans, *Domain-Driven Design* (Ubiquitous Language).

### ARCH-2 · Shared state has exactly one owner — everyone else keeps ids and their own read-models [MUST]

**Rule.** When multiple services touch the same conceptual state, make exactly one the authoritative owner where mutations commit. Others hold references (ids) and keep **local read-models / replicas of only the data they need** — they do not call back to the owner's record on the hot path. Re-assert the owner's invariants at its boundary at commit time, check all preconditions before mutating anything, and surface a downstream failure as an explicit return message rather than letting work strand. Services **never share tables or databases**, and synchronous cross-service calls are kept to an absolute minimum — communicate with messages that capture intent and what happened, so consumers track state independently.

**Why.** Two writers with two views of "who owns what" is the classic distributed-consistency trap; single authority makes a lost race *resolvable* — one winner commits, the loser detects and compensates. Local read-models keep a consumer fast and available even when the owner is slow or down. Shared databases couple deploys and destroy context autonomy.

**When not to apply.** The cost is eventual consistency: an accepted request can still fail downstream, so that outcome must be made explicit and recoverable, never swallowed. If a single database can legitimately hold both concerns, don't split into services *just* to apply this — the rule is about separately-owned state, not ceremony.

**Sources.** [microservices.io — Database per Service](https://microservices.io/patterns/data/database-per-service.html); Helland, *Life Beyond Distributed Transactions* (CIDR 2007); [Microsoft — Tactical DDD for microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/tactical-domain-driven-design).

### ARCH-3 · Choose the persistence style per case — CRUD, event sourcing, or projections [CONSIDER]

**Rule.** Don't default dogmatically in either direction. Use plain current-state rows (CRUD) when that is all a case needs; reach for **event sourcing where it genuinely adds value** (full temporal audit, replay-to-any-point, retroactive correction — see ARCH-4 for its disciplines); add **read-model projections for query performance** where the write model is awkward to read (see ARCH-10 for theirs). Evaluate per aggregate / use case. When you keep current-state rows, let the entity's named methods — one per legal transition — be the only mutators, each rejecting an illegal transition with a domain exception and mutating in place; no `Apply(event)` methods — those are only meaningful in event-sourced aggregates. Prefer to let each transition method **return the domain events it decided on**, and let the calling slice publish exactly what the aggregate returned. Every fact then has one minting site, and the bug "state changed but no event went out" cannot be written at all. Integration events are still composed at the boundary (ARCH-5, ARCH-9). If you compose events at call sites instead, accept the cost that comes with it: every mutation site needs a pinning test that it emits the matching fact, and event-id minting must be centralized.

**Why.** Treating event sourcing as a house-wide default bolts operational cost onto many cases that don't need it; a stubborn no-projections stance hurts read performance. The right answer varies by case.

**When not to apply.** The rule *is* "it depends" — just make the choice deliberately and write down why.

**Sources.** [Fowler, CQRS](https://martinfowler.com/bliki/CQRS.html); [Fowler, PolyglotPersistence](https://martinfowler.com/bliki/PolyglotPersistence.html); [Microsoft — CQRS pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs).

### ARCH-4 · If you event-source, the stream is the aggregate and the log is immutable truth [MUST]

**Rule.** Where ARCH-3 lands on event sourcing for an aggregate: one stream per aggregate instance, appended in one transaction — the stream *is* the consistency boundary, and the optimistic-concurrency check is the expected stream version on append. Current state is a fold of the stream (an `Apply` per event); **snapshots are a cache, never truth** — rebuildable and droppable at will. Stored events are **immutable forever**: evolve by transforming on read (upcasters / weak-schema mapping), never by rewriting the log.

**Why.** Everything the log buys — audit, replay-to-any-point, retroactive correction — rests on it being append-only and trustworthy; rewrite it once and every projection and every audit answer is suspect. *Example: an auditor asks why a renewal was priced at last March's rate — the event log answers; a current-state row cannot.*

**When not to apply.** Legal erasure (PII) is the recognized exception — handle it as a deliberate, documented migration (copy-transform into a new stream, or crypto-shredding), never an in-place edit. And don't add snapshots until replay time measurably hurts; most aggregates never need them.

**Sources.** [Fowler, Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html); Greg Young, *Versioning in an Event Sourced System*; Vernon, *Effective Aggregate Design*; [Microsoft — Event Sourcing pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing).

### ARCH-5 · The wire format is the contract — each side owns its own types and translates at the edge; no shared kernel [MUST]

**Rule.** For cross-boundary messages, let each side own its copy of the DTOs, route/deserialize by a stable message identity, and translate at a single anti-corruption seam so foreign vocabulary never reaches the core. **The contract is the serialized shape, not a shared assembly.** Avoid a shared "models" kernel; the duplication is a cost deliberately paid to keep contexts independently deployable.

**Why.** A shared kernel couples both contexts' deploy cadences and leaks one's types into the other; per-context copies plus a named translation seam keep them decoupled and independently refactorable.

**When not to apply.** You pay with hand-maintained copies, so lean on the framework's versioning for evolution (message identity + forwarders — see MSG-3) and add a drift gate only where copies can silently diverge (TEST-1). Cross-language / cross-org contracts warrant a schema registry or consumer-driven contracts instead.

*Sketch: Appendix A3.*

**Sources.** Evans, *Domain-Driven Design* (Anticorruption Layer); [Microsoft — Anti-corruption Layer pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer); [Fowler, BoundedContext](https://martinfowler.com/bliki/BoundedContext.html).

### ARCH-6 · Make illegal states impossible to build — value objects and typed sums [SHOULD]

**Rule.** Model concepts as Value Objects, and make invalid states impossible to construct. When a rule says "in mode A, field X must be absent," encode it as a typed sum (a discriminated union / sealed-record hierarchy) so the invalid combination cannot be built at all — rather than as a runtime `if` that someone will forget at one of its many call sites. Centralize construction in one factory. An aggregate or value object should never be constructable in an illegal state.

**⚠ Trap:** A type cannot close a concurrency race. The typed sum makes the illegal combination unbuildable at compile time; the database constraint is the race-free validity backstop at the commit boundary (CON-5). They are complementary, never substitutable — shipping the type without the constraint leaves the race open.

**Why.** Turns a whole class of caught-late, hard-to-trace bugs into a construction-time impossibility, and makes the model self-documenting.

**When not to apply.** Worth it for a load-bearing invariant easy to violate across many sites; over-applying sum types to every nullable field is ceremony. The over-engineering to avoid is narrow: reconstructing at every boundary a typed sum nothing branches on — a load-bearing type can still be persisted flat.

**Sources.** [King, "Parse, don't validate"](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/); Minsky, *Effective ML* (make illegal states unrepresentable); Evans, *Domain-Driven Design* (Value Objects).

### ARCH-7 · Organize code by use case (vertical slices), with its failure policy alongside [SHOULD]

**Rule.** Group code by use case, not technical layer: keep the input type, the handler, the route/subscription registration, and that operation's error-to-status mapping in one file. Each slice carries its own retry/error convention — and, where that convention dead-letters, the alert route per OPS-2 (a MUST there).

**Why.** A reader sees one behavior end to end without hopping a controllers/services/repositories tree; a long lifecycle or async flow stays legible, and each slice tunes its own resilience.

**When not to apply.** Cross-cutting policy that must be uniform — the central concurrency-token bump, transport routing, global serialization — still lives in one place. Slices own only what is specific to them.

**Sources.** [Bogard, "Vertical Slice Architecture"](https://www.jimmybogard.com/vertical-slice-architecture/).

### ARCH-8 · Name events for what happened, in the past tense — capture intent [SHOULD]

**Rule.** Emit a distinctly named, past-tense event per business transition (`OrderShipped`, not a generic `OrderChanged`). The specific name and payload let different subscribers use the same stream differently and carry facts the current-state row never persists (the *why* behind a transition).

**Why.** Different consumers need the stream for opposite purposes: one converges by re-reading current state; another records each event's specific intent, including reasons a current-state row would never store. A generic "changed" event serves neither and loses those facts forever.

**When not to apply.** More event types is more to version; granularity should track *meaningful business transitions*, not every field change. Don't collapse distinct intents into one event just to reduce type count — and don't split one transition into many events consumers can't tell apart. Past-tense naming applies to *events*; a command is a different kind of message with different ownership (MSG-7).

**Sources.** [Fowler, Domain Event](https://martinfowler.com/eaaDev/DomainEvent.html); [Hohpe & Woolf — Event Message](https://www.enterpriseintegrationpatterns.com/patterns/messaging/EventMessage.html).

### ARCH-9 · Distinguish internal domain events from cross-boundary integration events [SHOULD]

**Rule.** Treat "a fact happened inside this service" and "another service must be told" as two kinds of message. Internal events drive local side effects and may stay in the publisher's own vocabulary; integration events are an external contract jointly owned by both ends. Don't let one type serve both roles. Internal vocabulary may stay private — but internal *identity* is not free to change once envelopes persist (MSG-8).

**⚠ Trap:** Defer the second *type*, never the second *name*. Keep both roles named from day one — renaming after an external consumer exists is a negotiation, not a refactor.

**⚠ Trap:** Watch the reverse leak. A wire-contract type (a failure-reason enum, a contract DTO) embedded unmapped in internal events or persisted history couples your internal history to the contract's evolution. Translate it at the boundary, or record the accepted coupling explicitly.

**Why.** Lets you add an internal consumer (e.g. a read-model projector) without touching the cross-context wire contract, while internal names stay rich and wire contracts stay minimal.

**When not to apply.** In a small system the same record can wear both hats. The split trigger is an **external wire consumer**, not system size. Before that consumer exists, splitting is cheap. Once it exists, any rename or enrichment forces the split — and your internal vocabulary is held hostage to the external contract. Don't build tooling to enforce the split before a real second consumer exists.

**Sources.** [Microsoft — Domain events: design and implementation](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation); Vernon, *Implementing Domain-Driven Design*.

### ARCH-10 · Know what a duplicated event and a lost event each do to your consumer — and guard against the one that corrupts it [MUST]

**Rule.** Message delivery misbehaves in two ways: it can hand your consumer the same event **twice**, and it can **lose** one. Know what each does to your consumer, because the two common designs fail on opposite sides. A projector that handles an event by **re-reading current state from the authoritative store** is safe against duplicates and reordering — re-reading twice just writes the same answer — but a lost event can silence it forever: once an aggregate reaches its final state, nothing will ever emit that last event again. This design needs durable delivery. An **append-only recorder** (an activity feed, an audit trail) is the mirror image: a lost event is merely a missing row, but a duplicated event becomes a duplicated row. This design needs dedup on a stable event id. When the recorded history doubles as the audit or compliance trail, both guards become part of the evidence — and they extend to operators: a manual replay or re-emit must itself land a record naming who acted, on what, and why, or the trail silently attributes the operator's action to the system. (The replay procedure itself is OPS-3.) Events carry facts the current-state row never stores (ARCH-8) precisely for these history readers.

**⚠ Trap:** A projector that builds its view from the event's *payload*, instead of re-reading the store, has quietly switched sides: duplicates and reordering now corrupt it. It inherits the recorder's dedup discipline — a stable event id — plus ordering guards.

**⚠ Trap:** Never expose a projected version a client could mistake for the write-side concurrency token — name it as observed (e.g. `ObservedVersion`).

**Why.** The two designs fail on opposite sides: the re-reading projector survives duplicates but dies silently on loss; the append-only recorder tolerates staleness but writes every duplicate as a fresh row. A guard chosen without knowing which design you have protects neither. *Example: the event for a terminal "order delivered" transition is lost in a crash; nothing ever re-emits it, and the dashboard shows the order in transit forever.*

**When not to apply.** A projection rebuilt wholesale from authoritative state (on a schedule, or on deploy) tolerates both loss and duplication between rebuilds; neither guard is load-bearing there. Staleness until the next rebuild is the accepted cost. An ad-hoc cache is an unmanaged read model — the moment it must stay *correct* rather than merely fresh-enough, it inherits this rule's guards.

**Sources.** [microservices.io — CQRS](https://microservices.io/patterns/data/cqrs.html); [Khononov, "Event-Driven Architecture on AWS, Part III"](https://vladikk.com/2024/10/13/aws-eda-iii/); [Microsoft — Materialized View pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view).

### ARCH-11 · Prefer choreography over orchestration; orchestrate only when necessary [SHOULD]

**Rule.** Default to choreography — services react to each other's events with no central coordinator — and read the workflow's progress from the statuses of the entities themselves, as long as every contended step already elects exactly one winner. Reach for explicit orchestration (a process manager / saga coordinator) only when the workflow genuinely needs central coordination: fan-out/quorum, timeouts-as-state, or multi-participant compensation. A framework-native timeout that a coordinator awaits (e.g. a saga `TimeoutMessage`) is the legitimate case for *that* orchestration — awaiting a reply that may never come.

**Why.** A separate coordinator is extra state to keep consistent; when the natural entities already carry enough to express every step, choreography is simpler and more autonomous.

**When not to apply.** Choreography-on-entity-state only holds when every step has a single deterministic winner and statuses fully capture progress. A fan-out/quorum flow, deadlines, or multi-party rollback justify an explicit coordinator — don't contort choreography to fake those.

**Sources.** [microservices.io — Saga](https://microservices.io/patterns/data/saga.html); [Hohpe & Woolf — Process Manager](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html); [Microsoft — Choreography pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/choreography).

### ARCH-12 · Defer not-yet-needed components to a roadmap; ship a thin throwaway seam [SHOULD]

**Rule.** When a future component is desirable but not yet required, record its intended design in a roadmap and ship the thinnest interim seam that unblocks current work — with an **explicit, enforceable deletion criterion** (e.g. dev-only, gated to non-prod, removed before GA).

**Why.** Building it now buys complexity you must carry before you know the design is right; a cheap seam unblocks work without premature commitment.

**When not to apply.** Deferral only works if the interim seam is genuinely cheap to delete and does not leak into the real design; a "temporary" shim the product comes to depend on is worse than building the real thing. The deletion criterion is what keeps it honest.

**Sources.** [Fowler, Yagni](https://martinfowler.com/bliki/Yagni.html).

### ARCH-13 · Design simply, in priority order: tested, intention-revealing, duplication-free, minimal [SHOULD]

**Rule.** When two designs both work, prefer the simpler by Beck's four rules, **in priority order**: (1) it passes the tests; (2) it reveals intention — a reader can tell what it is for; (3) it contains no knowledge-duplication; (4) it has the fewest elements — classes, interfaces, indirections — that satisfy the first three. Build for what the system needs *now*. Speculative generality is a cost paid today for a guess: the parameter nobody passes, the interface with one implementation, the extension point for a future that may not come. Same principle as ARCH-12 and the README's trigger tables: add structure when its trigger arrives, not before.

**⚠ Trap:** The order is load-bearing. "Fewest elements" never licenses deleting a test or obscuring intent — minimality is the tie-breaker, not the goal.

**Why.** Every element is something a maintainer must read, a test must cover, and a change must navigate around; the caveats throughout these guidelines keep warning against over-applied patterns because unneeded structure is the most common self-inflicted complexity. *Example: a strategy interface with one concrete pricing strategy, "for flexibility" — every reader now traces an indirection that encodes nothing; inline it until the second strategy exists.*

**When not to apply.** Simple is not simplistic: a load-bearing invariant still earns its type (ARCH-6) and a hot path its structure. And known, dated requirements are not speculation — designing for next sprint's confirmed feature is just design.

**Sources.** Beck, *Extreme Programming Explained*; [Fowler, BeckDesignRules](https://martinfowler.com/bliki/BeckDesignRules.html); [Fowler, Yagni](https://martinfowler.com/bliki/Yagni.html).

### ARCH-14 · Tolerate duplication until the abstraction is proven — and inline the wrong one back [SHOULD]

**Rule.** Duplication is a hint, not an emergency. Abstract when the **third occurrence** proves the pattern (the rule of three) — two similar blocks may be coincidence; three reveal a shape worth naming. Deduplicate **knowledge** (a business rule, an invariant — one authority), not **incidental likeness** (two blocks that merely look alike today and will diverge tomorrow). And when an abstraction turns out wrong — you can tell because every new caller adds another parameter or conditional — **inline it back into its call sites and let the duplication re-emerge**. Starting over from duplication is cheap; forcing the wrong shape onto each new case gets more expensive every time.

**⚠ Trap:** The wrong abstraction survives on sunk cost — each new caller adds one more flag "because the helper already exists." The fix is never one more parameter; it is un-abstracting.

**Why.** Duplication costs a multiplied edit; the wrong abstraction costs every future reader and caller — they are not symmetric ("duplication is far cheaper than the wrong abstraction," Metz). *Example: two checkout flows share a "process payment" helper; the third flow needs a deposit, so the helper gains a flag, then a mode enum, then a callback — inlined back into three plain flows, each reads in one screen.*

**When not to apply.** Knowledge-duplication of an *invariant* doesn't wait for three occurrences — a business rule encoded twice can disagree, and one authority is the point. Duplicated *contract* shapes are governed by ARCH-5 and its drift gate, not by this rule.

**Sources.** [Metz, "The Wrong Abstraction"](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction); Fowler, *Refactoring* (rule of three); Hunt & Thomas, *The Pragmatic Programmer* (DRY is about knowledge, not text).

### ARCH-15 · Keep decision logic pure; push I/O to the edges (functional core, imperative shell) [SHOULD]

**Rule.** Separate *deciding* from *doing*. The core is **pure**: domain rules, transition legality, calculations, and the choice of which messages to emit, all computed from in-memory inputs and returned as outputs — no I/O, no clock, no randomness. (The seams of TEST-8 exist precisely to keep it so.) The shell loads state, invokes the core, and performs the effects the core decided; handlers, slices, and repositories all live there. These guidelines already apply that shape at the large scale — aggregates decide while slices do I/O, and handlers re-read then re-decide (IDEM-6); apply it at function scale too: a function that both computes a verdict and writes it somewhere is two functions.

**⚠ Trap:** A unit test that needs a container, a broker stub, or a real clock to check a *decision* is the seam detector — the decision is trapped in the shell. Move the logic, not the test (TEST-3).

**Why.** Pure decisions are the cheapest code to test exhaustively (the broad base TEST-3 wants), trivially safe under retry and replay, and readable without simulating the world in your head. *Example: "may this order ship?" as a pure function over order + inventory snapshots gets thirty table-driven tests in milliseconds; the same logic inline in the handler needs a booted host per case.*

**When not to apply.** Something must do the I/O; shell code is fine, and is tested at its own layer (TEST-4). Don't force genuinely I/O-shaped work (a streaming export) into purity ceremony. The rule targets decisions, not plumbing.

**Sources.** Bernhardt, "Functional Core, Imperative Shell"; Ousterhout, *A Philosophy of Software Design*.
