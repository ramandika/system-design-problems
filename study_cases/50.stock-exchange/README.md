# Case 50 — Stock Exchange Matching Engine

**The system**: the core trading system of an exchange — the component that takes orders
from member firms (brokers, market makers), matches buy against sell under price-time
priority, publishes the resulting executions and the evolving order book to the market,
and hands the trade record downstream to clearing. Orders flow through a gateway that
stamps them with a sequence number; the matching engine consumes the sequenced stream,
maintains one order book per symbol, and emits an outbound event stream (execution
reports, market data) of its own. The whole thing is a *deterministic state machine
wrapped around a log* — the same skeleton as [Case 13 — Kafka](../13.kafka/README.md),
but with a fairness contract and microsecond latency budgets where Kafka has throughput.

**Why it's hard**:

- **Correctness over availability, always**: a wrong fill is a regulatory incident and a
  market integrity incident, not a support ticket. When in doubt the exchange halts
  trading in a symbol — *stopping* is a designed, accepted outcome. This inverts the
  usual distributed-systems instinct (cf. [44.payment-system](../44.payment-system/README.md),
  which at least must keep the revenue path up).
- **The matching contract is strict serializability at heart**: every order occupies
  exactly one position in one total order per symbol, and price-time priority means that
  position, not the arrival wall-clock, decides priority. Any concurrency that lets two
  orders share or race for "arrival position" is a fairness bug by definition.
- **Microsecond budgets**: the matching engine turns a message around in tens of
  microseconds (several venues publicly report matching in that range); every layer —
  protocol, threading, memory layout, kernel — either serves that or is banned from the
  path. Unlike every other case in this corpus, cache misses and locks are counted.
- **Determinism is a requirement, not a property**: the state must be rebuildable by
  replaying the event log to the byte. A nondeterministic scheduler, hash map iteration,
  or thread race in the matcher breaks recovery, breaks the audit, and breaks the
  dispute answer ("which order was behind mine at the moment of the match?").
- **The hot symbol problem**: scale per-symbol is not uniform — one symbol can burst to
  a hundred times its average on a news event, while thousands of symbols sit idle.
  Sharding buys parallelism across symbols but does nothing for the single hottest one.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: the log as the source of truth and determinism as the
recovery strategy (deep dives §3 — sibling of Kafka's partition log,
[13.kafka/04-deep-dives.md](../13.kafka/04-deep-dives.md)); single-writer state machines
with no shared mutable state; idempotent client-supplied order IDs
([44.payment-system/04-deep-dives.md](../44.payment-system/04-deep-dives.md) §1, applied
where the retry budget is µs, not seconds); data structure choice driven by cache
locality (deep dives §1 — the flip side of Kafka's sequential-disk insight); fairness
under contention from a *position in a stream*, not timestamps
([37.ticketmaster/04-deep-dives.md](../37.ticketmaster/04-deep-dives.md)); fanout by
multicast with a replay tier (cousin of
[02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md)'s fanout, but with
subscribers who litigate against dropped quotes).

**Try it first**: before reading 03, answer: (1) two buy orders at the same price arrive
at the same instant on two different gateway connections — what decides which is "first",
and what single mechanism makes that decision reproducible after the fact? (2) the
matching engine crashes mid-second — how does the standby learn the exact book state,
and what would make its replay produce a *different* book? (3) market data subscribers
suddenly need 10× the quote rate during a burst — what can be throttled and what cannot?
