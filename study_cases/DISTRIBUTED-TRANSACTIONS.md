# Transactional boundaries across the case catalog

How the 50 study cases keep multi-part writes consistent — two-phase commit, saga and
compensation, the transactional outbox, exactly-once ambitions, idempotent retries.
Companion to [CONCEPTS.md](CONCEPTS.md): the "Idempotency keys / at-least-once delivery"
union (23/50, 46%) and the "Durable work queues (incl. saga/outbox)" row (11/50, 22%).
Those rows track the idempotency primitive; this doc owns the transactional-boundary
framing — what happens when a business operation spans store + log + external call.
Counts below are **mention-based** (grep over the corpus, verified in context) and are
not the union counts in [CONCEPTS.md](CONCEPTS.md).

Headline: **no case in the catalog implements distributed commit — every case that
considered 2PC rejected it, and the universal substitute is a two-move maneuver: first
shrink the transaction until it fits one store, then cover the residue with idempotency
plus saga compensation plus reconciliation.** The rejection is stated with unusual force
in [44.payment-system](44.payment-system/README.md): "No 2PC across banks — by fiat and
by physics" ([04-deep-dives.md](44.payment-system/04-deep-dives.md)) — and with unusual
casualness in [39.e-commerce](39.e-commerce/README.md), which drops it in one line
against compensation because "compensation is cheap here precisely because ... expiring
reservations". And the honest wording of "exactly-once" appears only as a negation:
"Exactly-once is a lie; effective exactly-once is the deliverable" (37). Nobody claims
the wire delivers once; they claim the *effect* happens once.

