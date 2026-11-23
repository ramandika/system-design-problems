# 02 — Scale estimates

Assumptions (adjust and redo arithmetic if they change):

- Google Workspace scale: publicly reported *billions* of accounts across Workspace; take
  ~1B accounts, ~300M DAU (est.); docs (word-processor product) edits ~50M docs/day (est.).
- **Concurrent editors**: peak ~10M users actively editing, average ~5M (est. — editing is a
  work-hours, work-day activity; 10M / 300M DAU ≈ ~3% of DAU at peak).
- Editing behavior: an active typist produces ~2 op-batches/s while typing (keystrokes
  coalesce into small batches); *typing duty cycle ~20% of editing time* (users read, scroll,
  think).
- Concurrency shape: most active docs have 1–3 editors; the tail (meeting agendas, lecture
  notes, interview docs) has 50+; per-doc fanout is the tail's problem, not the mean's.
- Doc size: mean text ~30 KB (tail into multi-MB); doc opens ≫ edits: ~2B opens/day (views
  ≈ 10–40× edit sessions).

## Operation throughput (the load that shapes the sync design)

- Fleet-wide op rate at peak: 10M editors × 2 batches/s × 0.2 duty ≈ **4M ops/s**.
- Op size: 1–3 chars + revision number + author ≈ ~100 B → **~400 MB/s** of op ingest at
  peak — *bytes are trivial; ordering and fanout are everything*.
- Per-doc math (the number that actually matters): a busy doc with 20 participants and 5
  simultaneous typists runs ~10 ops/s; the server's outbound fanout for that doc is
  20 subscribers × 10 ops/s = **~200 msg/s per busy doc** — cheap per doc; a 200-participant
  doc with 20 typists runs ~40 ops/s, so fanout is 40 × 200 = ~8K msg/s outbound — still what
  a single service node's connection set handles, which is why *per-doc* sharding works
  (03-architecture). Per-doc ops average 0.4 × k ops/s for k editors (2 batches/s × 0.2
  duty); fanout is 0.4 × k² outbound msg/s while that doc's editors are at duty (estimate).
- The naive client-to-client design (mesh) fails structurally, not at scale: N editors each
  transforming against an unconverged mix of others' ops is where divergence is born — the
  star topology (deep dives §2) is a *correctness* choice that happens to also make fanout
  linear.

## Read path

- Doc opens: ~2B/day ≈ **23K QPS average**, diurnal ~3× ≈ **~70K peak** — every open is: ACL
  check, doc fetch (snapshot + recent ops), presence join. Doc text is small (est. below) and
  cacheable *per viewer* after ACL evaluation (viewer-independent content, ACL at service —
  the same pattern as [42.linkedin/04-deep-dives.md](../42.linkedin/04-deep-dives.md) §3).
- Read:write is lopsided but *not* the design center: ops (4M/s peak, small) and opens (23K
  QPS, bigger payload) are different paths; both are small by fleet standards. The design
  effort concentrates in correctness (ops) and tail latency (opens at 200 ms with cold ACL).

## Storage

- Corpus: ~5B docs (est.) × ~30 KB mean text ≈ **150 TB of document text** — small.
- Revision/ops history: op logs ≈ ~10× current text (deleted/rewritten content lives in
  history; the mean doc's lifetime revisions exceed its current size) ≈ **~1.5 PB**; hot docs
  (recently edited) keep ops in a fast store; cold docs' history pages to object storage.
- Metadata (ACLs, comments, presence state, sharing graphs): tens of TB.
- Snapshot cadence: periodic compaction (log replay → full snapshot every ~1K ops or ~1 h
  per active doc) keeps open-time replay bounded (~[40.tinyurl](../40.tinyurl/README.md)'s
  lesson in a different shape: bound the work of the *read* path by precomputing).

## Latency budgets (the numbers the design must satisfy)

- Local keystroke → screen: < ~50 ms (client pipeline, no network dependency).
- Typist's op → collaborator's screen: p95 ≤ ~200 ms = network RTT + service hop + fanout +
  the collaborator's render; batching budget ~50 ms (a bigger batch window breaks the 200 ms
  bar; a smaller one multiplies op volume — deep dives §2).
- Presence: ≤ ~1 s, lossy, no retry requirement.
- Doc open (service-side): cold ~200 ms p95 (ACL check + snapshot + recent-ops fetch);
  warm ~50 ms (estimate — cached snapshot + hot ACL).

## Consistency check

- Op-rate sanity: 4M ops/s ÷ 10M editors ≈ 0.4 op-batches/s per editor *including* non-typing
  time; during typing at 2/s with 20% duty — the same number seen two ways ✓.
- Storage sanity: 50M docs edited/day × ~10 ops/active-minute... skip; direct check: ~1.5 PB
  history vs 150 TB text implies ~10× lifetime-op overhead per doc ✓ consistent with the
  "documents are rewritten more than they grow" reality of word processing.
- Fanout sanity: fleet op ingest 400 MB/s (est.); a doc with k editors sends 0.4k ops/s
  average to its k subscribers → 0.4k² outbound msg/s (estimate, per-op ~100 B). With the
  stated shape (most docs 1–3 editors, tail 50+), the editor-weighted mean is ~3–5
  subscribers → ~0.4 × 4 × 10M ≈ 16M msg/s ≈ **1.6 GB/s** of fanout traffic (est.) — real,
  and *still* small by fleet egress standards
  (compare [43.zoom/02-scale-estimates.md](../43.zoom/02-scale-estimates.md)'s ~40 Tbps media
  fleet); the per-doc extremes above (200 msg/s, 8K msg/s) are what stress a single node, and
  it is latency, not
  bandwidth, that makes this hard.
