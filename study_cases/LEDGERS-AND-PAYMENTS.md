# Ledgers & payment plumbing across the case catalog

Where money — double-entry ledgers, escrow/authorization holds, PSP integrations, settlement
reconciliation — appears in the 50 study cases, what role it plays, and how deep each case goes.
Companion to [CONCEPTS.md](CONCEPTS.md) (idempotency-keys union, Tier 2, 23/50 cases — but the
money-specific machinery around that union has no union of its own).

Headline: **every case that touches money converges on the same three-part shape.** An
*entries-not-states* ledger is the source of truth; one idempotency-key family threads through
every downstream call including the PSP; and the auth→capture→settlement sequence is treated as a
saga across rails you do not control, reconciled — never trusted. Only case 44 studies a payment
system itself; the other six discover the same design independently, which is the strongest
signal in the catalog that this shape is canonical.

- Cases with a designed ledger or money-movement component: **7 / 50** (06, 27, 29, 36, 37, 39, 44)
- The case itself: **1** (44); full payment plumbing inside another system: **6**
- Same pattern named but skipped (purchase = one PSP call, no ledger design): other product
  cases delegate payment to "a PSP" — a named dependency, not architecture.

## Summary table

| Case | Ledger role | Payment state machine | Reconciliation |
|---|---|---|---|
| [44.payment-system](44.payment-system/README.md) | The case itself: immutable double-entry postings, balances are queries | `created → authorized → (captured | voided | expired) → settling → settled` | Daily acquirer settlement-file ingest + breaks queue |
| [06.apple-app-store](06.apple-app-store/README.md) | Single-writer sharded ledger, append-only transitions | `AUTHORIZED → CAPTURED → (REFUNDED | SUBSCRIBED_RENEWED)` | Daily reconciliation against PSP records flags missing captures |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Double-entry billing ledger: CDRs, captures, refunds, partner settlements, VAT as entries | Deterministic tariff re-run from raw events | Inbound roam-CDR offsets (`Credit CDR`) in the same ledger |
| [29.uber-eats](29.uber-eats/README.md) | Settlement ledger derived from order events; payouts settle T+1–N | Charge authorized synchronously at checkout | Refunds reference ledger rows, never rewrite them |
| [36.airbnb](36.airbnb/README.md) | Ledger rows under the availability grid (grid reconstructible from them) | Two-phase with external PSP; authorize vs capture split | PSP timeout = *in-flight*, resolved by reconciliation |
| [37.ticketmaster](37.ticketmaster/README.md) | Immutable seat/order events with payment steps in the same flow | Authorize → capture-after-authorize, holds with TTL | Auth-hold expiry sweeper; webhooks reconciled |
| [39.e-commerce](39.e-commerce/README.md) | Explicitly contrasted with its own reservation system | Authorize (never capture) at checkout; capture on fulfillment | Saga compensations keyed by the same idempotency key |

## Per-case notes

### The case itself

**[44.payment-system](44.payment-system/README.md)** — [04-deep-dives.md](44.payment-system/04-deep-dives.md)
picks **Option B — immutable journal of double-entry postings + derived balances** over mutable
balance rows: replay reconstructs state, corrections are *new reversing postings*, per-account
sharding with a hot-account row-lock (money *pauses* on a partition, it does not go missing).
The auth→capture→settlement chapter is framed as "a saga across rails you don't control": the
state machine crosses acquirer boundaries, gateway adapters reuse the same bank-side uniqueness
reference (adapter-level idempotency), and concurrency inside one payment's lifecycle resolves
through CAS on the postings. Two further designed blocks: the **tokenization vault** (PCI-DSS
scope isolation — token↔PAN mapping held by a service with no bulk read path) and **settlement
reconciliation** — daily acquirer files matched by scheme reference + amount into a breaks queue
with classification and SLAs. Cross-referenced to [27.ev-charging-network](27.ev-charging-network/README.md)
for the CDR variant of the same reconciliation.

### Full payment plumbing inside another system

