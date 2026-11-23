# 02 — Scale estimates

Assumptions (adjust and redo arithmetic if they change):

- ~1B registered members (publicly reported ~1B since 2023); engaged members are a small
  fraction of registrations — assume ~300M DAU, ~400M MAU (estimates, not reported figures)
- Median connections modest (~100–300) but a visible heavy tail; product cap 30,000
  1st-degree connections. Assume average ~400 connections per active member
- Profile doc ~5–10 KB (positions, education, skills, endorsements — JSON-ish)
- Feed opens ~5×/member/day; profile viewing ~10 profiles/viewer/day (estimate)

## The graph

**Edge count**: symmetric graph, one row per direction in an adjacency list (or one
half-edge stored once, see deep dives):

- ~300M active × ~400 connections ≈ **120B edge endpoints** (≈ 60B unique pairs + tail).
- Adjacency storage: ~400 entries × 8-byte member ID ≈ 3.2 KB/member hot list → 300M ×
  3.2 KB ≈ **~1 TB** of raw adjacency; 2–4× with attributes (connected date, relationship
  context). Small *per member*, huge globally — one shard per member works.

**Dist-2 size is the real problem**:

- Median member: 300 friends × ~300 of theirs ≈ up to **~90,000 2nd-degree nodes** before
  dedup; after dedup order tens of thousands.
- Heavy member at the 30K cap: 30K × ~300 ≈ 9M endpoints pre-dedup → millions of distinct
  2nd-degree nodes. Materializing full dist-2 *lists* for such members is ~72 MB each
  (9M × 8-byte IDs, more with attributes) — impossible to serve hot; hence the
  precompute-aggregates-not-lists decision (deep dives §1).

## Read paths

**Profile views**: 300M DAU × 10 profiles/day ≈ 3B profile views/day ≈ **35K QPS average,
~100–150K QPS peak** against profile + graph annotation (is-2nd-degree badge adds a lookup
per view). Cache is what makes this cheap: hit ratio ~95%+ keeps origin reads at low single-digit
K QPS (→ [10.distributed-cache](../10.distributed-cache/README.md)).

**Profile writes**: profile edits are rare — assume 1 edit per active member / 2 weeks ≈
20M/day ≈ 230 QPS. **Read:write ratio ≈ 150:1** (3B views vs 20M edits) — still massively
read-dominated; any design that recomputes visibility or
rebuilds indices on write wins for free; on-write invalidation is affordable, on-read ACL
computation is not.

**Connection requests**: assume 1 invite per active member per week ≈ 43M/day ≈ **500 QPS**
average. Tiny write rate — but every accept triggers: 1 atomic two-way edge write +
invalidations (dist-2 caches, PYMK, feed features) + a notification. Bursts around
recruiting cycles are ~5–10×, still small; the *notifications* (email/push) are the fanout.

**Degree-bounded graph queries** ("how do I know X", search badges, "who from my company"):
assume 3 such lookups per DAU per day → 900M/day ≈ **10K QPS lookup QPS** (900M ÷ 86,400 s ≈ 10.4K), each answered from
precomputed dist-2 stores, not BFS.

## Feed

- 300M DAU × 5 opens/day = 1.5B feed requests/day ≈ **17K QPS average, ~80–100K peak**.
- Each request scores thousands of candidates: 17K QPS × ~2–5K candidates ≈
  **~35–85M candidate scores/second** before any optimization — this, not the request QPS,
  is the workload. Multi-stage funnel (retrieval → light rank top-hundreds → heavy model
  top ~tens) collapses it by ~10× per stage (deep dives §4, generic treatment in
  [15.ml-ranking-system](../15.ml-ranking-system/README.md)).

## Storage

- **Profiles**: 1B members × ~10 KB ≈ **10 TB** raw, ~3–5 TB compressed; sharded by member ID.
- **Connections adjacency**: ~1 TB (above) + pending-invite table (small: live invites for
  open sessions are tiny, ~10s of GB).
- **Dist-2 derived store**: cannot store full lists for heavy users (above); per-member
  aggregates + sampled paths: ~10–100 KB/member ≈ 300M × 50 KB ≈ **~15 TB** spread across
  shards; rebuilt continuously from events.
- **WVYP / event analytics**: view events 3B/day × ~500 B ≈ 1.5 TB/day before dedup →
  real-time OLAP for ~5–30 days (tens of TB), weekly rolls retained long-term.

## Consistency check

- 3B profile views/day ÷ 300M DAU = 10 views/viewer/day — matches assumption. Targets
  viewed/day is the same 3B (each view has exactly one target; no double counting either
  side); 3B targets ÷ 300M reachable targets ≈ 10 views/target/day on average, with heavy
  skew to recruiters and popular profiles — cache hit ratio concentrates there, consistent
  with the ~95% goal.
- Edge math sanity: 120B directed endpoints ÷ 1B members ≈ 120 endpoints/member average —
  lower than our 400 average for *active* members, right because most of the 1B members
  are dormant with near-zero connections. Redo the product with a bimodal distribution;
  conclusions unchanged, per-active-shard sizes just get the tail treatment.
- Feed scores/sec: our ~35–85M/s estimate is why ranking is a *systems* problem, not just
  an ML problem — a ranker that takes 1 ms per candidate is unusable at this rate.
