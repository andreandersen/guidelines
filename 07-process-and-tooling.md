# Process & Tooling

### PROC-1 · Test-first is mandatory (red-green-refactor) [MUST]

**Rule.** Write a failing test before any production code and let it drive the design; treat red-green-refactor as a non-negotiable discipline, not an aspiration applied when convenient. Carry it forward deliberately even when the codebase's own conventions don't mandate it. Put real effort into the **test's reasoning** — what exactly it pins, and why — not just mechanically writing a test: the failing test should be a **behavior test at the slice or contract boundary**, asserting an observable outcome, not mock choreography (a test-first mock-verification test satisfies the ritual and proves nothing).

**⚠ Trap:** The moment "wiring" acquires a conditional, it is behavior, and the mandate applies.

**Why.** Its force comes from an explicit decision; without deliberately carrying it forward, implementation-first habits erode the design-driving benefit.

**When not to apply.** The mandate is absolute for **production behavior** — anything a user, a consumer, or a later maintainer relies on. There is no per-task opt-out. Two things lie *outside* that scope; neither is an exception inside it.

*Spikes.* Code written to answer a question — "does the framework support this?", "how does this API actually behave?" — may be written test-free, under a fixed protocol: **timeboxed** up front; on a branch or scratch tree **labeled as a spike**; **never merged**. Keep the learning — a note, an ADR, the disproved assumption. Then delete the code, or rebuild it test-first. The spike is the research, not a first draft: even a spike that "works" is re-driven red-green-refactor, with the spike open beside you as reference.

*Scaffolding and pure wiring.* Code with no decision logic of its own — project skeletons, dependency registrations, route plumbing — needs no dedicated prior test. The slice-level behavior test that drives the feature (TEST-3–TEST-4) already fails until the wiring exists; that failing test *is* the test-first driver.

Adoption remains a social process: introducing the discipline to a team that hasn't committed is persuasion, not unilateral imposition.

