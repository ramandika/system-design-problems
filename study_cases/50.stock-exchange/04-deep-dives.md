# 04 — Deep dives

## 1. Order book data structure: array of price levels vs. pointer structures

The book does four things constantly: find best bid/ask, insert at a price, remove a
specific order, walk a level's FIFO queue to fill. At the burst budget of ~20 µs per
message on the hottest symbol (02-scale-estimates), the data structure is the matcher.

**Option A — tree/heap of price levels with linked order objects (the "textbook" book)**:
- wins: sparse price space costs nothing (no wasted slots); range queries (show me
  levels near price P) are natural.
- loses: every best-price lookup is a pointer chase (log depth); nodes are heap-
  allocated and scattered across memory — an average L1 cache miss (~few ns each, and
  many per lookup inside a 20 µs budget) per level hop and per order follow; deletion
  needs a find-then-unlink. On a hot day the top-of-book churns thousands of times per
  second per symbol and the pointer chasing dominates.

**Option B — flat array of price levels indexed by tick, each level a small inline
order queue (chosen)**:
- wins: level lookup by price is an O(1) array index — arithmetic, no pointers; the
  top levels (where all the action is) sit contiguous in cache; best-bid/ask maintenance
  is a decrementing/incrementing integer scan over an array of booleans/counts —
  sequential, prefetcher-friendly. Cost is bounded and *predictable*, which the µs
  budget cares about as much as the mean.
- loses: memory is proportional to the *tick grid*, not the filled levels — $500 max
  at $0.01 tick = 50K slots/side ≈ 400 KB/side (02-scale-estimates: ~6.4 GB for the
  whole venue — acceptable); changing tick size or handling corporate-action price
  resets means remapping the grid (rare, offline-maintained); a symbol split/stock
  split reindexes everything (acceptable: they are scheduled events).

**Chosen: Option B.** The pointer-free path wins because the matcher's per-message
budget is ~20 µs and Option B turns the two hottest operations (level find, queue
walk) into array indexing and sequential scans. This is the same insight as Kafka
betting everything on sequential disk access
([13.kafka/04-deep-dives.md](../13.kafka/04-deep-dives.md)): pick the operation that
dominates and make its physical access pattern optimal; everything else bends.

**Failure modes**: a level's inline queue overflowing (capped, spilled to a secondary
list — hot levels rarely exceed ~hundreds of concurrent orders); a price *outside* the
grid (penny stocks trading at $0.0001 or a symbol exceeding grid max — rejected with
"price out of range" rather than crashing the matcher); memory waste on 8,000 mostly-
quiet symbols (accepted: 6.4 GB is noise); a bug in best-price maintenance desyncing
the cached best-bid/ask from the arrays (bounded by an invariant check per N events +
halt, never by trusting the cache).

## 2. Threading: single-writer per symbol, sharded by symbol group

Someone must own book state. The choice is who, and how many of them.

**Option A — shared book, concurrent access, locks/lock-free structures**:
- wins: horizontal scaling feels free; any shard can handle any symbol.
- loses: every match step races with every other; contention on hot symbols turns the
  20 µs budget into a lock-wait distribution; worst, **matching is not per-symbol
  parallelizable internally without breaking sequence** — two threads pulling from one
  symbol's stream must re-impose the total order they just destroyed. Lock-free books
  exist but their *fairness* semantics ("who was first") become probabilistic —
  exactly the property an exchange cannot have (deep dives §4).

**Option B — single-writer per symbol; symbols grouped into shard groups, each group a
single-threaded deterministic event loop pinned to a core (chosen)**:
- wins: no locks, no atomics on state, no memory-ordering reasoning; the loop is a
  pure function sequenced-events → outbound events, which is what makes replay
  deterministic (deep dives §3) and testing exhaustive; ~8,000 symbols ÷ ~30-40 groups
  ≈ ~200-270 symbols per core (est.) comfortably fits the average (~2.3K msgs/s
  aggregate ≈ under 100 msgs/s per group).
- loses: **no horizontal scaling for one symbol** — the hottest symbol's capacity is
  one core's single-message speed (~20 µs/msg → ~50K msgs/s ceiling); per-shard load
  balancing is static (regime-dependent, needs repartition maintenance windows); a
  slow core (NUMA-locality miss, thermal throttle) degrades exactly its symbols.

**Chosen: Option B.** Determinism and fairness are load-bearing; locks are not
affordable anyway. The capacity answer for a single hot symbol is inside the core:
- **Batching**: drain up to N messages per loop iteration, process them in one cache-
  warm pass, emit batched outbound events — hides fixed per-message overheads; it does
  not change order semantics because batching preserves sequence.
- **Kernel bypass / busy-poll dispatch** (est. tens-of-µs→sub-µs ingress): standard
  venue practice, publicly described in low-latency engineering literature.