- **[06.apple-app-store](06.apple-app-store/README.md)** — *Single-writer sharded ledger* sharded by
  account_id; the ledger row is written once and transitions are append-only entries; balances and
  subscription state are projections (family sharing, restore). The PSP is called with the same
  idempotency key that keys the ledger, and a second net exists either way: daily reconciliation
  of charges against PSP records flags missing captures. The signature move is the **receipt**:
  the ledger is server truth, the receipt is a chain-signed device-held document that can prove
  entitlement *offline*, with a `REVOKED` ledger entry driving refresh — ledger and receipt answer
  different questions and are designed separately.
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — the billing ledger is double-entry,
  append-only, and deliberately *deterministically re-runnable*: re-running the tariff engine from
  the raw event store produces no second entry because CDRs are inserted keyed-if-absent. Partner
  (roaming) settlement flows through the same ledger in the other direction — inbound CDRs offset by
  Credit CDRs — so there is exactly one place where "what do we owe partner X" is answerable.
- **[29.uber-eats](29.uber-eats/README.md)** — a settlement ledger consumes order events and writes
  immutable rows (charge, merchant cut, courier payment); payouts settle in T+1–N batches and
  support refunds reference ledger rows, never rewrite them. Geography partitions everything in the
  case *except* the ledger — a global-authoritative, market-partitioned money table, the same
  shape 44 chose.
- **[36.airbnb](36.airbnb/README.md)** — booking is a two-phase transaction with an external system
  (the PSP) on the critical path; the case refuses to hold a row lock across the PSP round trip.
  Chosen shape: reservation row goes `on-hold` with a TTL'd authorization hold, PSP authorize,
  then commit. The saga enumerates all failure worlds explicitly — payment captured but confirm
  failed → compensating full refund; confirm won, authorization orphaned → explicit void; PSP
  timeout with unknown outcome → *in-flight*, resolved by reconciliation, not guessed.
- **[37.ticketmaster](37.ticketmaster/README.md)** — the sharpest statement of the capture problem:
  *authorized money is a promise; capture must succeed exactly once.* The capture call may time
  out, and retry is only safe because the idempotency key makes the rails' duplicate-capture a
  no-op — the same key family keys order creation, authorization, capture, and ticket issuance.
  Authorization holds carry TTLs with an expiry sweeper, mirroring 39's reservation doctrine.
- **[39.e-commerce](39.e-commerce/README.md)** — states the boundary the other cases imply:
  *a recorded order without captured money is recoverable; captured money without an order is a
  liability*. Checkout authorizes and never captures before fulfillment; the PSP adapter
  propagates the same idempotency key. The case also contrasts itself with the money ledger
  (06, deep dive 4): purchase reservations are *expiring* TTL records, ledger entries are
  irreversibly append-only — the two look alike and must not be conflated, exactly the
  ledger/receipt split of 06 at a different altitude.

## The portable pattern

1. **Entries, not states.** Append-only postings are truth; balances, subscription state,
   availability grids and payouts are derived views ([44](44.payment-system/README.md), 06, 27, 29, 36).
2. **One idempotency-key family, end to end.** The same client-generated key keys the ledger
   write, the PSP call, the saga compensation, and the reservation — so retries anywhere in the
   chain are safe ([37](37.ticketmaster/README.md), 06, 39, 44). This is the concrete instance of
   the [CONCEPTS.md](CONCEPTS.md) idempotency union; see also [KAFKA.md](KAFKA.md) for the same
   key discipline on the event plane.
3. **Authorize is not capture.** Capture is the dangerous step and happens late, after the
   business transaction is committed (fulfillment, seat commit, delivery) — never before
   (39, 36, 37, 44).
4. **External rails are a saga, not a transaction.** Compensating entries instead of edits;
   unknown outcomes go to *in-flight* and are resolved by reconciliation, not by choosing to
   trust either side (36, 44, 27).
5. **Reconciliation is the backstop, not the primary path.** Daily files from acquirers/PSPs
   matched against postings into a breaks queue; its existence is the honest answer to "what if
   our ledger is wrong" (44, 06, 27).
6. **Proof-offline artifacts are derived separately.** Receipts (06) are signed, device-held
   projections of ledger truth with their own revocation story — never a live query.
