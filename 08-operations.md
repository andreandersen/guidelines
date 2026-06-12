# Operations

> **Read this first.** Reliable messaging does not eliminate failure — it moves failure into durable buffers (an outbox, queues, a dead-letter store) where it waits for an operator. Nearly every rule in the preceding chapters creates such a buffer. This chapter is the other half of that bargain: the ids that let a human reconstruct what happened, the alerts that summon the human, the runbook the human follows, and the retention that keeps the bookkeeping from rotting. A guarantee nobody can observe is indistinguishable from one that quietly broke.

### OPS-1 · Mint correlation and causation ids at the front door; carry them across every async hop [SHOULD]

**Rule.** Every workflow carries a **correlation id** (the business transaction that started the chain) and every message a **causation id** (the message or request that directly caused it).

- **Mint** the correlation id at the first edge the workflow enters: the inbound request, the consumed integration event, the scheduled job.
- **Carry** both across every durable hop: the outbox row stores them, the broker headers carry them, and a consumer copies them onto everything it emits.
- **Two spots silently drop context**, for the same reason — propagation works by copying ids off the message currently being handled, and neither of these spots has one: (a) a *scheduled saga timeout* fires in a fresh context long after the original handling — persist the correlation id in the saga's durable state and restore it when the timeout fires; (b) a *reaper or background-loop re-emit* (IDEM-8–IDEM-9) derives work from rows, not from a message — persist the correlation id on the entity at first write and restore it onto the re-emit. If no original context survives, mint fresh and log the adoption — never emit with an empty one.
- **Log** both ids as structured fields on every line written during handling, not interpolated prose, so one query over aggregated logs replays the whole workflow in order. The log itself is an **event stream the platform owns**: write structured lines to stdout and leave collection, routing, and retention to the environment — a service that opens, rotates, or ships its own log files has taken on a platform job, and loses the files with the instance's disposable disk anyway (RUN-2).
- **Trace:** emit spans on the same lineage — the consumer's handling span links to the producer's emitting span across the broker hop, so an async workflow reads as *one* trace, with queue wait visible as the gap between spans. Verify the trace context survives the durable hops (outbox storage, scheduled delivery, background scopes) — exactly where propagation defaults drop it.

**⚠ Trap:** A hand-replayed dead-letter re-enters with whatever headers the operator happened to preserve. Replay through a path that restores the original envelope (OPS-3), or the workflow becomes untraceable at exactly its most suspicious moment.

**Why.** A workflow spanning services and async hops cannot be reconstructed from any one service's logs. The correlation id turns "grep five services and guess at the interleaving" into one query. The causation id distinguishes a redelivery of the same cause from a genuinely new cause — the exact distinction the dedup guard (IDEM-1) and the replay runbook (OPS-3) turn on. *Example: a shipment fails three hops after the order was placed; the correlation id pulls the whole chain — order, reservation, dispatch, failure — in one query, including the timeout that fired and the reaper pass that resolved it.*

**When not to apply.** Inside one process on one call stack, the stack is the correlation — instrument the boundaries, not every method. Prefer the platform's built-in propagation (e.g. W3C `traceparent`) over hand-rolled header plumbing (PROC-6). Ids in headers and logs remain the floor that works everywhere: tracing backends sample and expire — the ids must not.

