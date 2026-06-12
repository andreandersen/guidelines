# Runtime & Delivery

> **Read this first.** This chapter is the system's contract with the platform that runs it: what the *process* promises (statelessness, disposability), what the *artifact* promises (immutability, environment-blindness), and how anything else reaches production (as a release, never a hand edit). It restates the [12-Factor App](https://12factor.net/)'s runtime discipline in the vocabulary of these guidelines, updated for containers and orchestrators. Most of the twelve factors already live in earlier chapters; the table maps all twelve, and the four rules below carry the ones nothing else owned.

| Factor | Where it lives here |
|---|---|
| I. Codebase · II. Dependencies · V. Build, release, run | RUN-3 |
| III. Config · IV. Backing services · X. Dev/prod parity | RUN-1 (parity in tests: TEST-2) |
| VI. Processes · VIII. Concurrency · IX. Disposability | RUN-2 (the startup half: OPS-6; the scale-out levers: CON-6, OPS-5) |
| VII. Port binding | The platform's job today — containers made self-contained port-binding services the ambient default; no rule needed |
| XI. Logs | OPS-1 (structured, correlation-carrying lines to stdout; the platform aggregates) |
| XII. Admin processes | RUN-4 (migrations: MSG-9; replays: OPS-3) |

### RUN-1 · Config lives outside the artifact, secrets outside the repo — and a missing setting fails startup loudly [MUST]

**Rule.** Everything that varies between environments — connection strings, broker addresses, credentials, per-environment tuning — lives outside the build artifact, supplied by the environment at run time; the artifact itself is environment-blind (RUN-3). Backing services are **attached resources**: the service reaches its database, broker, or downstream API only through a configured address and credential, so swapping an instance — a local container, staging, a failover replica — is a config change, never a code change or a rebuild. Every environment, dev included, runs the **same backing-service engines** — containers make that parity cheap, and TEST-2 is this principle applied to tests. Secrets never enter the repo — not in settings files, not in compose files, not in test fixtures; the code reads them from the platform's secret mechanism (which mechanism is companion policy — README, Scope), and a leaked repo must leak no credential. The service **validates its configuration at startup and fails loudly**: missing a required setting, it crashes at boot and names the setting rather than entering traffic half-configured — the runtime twin of asserting silently-failing configuration in tests (TEST-5). Two litmus tests: could the repo be made public right now without compromising a credential? Could the artifact be pointed at a different environment without a rebuild? A "no" to the first means a secret is in the repo; a "no" to the second means config is in the artifact.

**⚠ Trap:** "Config" is not a license for a sprawling settings surface. A value that never varies between environments is code — a constant with a name — not config; every externalized knob is one more way production can differ from what was tested.

**Why.** An artifact with baked-in config can't be promoted unchanged (RUN-3), can't be shared without leaking, and hides its environment assumptions until the wrong one boots. Fail-fast at startup turns a misconfiguration into a crash at deploy time, when the deployer is watching and rollback is one command — instead of a runtime mystery an hour later. *Example: a renamed connection-string key ships; the service boots, serves health checks, and dead-letters every message for an hour — versus refusing to start with `Missing required setting: Messaging:BrokerUrl`.*