**Sources.** Beck, *Test-Driven Development: By Example*; [Fowler, TestDrivenDevelopment](https://martinfowler.com/bliki/TestDrivenDevelopment.html).

### PROC-2 · When behavior must not change, commit the test that proves it before you touch the code [MUST]

**Rule.** When a change must preserve an external contract or behavior, write and commit the pinning test **first**, green against the old code, then make the change under it. Build that test like this:

- **Pin what the code does, not what it should do.** A characterization test (Feathers) records actual current behavior. If you find a bug while pinning, write it down and pin the buggy behavior anyway — the fix is a separate change with its own test. Never mix a behavior change into a refactor.
- **Capture output as comparable data.** Assert on a serialized form — a response body, a generated document, the list of emitted events — and diff it against a committed snapshot. This is approval testing (also called golden master); libraries exist for it in every stack (ApprovalTests, Verify, Jest snapshots). Review the snapshot file like code. On a mismatch, a human decides — never regenerate (TEST-1).
- **Cover the inputs that matter.** One happy-path sample proves little. Run coverage to see which branches your samples reach, and add inputs until the code you are about to change is actually exercised.
- **Commit the test separately, before the change.** Any divergence the change introduces then fails a test that already existed — proof, not assertion.

**⚠ Trap:** A pinning test that cannot fail proves nothing. Before trusting it, sabotage one line of the code and watch the test go red, then revert. Mutation testing is the systematic version of this check.

**Why.** It proves the change preserved behavior instead of hoping it did. This is test-first (PROC-1) applied to refactoring.

**When not to apply.** Needs behavior capturable as stable, comparable data — pin randomness, timestamps, and generated ids through seams first (TEST-8). It doesn't apply where the old behavior *is* the bug being fixed; there the new test asserts the fix.

**Sources.** Feathers, *Working Effectively with Legacy Code* (characterization tests); [Bache, the Gilded Rose Refactoring Kata](https://github.com/emilybache/GildedRose-Refactoring-Kata) — the standard exercise for exactly this workflow; [Falco, ApprovalTests](https://approvaltests.com/); [Stryker — mutation testing](https://stryker-mutator.io/).

### PROC-3 · Prefer small iterations; reject large change-orders [SHOULD]

**Rule.** Default to small, independently-valuable, independently-shippable iterations. **Reject a large change-order** — decompose it into the smallest steps that each leave the system green and releasable. A big-bang staged megaplan is a **last resort**, only when a change is genuinely large and atomic; even then, give each stage its own plan + end-of-stage green gate + rollback point, and **delete dead code before any mechanical rename / sweep** so the sweep targets a smaller, unambiguous surface.

**Why.** Small iterations stay reviewable, reversible, and shippable; large staged plans carry real overhead and go stale fast.

**When not to apply.** A genuinely atomic large change (e.g. a contract split that must move both sides at once) is the staged-plan case — but that is the exception, not the default.

**Sources.** [Google Engineering Practices — Small CLs](https://google.github.io/eng-practices/review/developer/small-cls.html); Forsgren, Humble & Kim, *Accelerate* (small batches).

### PROC-4 · Mark superseded designs in place; don't silently delete — and ask when unsure [SHOULD]

**Rule.** When a design document or decision is replaced, keep it and add an unmissable banner pointing to its successor and explaining what changed, rather than deleting it. When you're unsure whether something is safe to delete, **ask the owner** rather than assume.

**Why.** The superseded reasoning is the cheapest way for a future reader to understand why the current design exists; a banner turns a stale doc into accurate history instead of a trap.

**When not to apply.** Accumulating superseded docs needs the banner unmissable and the link correct, or readers can no longer tell current from dead; periodically archive the truly dead ones.

**Sources.** [adr.github.io](https://adr.github.io/) (the superseded-by status).

### PROC-5 · Record decision reasoning and the rejected alternative, not just the verdict (ADRs) [SHOULD]

**Rule.** Capture hard-won decisions as short Context → Decision → Consequences records that preserve the rejected alternative, why it lost, any dissent, and the conditions under which to revisit. Make consulting them a precondition for changing the pattern they govern; inline a note at the code site where a known tension lives.

**Why.** On a system where the same tradeoff resurfaces, re-deriving the answer each time risks quietly reversing a deliberate choice.

**When not to apply.** Overhead worth paying only for genuinely contested or non-obvious decisions; a "the framework forced it" choice doesn't need a record, and one ADR per micro-decision is bureaucracy.

**Sources.** [Nygard, "Documenting Architecture Decisions"](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions); [adr.github.io](https://adr.github.io/).

### PROC-6 · When correctness rests on a framework, read its source and docs — then prefer its idioms to your own [SHOULD]

**Rule.** For framework behavior your correctness rests on, read the framework's **actual source / docs / samples** rather than guessing, recalling, or decompiling — and actively look for its idioms, best-practices, and samples. Then **prefer the idiomatic mechanism over a bespoke one** unless you have a concrete reason to diverge. Pin the verified fact where the decision that depends on it lives.

**Why.** Several subtle correctness bugs (transactional flush semantics, error-policy continuations, default serializer behavior) hinge on exact framework behavior; and many "grown out of hand" complications are bespoke reinventions of something the framework already offers idiomatically. **This is the single best antidote to over-engineering.**

**When not to apply.** Reading source has a real time cost and is overkill for well-documented stable APIs; reserve it for behavior that is load-bearing, version-sensitive, or contradicts the docs — and clone only the one or two frameworks your correctness actually rests on. Keep a bespoke mechanism only where it solves something the framework genuinely doesn't.

### PROC-7 · Gate expensive review automation behind explicit intent; auto-trigger only cheap checks [CONSIDER]

**Rule.** A review process that fans out into many agents (or any expensive automated gate) should be invoked manually by default; reserve auto-triggering on every change for cheap, fast, high-signal checks. The discriminator is per-run cost versus per-change value.

**Why.** Auto-firing a heavy multi-agent review on every save dominates the cost and latency of routine edits.

**When not to apply.** The opposite is right for cheap gates — a linter, a fast unit suite, a whitespace check should auto-trigger.

### PROC-8 · Decide whether a metric matters before chasing it; isolate the rig before benchmarking [SHOULD]

**Rule.** Before investing in moving a metric (throughput, latency), ask whether it matters for the system's stated purpose — don't optimize what the system isn't for. If you must benchmark, **isolate the rig**: a system-under-test co-located with its datastore, broker, and load generator confounds every conclusion (host-wide metrics hide the real bottleneck, and the generator's own footprint can become the ceiling).

**Why.** This is the meta-lesson behind a whole category of wasted effort — chasing a number that didn't matter, measured on a rig that couldn't answer the question.

**When not to apply.** When the metric genuinely *is* the product (a low-latency matching engine, a high-throughput streaming pipeline), invest fully — but still isolate the rig.

**Sources.** [Gregg, "The USE Method"](https://www.brendangregg.com/usemethod.html); Knuth, "Structured Programming with go to Statements" (premature optimization).

### PROC-9 · Write for the reader: names say intent, comments say why [SHOULD]

**Rule.** Code is read many times more often than it is written — optimize for the reader at the writer's expense. Names reveal intention at every scale: ARCH-1 governs the domain vocabulary; this rule extends the same discipline to locals, parameters, helpers, and tests — a name that needs a comment to explain it is the wrong name. Comments carry what the code **cannot** say: the *why* — the constraint, the rejected alternative, the surprise ("deliberately not cached — see ADR-7") — never a narration of *what* the next line does. A missing why-comment at a surprising line is a defect; a comment restating the code is noise that will rot into a lie.

**⚠ Trap:** When a comment and its code disagree, both are wrong — the comment lied, and the code lost its explanation. Comments update in the same change as the code they justify; a reviewer treats a stale comment like a failing test.

**Why.** The reader is usually you, later, without the context you hold now — and sometimes it is OPS-3's operator, reading under incident pressure. *Example: `if (retries > 3 && !order.IsGift)` — the code says what; only a comment can say "gift orders bypass the retry cap: fulfilment retries are free-of-charge replays (ADR-12)."*

**When not to apply.** Self-evident code needs no commentary — the discipline is *why over what*, not a comment quota. Generated code and labeled spikes (PROC-1's protocol) are exempt.

**Sources.** Ousterhout, *A Philosophy of Software Design* (a comment earns its place only by saying what the code cannot); McConnell, *Code Complete*; Hunt & Thomas, *The Pragmatic Programmer*.

### PROC-10 · Leave code better than you found it — refactor in the flow of work, never as a scheduled project [SHOULD]

**Rule.** Refactoring is a continuous part of delivering a change, not a separate work item: the *refactor* in red-green-refactor (PROC-1), plus the boy-scout pass — when a change drags you through code that resists it, improve what you touched (a clarifying rename, an extracted function, a deleted dead branch) and ship the improvement with the work, as its own behavior-preserving commits. Scope it to the campsite, not the forest: cleanup roughly the size of the change you came to make. Ship it in the same PR while the diff stays reviewable at a glance; give it an immediately-adjacent PR when it would drown the diff; and put it under a pinning test wherever external behavior must be preserved (PROC-2). This rule forbids the alternative: months of "no time to clean up" accumulating into a "refactoring sprint" — a large, behavior-free change-order that PROC-3 already rejects.

**⚠ Trap:** "While I'm here" is a budget, not a license — opportunistic cleanup that balloons past the ticket's scope is how a one-day change becomes a week-long rewrite nobody asked for. When the mess is bigger than the campsite, file it visibly (a roadmap entry, ARCH-12) instead of conquering it today.

**Why.** Each tolerated mess invites the next (broken windows); each small cleanup makes the next change cheaper. Deferred-cleanup projects are large, behavior-free, compete with features for priority, and go stale before they start. *Example: a developer adds parameter #13 to a function taking 12; the boy-scout move — introduce the parameter object now, one PR, fifteen minutes — versus the "parameter cleanup epic" that never gets scheduled.*

**When not to apply.** Not every touch obligates a cleanup — code you read but didn't change owes you nothing, and a hotfix under incident pressure ships the fix alone, with the cleanup filed as the follow-up. Don't refactor what you don't yet understand: the rule is "better than you found it," and to a reader who hasn't learned the code's reasons, "better" is often wrong (PROC-4's ask-the-owner instinct applies to code too).

**Sources.** Hunt & Thomas, *The Pragmatic Programmer* (the boy-scout rule; software entropy); [Fowler, OpportunisticRefactoring](https://martinfowler.com/bliki/OpportunisticRefactoring.html); Fowler, *Refactoring*.