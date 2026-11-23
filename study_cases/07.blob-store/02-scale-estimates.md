# 02 — Scale estimates

All numbers below are **rough estimates for a modeled deployment** (a large-public-cloud
scale S3-like store), chosen so the arithmetic is self-consistent. Public anchors, labeled
as such: AWS publishes S3's design targets (11 nines durability, 5 TB objects) and has
publicly reported S3 holding hundreds of trillions of objects with peak request rates in
the tens of millions per second; no primary source publishes per-day byte volumes, so
everything volumetric here is our own model.

## Capacity & object count (the binding constraint)

- Design capacity: **1 EB (1,000 PB) logical object data**.
- Object count: **~10^12 objects** (a trillion) → mean object size 1 MB
  (1 EB / 10^12 = 1 MB). Realistic distribution: median well under 100 KB, long tail to
  TBs — the mean comes from the tail, so the store must handle both extremes with one
  layout or two.
- Tenants: ~50,000 application accounts; there is no human "DAU" — load is what tenants'
  users and pipelines generate.

## Erasure coding overhead

- Layout: Reed–Solomon-style RS(k=8, m=4) per stripe → **1.5× storage overhead**
  (12 shards per stripe vs 8 needed; tolerates 4 lost shards).
- Physical capacity for 1 EB logical: **1.5 EB**, plus ~1.25× for free-space headroom and
  in-flight repairs → provision **~2 EB raw**.
- Drives: assume 20 TB HDDs → 2e18 / 2e13 = **~100,000 drives**, ~30 per node →
  **~3,300 storage nodes** globally, spread over ~30 datacenters (~110 nodes each).

## Write path

- New durable logical data: **100 PB/day** (net of deletes; overwrite/rewrite churn adds
  the ×1.2 background factor below).
- Physical shard writes: 100 PB/day × 1.5 (EC) × 1.2 (repair background traffic) ≈
  **180 PB/day of shard writes**.
- At ~4 MB shard size: 180e15 / 4e6 ≈ 4.5e10 shard writes/day ≈ **~520K shard writes/s**
  mean fleet-wide. This number is why the data plane is parallel-commodity hardware, not
  shared-nothing fancy disks.
- Single-object PUTs vs multipart: assume ≥70% of *bytes* arrive by multipart for objects
  >100 MB (part-parallel is the only way to move TBs over a real network) — the multipart
  path is the main upload path, not an edge case.

## Read path & bandwidth

- Egress: 1.5× ingest ⇒ **150 PB/day** delivered (GET-dominated; read:write ≈ 1.5:1 by
  bytes).
- Bandwidth: 150e15 / 86,400 s ≈ **~1.74 TB/s ≈ 14 Tbps** sustained mean, peak ~3× ≈
  42 Tbps. Per datacenter (~110 nodes): ~470 Gbps mean — the front-end fleet and NICs, not
  the disks, start being the constraint at this point for hot keys.
- Hot keys: assume a power-law skew (tiny % of objects take most GETs) — a "hot key"
  throttle + internal cache layer is mandatory, same shape as YouTube's origin-shield
  reasoning (../01.youtube/04-deep-dives.md).

## Request rate (metadata pressure)

- Assume **500B requests/day** (500e9) — ~5.8M req/s mean, **peak ~17M req/s** (3× mean,
  consistent with "tens of millions" publicly reported peaks at larger scale).
- Mix by *request count*: GET/HEAD ~80%, PUT ~15%, LIST ~3% but LIST is the expensive one
  per request (prefix scans, fanout over index shards) — cap it separately.
- Metadata record: ~1 KB/object → 10^12 objects ⇒ **~1 PB of index**, trivially sharded;
  metadata *write rate* = PUT rate ≈ 0.15 × 5.8M ≈ ~0.9M writes/s — this is what shapes
  the metadata store, not the record size.
- Multipart accounting: 500B requests include per-part PUTs; `CompleteMultipartUpload`
  with 10K parts is one request that fans out to ~10K metadata updates + one large stripe
  assembly — bounded burst, queued by design.

## The numbers that shape the design

1. **1.5× EC overhead vs 3× replication saves ~1.5 EB of raw storage at this scale**
   (1 EB logical = 1.5 EB raw under RS(8,4) vs 3 EB raw under 3×; with the 1.25× headroom
   that is ~100K provisioned drives vs ~190K — ~90K fewer 20 TB drives) — the margin
   that pays the entire metadata/control-plane engineering team. (Replication wins only
   for small objects where per-shard overhead dominates — see deep dive 1.)
2. **Repair capacity budget**: the physical ceiling on fleet read bandwidth is the disk
   fleet itself: ~100K HDDs × ~200 MB/s ≈ **~20 TB/s** of total disk reads — a full
   scrubber pass over 2 EB raw = 2e18 B / 2e13 B/s = 1e5 s ≈ **~28 hours even if
   scrubbing got every disk in the fleet**. Steady state must steal ~10–20% of disk I/O
   instead, so a full verification pass takes **~1–2 weeks** (2e18 / (2e13 × 0.1–0.2) =
   5e5–1e6 s ≈ 6–12 days); verification frequency sets detectable-corruption latency,
   which feeds directly into durability math (deep dive 5).
3. **Datacenter-loss recovery is a throughput problem, not a yes/no problem**: losing one
   DC (~1/30 of fleet ≈ 33 PB logical, i.e. ~50 PB of EC shard bytes at the 1.5× overhead)
   and rebuilding its shards from surviving DCs. The lost shards are already EC-inflated,
   so the 1.5× is not applied again — but rebuild is not write-only either: with 12 shards
   spread over ≥3 DCs, a lost DC removes ~4 of 12 shards per stripe, and rebuilding a
   stripe reads the k=8 survivors and writes 4 replacements, so cross-DC traffic is
   ~2–3× the lost shard bytes ≈ ~100–150 PB → even at a full cross-DC network budget
   (~10 Tbps ≈ 1.25 TB/s): 100–150e15 / 1.25e12 ≈ 8e4–1.2e5 s ≈
   **~1–1.5 days** — the layout must make full-DC loss survivable without waiting on
   that (deep dive 5).

## Consistency check

- 150 PB/day egress ÷ 1 EB logical stored ⇒ 150e15 B/day × 365 ≈ 54.75 EB/year over
  1 EB stored ⇒ every byte is read **~55×/year on average** (~6.6 days
  mean-time-between-reads per byte if uniform; real skew means much hotter) — consistent
  with "HDD-backed with a small flash cache for hot ranges".
- 100 KB median objects vs 4 MB shards: bundling ~40 median objects per stripe only works
  with a log-structured shard file; per-object 4 MB-shard overhead for a 100 KB object
  (~2.5%×12 efficiency and 12 drives touched per read) is exactly why small objects need a
  different path — sanity checks the two-path deep dive.
- 500B req/day over 10^12 objects ⇒ each object touched ~0.5×/day on average — request
  rate is *not* the scale wall; capacity and repair bandwidth are.