**When not to apply.** Defaults are fine for genuinely optional knobs with safe fallbacks — required-with-no-default is for settings the service cannot run correctly without. And parity has a price line: a managed cloud service with no faithful local equivalent gets the closest container stand-in plus a scheduled real-thing test (TEST-11's scheduled station), not a dogmatic local replica.

**Sources.** [12factor.net — Config](https://12factor.net/config), [Backing services](https://12factor.net/backing-services), [Dev/prod parity](https://12factor.net/dev-prod-parity); Hoffman, *Beyond the Twelve-Factor App*; [Microsoft — External Configuration Store pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/external-configuration-store).

### RUN-2 · Processes are stateless and disposable — keep state in backing services, shut down gracefully, survive sudden death [MUST]

**Rule.** A service process owns no state worth keeping: anything that must survive — domain rows, sessions, queued work, files — lives in a backing service, and process memory or local disk is at most a single-operation scratch pad or a cache that can vanish without correctness loss. Statelessness is what makes every scale and recovery lever in these guidelines real: scale-out (CON-6; OPS-5's capacity lever), restart-on-crash, and progressive rollout (OPS-7) all assume any instance can serve any request and any instance can die. Disposability is the same property in time. **Start fast**, declaring readiness from real dependencies (OPS-6). **Stop gracefully**: on the platform's stop signal, stop accepting new work, finish or abandon in-flight handlers within the grace period, and let unacked messages return to the queue. And **survive sudden death** — which these guidelines already guarantee by construction: a handler that dies mid-transaction rolls back, the message redelivers, and the dedup guard (IDEM-1) absorbs the repeat. Graceful shutdown reduces redelivery noise; the dedup guards are what make death safe.

**⚠ Trap:** Sticky sessions are process state by another name — an instance the load balancer must remember is an instance that can't be replaced. The same goes for a "warm" in-memory cache a request *depends on* for correctness rather than merely benefits from.

**Why.** The moment one instance holds something the others don't, every operation the platform performs routinely — rescheduling, scaling, rolling a deploy — becomes data loss. *Example: an upload endpoint buffers chunks on local disk across requests; the orchestrator reschedules the pod mid-upload and the file is gone — buffered to a blob store keyed by upload id, the next chunk lands on any instance.*

**When not to apply.** Deliberate stateful runtimes exist — an actor system holding per-key state in memory (CON-7), an in-process projection cache — but each is a *declared* exception with a recovery story (rehydration from the store; wholesale rebuild, ARCH-10), never an accident of convenience. A cache that only saves latency, and is correct when cold, is fine anywhere.

**Sources.** [12factor.net — Processes](https://12factor.net/processes), [Concurrency](https://12factor.net/concurrency), [Disposability](https://12factor.net/disposability); Nygard, *Release It!*.

### RUN-3 · Build once; promote the same artifact through every environment — config varies, bytes don't [SHOULD]

**Rule.** One commit produces **one immutable artifact** (a container image, a published bundle), built once in CI, uniquely identified, and promoted unchanged through every environment — staging runs the same bytes production will. Build, release, run stay strictly separated: a **release** is the artifact plus that environment's config (RUN-1), and a rollback is redeploying a previous release (OPS-7's revert lever), never an emergency rebuild. The build itself is **reproducible from the repo alone**: every dependency explicitly declared and version-pinned (lockfiles, pinned base images), nothing supplied by whatever happened to be on the build host. The named anti-pattern is the per-environment rebuild — environment-conditional compilation, "we rebuild with the prod flag" — which makes the artifact you tested not the artifact you run.

**⚠ Trap:** A floating tag (`latest`, an unpinned major version) makes two builds of the same commit differ silently. Pin, and let dependency *updates* be commits — visible, testable, revertible — not build-time surprises.

**Why.** Every environment-specific rebuild reopens the gap that testing closed: the binary that passed staging is not the binary in production, so a green pipeline proves nothing about what ships. Immutable, identified artifacts also make "what exactly is running?" answerable in one lookup — the precondition for MSG-9's "compatible with the currently-deployed binary" to even be checkable. *Example: an incident at 02:00; rollback is `deploy release-417`, one minute — versus rebuilding an old commit against today's floating base image and shipping a binary nobody has ever run.*

**When not to apply.** The pipeline mechanics — registries, promotion tooling, signing — are platform work (README, Scope); the engineering-side obligation is that nothing in the code or build *requires* a per-environment rebuild. Things that aren't binaries — a migration script, a dashboard definition — ride alongside the artifact, versioned the same way, rather than having this rule forced onto them.

**Sources.** [12factor.net — Codebase](https://12factor.net/codebase), [Dependencies](https://12factor.net/dependencies), [Build, release, run](https://12factor.net/build-release-run); Humble & Farley, *Continuous Delivery* ("only build your binaries once").

### RUN-4 · One-off admin work runs from a release in the environment, never as a hand-typed mutation [SHOULD]

**Rule.** A data correction, a backfill, a re-emit, an ad-hoc report — one-off admin work — runs as **released code in the target environment**: the same artifact (or one built and reviewed the same way), the same config (RUN-1), the same outbox enrollment (IDEM-9), the same time and id seams (TEST-8). The shapes these guidelines already provide are the menu: a migration (MSG-9) for shape-or-data changes, a replay through the runbook (OPS-3) for parked messages, a reaper-style task (IDEM-8) for stranded state — each reviewed, tested where testable, and attributable (ARCH-10's operator-attribution clause: the trail names who acted, on what, and why). An interactive prompt in production is for *reading*; the moment it writes, the change has bypassed every guarantee the chapters build — no test, no review, no outbox, no audit row, and no idempotency when the first attempt half-fails and someone runs it again.

**⚠ Trap:** A "one-off" performed twice is a feature with no tests and no name — promote it to a real, released task before the third.

**Why.** Hand-typed production mutations are the highest-risk writes a system ever takes — unreviewed, unrepeatable, performed under pressure by one person at a keyboard — applied to exactly the data that mattered enough to fix by hand. *Example: a hand-run UPDATE unsticks 400 stuck orders but skips the outbox; downstream services never hear, the read models drift, and next week's reconciliation finds 400 ghosts — the released backfill task would have emitted the events.*

**When not to apply.** Break-glass moments exist — an incident may genuinely require a manual write before a task can ship. The discipline is then OPS-3's: pair on it, record exactly what ran and why as a first-class outcome (IDEM-5), and file the released version of the fix as the postmortem action. Reading in production needs no ceremony.

**Sources.** [12factor.net — Admin processes](https://12factor.net/admin-processes).
