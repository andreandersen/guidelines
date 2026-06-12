# Engineering Guidelines

Practical engineering guidance for distributed, message-driven systems, focused on the failure modes that appear once services communicate asynchronously, deploy independently, and persist contracts beyond a single release. These guidelines help teams apply the right discipline at the right time: each rule states what to do, why it matters, and **when not to apply it**.

## Scope

These rules target distributed, message-driven systems: independently deployable services, asynchronous messaging, and contracts or stored data that outlive any single deploy. Inside a single service with one database, simpler methods often suffice — a local transaction already commits state and side effects atomically, a method call needs no wire contract, and a version column plus a unique index covers most concurrency. A rule activates on a structural fact about your system, not on plans for it: the first asynchronous message, the first independently-deployed consumer, the first serialized shape that must survive a deploy. The trigger table below maps each such threshold to its rules; the always-on principles apply at any scale.

These chapters govern how the team engineers the system: domain modelling, reliable messaging, concurrency, failure handling, performance investigation, testing, the security obligations the architecture itself creates, the contract with the platform that runs it (the [12-Factor App](https://12factor.net/)'s runtime discipline, restated for containers and orchestrators), and the working process around all of it.

They are deliberately **not a complete policy manual**. Four areas belong to companion policy; where a rule touches one, it states the engineering-side obligation and leaves the rest to the policy that owns it:

- **The organization's security baseline** — secrets management and rotation, dependency and image scanning, penetration testing, incident response.
- **Platform operations** — deployment pipelines, infrastructure provisioning, the monitoring and paging stack, on-call rotas. OPS-2 and OPS-5 define *what* must be measured and alerted; the stack that delivers the page is platform work.
- **Data governance**, beyond classifying the fields that go on messages (SEC-3).
- **CI infrastructure** — TEST-11 defines what gates a merge; the pipeline that runs it is platform work.

Treat an unlisted topic as out of scope, not approved by silence.

## How to use this on a new project

Don't read front-to-back, and don't adopt every rule on day one. A handful are always-on — the cross-cutting principles and non-negotiables below. The rest **activate at a structural trigger**: apply them when the project crosses that threshold, deliberately, not before.

| The first time you… | Reach for |
|---|---|
| Write production code at all | Test-first (PROC-1) · ubiquitous language (ARCH-1) · vertical slices (ARCH-7) · small iterations (PROC-3) · time/id seams (TEST-8) · simple design, abstraction earned by repetition (ARCH-13–ARCH-14) · pure core, I/O at the edges (ARCH-15) · names say intent, comments say why (PROC-9) · leave it better than you found it (PROC-10) |
| Configure a service for a second environment | Config outside the artifact, secrets outside the repo, fail-fast startup (RUN-1) · build once, promote the same artifact (RUN-3) |
| Deploy to a platform that restarts or reschedules instances | Stateless, disposable processes; graceful shutdown (RUN-2) · readiness from real dependencies (OPS-6) |
| Need to fix production data by hand | Don't — run it from a release (RUN-4) · migration-shaped changes (MSG-9) · replay runbook (OPS-3) |
| Choose persistence for an aggregate | Per-case choice (ARCH-3) · event-sourcing disciplines if chosen (ARCH-4, TEST-10) · optimistic concurrency (CON-1–CON-2) |
| Send an asynchronous message | Outbox (MSG-1) · command or event? (MSG-7) · stable identity (MSG-8) · redelivery-safe consumers (IDEM-1–IDEM-3) · poison vs business failure (IDEM-4–IDEM-5) · per-service broker credentials and topology ACLs (SEC-1) |
| Expose a contract to another service or team | Wire format is the contract (ARCH-5) · internal vs integration events (ARCH-9) · failure codes (MSG-2) · additive evolution (MSG-3) · drift gate (TEST-1) |
| Expose an endpoint to callers you don't control | Principal and permission declared (SEC-1) · front-door validation with bounded sizes (IDEM-10; SEC-2) · request idempotency key (IDEM-3) · stale vs illegal status codes (IDEM-7) · endpoint contract evolution (MSG-3) |
| Call another service synchronously | Timeout from the caller's deadline, bounded retries with jitter, circuit breaker (MSG-10) |
| Add a field to a message | Classify it first — identifiers by default (SEC-3) |
| Build a read model | What does a duplicated event do to it? A lost one? Guard whichever corrupts (ARCH-10) |
| Await a reply that may never come | Saga timeout (IDEM-8) · choreography vs orchestration (ARCH-11) · retry exhaustion becomes a compensating outcome (MSG-4) |
| Publish from a background or scheduled job | Explicit outbox enrollment (IDEM-9) |
| Change a schema or stored shape that has live data | Expand → migrate → contract (MSG-9) · stable stored identity (MSG-8) · drift gate (TEST-1) |
| Go to production with a consumer | Correlation and causation ids on every hop, in every log line (OPS-1) · error-queue alert routed to the owning team (OPS-2) · depth and oldest-age on every durable buffer (OPS-5) · readiness from real dependencies (OPS-6) |
| Ship a change you're not sure about | Progressive rollout with a pre-named rollback trigger (OPS-7) · mixed-version safety (MSG-9) |
| Hand-replay a dead-letter (or run any operator re-emit) | Three verdicts, preconditions checked, one message before the batch (OPS-3) · dedup beyond the inbox window (IDEM-1) · restore the original envelope ids (OPS-1) |
| Add a dedup, idempotency, or outcome table | Retention derived from the hazard window; purge owned and tested (OPS-4) |
| Watch a queue fall behind | Capacity or admission, never durability or discard (OPS-5) · the consumer concurrency knob (CON-6) · admission ceiling (IDEM-10) |
| Enforce "at most one active X" across instances | Set-wide constraints (CON-5) |
| Care about throughput or latency | Decide the metric matters first (PROC-8) · then Performance |
| Stand up CI for the repo | Suite stations, block-on-red, flaky policy (TEST-11) |
| Settle a contested design decision | ADR (PROC-5) · mark what it supersedes (PROC-4) |

The same routing, starting from how the ticket is worded:

| When the ticket sounds like… | Reach for |
|---|---|
| "Add an endpoint that writes a row and notifies another service" | Outbox (MSG-1) · compose from the persisted entity (MSG-1) · command or event? (MSG-7) · stable identity (MSG-8) |
| "We charged / shipped / emailed twice" | Dedup guards (IDEM-1–IDEM-3) · prove it with a replay test (TEST-6–TEST-7) |
| "Add a field to a message another team already consumes" | Additive evolution (MSG-3) · wire format is the contract (ARCH-5) · update the drift gate (TEST-1) · classify it first (SEC-3) |
| "This list screen is slow / awkward to query from the write tables" | Read-model projections (ARCH-3, ARCH-10) |
| "Messages are piling up in the error queue" | Poison vs business failure (IDEM-4–IDEM-5) · per-handler retry policy (MSG-6) · DLQ alert and replay runbook (OPS-2–OPS-3) |
| "Two users saved at once and one edit vanished" | Optimistic concurrency (CON-1, CON-4) · stale-read vs illegal-state status codes (IDEM-7) |
| "A nightly job / background sweep needs to send messages" | Explicit outbox enrollment (IDEM-9) |
| "This order has been stuck 'awaiting payment' since Tuesday" | Saga timeout (IDEM-8) · retry exhaustion becomes a compensating outcome (MSG-4) |
| "Just run an UPDATE in prod to unstick these orders" | Released one-off task, not a hand-typed write (RUN-4) · outbox enrollment (IDEM-9) · replay runbook (OPS-3) |
| "It works on my machine / it broke only in staging" | Same engines everywhere, config outside the artifact (RUN-1) · same bytes everywhere (RUN-3) · real engine in tests (TEST-2) |
| "We already have optimistic concurrency — why would we need idempotency keys?" | The version token is not a dedup guard (CON-1's trap) · choosing the dedup identity (chapter-4 opener; IDEM-3) |

Each rule ends with **when not to apply** — read that as part of the rule, not a disclaimer. Most over-engineering is a sound rule applied before its trigger (the same judgment [YAGNI](https://martinfowler.com/bliki/Yagni.html) asks of code). When a rule and its caveat pull against each other, the caveat wins: it marks where the pattern stops paying for itself.

Where a rule names a concrete technology — a serializer, message framework, database, or broker — the name is an illustration of a mechanism, never a stack recommendation. The rules assume only *capabilities*: a message framework with a transactional outbox, durable queues, and per-handler error policies; a datastore with concurrency tokens and unique constraints; a broker with redelivery and dead-lettering. Any stack that provides them fits. Microsoft's [cloud design pattern catalog](https://learn.microsoft.com/en-us/azure/architecture/patterns/) takes the same stance — its patterns are deliberately technology-agnostic — and many rules cite its pattern pages. Likewise the italicized *Example:* lines: they use deliberately neutral domains (orders, payments, subscriptions) to make a rule's stakes concrete — illustrative, not prescriptive.

## The non-negotiables

The ten chapters boil down to seven essentials — if you remember nothing else, remember these:

1. **A state change and the messages about it commit in one transaction** ([transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html)). — MSG-1
2. **Every consumer tolerates redelivery; an effect that must not happen twice gets a durable dedup guard.** — IDEM-1–IDEM-3
3. **Poison is a bug to investigate; a business "no" is an outcome to record and emit** — never confuse the two routes. — IDEM-4–IDEM-5
4. **One winner per contended write**: optimistic concurrency on the aggregate; a set-wide rule gets the database constraint. — CON-1, CON-5
5. **The wire format is the contract** — pin it with a drift gate wherever serialized bytes outlive the process. — ARCH-5, TEST-1
6. **Test-first, at the layer that actually enforces the behavior.** — PROC-1, TEST-3
7. **A durable buffer nobody watches eventually deletes data** — retention expires whatever nobody drains. Alert on depth and oldest-age, and route the alert to the owning team. — OPS-2, OPS-5

## Contents

1. [Architecture & DDD](01-architecture-and-ddd.md)
2. [Reliable Messaging](02-reliable-messaging.md)
3. [Concurrency](03-concurrency.md)
4. [Idempotency & Failure](04-idempotency-and-failure.md)
5. [Performance](05-performance.md)
6. [Testing](06-testing.md)
7. [Process & Tooling](07-process-and-tooling.md)
8. [Operations](08-operations.md)
9. [Security](09-security.md)
10. [Runtime & Delivery](10-runtime-and-delivery.md)

- [Glossary](GLOSSARY.md)
- [Appendix: pseudocode sketches](APPENDIX-sketches.md)
- [Rule register](rule-register.md)
- [PR checklist](PR-CHECKLIST.md)
- [Cheatsheet](CHEATSHEET.md)


## Your first two weeks

New to messaging, DDD, or distributed systems? The trigger table routes by concepts; this track builds those concepts, in dependency order. If the system you're joining is still a single service with no async messaging, items 1–3 plus the always-on row of the trigger table are all you need on day one — the rest of these guidelines activates as you cross the triggers above.

1. **This README, top to bottom.** Everything else hangs off the trigger table and the non-negotiables — skim now, return often.
2. **[The glossary](GLOSSARY.md).** The chapters use its ~20 terms without stopping to define them, so keep it open beside whatever you read next.
3. **PROC-1 and PROC-3 (test-first, small iterations).** How we work applies to your first ticket no matter what the ticket touches.
4. **MSG-1 (the transactional outbox).** Every other messaging rule silently assumes this guarantee, so nothing downstream makes sense before it.
5. **IDEM-1–IDEM-5.** The first thing async delivery does to a CRUD-trained developer is run their handler twice — these are the guards, and IDEM-4–IDEM-5 is the routing decision every consumer must make.
6. **CON-1 and CON-4, then IDEM-7.** Two writers, one winner: how a contended write is decided, and which status code tells the losing client what to do next.
7. **ARCH-5, ARCH-8–ARCH-9, then TEST-1.** What an event is, which kind may cross a service boundary, and the drift-gate test you will be asked to write for one.

After these, read on demand via the trigger tables. Performance and the rest of Testing can wait until a ticket pulls you there.

## Reading by role

- **Reviewing a PR?** Work the [PR checklist](PR-CHECKLIST.md); keep IDEM-1's guard table and TEST-3 (is the test at the enforcing layer?) at hand.
- **On call, staring at a dead-letter queue?** OPS-2 (who gets paged), OPS-3 (the runbook), OPS-5's diagnosis map (which buffer is growing tells you what broke), then IDEM-4–IDEM-5 and CON-6.
- **Reviewing a design?** The [rule register](rule-register.md)'s design-review rules, led by ARCH-2 (ownership), ARCH-3 (persistence per case), CON-5 (set-wide invariants), and ARCH-11 (choreography vs orchestration).

## Rule IDs and tiers

Every rule heading carries a permanent ID, the imperative title, and a tier tag — `### MSG-1 · Commit the state change and its outgoing messages in one transaction (transactional outbox) [MUST]`. Prefixes per chapter: ARCH, MSG, CON, IDEM, PERF, TEST, PROC, OPS, SEC. **[MUST]** blocks merge; **[SHOULD]** needs a written waiver in the PR; **[CONSIDER]** guides design. Cross-references use the bare ID — search the ID to find the rule and everything that cites it. The full regime — numbering, retirement, authoring conventions, enforcement vehicles — lives in the [rule register](rule-register.md).

## Cross-cutting principles (these govern everything below)

1. **Test-first is mandatory.** [Red-green-refactor](https://martinfowler.com/bliki/TestDrivenDevelopment.html), before production code, always. (PROC-1.)
2. **Prefer small iterations over big-bang change.** Reject large change-orders; decompose into small, independently-shippable steps. A staged, multi-PR plan is a last resort. (PROC-3.)
3. **Prefer framework idioms over bespoke mechanisms** — and verify the idiom from the framework's own source/docs/samples, not from memory. (PROC-6.)
4. **Decide whether a metric matters before chasing it.** Don't optimize what the system isn't for. (PROC-8.)