- **Pipelining**: journal-write, match, and emit overlap across iterations of the same
  loop — still single-writer, still in order.

**Failure modes**: hot symbol exceeds even batched capacity (est. >100K msgs/s on one
symbol) → the venue throttles *new orders* for that symbol via an explicit, journaled
throttle event (a designed, disclosed action — not silent queueing, which turns a
fairness problem into an unbounded-latency problem); shard-group crash → symbols of
that group halt, standby group takes over by replaying its slice of the journal
(deep dives §6); core pinning + noisy neighbor on the same host (co-located analytics
stealing cache) → mitigated by isolation (irq affinity, dedicated hosts), monitored,
because it degrades fairness-perceived latency, not correctness.

## 3. Sequencing & determinism: the journal is the venue's source of truth

Everything above the physical wire — fairness, recovery, audit, dispute resolution —
reduces to: **one total order of inbound events, and deterministic application of it.**

**Option A — distributed replicated inbound log (Kafka-style: 3-way replicated partition
commits per message, matcher consumes)**:
- wins: durability and failover engineering is inherited, battle-tested (cf.
  [13.kafka/04-deep-dives.md](../13.kafka/04-deep-dives.md) §2–3); no bespoke
  sequencer software to own.
- loses: per-message quorum commit is ~ms-class and bursty; the acking path adds at
  least one network RTT + a fsync-durable write *before* matching — that is the whole
  latency budget gone (02-scale-estimates table); partition-key hashing across
  partitions would break the per-symbol total order unless one symbol = one partition
  (then replication is still per-message). At µs budgets it is structurally disqualified.

**Option B — dedicated single-active sequencer + hot standby, journal-anchored (chosen)**:
- sequencer stamps a globally increasing sequence on every inbound message, in arrival
  order *as observed at one point* — the single serialization point that gives the
  total order; appends to the journal (RAM buffer first, fsync by a parallel writer —
  **buffered-first is a deliberate durability trade**: a machine-level loss window of
  microseconds of journaled-but-not-fsynced events exists and is reconciled on failover,
  deep dives §6); forwards to shards. Standby tails the same journal (Option B wins:
  failover = "you are now the sequencer at sequence N", no data structure to elect).
- Replay: a fresh process (recovery, or a dispute investigator) applies the *inbound*
  stream to the deterministic matchers and reproduces the *outbound* stream. Byte-equal
  output is a **release test**, run on every build; a replay that diverges grounds the
  release. This is the exchange-grade version of "event-sourced with a tested reducer".
- **Why client order IDs must be idempotent**: gateways and member systems retry (a
  lost ack will be resent — the same rule as
  [44.payment-system/04-deep-dives.md](../44.payment-system/04-deep-dives.md) §1).
  The dedup key is (session, clOrdID), held by the matcher's per-session table: a
  resubmit of a *live* order ID returns the original ack behavior (reject-with-dup-ID),
  not a second order. The key must be client-chosen, not server-generated — a
  server-generated ID cannot tie the retry to the original through the retry path.
- Cancel of an already-filled order returns "too late to cancel / already filled" — a
  full-fidelity journaled response, never silence.

**Chosen: Option B.** At microsecond budgets the log cannot sit on the synchronous
path with quorum semantics; the sequencer is instead *made safe by the journal* (the
standby replays it), and the total order is *made cheap by being a single stamp*, not
a distributed agreement.

**Failure modes**: sequencer crash between journal append and dispatch → standby
starts from the last dispatched sequence and replays the remainder (idempotent client
IDs absorb any double-apply of a resubmit); two gateways forwarding near-simultaneously
→ both land in the one arrival queue — order decided by the sequencer's position, full
stop (deep dives §4); any nondeterminism source (thread-id in output, map-iteration
order, wall-clock as decision input) → caught by the replay test, treated as a
release-blocking bug, because it silently corrupts recovery and audit alike.

## 4. Fairness & latency: position in the stream, not the clock; colocation; never reject-for-latency

Fairness is a *contract*: an order must get its position in the book strictly by
sequence, and equal-priced earlier orders must fill first. Everything else (speed,
hardware, fees) is secondary to not breaking this.

