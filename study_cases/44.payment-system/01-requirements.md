# 01 — Requirements

## Functional

Core v1 (a merchant-side API business; consumers of the API are merchants' systems):

- **Charges / payment intents**: `POST /v1/charges` (or intents) — authorize a card for an
  amount in a currency, capture later or immediately; every request carries a client-supplied
  **idempotency key**; replays return the original result.
- **Lifecycle operations**: capture, void (before capture), refund (full/partial, possibly
  multiple), disputes (chargeback received, evidence submitted, outcome).
- **Customers & tokens**: store a payment method *without* storing PANs — tokenization; a
  token represents the card everywhere downstream.
- **Payouts**: sweep collected balance to the merchant's bank account on a schedule; payout
  rows must trace to the movements that funded them.
- **Webhooks**: signed event deliveries (`charge.succeeded`, `charge.refunded`,
  `payout.paid`, …) to merchant endpoints, retried until acknowledged (delivery guarantees:
  → [03.whatsapp](../03.whatsapp/README.md)'s taxonomy; at-least-once + signature + replay
  window).
- **Dashboard/API reads**: balances, transaction lists, exports.
- **Declines and their taxonomy**: issuer declines with codes (insufficient funds, fraud,
  do-not-honor), soft vs hard — a decline is a full-cost successful API call, not an error.

Out of scope v1: issuing (create cards), subscription billing logic, currency FX *trading*
(pass-through FX as ledger postings), 3DS protocol internals (step-up flows referenced only),
ACH/SEPA rails besides cards beyond "another rail, same state machine".

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Auth latency | platform-internal p99 ≤ ~500 ms; end-to-end incl. issuer seconds | risk check budget ~150 ms inside the internal envelope (deep dives §5); the bank's RTT is not yours to fix |
| Duplicate-charge probability | zero by design (not "rare") | idempotency keys enforced at the write path, not best-effort dedup (deep dives §1) |
| Availability | auth is the revenue path; **fail conservative** — an outage must never be resolved by accepting twice | auth degrades to *rejecting*, never to double-processing |
| Auditability | every money movement reconstructs from the ledger alone, indefinitely | immutable journal, 7-year retention, no destructive edits; corrections are new postings |
| Consistency | balance reads: read-own-writes for the caller, eventually consistent for aggregates | derived rollups + reconciliation backstop (deep dives §2, §4) |
| Data protection | PANs never touch platform servers or logs unencapsulated | PCI scope reduction by tokenization (deep dives §6) |
| Scale | ~$1.4T/yr TPV design point; ~150M API write requests/day | load is small; stakes are absolute — see 02-scale-estimates |

## Scope decisions that shape everything

1. **The API is an intent machine, not a database of charges**: `authorized`/`captured`/
   `settled`/`refunded`/`disputed` are states of money's journey across *other* systems'
   rails; our state machine (deep dives §3) must match that reality, not idealize it.
2. **Write path is the product**: reads (dashboard) dwarf nothing important here; the design
   effort concentrates on the ~150M/day idempotent, stateful, externally-coupled writes.
3. **Reconciliation is a subsystem with an SLA**, not a monthly report: settlement files are
   the arbiter of truth once banks have spoken (deep dives §4).
