# 01 — Requirements

## Functional

Core v1 (continuous double-auction equities matching; consumers are member
firms and market-data subscribers):

- **One order book per symbol**, continuously open (with scheduled sessions), matching
  the best bid against the best ask the instant they cross.
- **Order types**: limit (price + quantity, the only type that rests), market (execute
  at the prevailing price, never rests), IOC (immediate-or-cancel — execute what it can,
  cancel the remainder instantly), FOK (fill-or-kill — execute the full quantity or
  reject the entire order). Day orders, fill-or-kill semantics and partial fills follow;
  exotic types (pegged, reserve/iceberg display quantities, sweep-to-fill) are
  acknowledged as real but out of the v1 core.
- **Price-time priority as the matching contract**: within a price level, earliest
  resting order first; across levels, best price first. Every fill and every queue
  position must be reconstructible and explainable after the fact.
- **Order lifecycle operations**: submit (new), cancel by order ID, mass-cancel per
  account/symbol (venue-required risk valve), cancel/replace. Every terminal order
  reaches an execution state or a cancel state — nothing is dropped silently.
- **Execution reports back to the placing firm**: ack, fill (with price, quantity, own
  order ID, counterparty anonymous), partial fill, reject (with reason), cancel
  confirm/reject. Drop-copy of all of the member's activity to their compliance systems.
- **Market data broadcast**: top-of-book (BBO) and full-depth quotes, trades (price,
  quantity, time, trade condition), sequence-numbered; snapshot + incremental model so
  a subscriber can join mid-stream and recover (deep dives §5).
- **Auction handling at open/close**: simple v1 form — an indicative-price phase
  followed by an auction match maximizing matched volume at a single opening price
  (design as an extension of the continuous book, same structures).
- **Post-trade feed to clearing**: after each execution, a trade record (buy side, sell
  side, price, qty, time, trade ID) flows to the clearing link. The exchange *stops
  there* — it does not net, margin, or settle.
- **Trading halts**: circuit-breaker-triggered or manual halt on a symbol (or the
  market); when halted, new matching stops, the state is frozen, and resumption
  re-opens with the same book. Halt is a first-class feature, not an exception path.

Out of scope v1: clearing/settlement internals (netting, CCP margin — the exchange hands
off a clean, complete trade record; see [44.payment-system](../44.payment-system/README.md)
for the post-trade money side), surveillance/regulatory reporting systems (they *consume*
our event log), options multi-leg matching, cross-venue order routing (this exchange is
one venue; smart routing is the client's problem), lending/borrow and short-sale
locates, crypto-style off-hours trading.

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Matching latency | gateway-in → execution-out p50 ≤ ~50 µs, p99 ≤ ~1 ms (several venues publicly report matching steps in the tens-of-µs range) | no locks on the hot path, no heap allocation on the hot path, no network hops of convenience (deep dives §2, §4) |
| Fairness | an order's position is decided *only* by its sequence number in the inbound stream, and that stream is one total order per symbol | no wall-clock timestamps as priority; no re-ordering between gateways; determinism makes it auditable (deep dives §3, §4) |
| Throughput | sustained 20K+ msgs/s aggregate inbound; bursts ≥ 500K msgs/s during volatility (est., see 02) | sized per-symbol for its worst day, not its average day |
| Correctness | fills are exactly-once by construction; the order book is a pure function of the inbound sequenced stream | replayability is a *test*: the engine's output from a replayed journal must be byte-identical to the original (deep dives §3) |
| Availability | continuous-trading sessions with failover so fast the session is rarely "down"; BUT the failure policy is stop-not-corrupt | on any condition that risks wrong state, halt the symbol, not the correctness (deep dives §6) |
| Auditability | every state transition is in the journal, retained for years (regulators require long books-and-records retention) | append-only log, no destructive edits — same discipline as the ledger in [44.payment-system](../44.payment-system/01-requirements.md) |
| Recovery | hot/warm standby with RPO ≈ 0 to ~µs of events; failover into a session mid-trade | recovery = replay journal, and replay MUST be deterministic (deep dives §6) |

## Scope decisions that shape everything

1. **The exchange never becomes unavailable into ambiguity about what it accepted**:
   rejecting an order is a full-fidelity, journaled outcome; losing sight of whether an
   order was accepted is not. This is the payment-system rule
   ([44.payment-system/01-requirements.md](../44.payment-system/01-requirements.md))
   under a stricter clock: "fail conservative" here means *halt*, not *degrade*.
2. **Sequence number is the physical unit of fairness**: all ordering, all recovery,
   all market data stitching hang off sequence numbers stamped centrally before the
   matcher sees anything (deep dives §3). Wall-clock time is metadata for regulators,
   never an input to matching decisions.
3. **The book is in-memory and rebuilt, never persisted as state**: the persistence
   format is the event log only. There is no "book snapshot DB" that can silently
   disagree with the log.
4. **Throughput is bought per-symbol by design, not per-cluster by load balancing**:
   the single hottest symbol during a burst is the real capacity target (02-scale-estimates
   and deep dives §2), and a shard boundary at the symbol is the only concurrency
   the fairness contract tolerates.
