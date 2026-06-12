# Rule register — tiers and enforcement

Every rule in these guidelines, its tier, and the vehicle that enforces it. The tier tag in each rule's heading is the tier of record; this register mirrors it and adds the enforcement vehicle. A PR that changes one changes both (README — Governance).

**Tiers.** **MUST** — a violation blocks merge. **SHOULD** — a deviation needs a written justification in the PR, citing the rule ID. **CONSIDER** — a judgment heuristic; cite it in design discussion.

**Vehicles.** **ci-test** — an executable CI test proves the behavior or pins the compliant configuration. **ci-lint** — a static CI check (regex or architecture-fitness rule). **pr-checklist** — a reviewer-verified checklist item (see [PR-CHECKLIST.md](PR-CHECKLIST.md)); for a MUST the reviewer blocks the merge, for a SHOULD the reviewer collects the justification. **design-review** — enforced where designs are discussed, before code exists.

**The regime.**

- A new rule takes the **next free number** in its chapter and is appended at the end of the chapter file. The number is an address, not a reading order.
- A retired rule — should the team ever choose to retire one — uses a supersession banner (README — Governance). IDs are never renumbered and never reused after a retirement.
- Cross-references use the bare ID: "(MSG-1)". Search the ID to find the rule and everything that cites it.
- Maintenance is manual and cheap: the next free number is visible at the chapter's end; adding a rule means one heading, one register row, one changelog row. Optionally, a one-regex CI check asserts ID uniqueness.

**Authoring.** The first sentence of every **Rule.** paragraph states the norm — one sentence that stands alone; detail, mechanism, and sub-cases follow it. Correctness traps go on a "**⚠ Trap:**" line directly under the Rule paragraph, never in "When not to apply" — that section is reserved for genuine scope boundaries. A trap must be **news** — a fact stated neither in the Rule above it nor in the rule it cites.

**Voice.** Plain language, short sentences, no performance. The reader may be mid-incident.

- Say it once, plainly. No aphorisms, no imagery. If a phrase sounds quotable, replace it with the literal statement.
- Short sentences, one point each. At most one aside — a second dash or parenthetical starts a new sentence.
- Keep subject, verb, object together. Qualify after, never in the middle.
- Titles are sentences someone would say in a review, using only concepts the reader already has.
- Cite a rule by stating its claim: "broker retention is finite (OPS-2)" — never "the timer-discard of OPS-2".
- Define non-[glossary](GLOSSARY.md) terms at first use. No slang. No symbol shorthand (`+`, `→`, `vs.`) in prose.
- At most one *Example:* per rule, only where the rule is abstract without it. **Why** is two or three sentences, not an essay.
- Test: read it aloud. If you backtrack, or it sounds like writing, rewrite.

The sources the rules cite are also the voice's models. Seven techniques, learned from their prose:

- **Define a term as a flat equation.** "Timeouts are the maximum amount of time that a client waits for a request to complete" (AWS). "An app's config is everything that is likely to vary between deploys" (12factor). Subject, *is*, definition — no motivation first.
- **Let the tier tag carry the obligation.** 12factor writes "The twelve-factor app stores config in environment variables" — a present-tense fact about the conforming app, no *should*. Write rule bodies the same way: "the service validates configuration at startup"; the [MUST] does the obliging.
- **Offer a litmus test where one exists** — a question the reader can answer in seconds that reveals whether they comply. 12factor: "whether the codebase could be made open source at any moment, without compromising any credentials."
- **Narrate a mechanism in execution order, actor first.** "The service that sends the message [first stores] the message in the database as part of the transaction that updates the business entities. A separate process then sends the messages" (microservices.io). Who acts, then what happens next — never the mechanism described backwards.
- **A vivid name is legal only as a working term: coin it, unpack it immediately, reuse it.** Fowler coins "cost of carry" and accounts in it for the rest of the essay; AWS writes "Retries are 'selfish.' In other words, when a client retries, it spends more of the server's time to get a higher chance of success." A vivid phrase that is never reused is decoration — the no-aphorism rule above applies.
- **Examples carry numbers and follow one scenario.** "The timeout was set very low, to around 20 milliseconds" (AWS) teaches what "very low" cannot. One scenario followed through beats three fragments.
- **Voice the objection in the reader's own words, then answer it.** "A bit of redundant checking can't hurt, right? Unfortunately, it isn't so simple" (lexi-lambda). This is the natural shape of a Trap.

