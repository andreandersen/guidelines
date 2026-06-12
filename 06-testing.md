# Testing

### TEST-1 · Pin contracts with the framework's versioning and handler tests; a golden sample catches silent drift [MUST]

**Rule.** For contract evolution, the primary mechanism is the framework's idiom: stable message identity, version forwarders, and **handler tests** asserting that old and new versions both deserialize and transform. (The idiomatic path is identity plus registered transforms — **not** snapshotting wire JSON.) For the wire *shape* itself, keep a **golden-sample JSON test as a drift gate wherever serialized bytes outlive the process or cross an ownership line** — persisted to an outbox, a durable queue, a dead-letter store, scheduled delivery, or read by an independently-deployed consumer (per-context contract copies can silently diverge; so can serializer options). It is genuinely optional only for ephemeral, in-memory-only messages, or when both sides are generated from one schema. Serialize with the production serializer's exact options and never auto-overwrite on mismatch.

**⚠ Trap:** "Internal" is not the exemption. An internal event on a durable queue still qualifies for the drift gate — the trigger is bytes that outlive the process or cross an ownership line, not the service boundary.

**Why.** Forwarders handle *intentional* evolution; a golden sample catches *accidental* drift — between independent copies, or from a serializer-option regression (an enum silently flipping from names to integers) — different problems, both load-bearing once bytes are durable or independently read. *Example: a serializer-options change silently flips an enum from names to integers; both sides' unit tests stay green while every consumer reads garbage.*

**When not to apply.** Over-applying golden samples to ephemeral, never-persisted internal DTOs is needless brittleness. Across separate repos, a published contract package or consumer-driven contracts replace the shared file.

*Sketch: Appendix A3.*

