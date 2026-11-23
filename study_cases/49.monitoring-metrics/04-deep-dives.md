# 04 — Deep dives

## 1. Pull vs push (and why "HA without consensus" works for alerts)

The direction of collection decides almost everything downstream.

**Option A — push**: targets emit samples to a receiver as they change (StatsD-style,
or OpenTelemetry push):
- wins: covers short-lived jobs and NAT/firewalled targets without an intermediary;
  near-zero latency from event to sample; no fan-out of scrape traffic from the monitor.
- loses: the receiver's input *shape* is up to the monitored party — cadence, batching,
  and completeness are per-target policies you cannot inspect; "target down" becomes
  "no data from target", which is indistinguishable from "target alive but idle"; and
  a sick fleet *pushes harder* (retry storms) exactly when the monitor should throttle.

**Option B — pull (chosen)**: monitor discovers targets, GETs `/metrics` every 15 s:
- wins: one enforced cadence (rate/averages are well-defined — §6); "no sample at t"
  is a *positive observation* (`up == 0` with a recorded timestamp and scrape error),
  which is the primitive alerting is built on; targets stay stateless and dumb; the
  monitor can be moved/redeployed with zero target-side change; health-check semantics
  come free — the scrape *is* the health check.
- loses: event-y changes between scrapes are lost or smeared (§6); NAT'd/batch jobs
  need help (the Pushgateway: jobs push to a cache which is *then scraped* — it patches
  the lifetime hole without giving up the direction); a target that exposes `/metrics`
  slowly can hold scrape slots (mitigation: per-target timeout + concurrency caps).

