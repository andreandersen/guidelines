# Performance

> **Read this first.** Performance work has two preconditions, both owned by PROC-8: decide whether the metric matters for the system's purpose, and isolate the benchmark rig. Keep the methodology below; skip the instinct to chase the number.

### PERF-1 · Profile before optimizing — idle resources mean you're not CPU-bound [CONSIDER]

**Rule.** Before tuning code, measure where wall-clock time actually goes. If the process's threads are mostly parked waiting on I/O while host resources sit idle, the bottleneck is orchestration / latency, not your code's CPU — optimizing the code will not move the ceiling. And before building a fix around the leading theory, **disprove it cheaply first**: a lock or wait event that dominates a profiler view is often a *symptom* of a different root cause — run a targeted, cheap experiment before committing to an expensive fix.

**⚠ Trap:** Convenience profiler exports can present the wrong time model (wall-clock vs on-CPU) and lie. Confirm what the tool actually measures before trusting its breakdown; when the default lies, count the raw sample events against the same source the trusted UI uses.

**Why.** Redirects effort from rewriting service code (wasted) toward the real levers, and kills expensive changes that wouldn't have helped.

**When not to apply.** Requires representative load with dependencies visible — if services run on separate hardware from their datastore, the parked-thread signature is normal and uninteresting. The disproving experiment must be cheap relative to the real fix; if validation itself is costly, weight your prior more heavily.

**Sources.** [Gregg, "The USE Method"](https://www.brendangregg.com/usemethod.html); Gregg, *Systems Performance*.

### PERF-2 · A saturated shared resource sets the ceiling — only fewer operations against it, or removing the saturation, moves it [CONSIDER]

**Rule.** When a shared resource (commit / fsync throughput, a connection pool, a single broker) is saturated, it sets the throughput ceiling. Two kinds of lever **move** it. The first is to *reduce the number of expensive operations against it* — every durable inbox, outbox, or dedup write is a commit. So pay durability only where loss is unrecoverable. And drop a durable layer when another mechanism already guarantees the same thing. The second is to *remove the saturation*: separate machines, or structurally less work per transaction. Levers that **don't** move it: cheaper per-operation work (fewer queries, less logging — lowers consumption per unit, not the ceiling) and scaling up co-located instances (they share the one box's budget). **Measure per-component** — host-wide metrics hide which resource is the wall.

**⚠ Trap:** Dropping the durable inbox is only loss-safe with an **inline, ack-after-handling listener**. Frameworks commonly default listeners to a buffered mode that acks the broker *before* handling; a crash then silently loses every acked-but-unhandled message, and no dedup row or reaper ever sees them. Durable outbox + inline listener is a paired configuration, not two independent choices.

**Why.** Prevents mistaking efficiency wins for throughput wins, and prevents wasted "split the dependency" effort that doesn't escape a single-box ceiling.

**When not to apply.** Keep the efficiency wins — they pay off the moment the bottleneck resource is no longer saturated. On slow or network storage, commit-flush tuning that is a dead end on fast local disks can matter again.

**Sources.** [Gregg, "The USE Method"](https://www.brendangregg.com/usemethod.html); Gunther, *Guerrilla Capacity Planning* (the Universal Scalability Law).

### PERF-3 · Spread test data the way production spreads it — or the ceiling you find belongs to the test, not the system [CONSIDER]

**Rule.** At high throughput, a synthetic workload concentrated on too few entities creates concurrency contention that doesn't exist in production's wider distribution. Match test-data cardinality to a realistic spread, or a plateau that looks like a system limit is really a *test-data* limit.

**Why.** Explains "ceilings" that are actually test-data artifacts, and unblocks the next scaling step.

**When not to apply.** Cuts both ways — if production genuinely concentrates load on a few hot entities, a wide-distribution test hides a real contention risk you must test separately.

**Sources.** Kleppmann, *Designing Data-Intensive Applications* (describing load; skew and hot spots).