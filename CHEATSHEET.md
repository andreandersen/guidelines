# Cheatsheet — one page

> **Derived artifact.** A printable condensation; the chapters are the authority. A changelog entry touching a summarized rule updates this page.

## The non-negotiables

1. **A state change and the messages about it commit in one transaction** (transactional outbox). — MSG-1
2. **Every consumer tolerates redelivery; an unrepeatable effect gets a durable dedup guard** keyed to its boundary's hazard. — IDEM-1, IDEM-2, IDEM-3
3. **Poison is a bug to investigate; a business "no" is an outcome to record and emit** — never confuse the two routes. — IDEM-4, IDEM-5
4. **One winner per contended write**: optimistic concurrency on the aggregate; a set-wide rule gets the database constraint. — CON-1, CON-5
5. **The wire format is the contract** — pin it with a drift gate wherever serialized bytes outlive the process. — ARCH-5, TEST-1
6. **Test-first, at the layer that actually enforces the behavior.** — PROC-1, TEST-3
7. **A durable buffer you don't watch is a discard on a timer** — retention eventually deletes whatever nobody drains. Depth and oldest-age alerted, routed to the owning team. — OPS-2, OPS-5

## The always-on principles

Test-first (PROC-1) · small iterations (PROC-3) · framework idioms, verified from source (PROC-6) · decide whether the metric matters (PROC-8) · leave it better than you found it (PROC-10).

## Tiers, in one line each

**[MUST]** blocks merge. **[SHOULD]** needs a written waiver in the PR, citing the rule ID. **[CONSIDER]** guides design. A MUST is never waived in a PR — if a MUST is wrong, amend it first.

## Which id do you dedup on? (chapter-4 opener, IDEM-3)

| Id | Minted | Catches |
|---|---|---|
| Broker / envelope message id | Fresh at every send | Redelivery of that envelope only — the inbox window |
| Stable event id, persisted with the fact | Once, at decision time | Every re-send and re-emit of that fact |
| Caller-supplied idempotency key | Once, by the caller | The caller's retries of that operation |
| Natural / business key | Derived from the operation itself | Duplicates on every channel — and legitimate repeats too; only for genuinely unrepeatable operations |

Mint-time decides what an id identifies: per send, the envelope; once per intent, the operation. The version token is not a dedup mechanism (CON-1).

## Choosing a redelivery guard (IDEM-1)

| Effect / hazard shape | Guard | Why this guard | Retention |
|---|---|---|---|
| Read-only, or naturally idempotent (set-to-value) | None | A duplicate re-produces the same state; a guard is pure cost | — |
| Monotonic transition; re-running from the same pre-state is harmless | State guard: no-op unless the entity is in the expected pre-state | The entity's own state *is* the dedup record — no extra row, no window | None — lives on the entity |
| Non-monotonic transition; a stale duplicate could undo newer progress | State guard widened to the correlation key (a discriminator on the subject) | Pre-state alone can't tell "retry of this" from "stale message about a superseded subject" | None — lives on the entity |
| Unrepeatable effect; duplicates plausible only within transport redelivery | Framework durable inbox (dedup by message id) | Already built, already transactional — don't hand-roll what the framework provides | Framework window — typically minutes after handling; verify the default |
| Unrepeatable effect; duplicates possible *beyond* any window (dead-letter hand-replay, operator re-run, reaper re-emit) | Application dedup row keyed on a business id, written as INSERT (IDEM-2; key per IDEM-3) | Only a durable row you own outlives the inbox window; the PK collision closes the race | Explicit policy — at least as long as a replay is possible; unbounded-by-default is the trap |
| The outcome itself must be queryable or reportable | Dedup row elevated to a first-class outcome entity (IDEM-5) | A guard row doubling as the outcome log serves neither role well | Domain-driven — the audit horizon |
| Synchronous client double-submit at the HTTP door | Client-supplied idempotency key replaying the stored response (IDEM-3, IDEM-10) | No message id exists yet at the door; only the caller can name the retry | At least the client's retry horizon |

## Write-conflict status codes (IDEM-7)

Stale read (the row moved under you) → **412**, re-fetch and retry. Illegal in the current state → **409**, terminal — retrying loops forever.

## On call: which buffer is growing? (OPS-5)

Outbox growing: the relay died. Queue growing: consumer capacity, or a down consumer. Error queue growing: poison — page per OPS-2, triage per OPS-3.
