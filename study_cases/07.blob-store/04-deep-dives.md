# 04 — Deep dives

## 1. Erasure coding vs replication — and chunk placement (THE dive)

The headline durability requirement makes the storage layout decision. **Option A — 3×
replication**: each object (or MB-chunk) copied to 3 hosts.

- wins: simplest possible reads (any replica answers), simplest repair (copy from a
  survivor), lowest read latency tail (pick nearest/liveliest replica; k-of-n
  recombination latency doesn't exist), lowest CPU.
- loses: 3× storage overhead — 1 EB logical costs 3 EB raw, ~1.5 EB more raw media than RS
  (from estimates: ~100K provisioned drives for RS(8,4) with headroom vs ~190K under 3×); per-object writes fan out ×3 full bytes.

**Option B — erasure coding, RS(k=8, m=4) shards per stripe**:

- wins: 1.5× overhead (1.5 EB for 1 EB logical) — the difference funds the whole
  engineering effort at this scale; every write spreads across failure domains instead of
  duplicating.
- loses: reads need ≥k of 12 shards in parallel — one slow disk stretches the tail of
  *every* read (mitigate by reading k+m' > k, e.g. 9 of 12, and racing); any write ≥ stripe
  size touches 12 drives (bad for small objects); repair reads 8 full shards to rebuild
  1 (network-heavy vs a replica copy); recombination costs CPU.

**Chosen: B for large objects (≥ ~4 MB), A for small objects.** For a 100 KB object,
RS(8,4) at MB-sized shards means 12 drives touched per read, and unbundled the stripe
is brutally inefficient: a 4 MB-shard stripe is 12 shards of 4 MB — ~48 MB of physical
bytes for one 100 KB object if it can't share a stripe (02's consistency check:
~2.5% fill × 12 shards). Bundling mends the bytes but not the 12-drive read tax.
Replication at 3× (300 KB of physical bytes, one drive per read) is cheaper per object
and orders of magnitude better at read tail. Small objects get replicated 3× or bundled; big objects, which carry most of
the bytes, get coded. This two-path split is not incidental — it's the direct consequence
of the mean-vs-median object size gap in the estimates.

### Chunk placement: where the 12 shards live

Placement is a reliability decision, not a load-balancing one. Rules:

- **Spread shards across at least 2 power supplies, 2 top-of-rack switches, ≥8 hosts**
  (approximate independent failure units; a rack can reboot entirely without touching
  availability).
- **Intra-datacenter for standard class** (latency); **cross-DC layout for the durability
  headline** (deep dive 5 decides exactly what goes where).
- **Deterministic pseudo-random placement** (hash of stripe id + replication slots),
  so that no two stripes' shards are correlated by machine — a "hot rack" is a latency
  annoyance, not a batch-durability event.
- **Small stripe count per (bucket, time) so the rebalancer only moves bounded data** —
  placement that scatters everything forever makes decommissioning unbounded.

- **Trade-off made explicit**: we accept per-read tail latency (k-parallel recombine,
  racing reads) and a repair protocol that reads k× the lost bytes, in exchange for
  ~1.5 EB of raw-media savings at target scale and better multi-failure tolerance per GB.
  For a cost-dominated service at exabyte scale, that trade is not close.
- **Failure modes**: (a) hot shard — one slow disk of 12 drags every stripe it's in;
  mitigated by racing an extra shard read; (b) correlated rack failure surviving <k shards
  — placement rules + DC-level layout cap this; (c) EC repair storm after a top-of-rack
  replacement — repair scheduler rate-limits per rack (see deep dive 4).

## 2. Metadata / control plane separation, and the small-delta list problem

**The split**: metadata (key→stripe map, versions, parts, ACLs, bucket config) lives in a
strongly consistent, ordered, bucket-sharded index. The data plane stores only bytes keyed
by stripe id and *never* participates in requests' consistency — front-ends consult
metadata, then talk bytes to data nodes. The data plane has no concept of "bucket", "key"
or "version".

- **Option A — one store for both** (index inline with data): wins conceptual simplicity
  (single system), loses both ways — the index's latency-sensitivity forces expensive
  media under exabytes of HDDs, and the hot listing path competes with ~520K shard
  writes/s; any index re-sharding becomes a byte-migration problem. Also couples data
  growth to index growth on one fleet.
- **Option B — split planes**: wins independent scaling (index: 1 PB and ~1M writes/s;
  data: 2 EB raw and 1.74 TB/s), lets index hardware be fast and data hardware be cheap,
  and makes the failure domains different (index loss = no new requests; data loss = real
  durability events — different runbooks). Loses: two systems to build and operate, and a
  consistency obligation between them (a committed index record must always point at
  durable bytes — the PUT ordering from architecture guarantees this).

**Chosen: B.** Every large object store converges here; the divergence you'd get from A
is unaffordable at exabyte scale.

- **Trade-off made explicit**: we accept cross-plane protocol complexity (two-phase PUT:
  bytes-then-index, plus orphan garbage collection) in exchange for planes that scale and
  fail independently. The PUT's "data durable before index commit" ordering converts all
  crashes into invisible orphans instead of broken users — one-sided by construction.

### Listing: the small-delta list problem

LIST must return keys in lexicographic order, paginated, against a bucket being
simultaneously written. Consider an index shard holding 1M keys that absorbs a *small*
delta of new writes: hundreds of concurrent list calls then each re-scan the full shard
(or re-read the 1M-key base) to find a few thousand new entries. The list load multiplies
by the whole-shard size instead of the delta.

- **Option A — naive index scan per list page**: correct, but a bucket's list cost scales
  with shard size, not result size; a hot bucket being listed while ingesting can
  monopolize index IOPS.
- **Option B — small-delta optimization**: each index shard (or bucket partition) keeps a
  small, recently-absorbed delta buffer explicitly surfaced to the list path; a list page
  = base (immutable/snapshot region) + delta (small, hot) merge. When the delta crosses a
  threshold, a background compaction folds it into the base. Lists pay for what the
  request returns.
- **Also needed**: pagination by key-marker into the snapshot so a rolling compaction
  cannot duplicate or skip keys across pages; per-bucket list rate limits to keep one
  tenant's enumeration from DoSing others.

**Chosen: B.** The cost arithmetic from estimates makes it obvious: LIST is ~3% of
requests but each unbounded scan would cost ~1,000× a GET's index work.

- **Trade-off off the consistency lever**: real object stores serve lists *read-*
  consistent (base snapshot), while a just-completed PUT is visible to GET immediately
  but may lag LIST by a bounded window (delta absorption) — the S3-documented "eventual"
  listing behavior for *new* keys vs the stronger *read-your-writes* GET guarantee.
  We accept list lag in exchange for bounded list cost; we do *not* accept PUT-visible-GET
  inconsistency — that's a contract break users' retry logic depends on.
- **Failure modes**: (a) hot-prefix listing (thousands of keys under one prefix on one
  index shard) — shard the index by (bucket, key-prefix-bands) so hot prefixes don't pin
  one shard; (b) compaction + list marker races causing skipped/duplicated keys — marker
  into snapshot generation id, tested invariant, not best-effort; (c) index shard
  overload during tenant bulk-import — per-bucket admission limits.

## 3. Multipart upload resumability

A 5 TB object cannot move in one request over any real network, so multipart exists —
and its real job is *resumability under failure*, which shapes everything.

Design:

- **Init**, a 5 GB part PUT, each an independent, idempotent transaction: part N's bytes
  are durable in the data plane and registered in an upload's part registry *before* the
  part-level ack. Re-uploading the same part number replaces it (idempotent by
  (upload_id, part_number)).
- **Complete** is a pure metadata transaction: the object record becomes a composition of
  part stripes; no data is copied or rewritten at commit. Commit must be atomic per
  upload: it either fully succeeds (object exists) or not (client retries Complete — the
  re-run sees "already complete" and returns the object; idempotent).
- **Resume story**: client restarts / connection dies mid-upload → client lists recorded
  parts (`ListParts`), re-uploads only what's missing, and continues. A 5 TB upload
  interrupted after 90% finishes with the missing 10%, not a from-scratch retry.
- **Abort/lifecycle**: abandoned in-progress uploads — parts are *physical bytes* — are
  reclaimed after a (configurable, default e.g. 7-day) idle expiry, since unreferenced
  bytes at exabyte scale are real money. The expiry must reset on part-activity, so an
  upload paused 6 days still resumes.

- **Trade-off**: storing parts durably per-part (with real stripe layout, i.e., EC'd)
  rather than staging them unreliably costs EC'd bytes for potentially-abandoned uploads;
  in exchange, a multi-day, multi-connection upload of 5 TB is a supported *use case*, not
  a gamble. Staging cheaply then re-Encoding at commit would halve the durability spend
  but break resumability (uncommitted parts not readable/recoverable) — for a product
  whose whole pitch is "big data lives here unharmed", resumability wins.
- **Same pattern, other case**: each part is an independently-idempotent retryable unit
  assembled into a whole — exactly the discipline of [YouTube's segmented transcode DAG
  and resumable upload](../01.youtube/04-deep-dives.md) (per-segment retry + final
  publish), and of at-least-once-then-dedupe delivery in
  [WhatsApp deep dives](../03.whatsapp/04-deep-dives.md). Make retries cheap and safe at
  the smallest unit; make the top-level commit atomic.
- **Failure modes**: (a) client keeps initiating uploads it never completes → orphan part
  bytes; lifecycle expiry is not optional hygiene, it's the metering of the failure mode;
  (b) Complete with a missing/corrupt part name — validation must check the part registry
  strictly and fail closed; a mismatched part size in the manifest must abort, not
  assemble a corrupt object; (c) Complete retry storm after a timeout on a huge upload —
  idempotency handles it, but the registry lookup must be cheap (indexed by upload_id).

## 4. Background repair, scrubbing, and rebalancing

At this fleet size hardware failure is a *continuous* stream, not an outage: at 2% AFR
per drive (industry-rough), ~100K drives ⇒ ~2,000 drive failures/year ⇒ **~5–6 failures/
day fleet-wide**, before host and NIC failures. Durability is then not a layout property
alone — it's a race between repair and the next failure (deep dive 5 does the math).

Design:

- **Scrubber**: continuously re-reads stored shard bytes and verifies checksums. This is
  what catches silent corruption — without it, parity shards are faithfully encoding
  garbage. Full pass ≈ 1–2 weeks (estimates §2); checksum of every byte is verified at
  least every few weeks.
- **Repair**: on shard loss (drive removed, disk bad, checksum mismatch), rebuild the
  shard from k survivors, write into a new stripe slot. Unit of work = (stripe, lost
  shard id) — idempotent, retriable, queued.
- **Repair scheduler**: rate limits per-rack/per-host repair bandwidth, prioritizes by
  stripe exposure — a stripe at 2 lost shards is a bigger durability risk than one at 1
  (closer to the <k cliff) — and dedupes with the rebalancer.
- **Rebalancer**: drains/decommissions nodes, fixes capacity and load skew, moves shards
  *without* changing their failure-domain spread. Strictly lower priority than repair —
  a decommission that delays durability repair is a real (if slow-motion) incident.

- **Trade-off**: aggressive repair (start within minutes of first lost shard, saturate I/O)
  minimizes the exposure window and best-uses durability math, but steals foreground I/O
  and can cascade (a repair storm after a rack reboot hammers the survivors' disks, adding
  *more* latency for users — and more failures if the fleet is already degraded).
  Gentle repair keeps service smooth but leaves stripes under-repaired longer — durability
  math suffers. **Chosen: priority-weighted, budgeted repair** — a per-fleet I/O budget
  for background work (~10–20% from estimates), burstable for high-exposure stripes, and
  hard per-rack caps to make the post-rack-recovery storm self-throttling.
- **Rebalance/repair as O(fleet) fanout**: both are coordinated write fanouts against
  live foreground traffic — the scheduling problem rhymes with keeping background
  deletion cheap via tombstones in [Twitter's delete-across-millions-of-timelines
  dive](../02.twitter/04-deep-dives.md): choose the mechanism whose cost sits *outside*
  the user-visible path, and bound its worst case.
- **Failure modes**: (a) scrubber falls behind — corruption accumulates as "verified
  lies" and the durability model silently degrades; track scrubber lag as an SLO, not a
  cron log; (b) repaired-but-not-cleaned old stripe → duplicate physical bytes; bounded by
  GC correctness, tested with the same invariant tests as index→stripe orphan collection;
  (c) rebalancer + repair racing on the same stripe — one owner per stripe (lease), since
  two writers rebuilding the same shard slot is a classic corruption source; (d) a bug in
  the EC math itself — the only defense is periodic full-verify (restore-and-compare
  drills on sampled stripes).

## 5. Durability math and datacenter loss recovery

**Naive math (do the arithmetic so you know which assumptions carry the promise).** Per a
single stripe: drive-level failure rate ~2e-6/hour (2% AFR, rough), repair completes in
T = 24h worst case. Over the exposure window, one shard's loss probability p ≈ 2e-6×24 =
4.8e-5. RS(8,4) tolerates 4 shard losses; failure of the stripe = losing ≥5 of 12 shards
within the window:

- P(≥5 of 12) ≈ C(12,5) · p^5 ≈ 792 × (4.8e-5)^5 ≈ **2e-19** per window.
- Windows/year = 365 → per-object annual loss rate ~**7e-17** — "well past 11 nines" on
  paper.

This computation is *necessary but nowhere near sufficient*; if it were the whole story we
would be at ~16 nines, not 11. Reality's corrections:

- **Correlated failures dominate**: same-firmware drives, same-rack PDUs, buggy scrubber,
  a bad batch — the binomial's independence assumption breaks exactly when it matters
  (5 drives from one bad batch of 500 in one window).
- **Operator and software error are first-order**: at this scale, an errant delete of a
  shard subtree or a buggy migration out-runs hardware math. Which is why *soft delete
  + repairable tombstones* (architecture) is part of durability, not an ops nicety.
  (For the same reason at the request path, see how [WhatsApp treats MUST-NOT-LOSE vs
  MUST-NOT-COST-ANYTHING state](../03.whatsapp/04-deep-dives.md) — know which signals
  get durability and which don't.)
- **Scrubbing latency is part of the math**: silent corruption found a month late
  multiplies the exposure window by ~30 beyond the naive 24 h.
- Net: the published 11-nines S3 target is the *engineering design target* including all
  this, not the naive binomial — and our design matches it by construction (redundancy +
  scrub cadence + repair budget + DC layout + soft-delete).

### Datacenter loss recovery

Intra-DC RS(8,4) is destroyed by total-DC loss — the entire deep-dive-1 layout must then
be layered to survive it. Options:

- **Option A — cross-DC EC** (e.g., RS(8,4) with shards placed across ≥3 DCs): wins
  single-layout simplicity and best per-byte efficiency; loses read latency (spans DCs),
  and normal GETs now traverse WAN — unaffordable for the standard hot read path.
- **Option B — intra-DC EC + cross-DC replication of a resilience class**: wins local
  latency and decoupled classes (storage class = layout policy: Standard keeps a 2nd/3rd
  copy in another DC; Archive uses DC-spread parity instead of full copies); loses extra
  bytes for the hot class.
- **Option C — intra-DC EC + async cross-DC EC re-encode** (shards re-encode to a
  DC-spread stripe in the background): bytes-efficient; loses a bounded cross-DC async
  window — the copy lag is a real (hours-scale) durability hole for a total-DC loss.

**Chosen: B/C hybrid by storage class** — Standard objects: intra-DC RS + sync-ish
cross-DC copy (Option B bytes, small window); cold/archive objects: DC-spread parity
(Option A) since their latency budget forgives WAN. The class system makes this a
*per-object* choice instead of a fleet-wide bet.

- **Trade-off made explicit**: we pay extra bytes on the hot class (and a latency tax on
  archive gets) to buy a layout where full-DC loss never becomes a bulk-restore emergency.
  The recovery capacity math in estimates (§3: ~1–1.5 days to re-encode one lost DC's
  shards — read k=8 survivors + write 4 replacements per stripe — at full cross-DC
  network budget) is the sanity bound: recovery from a DC loss is a
  throughput problem measured in hours-to-days, so the *steady-state layout* must already
  tolerate the loss — not recover into it.
- **Failure modes**: (a) losing 2 DCs at once (region event) — the archive class's DC-
  spread parity must include a DC far outside the affected region pair, so placement
  policy per class needs a DC-diversity constraint review, not just "spread shards"; (b)
  after a big loss event, repair backlog vs new-failure race inverts — recovery
  orchestration should throttle *new* ingest ahead of durability rebuild (write-pause is
  the right trade: users wait hours for PUT, durability doesn't wait);
  (c) stale cross-DC copy (async lag) + immediate DC loss = small data-loss event — bound
  and *advertise* the lag per class instead of pretending it's zero; users who need a
  tighter window choose a stronger class, which is the honest version of the durability
  promise.
