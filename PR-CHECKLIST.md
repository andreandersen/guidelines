# PR checklist

> **Derived artifact.** Generated from the [rule register](rule-register.md)'s `pr-checklist` rows; the chapters are the authority. Check only the sections your PR touches — most PRs check 3–5 boxes. A **[MUST]** box left unchecked blocks the merge; a **[SHOULD]** deviation needs a one-line waiver citing the rule ID (see README — Governance).

## Every PR

- [ ] A failing behavior test came first, written at the slice or contract boundary (PROC-1) **[MUST]**
- [ ] The merge-gate suites are green — no test quarantined or blindly retried to get past the gate (TEST-11) **[MUST]**
- [ ] Names use the business's vocabulary; renames follow when the vocabulary shifts (ARCH-1) **[SHOULD]**
- [ ] No speculative structure; abstractions added only once repetition proves the need; comments say *why*, not *what*; touched code left a little better, with cleanup scoped to the change (ARCH-13, ARCH-14, PROC-9, PROC-10) **[SHOULD]**

## Changing code whose behavior must not change?

- [ ] The test that pins current behavior was committed *before* the change, and passes against the old code (PROC-2) **[MUST]**

## Touched a message contract or the HTTP door's shapes?

- [ ] The change is additive, or ships as a new message version alongside the old; a change in meaning counts as breaking (MSG-3) **[MUST]**
- [ ] The drift gate (golden sample / endpoint shape test) is updated deliberately, never regenerated (TEST-1) **[MUST]**
- [ ] New fields classified — identifiers by default; personal data is a recorded exception (SEC-3) **[SHOULD]**

## New consumer or message handler?

- [ ] Redelivery guard chosen from IDEM-1's table; a replay-twice test proves the second delivery changes nothing (TEST-6) **[MUST]**
- [ ] Poison messages and business rejections take different routes: dead-letter the first visibly, record the second as a coded outcome (IDEM-4) **[MUST]**
- [ ] The slice declares its own retry/error policy, covering every fault mode (MSG-6, ARCH-7) **[SHOULD]**
- [ ] The consumer's dead-letters raise an alert routed to the owning team (OPS-2) **[MUST]**
- [ ] Correlation and causation ids propagated, including on timeouts and re-emits (OPS-1) **[SHOULD]**
- [ ] Readiness declared from real dependencies — topology bound before consuming (OPS-6) **[SHOULD]**

## New endpoint, or calling another service?

- [ ] Principal and required permission declared; anonymous is an explicit marker (SEC-1) **[MUST]**
- [ ] Front-door validation with bounds, paginated collections, one error shape (IDEM-10, SEC-2) **[MUST]**
- [ ] Every synchronous out-of-process call has a timeout from the caller's deadline; bounded retries; circuit breaker where warranted (MSG-10) **[MUST]**

## New bookkeeping table (dedup, idempotency, outcome, outbox)?

- [ ] Retention derived from the hazard window — how long a duplicate can still arrive; purge job owned and tested (OPS-4) **[SHOULD]**
- [ ] The buffer exports depth and oldest-age, with alerts on both (OPS-5) **[MUST]**

## Schema or stored-shape change with live data?

- [ ] Expand → migrate → contract; compatible with the currently-deployed binary; forward-only, run in CI against the real engine (MSG-9) **[MUST]**

## Touched tests or test infrastructure?

- [ ] Tests sit at the layer that enforces the behavior (TEST-3) **[SHOULD]**
- [ ] Tests that touch infrastructure run against the real engine, not a fake (TEST-2) **[MUST]**
- [ ] Endpoint behavior is asserted through the full pipeline, not by calling the handler directly (TEST-4) **[SHOULD]**
- [ ] Tests sharing a host use fresh ids and never assert on whole-table state (TEST-9) **[SHOULD]**
- [ ] Configuration that would fail silently when wrong has a direct assert (TEST-5) **[SHOULD]**
- [ ] Event-sourced behavior is tested in the given-events / when-command / then-events shape (TEST-10; only where ARCH-4 applies) **[SHOULD]**

## Added or changed a config setting?

- [ ] The setting lives outside the artifact; no secret enters the repo; a required setting with no default fails startup, and the error names the setting (RUN-1) **[MUST]**

## Introduced an interim seam or dev-only shortcut?

- [ ] The PR states the condition under which the seam gets deleted (ARCH-12) **[SHOULD]**