**Sources.** [Fowler, ContractTest](https://martinfowler.com/bliki/ContractTest.html); [Robinson, "Consumer-Driven Contracts"](https://martinfowler.com/articles/consumerDrivenContracts.html).

### TEST-2 · Integration-test against a real throwaway database, not an in-memory fake [MUST]

**Rule.** Boot the real application host against an ephemeral, containerized instance of the **real** database (spun up and torn down per run), not an in-memory or mocked store. Only this exercises real schema, real SQL, real concurrency tokens, and real constraint enforcement.

**Why.** The headline behaviors under test — concurrency-token rejections, unique-constraint inserts, monotonic projection guards — are properties of the real engine, invisible against an in-memory provider.

**When not to apply.** Do this **when the container infra cost is acceptable** and doesn't block the suite; pure-domain logic that touches no persistence shouldn't pay it. Reuse one container per test collection to amortize startup.

**Sources.** [Vocke, "The Practical Test Pyramid"](https://martinfowler.com/articles/practical-test-pyramid.html). Pairs with dev/prod parity ([12-Factor X](https://12factor.net/dev-prod-parity)): the same engine in tests as in production.

### TEST-3 · Match each test's layer to where the behavior is actually enforced [SHOULD]

**Rule.** Decide each test's layer by where the behavior lives: pure-domain (plain objects, no I/O) for invariants and state machines; infrastructure-backed for engine-enforced concurrency tokens and constraints; full-pipeline in-process for HTTP / status mapping; serializer-level for wire shape. Don't push a behavior to a layer that can't exhibit it, and keep cheap pure tests as the broad base.

**Why.** A pure-domain test can't exercise a concurrency token or a unique-index violation; an HTTP test can serialize and miss the token entirely. Encoding invariants in cheap pure tests keeps the slow infra-backed tests focused on wiring and persistence.

**When not to apply.** Splitting one behavior across two layers adds tests; justified only when both layers genuinely enforce something, wasteful when one is a pass-through.

**Sources.** [Fowler, TestPyramid](https://martinfowler.com/bliki/TestPyramid.html); [Vocke, "The Practical Test Pyramid"](https://martinfowler.com/articles/practical-test-pyramid.html).

### TEST-4 · Test endpoints in-process through the full real pipeline, asserting client-observable output [SHOULD]

**Rule.** Exercise endpoints by sending real requests through the booted app's full middleware / routing / serialization pipeline in-process (no external server, no mocked controllers), asserting on status codes and bodies exactly as a client would observe — including the deliberate error-to-status mapping and idempotency-header handling.

**Why.** Status-code mapping, model binding, validation, and header handling are real behaviors that unit-testing a handler in isolation skips.

**When not to apply.** Still relies on a booted host (and real database), so it's an integration test, not a unit test; it won't catch issues that appear only across a real network boundary or reverse proxy.

**Sources.** [Fowler, SubcutaneousTest](https://martinfowler.com/bliki/SubcutaneousTest.html).

### TEST-5 · Assert configuration that would fail silently at runtime [SHOULD]

**Rule.** When a misconfiguration fails *silently* rather than throwing — a dropped route, a queue that should be durable but isn't, a property that must *not* be marked as a concurrency token — assert the configuration directly and deterministically instead of trying to observe the runtime symptom.

**Why.** A missing publish registration drops messages with no error; a non-durable projection queue loses state on a crash; an accidentally-versioned read-model row freezes the view — each invisible until production.

**When not to apply.** Config-shape assertions are framework-coupled and can ossify internals; reserve them for genuinely silent failure modes.

### TEST-6 · Prove idempotency by replaying the identical operation and asserting exactly once [MUST]

**Rule.** Perform the identical operation twice and assert the side effect happened a single time (one row, one resource id replayed, one processed record). Make "exactly once under replay" an executable guarantee, not a hopeful comment.

**Why.** Idempotency guards exist precisely so a retry never double-acts; a doubled effect produces an incorrect, non-idempotent result.

**When not to apply.** Sequential double-invoke proves the dedup-record path but not behavior under truly concurrent duplicates — that needs the parallel test below.

**Sources.** [AWS Builders' Library, "Making retries safe with idempotent APIs"](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/).

### TEST-7 · Fire concurrent and redelivered copies at the consumer; expect one correct outcome and an empty error queue [SHOULD]

**Rule.** Fire concurrent *and* redelivered copies of the same message; assert they converge to one correct state and leave nothing on the error queue, using the framework's activity tracking. Relax naive "no exceptions" assertions deliberately where a constraint loser legitimately throws before its retry resolves.

**Why.** Find-then-insert and double-effect races resolve via the writer that loses the unique-constraint race being retried into an idempotent update; without the test, a regression that dead-letters that retry or duplicates the effect ships green.

**When not to apply.** Needs a framework that surfaces an error-queue / tracking hook and lets you await settling (a tracked-session test harness); with a raw broker you'd assert on the DLQ and final state.

### TEST-8 · Inject seams for time and identity so tests are deterministic [SHOULD]

**Rule.** Inject the clock and id / random generators rather than calling them statically, so tests can pin timestamps and assert that generated identifiers came from the controlled seam. Production keeps real implementations; tests substitute recording or fixed ones.

**Why.** Time-based expiry and stranded-state sweeps need to age data deterministically, and load-bearing generated ids must be pinnable.

**When not to apply.** Only add a seam a test actually exercises — don't inject a generator nothing pins.

**Sources.** Feathers, *Working Effectively with Legacy Code* (seams).

### TEST-9 · Share one booted host per test collection; isolate tests by fresh ids [SHOULD]

**Rule.** Boot the expensive host / container once per collection and share it for speed, but keep tests independent by generating fresh unique ids per test and seeding per-test data, rather than relying on a clean database between tests.

**⚠ Trap:** A shared-host test must never assert on global counts or mutate shared rows. It passes alone and flakes in the suite — and the flake implicates whichever test runs alongside it, not the offender.

**Why.** Re-creating the container / host per test makes the suite far too slow; sharing it requires that no test depend on global state, which fresh-id seeding guarantees.

**When not to apply.** A test that genuinely needs an empty store needs its own host, or a dev-only reset endpoint.

### TEST-10 · Test event-sourced behavior as given-events → when-command → then-events [SHOULD]

**Rule.** For an event-sourced aggregate (ARCH-4), the natural unit test is: arrange a history of events, fold them to state, dispatch a command, and assert on the **newly emitted events** — their types and payloads — not on internal state. Pure domain, no store.

**⚠ Trap:** An upcaster test must feed the old *serialized* form, pinned as a golden sample (TEST-1). Tested only against freshly-serialized old types, it never sees what is actually stored — green in CI, broken on real replay.

**Why.** Events are simultaneously the aggregate's observable output and its persistence; asserting them tests exactly the contract that replay, projections, and consumers all rely on — cheap, deterministic, and store-free.

**When not to apply.** Projection correctness still needs its own tests: a projector test feeds events and asserts the read model.

**Sources.** Young, *Versioning in an Event Sourced System*.

### TEST-11 · A suite either blocks the merge or it is decoration — declare which, and stop the line on red [MUST]

**Rule.** Every suite declares its station. **Merge-blocking:** pure-domain tests, the infra-backed slice and consumer tests against the throwaway container (TEST-2), contract drift gates (TEST-1), and configuration assertions (TEST-5) — everything that pins the correctness guarantees these chapters define runs on every merge and blocks it on red. **Scheduled:** suites whose cost genuinely cannot be paid per merge — long load and chaos runs, real-broker topology checks — run nightly or pre-release, each with a **named owner who reads every result**; an unowned nightly suite is decoration. A red merge gate stops the line: the breaking change is fixed or reverted before unrelated work merges over it, and whoever merged it owns the red until green.

A **flaky test is a defect** — in the test or in the system, and often the system, in exactly the races these guidelines exist to guard. Quarantine it within a day, visibly (a named quarantine list with an owner, not a deleted assertion); within the iteration, fix the root cause or delete the test deliberately, recording the coverage lost. Never bare-retry-until-green — above all on the concurrency and idempotency suites (TEST-6–TEST-7), where intermittent failure is the very signal the test exists to raise.

**Why.** The guarantees in these chapters — one winner per contended write, exactly-once effects, no lost messages — are only as strong as the suite that enforces them on every merge. A gate that can be ignored trains the team to ignore it; after the first tolerated red, every later red is noise. And a retried concurrency test converts a real, shipping race into a green checkmark. *Example: a claim-contention test fails one run in twenty; wrapped in a 3× retry it stays green for a quarter — while production loses exactly that race under load.*

**When not to apply.** Moving a genuinely *slow* suite to the scheduled tier is healthy; moving a *flaky* one there to escape the gate is the dodge this rule exists to block. Awaiting eventual convergence inside a test (a tracked session settling — TEST-7) is an assertion with a deadline, not a retry of the test. And a deliberately red work-in-progress branch is fine — the gate governs merge, not exploration.

**Sources.** [Fowler, "Eradicating Non-Determinism in Tests"](https://martinfowler.com/articles/nonDeterminism.html); [Google Testing Blog, "Flaky Tests at Google and How We Mitigate Them"](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html); [Fowler, SelfTestingCode](https://martinfowler.com/bliki/SelfTestingCode.html).