- Cases naming distributed commit (2PC / distributed transaction) in any way: **7** (10, 12, 28, 36, 39, 44, plus [ZOOKEEPER.md](ZOOKEEPER.md)'s scope-row for 11)
- Cases where it is a designed-away substitute (saga / hold-then-confirm / collapse-to-one-writer): **4** (28, 36, 39, 44)
- Cases where it is a scope line ("not for this tier"): **2** (10, 12)
- Cases that implement it: **0 / 50**
- Saga / compensation as a designed mechanism: **4** (29, 36, 39, 44)
- Transactional outbox, server-side (status write + event publish in one local transaction): **1** (04; cited by 27 as "the outbox pattern")
- Client-side outbox / send queue: **5** (21, 19, 24, 25, 22) — owned by [SYNC-AND-CRDT.md](SYNC-AND-CRDT.md)
- Directories where "exactly once" appears: **17** by grep; **15** genuine after
  filtering the two negative-scoped requirements (11, 49); within those, **5** state the
  wire/effect split in so many words (03, 06, 37, 44, 35)

## Summary table

| Case | Boundary mechanism | Depth |
|---|---|---|
| [44.payment-system](44.payment-system/README.md) | 2PC rejected by fiat; saga across bank rails (void/refund); idempotency keys (the dive); reconciliation as backstop | Full case |
| [39.e-commerce](39.e-commerce/README.md) | Checkout saga, PENDING order row as idempotency anchor, keyed compensations, TTL sweeps | Full case |
| [36.airbnb](36.airbnb/README.md) | Hold→confirm two-phase against the PSP; saga both failure worlds; unknown-outcome = in-flight + reconcile | Core component |
| [37.ticketmaster](37.ticketmaster/README.md) | Two-phase auth-then-capture, hold_id as idempotency key, "effective exactly-once" invariant, hourly reconciliation | Core component |
| [06.apple-app-store](06.apple-app-store/README.md) | Idempotency key end-to-end, single-writer ledger, PSP keyed one layer down, offline receipts | Core component |
| [27.ev-charging-network](27.ev-charging-network/README.md) | At-least-once inputs → exactly-once effect: deterministic engine + insert-if-absent ledger | Core component |
| [04.deployment-system](04.deployment-system/README.md) | Transactional outbox (status + event one transaction) + fencing tokens on re-claim | Core component |
| [03.whatsapp](03.whatsapp/README.md) | At-least-once wire + client dedup = exactly-once as-experienced; the catalog's origin contract | Core component |
| [29.uber-eats](29.uber-eats/README.md) | Compensating ledger rows (never edit history), validating-owner state machine, force-forward policy | Moderate |
| [34.reddit](34.reddit/README.md) | Synchronous transactional vote writes rejected for hot-key contention; event log + idempotent consumers | Designed |
| [35.tiktok](35.tiktok/README.md) | End-to-end exactly-once priced and rejected; at-least-once + event UUID + consumer dedup | Moderate |
| [09.cdn](09.cdn/README.md) | Exactly-once billing via per-POP sequence numbers, not per-request dedup + daily reconciliation | Moderate |
| [28.uber](28.uber/README.md) | Reserve = CAS in the cell's geo shard ("no distributed transaction needed"); timeout as timer | Moderate |
| [42.linkedin](42.linkedin/README.md) | CAS-guarded invite accept; transactional rows for pending state | Brief |
| [16.dalle](16.dalle/README.md) / [18.sora](18.sora/README.md) | Idempotent job effects: keyed artifacts overwrite the same output; retries safe | Designed (shared shape) |
| [05.discord](05.discord/README.md) | Client dedup by message snowflake — retry idempotent at the store | Brief |
| [33.instagram](33.instagram/README.md) | Client-supplied idempotency token makes a second write a no-op | Brief |
| [08.chatgpt](08.chatgpt/README.md) | Tokens idempotent-per-offset in Sse delivery | Brief |
| [41.web-crawler](41.web-crawler/README.md) | Consumer dedup by URL+content-hash over at-least-once replay | Brief |
| [10.distributed-cache](10.distributed-cache/README.md) | Scope line: no cross-shard transactions, "a job for the origin stores" | Boundary |
| [12.bigtable](12.bigtable/README.md) | Row = unit of atomicity by design; losing is "no cross-row" consistency — accepted | Boundary |
| [07.blob-store](07.blob-store/README.md) | Two-phase PUT (bytes durable before index commit) — one system, ordering protocol not commit protocol | Boundary |
| [13.kafka](13.kafka/README.md) / [50.stock-exchange](50.stock-exchange/README.md) | Exactly-once transactions named (13) or mooted by deterministic replay (50) — log-replication domain | Boundary |
| [21.frontend-calendar](21.frontend-calendar/README.md) + 19, 24, 25 | Client outbox + idempotent ops — the client half of the boundary | Boundary → [SYNC-AND-CRDT.md](SYNC-AND-CRDT.md) |

## Per-case notes

### The rejection is the pattern

**[44.payment-system](44.payment-system/README.md)** is the only case where distributed
commit is physically impossible rather than merely unpriced: the other participants are
issuers. [04-deep-dives.md](44.payment-system/04-deep-dives.md) §3: "no issuer will lock
an account pending a distributed-commit from our side. The only viable protocol is saga +
compensations: `void` compensates a not-yet-captured auth (no network fee); `refund`
compensates a capture (fees partially lost — so design for *pre-capture cancellation*
whenever possible)". The economics even shape the design: refund loses fees, so 44 pushes
merchants toward late capture. The ambiguous-timeout case is refused any clever answer —
timeout on the acquirer "does *not* mean failure — it means unknown" — resolved only by a
status inquiry and the daily settlement-file reconciliation (§4): "Real-time says what we
*believe*; the file says what *happened*; both are stored". The idempotency dive (§1) is
the boundary's write-path half: a gateway replay cache with ~24 h TTL, "same key,
*different* request body → explicit error", and store outage → "reject writes
conservatively (503), never process unkeyed" — fail closed at the boundary because an
unkeyed write is the one thing no later mechanism can fix.

**[39.e-commerce](39.e-commerce/README.md)** runs the same protocol a tier down and
states the cheapness directly: the saga `reserve → authorize → redeem promo → commit →
emit` runs "failure at any step runs release/refund/release in reverse. Compensations
are keyed by the same idempotency key so the saga itself is retry-safe. No cross-service
two-phase commit — compensation is cheap here precisely because of deep dive 3's expiring
reservations" ([04-deep-dives.md](39.e-commerce/04-deep-dives.md)). Two boundary decisions
stand out. First, the **idempotency anchor is written first**: "the order row is written
first with state `PENDING`, before payment and before reservation commit — it is the
idempotency anchor: every retry sees the row and resumes or replays its result" — which
converts a crash mid-checkout from an unknown outcome into a resumable one. Second, the
**worst case is bounded by design**: "payment is only authorized, never captured
pre-fulfillment (01's N1 'fail closed'), so a stuck saga's worst case is a voided hold".
That is the same doctrine as 06: "a recorded order without captured money is recoverable;
captured money without an order is not" (a paraphrase of 39's failure-mode note, cross-cited to 06). Arithmetic
for the sweep cost: TTL sweeps for reservations and orphaned promo redemptions are "the
same job, different table" — O(pending orders) rows, trivially small against ~15B
events/day on the read models.

**[10.distributed-cache](10.distributed-cache/README.md)** is the purest scope line:
"No global lock manager, no distributed transactions, no cache-wide 'consistency
oracle': the tier's consistency story is per-class TTL + invalidation discipline +
replication mode, not a protocol" ([03-architecture.md](10.distributed-cache/03-architecture.md)).
[01-requirements.md](10.distributed-cache/01-requirements.md) gives the escape hatch:
multi-key atomicity is limited to keys sharing a hash tag — collapsing the boundary into
one shard instead of paying a protocol. **[12.bigtable](12.bigtable/README.md)** makes
the identical move at the table's design layer: "One tablet owns a row's data; rows are
the unit of atomicity — no row spans tablets, so per-row atomic read-modify-append is
free (single machine). Losing: no cross-row" transactions. Both state the trade-off as a
requirement boundary, not an apology.

### The collapse move: shrink the transaction until it is local

Before any saga, the cases try to *abolish* the boundary by giving one store one
uncontended writer over everything that must commit together:

- **[36.airbnb](36.airbnb/README.md)** — hold critical section for the nights: re-check
  and mark inside one hash-partitioned writer per listing, PSP authorize *inside* the
  hold, then commit. Option B (conditional row checks from any instance) loses because of
  the PSP call: "holding a PSP call across a row lock (lock hold time balloons to the PSP
  round trip, or you re-check after the PSP and open a race)". The chosen shape: 01
  README — "Two-phase reservation (hold → confirm) with payment authorization inside the
  hold". The hold's TTL "buys time for the money, not convenience" (~10–15 min for
  request-to-book, seconds for instant book) — bearing the authorization in a
  timer-resolved state rather than locking anything at the PSP.
