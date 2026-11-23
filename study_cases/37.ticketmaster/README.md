# Case 37 — Event ticketing (Ticketmaster-shaped on-sale system)

**The system**: primary-market event ticketing. Most of the year it is an ordinary
e-commerce catalog — browse 50,000 events, buy 4 seats, no drama. Then one morning at
10:00 a stadium show goes on sale and several million people arrive in the same few
minutes for 20,000 physically finite seats. The system is two products wearing one URL:
a quiet catalog that must never be down, and a burst machine that must convert a one-shot spike
(~2M arrivals in a 15-minute window, front-loaded into the first minutes — see
[02-scale-estimates](02-scale-estimates.md)) into finished purchases without selling a seat twice, selling a seat
that doesn't exist, or letting the strongest bot fleet buy everything.

**Why it's hard**:

- **Demand vastly exceeds supply by design.** Unlike a web app that scales capacity to
  meet demand, ticketing *cannot* scale seats. The correct behavior under ~300×
  over-subscription is not "serve everyone" — it is structured rejection: a waiting room,
  a queue, and a bounded active-sale population. Most of the engineering is about
  *failing fairly and cheaply* for the millions who will not get tickets.
- **The unit of inventory is a row in a seat map, and neighbors are load-bearing.** Users
  want contiguous groups ("2 or 4 together, not alone"). Selling seats one at a time
  fragments the map so badly that later buyers find only singles left even at half
  capacity. Inventory design is a spatial-fragmentation problem, not a counter problem.
- **Every seat is a hot key during on-sale.** Sharding by user, by geo, by anything
  doesn't help — all demand converges on one event's inventory, often on a handful of
  price tiers within it. This is the extreme version of the hot-partition problem.
- **Holds are the product.** Between "I want that seat" and "my card cleared" there is a
  multi-minute checkout carrying a payment. The seat must be invisible to everyone else
  during that window (TTL hold), automatically returned if the buyer walks (reaping),
  and commit atomically with the payment (exactly-once purchase).
- **The adversary is automated.** A meaningful fraction of on-sale traffic is bots with
  home-farm IPs, pre-warmed accounts, and headless browsers. Any fairness mechanism
  (queue position, purchase caps) is only as good as its bot defense, and every defense
  taxes legitimate fans too.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**What this case teaches that the other cases don't**: Twitter (see [02.twitter](../02.twitter/04-deep-dives.md))
scales fanout and absorbs spikes by degrading freshness; WhatsApp scales sockets; Kafka
scales ordered throughput. Ticketing teaches **scarcity engineering**: demand-side
admission control (a queue that is a product feature, not an error page), inventory
correctness under contention (TTL holds, compare-and-swap on seats), and a money
pipeline that must reconcile a reservation against a payment under retries. It is also
the clearest case here of the **single-hot-shard problem**: sharding solves nothing when
all keyspace demand lands on one event.

**Cross-references**: locks and fencing for seat holds build on the same machinery as
[11.zookeeper](../11.zookeeper/04-deep-dives.md) (lease semantics, why a lock is never
sufficient alone). The checkout/payment reconciliation mirrors
[06.apple-app-store](../06.apple-app-store/04-deep-dives.md) (at-least-once money
messages plus idempotent receivers). The order-event log and hold-expiry reaping use
[13.kafka](../13.kafka/04-deep-dives.md) exactly as the seat of a state machine.
Cache-layer structure for the steady-state catalog is the same shape as
[10.distributed-cache](../10.distributed-cache/04-deep-dives.md).

**Try it first**: before reading 04, answer: (1) 2 million people want 20,000 seats —
where in the request path do you *want* most of them to spend their time, and what does
that component look like? (2) two buyers request seats 12–14 of the same row at the same
millisecond — name every mechanism between them and a double sale. (3) a buyer's card
authorizes but the confirmation write times out — what does the buyer see, and what
stops the seat from being sold twice?

**Estimates & public anchors**: numbers in this case are estimates and marked as such.
Publicly reported anchors used for sanity: Ticketmaster publicly said the Taylor Swift
Eras Tour presale (November 2022) drove roughly **3.5 billion system requests** over the
day and blamed a several-fold surge in bot traffic; press coverage of other on-sales
routinely describes sellouts in minutes. Use those as orders of magnitude, not facts
about any specific show.

**Primary context**: Ticketmaster engineering blog posts and the 2023 Senate Judiciary
hearing testimony (publicly reported figures on bot traffic and the Eras presale);
StubHub/Pollstar reporting on the resale market; the general literature on flash-sale
and virtual-waiting-room design.