Chapter files: 01-architecture-and-ddd.md, 02-reliable-messaging.md, 03-concurrency.md, 04-idempotency-and-failure.md, 05-performance.md, 06-testing.md, 07-process-and-tooling.md, 08-operations.md, 09-security.md, 10-runtime-and-delivery.md.

Distribution: 32 MUST · 39 SHOULD · 9 CONSIDER — 80 active rules.

| ID | File | § | Rule | Tier | Vehicle | Notes |
|---|---|---|---|---|---|---|
| [ARCH-1](01-architecture-and-ddd.md) | 01 | 1 | Ubiquitous language in code | SHOULD | pr-checklist | |
| [ARCH-2](01-architecture-and-ddd.md) | 01 | 2 | Shared state has exactly one owner | MUST | design-review | |
| [ARCH-3](01-architecture-and-ddd.md) | 01 | 3 | Persistence style per case | CONSIDER | design-review | |
| [ARCH-4](01-architecture-and-ddd.md) | 01 | 4 | Event-sourced stream is the aggregate; log immutable | MUST | design-review | |
| [ARCH-5](01-architecture-and-ddd.md) | 01 | 5 | Wire format is the contract; no shared kernel | MUST | ci-test | Drift gate (TEST-1) is the teeth; the no-shared-kernel half rides design-review. |
| [ARCH-6](01-architecture-and-ddd.md) | 01 | 6 | Make illegal states unrepresentable | SHOULD | design-review | |
| [ARCH-7](01-architecture-and-ddd.md) | 01 | 7 | Vertical slices with failure policy alongside | SHOULD | pr-checklist | The alert-route clause is OPS-2 MUST, referenced here. |
| [ARCH-8](01-architecture-and-ddd.md) | 01 | 8 | Past-tense, intent-capturing event names | SHOULD | pr-checklist | |
| [ARCH-9](01-architecture-and-ddd.md) | 01 | 9 | Internal vs integration events | SHOULD | design-review | |
| [ARCH-10](01-architecture-and-ddd.md) | 01 | 10 | Know what duplicates and lost events do to each consumer; guard accordingly | MUST | ci-test | |
| [ARCH-11](01-architecture-and-ddd.md) | 01 | 11 | Choreography first; orchestrate only when necessary | SHOULD | design-review | |
| [ARCH-12](01-architecture-and-ddd.md) | 01 | 12 | Defer to roadmap; thin seam with deletion criterion | SHOULD | pr-checklist | The PR introducing an interim seam must state its deletion criterion. |
| [ARCH-13](01-architecture-and-ddd.md) | 01 | 13 | Simple design, in priority order | SHOULD | pr-checklist | "Fewest elements" is the tie-breaker, never the goal. |
| [ARCH-14](01-architecture-and-ddd.md) | 01 | 14 | Duplication until proven; inline the wrong abstraction back | SHOULD | pr-checklist | |
| [ARCH-15](01-architecture-and-ddd.md) | 01 | 15 | Functional core, imperative shell | SHOULD | design-review | |
| [MSG-1](02-reliable-messaging.md) | 02 | 1 | Transactional outbox; compose from the persisted entity | MUST | ci-test | |
| [MSG-2](02-reliable-messaging.md) | 02 | 2 | Coded cross-boundary failure reasons | MUST | ci-test | |
| [MSG-3](02-reliable-messaging.md) | 02 | 3 | Additive contract evolution; version only on break | MUST | ci-test | Covers the HTTP door's request/response shapes too. |
| [MSG-4](02-reliable-messaging.md) | 02 | 4 | Retries exhausted: emit the coded failure | SHOULD | ci-test | Tier debated: "callers never strand" is MUST-grade, but the durable guarantee lives in IDEM-8; this best-effort half is the SHOULD. |
| [MSG-5](02-reliable-messaging.md) | 02 | 5 | Retry count on entity; ceiling in policy | CONSIDER | design-review | |
| [MSG-6](02-reliable-messaging.md) | 02 | 6 | Per-handler retry/error policy, backoff + jitter | SHOULD | pr-checklist | |
| [MSG-7](02-reliable-messaging.md) | 02 | 7 | Command vs event — don't blur | SHOULD | design-review | |
| [MSG-8](02-reliable-messaging.md) | 02 | 8 | Stable message identity once envelopes persist | MUST | ci-lint | |
| [MSG-9](02-reliable-messaging.md) | 02 | 9 | Expand → migrate → contract for every durable shape | MUST | pr-checklist | |
| [MSG-10](02-reliable-messaging.md) | 02 | 10 | Sync calls: timeout from caller's deadline, bounded retries, circuit breaker | MUST | pr-checklist | Tier debated: the timeout's presence is checkable (hence MUST), but deriving it from the caller's deadline is judgment. |
| [CON-1](03-concurrency.md) | 03 | 1 | Optimistic concurrency on aggregates | MUST | ci-test | |
| [CON-2](03-concurrency.md) | 03 | 2 | Central concurrency-token bump | SHOULD | ci-test | Native store tokens are the licensed deviation. |
| [CON-3](03-concurrency.md) | 03 | 3 | Reject doomed work at the earliest authoritative step | CONSIDER | design-review | The rule calls itself "folklore, not doctrine". |
| [CON-4](03-concurrency.md) | 03 | 4 | Client precondition; store token is the real guard | MUST | ci-test | The MUST is never relying on the client half for correctness. |
| [CON-5](03-concurrency.md) | 03 | 5 | Aggregate invariant: version token; set-wide rule: database constraint | MUST | ci-test | |
| [CON-6](03-concurrency.md) | 03 | 6 | Find the consumer concurrency knob | CONSIDER | design-review | |
| [CON-7](03-concurrency.md) | 03 | 7 | Hot-key bulkhead, no global cap | CONSIDER | design-review | |
| [IDEM-1](04-idempotency-and-failure.md) | 04 | 1 | Every consumer survives the same message twice; guard per the hazard table | MUST | ci-test | |
| [IDEM-2](04-idempotency-and-failure.md) | 04 | 2 | Dedup guard as INSERT, fail loud | MUST | ci-test | |
| [IDEM-3](04-idempotency-and-failure.md) | 04 | 3 | Dedup key: message id or client token, not content | SHOULD | design-review | |
| [IDEM-4](04-idempotency-and-failure.md) | 04 | 4 | Poison vs business failure, routed differently | MUST | ci-test | |
| [IDEM-5](04-idempotency-and-failure.md) | 04 | 5 | DLQ is not the system of record | SHOULD | design-review | |
| [IDEM-6](04-idempotency-and-failure.md) | 04 | 6 | Re-read and re-decide on OCC retry | MUST | ci-test | |
| [IDEM-7](04-idempotency-and-failure.md) | 04 | 7 | 412 (stale read) vs 409 (illegal state) | SHOULD | ci-test | |
| [IDEM-8](04-idempotency-and-failure.md) | 04 | 8 | Backstop awaited replies with a saga timeout; a reaper for stranded state no saga watches | MUST | ci-test | |
| [IDEM-9](04-idempotency-and-failure.md) | 04 | 9 | Explicit outbox enrollment in background work | MUST | ci-test | |
| [IDEM-10](04-idempotency-and-failure.md) | 04 | 10 | Validate at the front door; bounds, admission ceiling, pagination | MUST | ci-test | |
| [PERF-1](05-performance.md) | 05 | 1 | Profile first; disprove the theory cheaply; know the time model | CONSIDER | design-review | |
| [PERF-2](05-performance.md) | 05 | 2 | Saturated shared resource sets the ceiling | CONSIDER | design-review | The inline-listener pairing is this rule's ⚠ Trap. |
| [PERF-3](05-performance.md) | 05 | 3 | Spread test data the way production does | CONSIDER | design-review | |
| [TEST-1](06-testing.md) | 06 | 1 | Wire-shape drift gate (golden samples) | MUST | ci-test | |
| [TEST-2](06-testing.md) | 06 | 2 | Real throwaway database for integration tests | MUST | pr-checklist | |
| [TEST-3](06-testing.md) | 06 | 3 | Test at the layer that enforces the behavior | SHOULD | pr-checklist | |
| [TEST-4](06-testing.md) | 06 | 4 | In-process full-pipeline endpoint tests | SHOULD | pr-checklist | |
| [TEST-5](06-testing.md) | 06 | 5 | Assert silently-failing configuration | SHOULD | pr-checklist | |
| [TEST-6](06-testing.md) | 06 | 6 | Replay twice, assert exactly once | MUST | ci-test | |
| [TEST-7](06-testing.md) | 06 | 7 | Concurrent and redelivered copies, one outcome, empty error queue | SHOULD | ci-test | Tier debated: MUST-grade for unrepeatable effects, but it needs framework tracking hooks, so SHOULD. |
| [TEST-8](06-testing.md) | 06 | 8 | Inject time and identity seams | SHOULD | ci-lint | |
| [TEST-9](06-testing.md) | 06 | 9 | Shared host per collection; fresh ids | SHOULD | pr-checklist | |
| [TEST-10](06-testing.md) | 06 | 10 | Given-events / when-command / then-events | SHOULD | pr-checklist | Active only where ARCH-4 is in play. |
| [TEST-11](06-testing.md) | 06 | 11 | The suite is a gate or it is decoration | MUST | pr-checklist | |
| [PROC-1](07-process-and-tooling.md) | 07 | 1 | Test-first is mandatory | MUST | pr-checklist | No per-task opt-out. Spikes and pure wiring are outside its scope, not exceptions inside it. |
| [PROC-2](07-process-and-tooling.md) | 07 | 2 | The behavior-pinning test is committed before the change | MUST | pr-checklist | |
| [PROC-3](07-process-and-tooling.md) | 07 | 3 | Small iterations; reject large change-orders | SHOULD | design-review | |
| [PROC-4](07-process-and-tooling.md) | 07 | 4 | Supersession banners, never silent deletion | SHOULD | pr-checklist | |
| [PROC-5](07-process-and-tooling.md) | 07 | 5 | ADRs preserve the rejected alternative | SHOULD | design-review | |
| [PROC-6](07-process-and-tooling.md) | 07 | 6 | Verify framework behavior from source; prefer idioms | SHOULD | design-review | |
| [PROC-7](07-process-and-tooling.md) | 07 | 7 | Gate expensive automation behind explicit intent | CONSIDER | design-review | |
| [PROC-8](07-process-and-tooling.md) | 07 | 8 | Decide the metric matters; isolate the rig | SHOULD | design-review | |
| [PROC-9](07-process-and-tooling.md) | 07 | 9 | Names say intent, comments say why | SHOULD | pr-checklist | |
| [PROC-10](07-process-and-tooling.md) | 07 | 10 | Refactor in the flow of work; leave it better than found | SHOULD | pr-checklist | Forbids the deferred "refactoring sprint"; cleanup scoped to the campsite. |
| [OPS-1](08-operations.md) | 08 | 1 | Correlation + causation ids + tracing across every async hop | SHOULD | pr-checklist | |
| [OPS-2](08-operations.md) | 08 | 2 | Error-queue depth alerts, routed to the owning team | MUST | pr-checklist | An unalerted DLQ is a slow silent discard — lost messages. |
| [OPS-3](08-operations.md) | 08 | 3 | Replay by runbook: three verdicts, preconditions, one before the batch | SHOULD | design-review | |
| [OPS-4](08-operations.md) | 08 | 4 | Declared retention per bookkeeping row; owned, tested purge | SHOULD | pr-checklist | |
| [OPS-5](08-operations.md) | 08 | 5 | Depth + oldest-age on every durable buffer, alerted | MUST | pr-checklist | A dead relay silently strands committed facts. |
| [OPS-6](08-operations.md) | 08 | 6 | Readiness from real dependencies; topology before consuming; broker excluded at the door | SHOULD | pr-checklist | |
| [OPS-7](08-operations.md) | 08 | 7 | Decouple deploy from release; progressive rollout, pre-named rollback trigger | SHOULD | design-review | |
| [SEC-1](09-security.md) | 09 | 1 | Principal + permission declared at every entry boundary | MUST | pr-checklist | |
| [SEC-2](09-security.md) | 09 | 2 | Validate at the door, classify at the consumer | MUST | pr-checklist | |
| [SEC-3](09-security.md) | 09 | 3 | Classify message fields; identifiers by default | SHOULD | pr-checklist | |
| [RUN-1](10-runtime-and-delivery.md) | 10 | 1 | Config outside the artifact, secrets outside the repo; fail startup loudly | MUST | pr-checklist | |
| [RUN-2](10-runtime-and-delivery.md) | 10 | 2 | Stateless, disposable processes; graceful shutdown, crash-safe by construction | MUST | design-review | Declared stateful runtimes (actors, caches) need a recovery story. |
| [RUN-3](10-runtime-and-delivery.md) | 10 | 3 | Build once; promote the same artifact; reproducible from the repo | SHOULD | design-review | Pipeline mechanics are platform work; the obligation is that nothing requires a per-environment rebuild. |
| [RUN-4](10-runtime-and-delivery.md) | 10 | 4 | One-off admin work runs from a release, never a hand-typed mutation | SHOULD | design-review | Break-glass writes follow OPS-3's discipline and file the released fix. | |