**Chosen: B.** The requirement that decides it ([01](01-requirements.md)): the monitor
must *distinguish dead from idle* and keep a uniform cadence, and it must keep working
when the fleet is sick. Pull wins those; the losses (batch jobs, NAT'd edges) are the
two cases push fans actually care about, patched by dedicated mechanisms, not by
inverting the direction.

**HA without consensus** rides on this: two servers both pull the same targets, each
with its own WAL and head. Option A (leader-elected single scraper) loses — scrape
gaps during elections and shared failure domain for the most critical loop in the
system. Option B (both scrape, dedup downstream) loses storage ×2 and duplicate alert
delivery — accepted, because Alertmanager's dedup-on-fingerprint is cheap and a
30-second election window is exactly the scenario the HA pair exists to survive. The
monitor is the *only* component in this system allowed to be a redundant dumb pair,
because its output (alerts) has natural idempotent semantics downstream. Failure modes:
(i) both servers scrape-offset-drift differently and see *different* counter-reset
moments → their `rate` graphs disagree slightly; cosmetic, accept; (ii) a target's
`/metrics` degrades to 2 s responses → both servers burn slot capacity; caps +
per-target timeouts + `up{}` makes it *visible* rather than silent; (iii) an operator
"fixes" HA by load-balancing dashboards across both servers behind one VIP — now each
panel render lands on a server with *different* staleness, and graphs flip between
them; dashboards should pin one server, and the querier layer (§4) is the fix if
unified reads are needed.

## 2. TSDB write path: head block, WAL, chunk encoding, compaction

Per sample: append to WAL (fsync amortized over the scrape batch) → append to the
series' active in-RAM chunk in the head block → make queryable. On the 2 h boundary,
the head's chunks persist as an immutable block directory — chunk file + per-block
index (label index + postings) + meta.json — and the WAL truncates. A background
compactor merges 2 h blocks into 8 h, then daily blocks, and builds downsampled
variants for the remote layer.

**Option A — general LSM (Bigtable-style MemTable/SSTable, tombstones, merge-on-read)**:
- wins: handles out-of-order/deleted data; one proven engine.
- loses: read amp on a path where read amp is the scarce budget (§3: a selector query
  already does posting intersections; adding k-file merges per chunk makes p95 panel
  latency unownable); delete/tombstone machinery is waste — retention deletes whole
  blocks by time range, never individual samples.

**Option B — purpose-built time-partitioned structure (chosen)**: data is *already
sorted* — samples arrive ~in order per series, and the partition key is time itself.
So: head = per-series memchunks covering the 2 h window (~480 samples per series at
15 s, cut into ~120-sample chunks); samples packed with double-delta timestamps and
XOR'd float values, ~1.8 B per sample, publicly reported; blocks = immutable sorted chunk files; "compaction"
is block-merge *in time order*, not key reconciliation.
- loses: late/out-of-order samples near the head (handled: bounded out-of-order window
  into the previous chunk; anything older is dropped); arbitrary deletes (handled:
  tombstoning exists but is a maintenance operation, not a query-time property); no
  reuse for non-time data (which we don't want).

**Chosen: B.** The write stream is monotone by construction because *pull imposes the
clock* (§1) — the storage engine is simple *because* the ingestion model already paid
for ordering. This is the same causal family as Kafka: single-writer-per-partition
appends make the log cheap ([13.kafka](../13.kafka/04-deep-dives.md) §1), and the
block/compaction story is Bigtable's SSTable layering with the tombstone machinery
deleted ([12.bigtable](../12.bigtable/04-deep-dives.md) §2).

Compaction details that carry weight:
- **Vertical (time) compaction only**: blocks only merge along the time axis; the
  series set within a block is fixed. This is why block boundaries are also the
  retention/deletion and remote-upload units — an immutable 2 h file of everything
  is a cheap thing to ship to object storage (§4).
- **Chunk size discipline**: chunks target ~120 samples; smaller chunks inflate index
  and per-chunk overhead; bigger chunks hold RAM longer and delay head flush. The
  trade is per-chunk metadata vs head-set memory, tuned to the 2 h window.
- **fsync policy**: WAL flushes are batched per scrape; a full fsync per sample would
  collapse throughput for zero operational gain — one second of samples is the loss
  bound (§ non-functional in [01](01-requirements.md): losing ≤ 2 h is acceptable, so
  even WAL is a *bonus* durability).

Failure modes: (i) **compaction debt** — head flushes faster than compactor merges
(after a churn burst created 100+ blocks) → disk fills with un-merged blocks, index
queries slow down block-scanning; monitor `compaction debt` and backpressure head
cuts; (ii) WAL replay storm after crash — a crashed server replays ~2 h of WAL
(~10 GB, [02](02-scale-estimates.md)) before it is healthy; recovery is a scheduled
event, alerts for that server's own health come from the *peer*, which is one of the
actual reasons two scrape servers beats one; (iii) slow-block uploads to the remote
layer block retention deletes locally if sidecar upload queue saturates → disk death;
the upload queue must have bounded depth with explicit drop policy (drop remote
copies before dropping local retention).

## 3. Label index: inverted postings, selectors, and why arbitrary joins hurt

Query-time problem: `sum by (job) (rate(http_requests_total{code=~"5.."}[5m]))` over
4M active series must, in ~1 s, find "which series", "which chunks", and aggregate.
The answer is the same inverted index every search engine uses, applied per label.

Structure: each label pair `code="500"` → a sorted list of series IDs (the *posting
list*). Each series → sorted list of chunk refs. A selector's matchers intersect/
union postings (`=` → binary search into one posting, `=~` regex → union of matching
postings, negative matchers → all-postings minus the matched). Result: a sorted ID
set → chunk fetches are contiguous reads from chunk files by ref.

**Option A — row-key/sorted-map lookup only** (`__name__ + job + instance` must be
fully specified, like a K/V store):
- wins: trivially fast point lookup, no index maintenance write amp.
- loses: every dashboard query — "all 5xx across all instances of job=api" — becomes a
  full scan; the label selector *is* the product here.

**Option B — inverted label index with postings (chosen, this is what Prometheus
ships)**:
- wins: selector resolution is set algebra over sorted lists; a query touching 500 of
  4M series reads only those 500 series' chunks. Regex matchers degenerate gracefully
  (union of postings), and the *index itself is per-block*, so old blocks carry their
  own index and retention deletes the index with the block.
- loses: write amp — every distinct label value updates a posting list per sample
  batch (the index is why head RAM tracks *distinct* series, [02](02-scale-estimates.md));
  and posting lists for high-cardinality labels are long → intersections get expensive
  (§5's cost model is fundamentally about these lists).

**Chosen: B.** Pay at write (which is a monotone append and can be batched) to buy the
read pattern (which dominates: [02](02-scale-estimates.md) — order 10^9 point touches/s
across queries). Same ledger as Bigtable's bloom filters + locality groups: the LSM
pushed cost to read time, auxiliary structures buy it back there
([12.bigtable](../12.bigtable/04-deep-dives.md) §6); here the structure itself is
read-shaped from the start.

**Why arbitrary joins are the cliff.** Two series sets joined on an arbitrary predicate
means: materialize set A, materialize set B, then per-element probe a relation that the
index does not represent. PromQL stops *short* on purpose: vector matching only on
identical or *listed* labels (`on (job, instance)` / `group_left`), one-to-many in a
known direction, and *no recursive or transitive matching*. The cost cliff is real:
matching on a high-cardinality label (say `user_id`, 500K values) means holding both
series sets in memory and hashing per element — a join can allocate gigabytes where the
selector path allocated megabytes, and its cost is proportional to *cardinality*, not
result size. The escape hatches are the honest alternatives: recording rules
pre-materialize the join into a series; or push relational work to the systems built
for it (logs/traces/warehouse, out of scope per [01](01-requirements.md)). Failure
modes: (i) a regex `.*` matcher on `instance` → union of a 10,000-long posting list per
query, every dashboard render — the fix is label *design* and per-query limits (§5),
not query optimization; (ii) `group_left` against a mis-sized label → many-to-many
silently duplicated/errored — the engine errors many-to-many (good), but `group_left`
*chooses* to accept it, and a wrong "many" side doubles counts silently; (iii) stale
postings for series that churned hours ago slow intersections in the head — bounded by
head window (2 h) and by churn limits.

## 4. Scaling past one server: vertical federation vs horizontal sharding (and what
Thanos/Cortex/Mimir add)

At 4M active series one HA pair suffices ([02](02-scale-estimates.md)). At 10×, single-
server ingest and query budgets break (public operator reports put single-server
comfort in the low millions of active series — beyond that, degradation).

**Option A — horizontal sharding**: split *series* across N servers (by function or
label hash), each scraping/storing a subset:
- wins: linear ingest and storage scaling; each server's RAM/index bounded.
- loses: **every** aggregation query loses locality. `sum by (job)` over the fleet must
  hit every shard, fetch partial vectors, and merge — a query layer with scatter-gather,
  partial-failure semantics, and a coordinator; alert rules need the same fanout; and a
  hot metric's shard assignment is a data-model decision (the sharding key *is* a label)
  that operators must get right forever. The query cost is paid on every panel render
  forever, for a ingest-side win you may not need at this scale.

**Option B — vertical federation (chosen for the base design)**: one server per
functional domain / team / datacenter, each independently scraping its targets,
plus a thin query layer that can fan out *when the operator asks for a cross-domain
query* (federation `/federate` endpoint or a federating querier):
- wins: each server stays a simple, self-contained system with local queries (no
  coordinator in the happy path — the same "data plane is masterless" pattern as
  [Bigtable](../12.bigtable/04-deep-dives.md) §3); blast radius is the domain; scaling
  is a *re-assignment of targets to servers*, a human decision made once per re-org.
- loses: no automatic load balancing (a domain outgrows its server, someone must
  re-cut); cross-domain `sum` queries are explicit and pay the fanout only when used;
  a "global alert" needs either rules on every server or the federating layer.

**Chosen: B as the base, with the pain kept on purpose.** Prometheus shipped without a
distributed query coordinator for years — not because fan-out was unknown, but because
sharding-first would have turned every single-server deployment into a cluster
deployment (consensus, rebalancing, coordinator SPOFs) for the majority who never cross
1M active series. The cost is real: the *long-term* story (beyond 30-day local disk)
has no home in the base system, which is exactly the hole the ecosystem filled.

**What Thanos/Cortex/Mimir add** (three public projects converging on the same shape):
- **Object storage as the block store**: the compacted immutable 2 h blocks (§2 make
  them immutable precisely so this is possible) are uploaded as-is; a 13-month
  archive costs object-storage money instead of replicated-DB money (~7–8 TB,
  [02](02-scale-estimates.md)); the compactor on the store side merges and
  downsamples (raw 15 s → 1 min → 5 min tiers).
- **A store-gateway/querier that makes old blocks queryable** without holding them in
  RAM: index probes hit object storage, chunks stream in via page cache — "read old
  data slower" rather than "keep all data hot".
- **Global query fanout + dedup**: the querier fans out to every Prometheus + store
  gateway and merges duplicate series (from the HA pair) by label+timestamp —
  effectively implementing Option A's query layer, but *opt-in* and only on the read
  side, while writes remain per-server simple.
- **Dedup**: two servers' identical series collapse at query time; a
  `--cluster.label`/replica label distinguishes them so queries can dedup deliberately.
  This is the price of no-consensus HA (§1) — every consumer of cross-server data must
  know about the duplication.
- **Remote-write ingestion** (Cortex/Mimir): a real multi-tenant ingest tier if you
  want receivers, not scrapers, at the top — re-opening the push question (§1) but only
  for the long-term tier, where cadence is already fixed upstream.

Failure modes: (i) **federation fanout storms** — one dashboard panel that queries
every domain's server × every panel render turns the query layer into the fleet's
hottest component; mitigate: recording rules per-domain, federation reserved for
global aggregates; (ii) **remote-write backpressure** — store-side compactor falls
behind during an ingest spike, block upload queue fills, sidecar either blocks local
retention (§2) or drops blocks — the drop policy must be explicit and monitored,
because "which 2 h window silently vanished" is a question on-call will eventually ask;
(iii) **dedup lies after a target moves servers** — a re-sharded target's series show
a seam (two servers with different scrape offsets) that dedup cannot paper over; the
honest fix is accepting one-scrape-cycle seams in global graphs at re-org boundaries.

## 5. Cardinality explosion: label design, enforcement, and the cost model

The data model makes series count multiplicative in label *values* —
`{a: 10 values} × {b: 10 values} × {c: 10 values} = 1,000` series from three
innocuous labels. The canonical accidents, in rough order of damage:

- **IDs as labels**: `customer_id`, `user_id`, `request_id`, `pod_uid` on a hot
  metric — tens of thousands to millions of values, and the label *rotates*
  continuously as new users churn in (the churn line in [02](02-scale-estimates.md)).
- **Unbounded host/path labels**: HTTP paths (`/users/12345/orders`), URLs, email
  domains, error strings-as-labels.
- **Histogram buckets multiplying everything**: a histogram with 11 buckets × its
  label domains — the 5B-combination example in [02](02-scale-estimates.md) (55B with
  buckets).

The blast radius is not one metric — it is system-wide and ordered: (1) head RAM
(chunks + per-series metadata) — the first thing to die, and it takes *querying* down
with it, not just ingestion of that metric; (2) WAL/block size + index postings —
disk and block compaction debt (§2); (3) every selector that matches the metric now
intersects posting lists with hundreds of thousands of entries — panel p95s climb for
*unrelated* dashboards because the engine shares one process; (4) churn: each new pod
in a deploy re-creates the full label set — a 500-pod rolling deploy of a high-
cardinality-metric app pushes *distinct series* (which the index, not the active set,
tracks) even when *active* series looks fine on the graph.

**Option A — enforce limits at ingestion**: per-scrape-target series limits
(`sample_limit`, `label_value_length_limit`), per-metric keep/drop relabeling, and
rejecting a scrape that exceeds the limit (the extra samples dropped, `up` annotated):
- wins: the cost model becomes *bounded* — worst case per scrape is known, so head RAM
  and index sizes degrade linearly with fleet size, not with a contractor's bad pull
  request.
- loses: silent data holes when limits fire (a dashboard goes flat mid-incident);
  limits are per-target blunt instruments when damage is per-label; enforcement shifts
  the failure from "monitoring dies" to "monitoring lies a little" — which must be a
  deliberate, visible choice (`prometheus_target_scrapes_exceeded` alerts fire).

**Option B — self-regulation: document conventions, watch dashboards, fix on call**:
- wins: no data holes; flexible.
- loses: the failure mode is head OOM of the monitor itself — taking *all* metrics,
  all dashboards and all alerting down at once. The cost curve is convex (§ above),
  and one label change can cross the knee in a single deploy.

**Chosen: A with explicit limits + alerting on the limit itself**, because the system
being monitored is the system whose absence is most expensive during an incident: a
monitoring stack that dies from cardinality has its failure coincide exactly with the
traffic surge that makes monitoring valuable. Option B's flexibility is real but
belongs to *label design conventions* (below), not to *runtime* unboundedness.

The cost model operators should run (all rough, arithmetic in [02](02-scale-estimates.md)):
~3 KB heap per active series, ~1.8 B per sample on disk, ~5 B WAL per sample; a label
value domain of size V on a metric with M active base-series creates ~M × V active
series and ~M × V × churn-rate distinct-per-day. Before adding a label, do that
arithmetic; if the answer is over ~10% head growth, the label should be on a
*recording rule's* output, not the raw metric (aggregation happens once, downstream
consumers see the bounded result). This is the same lesson as Bigtable's row-key
design ([12.bigtable](../12.bigtable/04-deep-dives.md) §1): the *data model is the
operator's* responsibility, the engine enforces floors but cannot fix a bad key/label
choice after the fact. Failure modes: (i) limit-fire storms during deploys — every
scrape exceeds, dashboards flatten, alerts for `limit exceeded` drown the page;
mitigation: keep the limit-alert quiet on rate, page only on sustained saturation;
(ii) **recording-rule cardinality is not free** — the pre-aggregated series inherit
their own `by` groups; a recording rule that groups `by (customer_id)` just moved the
explosion one hop; (iii) deleted-but-recent series keep costing: head index holds
churned series for the head window and blocks hold postings for their lifetime —
cardinality cleanup shows up in hours (head) and only at retention expiry (blocks),
so the graph lags the fix; this is why *prevention* is the only cheap lever.

## 6. Staleness, rate windows, counter resets, and scrape gaps

The ingestion model (§1) makes input data lossy and non-uniform in exactly three
realistic ways: scrapes fail (gap), targets are replaced (series churn: a counter
"resets" by *restarting at 0 under a new identity*), and monitors lag (query reads
head data whose last sample is older than the interval). Every semantic below is a
choice about what a query *should* return given lossy input — and the choices are
consistent with one design goal: **make alerting expressions stable enough to compute
a rate and page on it.**

**Counters and `rate()`**: counters can reset (process restart, pod replacement) or
wrap (uint64 wraparound, rare at 15 s). `rate(counter[5m])` computes the slope over
the window and handles resets by *assuming a reset*: if the last delta is negative,
carry the counter's pre-reset value forward as `previous_value + reset_to` (treat
unknown as max-known); the estimated rate averages the extrapolated delta over the
window. Trade-off made explicit: this wins *alert stability* (a restart does not
produce a fake negative or a fake spike), loses *accuracy at the exact reset sample*
(one wrong delta among ~20 in the window — error ~1/window-size) and can hide a
*legitimately negative* phenomenon — but counters are by definition monotonically
non-decreasing, so there is no legitimate negative phenomenon to lose.

