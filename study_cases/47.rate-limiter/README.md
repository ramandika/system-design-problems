# Case 47 — Distributed rate limiter

**The system**: every API request asks a question — "is this caller (API key / user / IP)
allowed to do this thing, right now?" — and demands an answer in ~1 ms. The limiter is a
*stateful decision service* bolted onto the front of a multi-tenant API platform
(Stripe/GitHub/Cloudflare-class): it counts usage across a fleet of stateless gateway
nodes, against limits that change on short notice, and it must decide at a QPS approaching
the platform's own — because almost every request needs at least one check.

**Why it's hard (all that there is)**:

- **It is a consistency problem wearing a latency costume.** A counter shared across
  gateway nodes is either consulted synchronously (you pay a network round trip on the
  hottest path in the platform) or cached locally (you enforce slightly stale limits —
  nodes over-admit between syncs). Every design point lives on this dial.
- **Read-modify-write under concurrency.** "Count the request, compare to the limit, admit
  or reject" is three steps that race unless they are one atomic step. The naive version
  under-admits, over-admits, or — worst — leaves a counter wedged so a customer is
  permanently locked out.
- **The state check QPS dwarfs the API QPS it guards** (multi-dimensional keys, plus every
  rejected request still costs a check), and the key distribution is brutally skewed: a
  handful of customers account for most of the traffic, so the counter store inherits the
  hot-key problem at its worst.
- **Failure posture is a product decision.** When the counter store is down, "allow
  everything" and "reject everything" are both defensible — for different limit classes —
  and the wrong default is both a security hole and an outage amplifier.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: choosing an enforcement algorithm by burst behavior (token
bucket vs sliding window vs fixed window), quota leases / local enforcement with bounded
drift over a centralized authority, atomic decision scripts for read-modify-write,
chunked counters for hot keys, tiered fail-open/fail-closed defaults, and versioned
config distribution to a fleet.

**Siblings**: the counter store is a hot-key-infected KV cluster — deep dive 5 of
[10.distributed-cache](../10.distributed-cache/04-deep-dives.md) is the general theory;
the crawler faces the same counters from the *consumer* side (politeness per-host limits,
[41.web-crawler](../41.web-crawler/04-deep-dives.md) dive 1); the hot-key and
top-k-skew math mirrors [45.typeahead](../45.typeahead/04-deep-dives.md) dive 3;
config broadcast to a fleet uses the coordinator/sequencing ideas of
[11.zookeeper](../11.zookeeper/04-deep-dives.md); the decision telemetry pipeline is a
[13.kafka](../13.kafka/README.md) fire-hose, same shape as tinyurl's click stream
([40.tinyurl](../40.tinyurl/03-architecture.md)).

**Try it first**: before reading 03, answer: (1) why can't every check just hit a shared
Redis — do the latency arithmetic for an edge PoP; (2) a customer is allowed 1,000
requests/s and your limit is enforced on 100 gateway nodes — how over-admitted can you be
between syncs; (3) the counter store just died — write down which of your limit classes
fail open, and which one must not.