- **[28.uber](28.uber/README.md)** — "a second trip cannot reserve the same driver
  because the CAS fails. This CAS lives in the geo index shard that owns the driver's
  cell, which is also single-writer by cell partition — no distributed transaction
  needed" ([04-deep-dives.md](28.uber/04-deep-dives.md)). Driver timeout
  (`offered-to → available`) "is a timer, not a message, because the failure mode (silent
  driver) is the common one" — same shape as 29's wedged-order sweep.
- The catalog frames this choice as a *preference* in the store docs too: 39's
  transactional plane is deliberately small ("the *only* link from the transactional
  plane to every read model" is the event spine), and 10's cross-shard escape hatch is a
  hash tag — both are the same shrink.

### The saga family, and what compensation costs

Four cases run sagas, and they differ in the price of the compensating action — the
catalog's implicit rule is **order the steps so the compensations get cheaper, and
authorize, never capture, first**:

- **44** — over uncooperative rails: void-before-capture is free, refund-after-capture
  costs scheme fees; therefore "capture late".
- **36** — the two failure worlds are enumerated: payment captured and confirm failed →
  "compensating full refund via the original charge id"; authorized and hold commit
  failed → "explicit void of the authorization"; and the third world — PSP timeout with
  unknown outcome — "treated as *in-flight*, resolved by reconciliation of PSP statements
  against ledger intents". Note the progression: each case keeps intents as truth and
  demotes external responses to "facts" (44: PSP responses are facts; 36: same).
- **29.uber-eats** — the compensation is a *refund of state, not of money intent*:
  "reverses via a compensating ledger row — never edits history. Same ledger discipline
  as [06.apple-app-store](06.apple-app-store/README.md)"
  ([04-deep-dives.md](29.uber-eats/04-deep-dives.md)). Its customer-cancels branch is a
  business rule ("customer refunded, merchant compensated — an explicit business rule,
  implemented as its own transitions, not as an exception"). Its wedged-order policy is a
  pure trade-off pair: Option A (always compensate the courier) is exploitable, Option B
  (never) sours supply, chosen bounded compensation — compensation here is product
  design, not just consistency machinery.
- **39** — the fullest saga; keyed compensations make the saga itself idempotent, and the
  TTL sweep closes the "saga crashed mid-compensation" hole.

The outbox — publish and persist in one local transaction — is the saga's
event-emission half. **[04.deployment-system](04.deployment-system/README.md)** is the
only case that designs it server-side: "Writes happen transactionally with an **outbox
row** consumed by a relay into the event bus — status and its event are published
atomically, so consumers see a faithful stream without making the bus authoritative"
([03-architecture.md](04.deployment-system/03-architecture.md)). [27.ev-charging-network](27.ev-charging-network/README.md)
names it: "the same at-least-once → exactly-once contract as the outbox pattern in
[04.deployment-system](04.deployment-system/04-deep-dives.md)". The pattern costs a
relay and dedup on the read side ("every consumer dedups (run id, transition counter)"),
and buys freedom from the dual-write bug — the counter-example that never appears
explicitly in a case file, only implicitly in 04's design. Dual-write does appear
elsewhere, deliberately and in a different role: [13.kafka](13.kafka/README.md) uses it
for partition-count migration (old and new partitions both written while the consumer
group catches up) — a data-plane expand-contract covered by
[DEPLOYMENT-SAFETY.md](DEPLOYMENT-SAFETY.md), not a consistency mechanism.

### The exactly-once ledger: at-least-once inputs, once effects

The catalog's money-shaped cases all make the same decomposition, and 06 states it
cleanest: "The wire can't be exactly-once; the system must make the *user-visible
outcome* exactly-once" ([04-deep-dives.md](06.apple-app-store/04-deep-dives.md)).
Mechanics converge to a triple:

1. **A client-supplied key with a uniqueness constraint** (the write-side dedup —
   [CONCEPTS.md](CONCEPTS.md) owns the union; 44's "the key is proof of intent, not a
   guess" is the difference from fingerprint dedup, which wrongly suppresses two same
   gumballs and misses a changed amount).
2. **A deterministic engine or an idempotent write** so replay produces the same result:
   27's tariff engine is "a deterministic pure function `f(raw event history for session)
   → CDR`; the ledger takes CDRs keyed by `(session ID, ledger shard)` with
   insert-if-absent semantics. Re-running the engine on the raw store ... produces
   byte-identical CDRs but cannot produce a second ledger entry." [18.sora](18.sora/README.md)
   and [16.dalle](16.dalle/README.md) get the same property in the compute domain:
   "This is at-least-once execution whose artifacts" overwrite the same key; uploads
   "keyed by (job ID, image [index])".
3. **A reconciliation job as the last net** — 44's settlement files, 37's hourly
   "money-in vs orders vs seat transitions vs holds-timed-out", 09.cdn's daily
   log-vs-PSP match, 06's daily ledger-vs-PSP match. Not one money case trusts the happy
   path alone, and not one asks the ledger for a lock.

Two variants are priced explicitly. [37.ticketmaster](37.ticketmaster/README.md)
compares synchronous capture (Option A: loses "the retry surface is exactly where a
naive implementation double-charges") against two-phase auth-then-capture where the
hold_id is the idempotency key end-to-end and the invariant is "one (hold_id, account,
event) → at most one PAID order → at most one capture → at most one issued ticket set".
The stated cost: "a capture can fail after the user *believes* the purchase completed ...
a real, ugly edge case that must be explicit UX, not swallowed"
([04-deep-dives.md](37.ticketmaster/04-deep-dives.md)).
[35.tiktok](35.tiktok/README.md) prices the other polarity at ~100M events/s peak:
end-to-end exactly-once "loses throughput and adds coordination cost at exactly the tier
where ... no headroom, and exactly-once at the *client* boundary is unsolvable anyway
(phones retry POSTs)" — so at-least-once + event UUIDs + consumer dedup, with the double-
counted-watch-signal skew accepted as an explicit cost.

**[09.cdn](09.cdn/README.md)** earns exactly-once billing on a firehose without
per-request dedup at all: "Exactly-once billing is enforced by per-POP sequence numbers,
not per-request dedup" ([03-architecture.md](09.cdn/03-architecture.md)). The arithmetic
justifies it: at 09's scale (~2×10^12 req/day) a per-request dedup table holds ~10^12
entries/day of keys for exactly-once that a per-POP monotone counter gets with O(#POPs)
state — thousands of sequence numbers instead of a trillion keys; gaps in a POP's
sequence detect loss, duplicates are dropped by the consumer, and daily reconciliation
corrects the residue. Same trick family as 50's "fills are exactly-once by
construction; the order book is a pure function of the inbound sequenced stream"
([01-requirements.md](50.stock-exchange/01-requirements.md)) — sequence the inputs, make
the reducer deterministic, and exactly-once stops being a delivery problem.

### Boundary rows

- **[07.blob-store](07.blob-store/README.md)** — the only two-phase protocol in the
  corpus that actually spans writes to two planes ("two-phase PUT: bytes-then-index,
  plus orphan garbage collection"), and it is *inside one system*: data durable before
  index commit "converts all crashes into invisible orphans instead of broken users".
  One-sided by construction; no coordinator exists, and none is needed. Owned at depth by
  [OBJECT-STORAGE.md](OBJECT-STORAGE.md).
- **[13.kafka](13.kafka/README.md)** vs **[50.stock-exchange](50.stock-exchange/README.md)** —
  13 declares exactly-once transactions ("spanning produce + consume + processing") out
  of scope for its case; 50 rejects the log's replication for a different boundary
  reason (per-message quorum commits vs a ~20 µs budget). Replication-commit is
  [CONSENSUS-AND-QUORUM.md](CONSENSUS-AND-QUORUM.md); log-as-truth is [KAFKA.md](KAFKA.md);
  what is left here is only that *no case asks the log to co-commit* — the bus stays
  "for fanout, not authority" (04's own words in 02-scale-estimates).
- **[21.frontend-calendar](21.frontend-calendar/README.md)**, 19, 24, 25 — the client
  half runs the same contract mirrored: outbox-first writes, client op IDs, reconnect as
  one reconciliation point. [SYNC-AND-CRDT.md](SYNC-AND-CRDT.md) owns it; a link
  suffices.
- **[11.zookeeper](11.zookeeper/README.md)** and **[49.monitoring-metrics](49.monitoring-metrics/README.md)**
  appear in grep only as negations ("watches are one-shot and can miss events";
  "Exactly-once ... one sample per series per scrape" listed as a requirement not met) —
  boundary noise, not mechanism.

## The portable pattern

Every multi-part write in the corpus reduces to the same ordered procedure:

1. **Shrink it first.** Find the store that can own the whole atomic core in one shard
   or one row (28's cell CAS, 36's per-listing writer, 39's order row, 10's hash tag,
   12's row-granularity doctrine), and accept "no cross-X atomicity" as an explicit
   requirement boundary when it cannot. This is the case against skipping: every case
   that names 2PC rejects it, and *every* one of them rejects it before falling back to
   saga — none uses 2PC as a fallback among options.
2. **Persist the idempotency anchor before any side effect.** The order row
   `PENDING` (39), the hold record "durable before any PSP call" (36), the key in a
   transaction table (06). This is what makes a crash resumable rather than dual-write.
3. **Write store + event in one transaction: the outbox.** One designed instance (04),
   cited as the pattern by 27; the alternative (fire-then-store) appears only where the
   event is droppable telemetry (40, 47).
4. **Saga with keyed compensations, ordered cheap-compensation-last.** Never capture
   before you can void (44, 39, 37); key every compensating action with the same
   idempotency key so compensation replay is safe (39).
5. **Effectively-once = at-least-once + idempotent receiver + deterministic reducer.**
   03 (client dedup), 06/44 (keyed ledger), 27 (insert-if-absent + pure function),
   09/50 (sequence numbers + deterministic reducer). "Effective" is the honest word —
   37: "Exactly-once is a lie."
6. **Reconciliation is the correctness backstop, not an ops artifact.** Every money case
   runs one (44 daily files, 37 hourly cross-checks, 29's ledger-vs-verification, 06's
   daily ledger-vs-PSP). Disagreement is "a first-class product state, not an error log".
7. **Where the pattern is the wrong tool:** when the other endpoint cannot hold state
   for you (banks refuse the lock, so 44's saga is protocol, not preference), when the
   event rate leaves no headroom for coordination (35), and when the atomicity must span
   planes inside one system — there the pattern is a *one-sided ordering* (07's PUT) or
   a shrunk scope (10, 12), never distributed commit.