**Window length**: `rate([5m])` over a 15 s cadence = ~20 samples. Shorter windows
(1 m) are jumpy (few samples; a single missed scrape eats 25% of them) and more
reset-sensitive; longer windows (1 h) smooth away real signals and delay detection.
5 m is the operational sweet spot and the default guidance; window choice is a
detection-latency vs noise trade, stated per rule ([02](02-scale-estimates.md)'s
alert budget).

**Staleness and the 5-minute lookback**: at query time, a series' value "as of t" is
its last sample up to t — *if that sample is no older than 5 minutes*; older, the
series is absent from the instant result (and marks staleness explicitly after a
marker sample when a series stops being scraped). Trade-off: this wins *honest
holes* — a dead target's panel shows a gap or a stale-but-labeled point instead of a
silently-extended line that looks alive; loses *convenience* — a `sum` across a
partially-scraped fleet changes magnitude as members go stale mid-window, so
aggregates shift for semantically-bounded reasons. Compare the alternative (carry
last value forever, MRTG-style): graphs never lie but never tell you anything
either; the whole point of `up`, gap-as-value (§1) and bounded lookback is that
*absence is representable*.

**Scrape gaps in range queries**: a 2 h scrape outage followed by recovery produces —
- for **gauges**: a hole (no samples → no points at those steps);
- for **counters with `rate()`**: the two straddling samples produce ONE huge (correct
  total-over-gap ÷ gap-length) rate point, which reads as an average over the outage —
  the *honest* number (total errors did accrue) — then normal rates resume. Alert
  rules using `rate` therefore *detect* the outage as a blip, but a rule on
  `absent()` or `up == 0` is the dedicated detector (§1's `up` semantics). The
  trap: an alert `rate(errors[5m]) > 100` *fires correctly* through most outages but
  a gap longer than the lookback window makes rate return **no data** — and most
  alerting frameworks treat "no data" as NOT firing. Long outages thus need
  complementary rules (`up == 0 for 2m`, `absent(metric)`) — a multi-rule patch over
  an input-data hole, same shape as the typeahead fallback stack
  ([45.typeahead](../45.typeahead/04-deep-dives.md)).

**Chosen semantics, in one sentence each**: resets are modeled not surfaced (stability
wins, one-sample accuracy loses); staleness is bounded not eternal (honesty wins,
aggregate continuity loses); gaps in counters are averaged over (totals win, event
shape loses); `up` makes scrape failure itself a first-class series (observability
wins, nothing is traded away — this is the single highest-leverage semantic in the
system). Failure modes: (i) an off-schedule *double scrape* (target exposes the same
counter value twice within one interval) → rate underestimates for one window;
self-heals, accept; (ii) **counter reset on a *new* series identity** (pod replaced)
reads as series churn, not reset — the *new* series has its own 5 m warm-up during
which `rate` sees too few samples (no result or a low estimate) → warm-up windows in
rules or `clamp_min`-style guards; (iii) DST/timezone and step misalignment: a range
query with `step` not aligned to the scrape grid can alias between sample and gap
positions, making rate graphs look different at different resolutions — keep step ≥
2× scrape interval; and (iv) the 5-minute staleness window vs a 60 s scrape class:
a slow scrape target that exceeds one interval regularly produces jagged staleness
boundaries in instant queries — the fix is per-target cadence honesty ([02](02-
scale-estimates.md) § bend), not query-side hacks.
