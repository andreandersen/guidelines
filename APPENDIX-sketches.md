# Appendix: Pseudocode sketches

The chapters state rules in prose; these five sketches show the shapes the rules keep referring to. They are stack-neutral pseudocode — the point is the shape, not code to paste. Each sketch names the rules it illustrates; the rules remain the authority.

## A1. The dedup guard: try-insert, let the primary key win the race

Illustrates IDEM-1, IDEM-2, IDEM-3.

```
-- one row per handled message; the primary key IS the guard
TABLE processed_message (
    message_id   PRIMARY KEY,   -- the message's own id, not business content (IDEM-3)
    outcome      text,          -- the coded result, e.g. "Reserved" / "InsufficientFunds"
    handled_at   timestamp
)

handle(message):
    begin transaction
    if exists(processed_message where message_id = message.id):
        return                                     -- fast path: a duplicate we already handled
    ...do the real work: load entity, mutate, enlist outgoing messages...
    insert processed_message(message.id, outcome, now())   -- INSERT, never upsert
    commit
```

If two copies of the message run truly in parallel, both pass the `exists` check. The second `insert` then violates the primary key at commit and rolls back its entire transaction — work, guard row, and outgoing messages together. The lookup is the fast path; the PK violation is the race-proof backstop.

## A2. State change + outgoing message in one transaction (the outbox)

Illustrates MSG-1 (which also owns compose-from-the-persisted-entity, formerly ARCH-11).

```
handle(ClaimOrder command):
    begin transaction
    order = load Order by command.orderId        -- read current state; never trust the request
    order.Claim(command.buyerId)                 -- the entity enforces the legal transition
    event = OrderClaimed(                        -- composed from the post-write entity,
        orderId: order.id,                       --   not echoed from the inbound command
        buyerId: order.buyerId)
    outbox.enlist(event)                         -- written as a ROW in the same database
    commit                                       -- state change + outbox row commit atomically
```

After commit, a relay reads the outbox table and publishes to the broker. On rollback the outbox row vanishes with everything else. So a crash can neither lose a committed fact nor leak a message about an uncommitted one.

## A3. The golden-sample wire-shape test (drift gate)

Illustrates TEST-1 and ARCH-5.

```
test "OrderClaimed v1 wire shape has not drifted":
    sample = read("tests/contracts/order-claimed.v1.json")    -- committed by hand, never regenerated
    event  = OrderClaimed(orderId: FIXED_ID,
                          buyerId: FIXED_BUYER,
                          reason: ClaimReason.Gift)            -- pinned values, no randomness
    actual = serialize(event, PRODUCTION_SERIALIZER_OPTIONS)   -- the exact options production uses

    assert json_equal(actual, sample)      -- catches renames, type changes,
                                           -- an enum silently flipping names → integers
    assert deserialize(sample, PRODUCTION_SERIALIZER_OPTIONS) == event
                                           -- yesterday's bytes still readable today
```

When it fails, a human decides: intended evolution (update the sample as a deliberate contract change) or accidental drift (fix the code). Never auto-overwrite the sample on mismatch — that turns the gate into a rubber stamp.

## A4. Poison vs. business failure inside one consumer

Illustrates IDEM-4–IDEM-5 and MSG-2. (The A1 dedup fast-path at the top of the handler is omitted here for focus.)

```
handle(ReserveStock command):
    -- POISON: structurally invalid; no retry can fix a bug upstream
    if command.eventId is empty
       or command.quantity <= 0
       or command.sku is unknown:
        throw InvalidInstruction         -- zero retries → dead-letter for a human;
                                         -- NO business verdict is issued

    stock = load Stock by command.sku

    -- BUSINESS "no": a normal, expected outcome — not an exception
    if stock.available < command.quantity:
        publish StockReservationRejected(command.eventId,
                                         reason: InsufficientStock)   -- stable code, never free text
        insert processed_message(command.eventId, "InsufficientStock", now())
        return                           -- the caller's workflow resolves cleanly

    stock.Hold(command.quantity)
    publish StockReserved(command.eventId)
    insert processed_message(command.eventId, "Reserved", now())
```

The two routes never mix: poison gets investigation and no verdict; a business "no" gets a coded verdict and no investigation. Mis-routing either way is the bug (IDEM-4).


## A5. The concurrency-conflict retry: re-read and re-decide

Illustrates IDEM-6 (with IDEM-4–IDEM-5) and CON-1.

```
-- the concurrency-conflict retry: re-decide from fresh state, never replay the old intent
handle(ClaimOrder msg):
    order = load(msg.orderId)                     -- fresh read on every attempt
    if order is missing:
        throw InvalidInstruction                  -- poison → dead-letter (IDEM-4)
    if order.state ≠ Open:
        publish OrderClaimRejected(msg.eventId,
                                   reason: AlreadyClaimed)  -- business "no": recorded + emitted (IDEM-4–IDEM-5)
        insert processed_message(msg.eventId, "AlreadyClaimed", now())
        return
    order.Claim(msg.buyerId)                      -- decide against current state
    save(order)                                   -- version token: the losing writer throws (CON-1)
    -- a concurrency conflict re-runs the WHOLE handler: it re-reads, and may now
    -- legitimately land in the rejection branch instead of double-applying
```

The retry's value is the re-read: the second attempt sees the world the winner left behind and can reach a different, correct outcome (IDEM-6).