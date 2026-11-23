# Case 44 — Payment System

**The system**: a payments platform for internet businesses (Stripe-shaped) — an API that
charges a card, holds money between authorization and settlement, pays it out, and answers
for every cent, forever. The QPS is tiny by the standards of
[Case 40 — TinyURL](../40.tinyurl/README.md); the correctness bar is unmatched by anything
else in this corpus.

**Why it's hard**:

- **Money must be exactly-once**: a duplicate charge is a legal incident and a trust
  incident, and the network *will* deliver your auth requests twice (retries, timeouts,
  provider replays). The idempotency design (deep dives §1) is the heart of the case.
- **Balances cannot be a column you UPDATE**: the ledger must be an immutable double-entry
  journal from which every balance and every audit answer reconstructs (deep dives §2).
- **Your system is not the only authority**: banks and card networks operate on their own
  batch rails, their own timelines, their own outages — auth→capture→settlement is a saga
  with compensations, never a 2PC (deep dives §3).
- **The outside world's truth wins**: acquirer settlement files tell you what actually moved;
  reconciliation is a first-class subsystem, not a report (deep dives §4).
- **Fraud decisions happen inline** in ~150 ms, and the biggest cost is usually *false
  declines*, not missed fraud (deep dives §5).

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: idempotency keys + serialization for exactly-once money effects
(sibling of [27.ev-charging-network](../27.ev-charging-network/README.md) CDR billing and
[36.airbnb](../36.airbnb/README.md) hold→confirm), double-entry ledger as source of truth,
external-system sagas with void/refund compensations, reconciliation as the correctness
backstop, tokenization to shrink PCI scope.

**Try it first**: before reading 03, answer: (1) the client retries a charge 3× with the same
idempotency key — how many charges can *ever* hit the bank? (2) capture succeeds but the
response is lost before your DB commits — what does your system do on the next retry?
(3) the acquirer's settlement file disagrees with your ledger on 400 payments — who is right?