**Sources.** [Hohpe & Woolf — Correlation Identifier](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html); [W3C Trace Context](https://www.w3.org/TR/trace-context/); [OpenTelemetry — Context propagation](https://opentelemetry.io/docs/concepts/context-propagation/). The logs-as-a-stream line is [12-Factor XI](https://12factor.net/logs).

### OPS-2 · An unmonitored error queue is a slow, silent discard — alert when it grows, and page the team that owns the handler [MUST]

**Rule.** Dead-lettering is only "visible to a human" (IDEM-4) if a human is summoned. Alert when the error queue is non-empty or its depth grows, and route that alert to the **team that owns the failing handler** — not a central operations queue where it is one anonymous graph among hundreds. Extend the slice idiom (ARCH-7): the slice file that declares a dead-letter policy also names **where its dead letters alert** — the owning team's channel and the severity. A consumer reaching production with a dead-letter route but no alert route is incomplete, the same way a slice without an error-to-status mapping is incomplete.

**⚠ Trap:** Broker dead-letter retention is finite. An unalerted dead letter is not "parked indefinitely" — it is on a timer to deletion, which makes it a discard, just a slower one.

**Why.** The entire distinction between *move-to-error-queue* and *discard* (IDEM-4) is that somebody investigates. Ownership routing is what makes the page actionable: the owning team can read the message, knows the invariant it violated, and ships the fix; a central queue produces tickets that age. *Example: a malformed refund instruction dead-letters at 2 a.m.; the payments team — not a network operations center — is paged, recognizes yesterday's producer deploy as the cause, and fixes it before the morning batch sends a thousand more.*

**When not to apply.** Scope paging to production-like environments; depth alerts in dev are noise that trains people to ignore the real one. A backlog already under active investigation may be snoozed with an expiry — never unrouted.

**Sources.** [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/); [Hohpe & Woolf — Dead Letter Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DeadLetterChannel.html).

### OPS-3 · Replay dead letters by runbook: one of three verdicts, preconditions checked, one message before the batch [SHOULD]

**Rule.** Write the dead-letter runbook before the first production dead letter. Its triage gives every message exactly one of three verdicts:

- **(a) Replay as-is** — the cause was transient or environmental and is gone.
- **(b) Fix-then-replay** — a consumer or producer bug; deploy the fix first, replay after.
- **(c) Discard with a recorded outcome** — the message can never be validly processed; record the decision as a first-class outcome (IDEM-5), a row or compensating event with who decided and why, and only then delete.

There is no fourth verdict: "leave it for later" only waits for broker retention to delete the message (OPS-2) — a discard nobody decided. Before any replay, walk the preconditions:

- [ ] the dedup guard covers a replay arriving *outside* the framework's inbox window (IDEM-1);
- [ ] the fix, if any, is deployed everywhere the message touches;
- [ ] the stored envelope still deserializes under the current contract (MSG-9);
- [ ] any partial side effects of the original failed handling are accounted for;
- [ ] the replay path restores the original message id and correlation headers (OPS-1), rather than minting a fresh envelope.

Then **replay one message and verify the outcome end to end before batching the rest**.

**Why.** Replays happen rarely, under pressure, often performed by someone who didn't write the consumer — the exact conditions under which an unwritten procedure produces the second, larger incident. A wrong triage applied to one message is an incident; applied to ten thousand it is a catastrophe. And a discard without a recorded outcome silently reintroduces the lost-message hazard the whole messaging discipline exists to prevent.

**When not to apply.** Triage is per diagnosed *cause*, not per message — a batch sharing one cause gets one verdict, applied to one message first and then the rest. A low-value, high-volume message class (a metrics ping) may carry a standing, pre-approved discard policy — that is still verdict (c), decided once and written down, not an exemption from deciding.

**Sources.** [Google SRE Book — Being On-Call](https://sre.google/sre-book/being-on-call/) (playbooks over improvisation).

### OPS-4 · Every bookkeeping row declares its retention, derived from its hazard window — and the purge is owned and tested [SHOULD]

**Rule.** Every bookkeeping row the messaging discipline creates — consumer dedup rows, idempotency records, processed-message guards, relayed outbox envelopes — is born with a **declared retention period and the reasoning behind it**. Derive it from the hazard the row guards, never from a default: an idempotency record must outlive the longest client retry horizon; a consumer dedup row must outlive the longest possible redelivery — which includes a hand-replay (OPS-3), so the floor is the dead-letter store's retention plus operational slack, not the broker's redelivery window. Purge earlier, and a dead letter replayed after the purge executes as if it had never been handled — the effect lands twice, months apart, and the trail points at the operator who replayed it. A row that doubles as a business outcome record (IDEM-5) is a domain row whose retention the business sets — declare that too. The purge job is **owned by the team that owns the table**, runs in production from day one, and is tested: the time seam (TEST-8) ages rows deterministically; assert both what goes and what survives.

**Why.** Unbounded bookkeeping tables slow the hot paths they protect: the dedup lookup runs before every handle, and its index bloats. The decay is gradual enough that nobody connects cause to effect.

**When not to apply.** A low-volume table may genuinely never hurt — but make that a declared decision ("retain forever; bounded at ~N rows/year"), not a default you fell into. Domain rows and event streams are not bookkeeping: their retention is a business and regulatory question (for event-sourced streams, see the erasure carve-out in ARCH-4).

### OPS-5 · Watch depth and oldest-age on every durable buffer; answer backpressure with capacity, never with discard [MUST]

**Rule.** Every durable buffer the system writes through exports two metrics: **depth** (how many are waiting) and **oldest-age** (how long the head has waited). That means the outbox's unrelayed rows, every queue, the error queue (OPS-2), and any scheduled-message store. Alert on both, because they catch different failures: depth catches a surge or a slow drain; oldest-age catches the stuck or stopped drain that depth hides at low traffic. The outbox's unrelayed depth and oldest-age are the canary for a **dead relay**: every write commits, every test passes, the service looks healthy — and nothing it commits is actually reaching anyone. Set thresholds from each consumer's business deadline — how slow a drain it can tolerate (MSG-6) — not from a generic default. When queue lag grows because consumers can't keep up, the levers are: raise consumer concurrency (CON-6), scale consumers out, or admit less work at the front door. Never strip message durability mid-incident to drain faster: removing a durable layer is a designed, paired configuration change (durable outbox + inline listener, PERF-2), not a backpressure lever. And never discard backlog wholesale: discard is a per-cause triage verdict with a recorded outcome (OPS-3).

**Why.** A durable buffer converts an outage into a delay — but only an observed buffer converts it into a *bounded* delay. Each buffer's growth also implicates a different component: outbox growth means a relay problem; queue growth means consumer capacity, or a consumer that is down; error-queue growth means poison. These metrics are therefore the first-line diagnosis map, not just alarms. *Example: subscriptions renew nightly; the relay host dies at 1 a.m.; renewal rows keep committing, outbox oldest-age crosses its threshold, the page fires at 1:10, and the relay is restarted before any customer notices — versus discovering at month-end that three weeks of renewal events never left.*

**When not to apply.** Ephemeral in-memory channels have no durable buffer to measure — losing them on a crash was the accepted contract (MSG-8). And this rule prescribes signals, not a platform: wire the two metrics when the buffer exists; tune thresholds once real traffic teaches you the baselines.

**Sources.** [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) (the four golden signals); Nygard, *Release It!* (back pressure); [Microsoft — Queue-Based Load Leveling pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling).

### OPS-6 · Readiness reflects what each path really depends on — bind the topology before consuming; keep the broker out of the HTTP probe [SHOULD]

**Rule.** Readiness is **per path, derived from real dependencies**. A consumer declares ready only after its topology is declared and bound — queues exist, bindings made. Consuming before the binding exists silently discards messages (the cold-start race); the reaper (IDEM-8) eventually repairs the stranded work that leaves behind, but the loss itself was preventable. The HTTP door's readiness **excludes the broker**: the outbox is its shock absorber (MSG-1), so the door can accept and commit work through a broker outage — a readiness probe that includes the broker converts a buffered outage into a full HTTP outage, throwing away exactly the availability the outbox bought. The store the door commits to *is* in its readiness.

**Why.** Cold-start ordering failures look like lost messages and stranded workflows, but they are configuration — preventable at startup, not reapable after. *Example: a fresh deploy starts consuming before its queue binding exists; the first dozen events vanish silently, and the gap in the dashboard is blamed on the network.*

**When not to apply.** A consumer-only process with no HTTP surface has nothing to exclude — its readiness is its topology plus its store. And liveness is a different probe with a different job (restart-worthiness); don't fold the two.

**Sources.** [Kubernetes — Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/). Pairs with disposability ([12-Factor IX](https://12factor.net/disposability)) — the startup half of RUN-2; [Microsoft — Health Endpoint Monitoring pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/health-endpoint-monitoring).

### OPS-7 · Decouple deploy from release; ship a risky change progressively, with the rollback trigger named before the rollout [SHOULD]

**Rule.** Deploying the binary and releasing the behavior are two events. A risky change reaches production **progressively** — a small slice of traffic first, widened only while health holds — and the **regression signal and the rollback action are chosen before the rollout starts**, never improvised mid-incident: name the metrics that must hold (error rate, latency, the buffer signals of OPS-5) and what reverts (traffic shifted back, N−1 redeployed). MSG-9 is the enabling discipline — mixed versions already coexist safely mid-roll, so a canary is a roll that pauses while you read the verdict.

**⚠ Trap:** A canary without a pre-named abort signal is just a full rollout in slow motion — by the time someone decides what "unhealthy" means, everyone has the change.

**Why.** Blast radius is the one lever fully in your control at release time: the same bug at 5% of traffic for five minutes is an incident; at 100% it is an outage. *Example: a pricing change ships to 5% of sessions; the error-rate gate trips after two minutes and traffic reverts — versus discovering at full rollout that every checkout in one currency returns 500.*

**When not to apply.** The mechanics — traffic splitting, flag infrastructure — are platform work; this rule is the design obligation that the change be safely partial (MSG-9) and that its verdict signals exist before rollout (OPS-5). A change that cannot be partial (a destructive contract step) takes the MSG-9 path instead, scheduled outside the rollout window. And not every change earns the ceremony — "risky" is a judgment the PR states.

**Sources.** [Sato, CanaryRelease](https://martinfowler.com/bliki/CanaryRelease.html); [Hodgson, "Feature Toggles"](https://martinfowler.com/articles/feature-toggles.html); [Google SRE Workbook — Canarying Releases](https://sre.google/workbook/canarying-releases/). Pairs with build-once-promote (RUN-3; [12-Factor V](https://12factor.net/build-release-run)).