**Option A — wall-clock/timestamp ordering (timestamp each order at the gateway, order
by timestamp)**:
- wins: lets gateways be parallel and distributed; "fair" sounds like "earliest time".
- loses: clocks are not comparable at µs granularity even in one DC (drift, NIC
  timestamping jitter); two orders 200 ns apart get a *random* winner; the decision
  cannot be re-derived after a failover ("what was the clock at the moment X entered
  the queue?" is unanswerable). It converts an auditable property into an
  unreconstructible one — the exact opposite of what a regulator's dispute process needs.

**Option B — sequence-position ordering with one serialization point (chosen)**: the
sequencer's stamp IS the fairness primitive (deep dives §3); FIFO within a level is
just the position in the sequence. Consequences embraced rather than fought:
- **Colocation for everyone who pays**: members host in the same DC on the same terms;
  access tiers (connectivity, feed proximity) are disclosed and priced, but *ordering*
  never depends on path — the single queue erases path differences. A slower member's
  order still occupies its true sequence position and fills ahead of a faster later
  order when priority allows.
- **Latency is never an input to validity**: an order is valid because of its content,
  not its speed. The venue does NOT reject an order for arriving "late" — slow arrival
  just gets its (worse) queue position. Rejection as a fairness action is reserved for
  rate-limit throttle events, journaled and disclosed, and only on aggregate risk
  grounds (deep dives §2), never as a hidden latency policy.
- **No reordering between gateways**: the gateway→sequencer hop is one FIFO queue per
  ingress; nothing downstream may look ahead.

**Chosen: Option B.** The sibling case
[37.ticketmaster/04-deep-dives.md](../37.ticketmaster/04-deep-dives.md) shows the
consumer version of the same problem: a fair queue under contention wants *one place*
where relative position is decided, not a distributed race plus apologies afterwards.
Here the stakes are higher (litigable) and the budgets stricter, which forces the
single-point answer rather than the compromise.

**Failure modes**: an ingress path with two independent FIFOs (e.g. two NICs feeding
two queues into the sequencer) silently forks the total order — the invariant is
"exactly one serializer, exactly one queue per serializer input", and per-sender
sequence numbers from gateways let the sequencer *detect* ingress reordering and alarm
rather than absorb it; a member gaming intra-session ordering (pipelining many orders
into one burst ahead of others on the same connection) is behaviorally handled by
fairness rules operating on the *stream*, which is exactly what makes it auditable;
microburst unfairness across *symbols* (shard groups of different busyness) is
explicitly out of the fairness contract — the contract is per-symbol queue position,
not global latency equalization.

## 5. Market data fanout: sequenced multicast + replay tier; throttle depth, never trades

Subscribers: hundreds of firms taking the direct feed (02-scale-estimates: ~500),
from colocated HFT to vendor networks. Egress must be O(1); delivery must detect loss
and recover; and *some* messages are droppable under burst while others absolutely
aren't.

**Option A — per-subscriber TCP unicast**:
- wins: TCP gives reliability, ordering, per-subscriber pacing for free; trivially
  correct recovery.
- loses: 500 × 260 Mbps ≈ 130 Gbps egress and, worse, 500 TCP connection states whose
  slowest member paces backpressure into the feed writer — the feed's worst subscriber
  becomes everyone's jitter source (02-scale-estimates arithmetic); recovery and
  congestion control per subscriber.

**Option B — sequence-numbered UDP multicast (A/B redundant paths) + snapshot/replay
tier (chosen)**:
- one multicast stream per market-data channel (trades, BBO, full-depth per symbol
  groups); every message carries the feed's sequence; A/B independent paths let
  receivers detect and bridge short gaps; missed ranges are served by **snapshot
  (current book image per symbol) + TCP replay** servers, exactly the
  snapshot-plus-delta shape used by real feeds.
- wins: egress constant; receivers independent — a slow subscriber damages only itself;
  the feed writer stays deterministic (it just writes the journal stream onto the wire).
- loses: UDP requires the venue to *engineer* loss recovery (the replay tier, gap
  detection, snapshot cadence) instead of borrowing TCP's; multicast reach is
  network-dependent (works well colocated, not across the public internet —
  non-colocated subscribers get TCP delivery of the same sequenced stream, same
  content, different transport).

**Burst discipline — throttle vs. drop-last-quote**: the two message classes have
opposite rules:
- **Trades and executions are never dropped or coalesced** — they are the venue's
  authoritative record; every one is sequence-stamped and delivered (slow tiers may
  *lag*, they may not skip).
- **Depth updates may be coalesced** for slow transport tiers: if 10 depth changes for
  a symbol land in one 10 ms burst, a throttled channel may carry the *last* state
  (the coalesced "final" book level) instead of all 10 increments. Depth is state, not
  history: the last value supersedes. The coalescing rule is "latest-write-wins per
  (symbol, price level, side)", which keeps a coalesced stream reconstructible from
  any snapshot *after* the join point.
- This split is the same principle as
  [02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md) fanout (per-item
  delivery vs. coalesced-by-key delivery), but with a litigable correctness boundary
  on the coalesce.

**Chosen: Option B with the trades/depth split.** 130 Gbps of TCP fanout is not
affordable; TCP-only recovery is not sufficient when a 10-ms burst arrives faster
than any single WAN link drains; and the trades-cannot-drop rule is what makes the
throttling defensible publicly.

**Failure modes**: replay-server overload right after a multicast path flap (everyone
replays at once — rate-limit and stagger, or a recovery storm becomes an outage);
snapshot staleness during a frenzy (snapshot may be µs-old when taken and ms-old when
delivered — mitigated by snapshot+replay-from-snapshot-sequence, never by
snapshot-only join); silent gap-eating by a subscriber that ignores sequence numbers
(the protocol's gap detection is the contract; vendors that bridge gaps without
replaying state are their own bug); feed writer failing while the matcher continues
(a journaled event not on the wire — detected by end-to-end sequence reconciliation,
an alarm-and-halt condition, since market data integrity is itself regulated).

## 6. Recovery & HA: active–standby with journal replay; RPO zero-or-µs

The matcher is a deterministic state machine (deep dives §3), so HA reduces to: keep a
standby's deterministic copy of the state, and switch the sequence stream when the
active dies.

**Option A — synchronous replication of *state* (ship every book mutation to the
standby's book before ack)**:
- wins: standby is always hot, RPO = 0 by construction, failover = swap pointers.
- loses: a synchronous hop inside the ~20 µs per-message budget (a shared-memory ship
  is feasible on one host but then the standby shares the host's fate — the one thing
  failover must not do; a cross-host ship adds network-class latency); doubles
  matching work; and *state* replication still doesn't prove the standby interpreted
  events identically (a replicating engine that interpreted an event differently is
  corrupt, silently).
- Note: replicating *outbound events* rather than internal state mutations is nearly
  free for active–standby on the same sequencer stream (Option B's world), which is
  what A's "always hot" actually buys safely.

**Option B — hot/warm standby consuming the same journaled inbound stream; deterministic
replay to join (chosen)**:
- the standby runs the identical matcher code on the identical sequence stream
  (journal-tailed, lagging µs–ms) — its state is hot by *construction*, and it is
  continuously verifying that its derived outbound events match the active's (a
  free-running divergence detector: any mismatch = halt, because one of the two is
  wrong).
- **Failover**: the standby resumes dispatching at the last sequence it *verified*
  (or, for cold recovery, replays the journal from a snapshot checkpoint + events);
  RPO ≈ the journal's not-yet-fsynced µs-window (deep dives §3) — bounded, measured,
  reconciled post-failover; execution reports of the gap re-dispatch (idempotent
  client order IDs absorb re-acks; a member that got an ack before failover sees no
  contradiction: the ack is in the journal, either it was really sent or it wasn't).
- **RPO zero vs. microseconds** — the explicit trade: synchronous durable journal
  before dispatch would close RPO to 0 at the cost of an inline fsync in the ~50 µs
  envelope (02-scale-estimates); the design instead journals in a parallel-writer
  RAM buffer, keeps the not-durable window to single-digit µs at burst depth, and
  treats a crash inside that window as a *bounded, disclosed* loss of "we accepted,
  journaled in memory, did not fsync" — the recovery procedure re-derives and any
  gap is surfaced by reconciliation, never silently bridged. Whether that window is
  acceptable is a venue policy decision, publicly spec'd in the rulebook — made once,
  here, explicitly.
- **Faster than rebuild-from-scratch**: a cold replay of a full day's ~200M inbound
  events (02-scale-estimates; the journal also holds ~500M outbound events, which replay
  recomputes rather than re-applies) at ~1M events/s/core (est., batched replay) ≈ 200M ÷
  1M ≈ ~3–4 min for the whole venue,
  per-shard a fraction of that — minutes, not sessions; warm standby makes the point
  mostly moot for failover, but cold recovery bounds matter for disaster tiers.

**Chosen: Option B.** Determinism is the only replication mechanism that also *proves*
correctness (the standby isn't just up-to-date, it is provably identical); synchronous
state plumbing spends the latency budget to lose that proof. This is the exchange
instance of the general rule: replay beats sync-replicate whenever the reducer is
deterministic — cf. [13.kafka/04-deep-dives.md](../13.kafka/04-deep-dives.md) log-as-
truth, and [46.google-docs](../46.google-docs/README.md)'s op-log thinking — with the
twist that here the proof property is load-bearing, not incidental.

**Failure modes**: journal disk corruption (checksummed segments; a bad segment halts
replay loudly → fail to the standby's verified state rather than "apply what parses");
split-brain (two writers dispatching after a failed failover — prevented by fencing:
the old active's dispatch credential is revoked at journal level, and a journaled
epoch/leadership event makes double-dispatch detectable in replay); standby silently
lagging for days under a memory bug (continuous verification doubles as a live
canary — divergence alarms long before a failover is needed); recovery *during* a
volatility burst (the worst moment to fail over — throttled new-order events after
failover are the disclosed, journaled stabilizer, per deep dive §2's throttle
discipline).
