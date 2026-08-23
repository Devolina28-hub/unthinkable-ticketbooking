# System Design Write-Up

## Seat hold & TTL mechanism

Each seat's state lives on a single `event_seats` row (one row per seat per show,
created by snapshotting the venue's layout when an event is published). A seat has
one of four statuses: `available`, `held`, `offered`, `booked`. When a customer
selects seats, the API stamps each selected row with `status='held'`, `held_by=<user>`,
and `hold_expires_at = now + SEAT_HOLD_TTL_MINUTES` (default 10 minutes,
configurable via `.env`). This TTL is enforced two ways rather than one, to avoid a
single point of failure:

1. **A background sweeper** (`node-cron`, default every 30s) runs
   `UPDATE event_seats SET status='available', held_by=NULL, hold_expires_at=NULL
   WHERE status='held' AND hold_expires_at < now`. This is the durable, database-level
   guarantee — even if every client disappears, the seat map self-heals.
2. **A synchronous check on every read path** (seat map fetch, event detail, booking
   confirm) calls the same release function before answering, so a client can never
   observe a stale "held" seat that has actually expired, even in the gap between
   sweeper ticks.

On the frontend, the seat map polls every 4 seconds and shows a live countdown for the
customer's own active hold, clearing the selection client-side when it hits zero — but
the server is always re-checked on confirm, so a client-side clock drift can never
result in booking an expired hold.

## Concurrency prevention

The core guarantee — two customers can never both hold or book the same seat — comes
from treating every state transition as a **conditional UPDATE**, not a read-then-write:

```sql
UPDATE event_seats SET status='held', held_by=$1, hold_expires_at=$2
WHERE id=$3 AND event_id=$4 AND status='available'
RETURNING id;
```

If the row is no longer `available` when this executes, the query returns zero rows and
the request fails cleanly with `409 Conflict`. There is no "check status, then write"
window for a race condition to exploit, because the check and the write are the same
atomic statement. This pattern is reused identically for hold → booked, cancel →
available, and available → offered (waitlist), so every transition in the system has
the same safety property.

The database (Postgres, hosted on Neon) serializes concurrent `UPDATE` statements
targeting the same row under the hood: if two requests race for the same seat, the
second `UPDATE` blocks until the first transaction commits or rolls back, then
re-evaluates its `WHERE status='available'` clause against the now-committed row — so
it correctly finds zero matching rows with no explicit locking code required on the
application side. Multi-seat holds are wrapped in a single transaction, so a customer
either secures every seat they asked for or none of them — no partial holds that leave
orphaned reservations. This was validated with a genuinely simultaneous test (both
requests fired via `Promise.all`, not sequentially): exactly one request returned `200`,
the other `409`, and the seat's final state in the database showed it held by exactly
one customer, with no double-assignment.

## Waitlist auto-assignment flow

Waitlist entries are queued FIFO per `(event_id, category)`, ordered by `joined_at`.
The assignment logic lives in one function, `offerSeatToNextInLine`, called from two
places: booking cancellation, and offer expiry (see below) — so cancellations and
expired offers cascade through the exact same code path. When a seat becomes free:

1. The oldest `waiting` entry for that event+category is selected.
2. The seat is atomically flipped `available → offered` (same conditional-UPDATE
   pattern), reserved for that customer, with `hold_expires_at = now +
   WAITLIST_OFFER_TTL_MINUTES` (default 15 minutes).
3. The waitlist row is marked `offered`, and the customer is emailed a link to
   `/waitlist-offer/:id` with the deadline stated explicitly.
4. The customer calls `POST /waitlist/:id/complete`, which re-validates the seat is
   still `offered` and still theirs, then converts it into a real booking (QR
   generated, confirmation emailed) using the same transactional confirm logic as a
   normal checkout.

A customer can only have one active (`waiting`/`offered`) entry per event+category,
enforced by a partial unique index — no duplicate queue-jumping.

## Time-limited offer handling

Offer expiry reuses the same sweeper that handles hold TTLs. Each pass also scans for
`waitlist` rows with `status='offered'` and `offer_expires_at < now`. For each one, it:
marks the waitlist entry `expired`, releases the seat back to `available`, and then
immediately calls `offerSeatToNextInLine` again for that event+category — so an unused
offer doesn't dead-end the queue; it cascades to whoever is next, recursively, until
either someone accepts or the queue is empty. Because this reuses the identical
conditional-update primitive as every other transition, an offer that's completed in
the same instant the sweeper tries to expire it cannot be double-processed: whichever
operation's UPDATE lands first on the `status='offered'` row wins, and the other finds
zero matching rows and no-ops.
