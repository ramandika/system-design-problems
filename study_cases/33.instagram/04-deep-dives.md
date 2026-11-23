# 04 — Deep dives

## 1. The upload + transcode pipeline: what does "posted" actually mean?

The tweet pipeline stored a string and fanned out an ID. Here the unit of content is a multi-MB
file that is *unusable* until it has been re-encoded: a 6MB 4032×3024 original would blow every
mobile data plan and every CDN cache key. So the first design question is what the synchronous
path promises.

**Option A — synchronous pipeline** (post blocks until all renditions are ready):
- wins: correctness is trivial — the post is fully servable the instant the ack lands; no
  intermediate states to explain to users; cheap on infrastructure (no queue state).
- loses: post latency = transcode latency (multi-seconds for a video ladder at best), and any
  transcode worker outage means *no posting at all* — the write path inherits a compute farm's
  reliability; the user is staring at a spinner during their 3-second window of attention, which
  is the one UX crime Instagram cannot commit.

**Option B — fully async, ack first** (ingest stores original, ack, transcode later):
- wins: the ack is user-perceived-latency bound only by the upload itself (~seconds); a transcode
  backlog delays *visibility* but never *acceptance*; worker fleet can be right-sized for average
  load with retries for peaks.
- loses: the post exists in a `PENDING` state where it is stored but not shown — you need an
  explicit state machine (PENDING → SERVING → FAILED/DELETED) and a rule for what readers see
  (rule: not shown at all, i.e. absent, which is *not* a lie as long as it converges within
  seconds); also you accept the original blob possibly twice (if the client retries) —
  content-addressed keys dedupe that for free.

**Option C — client-side encoding** (the phone produces the first rendition; server only repacks):
- wins: the uplink bytes drop (send a 1.5MB compressed draft instead of 6MB) and the first
  "good" rendition is ready with no server round-trip; the original, if we still want it, rides
  behind the draft at low priority.
- loses: device fleet diversity means inconsistent output quality across phones; a phone
  encode of a video can take real battery; bugs in client encoders ship to a fleet of ~billions
  and cannot be hot-fixed as fast as a server fleet.

**Chosen: B with C for photos.** Ack on original stored (photos: client-compressed draft is *the*
first rendition and the original follows async; videos: client sends a low-res draft and the
server builds the ABR ladder async). We pick B+C because the user's intent — "this is posted, I
can close the app" — is decoupled from a compute farm's latency, and the async state machine is
reusable for stories and deletion alike. Option A survives only as a subset (thumbnails ARE
synchronous-ish: a post should render *something* within seconds).

**Failure modes**:
- Transcode worker crash mid-ladder → partial renditions. Mitigation: renditions are written
  immutably and the state machine only flips to `SERVING` when the *required* member set exists
  (at minimum: one image rendition / one small video rendition); the rest fills in lazily and the
  client ABR-manifest is regenerated when the ladder completes.
- Client uploads original bytes twice (retry) → duplicates. Mitigation: content-addressed object
  keys means the second write is a no-op; metadata row keyed by client-supplied idempotency token.
- Orphaned `PENDING` rows (crash between "stored" and "event emitted") → posts that never serve.
  Mitigation: a reconciliation sweep finds `PENDING` older than a few minutes and re-emits.
- Deleted post still on CDN from stale URLs → takedown must invalidate CDN + indexes; because
  URLs are immutable per rendition, the invalidation set is enumerable from the metadata row.

Cross-refs: blob durability and layout in ../07.blob-store/04-deep-dives.md; CDN mechanics in
../09.cdn/04-deep-dives.md. Contrast ../18.sora: that case *produces* the bytes and its pipeline is
generative latency; ours is re-encoding at fleet scale.

## 2. Stories: TTL mechanics at ~1B items/day

The feed's core invariants are durability and slow change. Stories invert all of it: every item
has a hard death time, ordering is "active stories of people I follow, recently-posted first",
and there is per-viewer seen state plus a per-viewer view log *for the author* (Instagram shows
authors who watched — a product decision that makes this per-viewer state, not an aggregate).

**Option A — sweep/purge** (a background scanner enumerates expired items and deletes them):
- wins: conceptually simple; deletion logic lives in one place and can be stopped for maintenance.
- loses: at 1B items/day you need a full-table scan every few minutes over a billion-row churning
  index, and correctness under this model lags — a story that dies at 24h but is purged at 24h+45m
  has *leaked* for 45 minutes, which is a real product bug (users rely on stories disappearing —
  that's the product). Sweep cost also scales with table size, not with change rate.

**Option B — TTL-based storage** where the storage engine owns expiry (rows/members auto-expire;
readers can never see an expired item) + a *lazy* reaper that reclaims bytes:
- wins: read-path correctness is structural — visibility requires a live key, expiry is enforced
  by the store at read/write time (e.g. a per-viewer active-story list where membership entries
  carry `expire_at` and are filtered/skipped on read; blob objects carry a TTL at the storage layer
  so bytes die without an operator touching them); cost scales with change rate.
- loses: deferred reclamation needs a compaction/reaper process (else you pay storage for dead
  items until compacted — with ~1.5PB/day churn, that reaper is not optional); archive/highlights
  require an explicit promotion to a *permanent* key before expiry, i.e. a copy/rewrite step, and
  if that promotion misses the deadline the story is gone, full stop — best-effort archive is a
  product-accepted loss (a user-opted-in archive must at least be retried before the window ends).

**Option C — event-scheduled expiry** (each item's birth writes a "kill at t+24h" task to a
distributed scheduler):
- wins: exact per-item timing; easy to extend ("kill at 24h", "kill at 48h for close-friends").
- loses: 1B scheduled tasks/day adds a whole scheduler system to the critical path; task loss
  (a missed or duplicated timer task) silently becomes a leaked story — the weakest of the three
  for correctness under churn, and its cost is O(n) scheduled tasks, similar to A but spread
  instead of swept.

**Chosen: B, with C as the *fallback* (a small timer job that force-expires anything the store's
TTL missed, e.g. during a replication bug).** B wins because correctness lands in the read path
(a dead story cannot be served if serving requires a live entry — the best defense is not needing
a cleanup to be *right*), and the reaper only has to be *eventually* right about bytes, which is
the right trade: correctness for visibility, effort for storage reclaim. The 2024-era reality of
any big stories system is exactly this split of "structural read-time expiry + async reclaim".

The second half of the problem is **seen/view state**: `seen(author, item, viewer)` and the author's
view list. At 7.5B view-marks/day, per-viewer lists must be compact (viewer+timestamp, batched by
item, hot window of ~24–48h, then evicted — after expiry nobody can view, so the state itself can
shrink to the same TTL plus a grace period). Tray ordering uses recency + relationship strength —
i.e. tray is a *ranking* surface too, just a cheap one, so it reuses the feature store instead of
getting a bespoke system.

**Failure modes**:
- Clock skew between ingestion paths shifts expiry by minutes. Mitigation: the store's wall-clock
  TTL is set server-side only (client timestamps are advisory), with a small grace buffer added
  *in the safe direction* (display slightly longer, never shorter — a story vanishing early is a
  much angrier bug than lingering 30s).
- Fanout to viewers lags → a story is posted but not in followers' trays. Mitigation: trays are
  assembled on read from an active-story index (writer's index, not per-viewer copies) — a story
  appears as soon as *it* exists, not as soon as fanout caught up; per-viewer materialization is
  limited to seen-state, which can lag harmlessly.
- Archive-promotion misses the expiry boundary → lost archive. Mitigation: retry before expiry;
  accept residual loss as documented product behavior (best-effort).
- View-log privacy edge: private accounts must re-check viewer authorization at view-log serve
  time (a follow could have been revoked) — authorization is strong-consistent at serve time.

## 3. Hybrid fanout under a *ranked* feed (what Twitter's design can't give us as-is)

The celebrity problem is the same as ../02.twitter/04-deep-dives.md: fanout-on-write for everyone,
on-read merge for a small set of high-follower authors. But ranking changes both halves.

**Option A — reuse Twitter's design verbatim** (materialize a reverse-chron timeline cache, then
rank on read):
- wins: simplest possible fanout (pure append of IDs), simplest celebrity merge ("give me your
  recent 20 tweets"), the timeline is trivially inspectable/debuggable as a log.
- loses: on-read ranking means every feed open re-scores ~hundreds/thousands of candidates with
  fresh ML calls; at ~56K QPS average feed opens that re-scoring is repeated per *session* even for
  unchanged content — and worse, a reverse-chron cache truncates to "recent N ids", so a post's
  *lifetime in the feed* ends when N scrolls past, which caps how stale-but-relevant content can
  ever resurface; the ranker never gets a real candidate pool, only a recency window.

**Option B — rank-first fanout** (compute scores in the fanout; materialize a *ranked* timeline):
- wins: reads are fast (just pop the ordered list) and cheap.
- loses: scores decay — a like 10 minutes after posting flips the order, so you need continuous
  re-scoring of billions of entries; also ranking *quality* degrades, since the ranker at
  fanout time doesn't know the *viewer* (engagement is inherently viewer-content-pair scoring); a
  post-score can't express "this is a great photo for Alice specifically."

**Option C — candidate-pool fanout + rank-on-read-with-precompute** (materialize a per-user pool
of candidate items + *precomputed pair features*; a lightweight ranker scores the top slice at
session time, the heavy model re-scores only the first screenful):
- wins: decouples content-availability (fanout's job) from ordering (the ranker's job) — the pool
  is a *candidate set*, cheap to maintain by append + trim, and pair-featurization is done
  ahead-of-time in batch so session-time ranking only runs shallow scoring on ~1–2K candidates and
  deep scoring on tens; stale celebrities are merged on read and scored like anyone else.
- loses: two-stage ranking machinery is real ML-infra cost (a feature store, a scorer with
  SLAs, retrieval/recall budgets); worst case the feed API must degrade to reverse-chron over the
  pool when the ranker is down (that fallback is now a required product path, not a nice-to-have).

**Chosen: C.** The deciding factor is that ranking quality is the product (engagement-driven feeds
are Instagram's whole point), and ranking needs viewer-specific scoring which only exists at read
time — so fanout's contract is "keep each user's *potential* posts fresh", while ordering is a
separate, read-time concern. The candidate pool also caps storage: ~1–2K IDs + scores + packed
features per user at ~100B/item ≈ ~0.1–0.2MB/pool, i.e. ~100–200TB for a billion users (estimate)
— the media bytes never enter the timeline, and each pool is small enough that the hot front
slice sits in RAM.

**Failure modes**:
- Pool cache eviction mid-session → feed briefly shrinks. Mitigation: pools are cache-aside with
  a durable backing log replay (a lost pool is rebuilt from the event stream, with recency
  backfill of the newest N directly from the author's own post index).
- Celebrity merge misses a celebrity post (the "did you follow Taylorswift and not see her post"
  bug, same as Twitter's Earlybird gap). Mitigation: an author-level recent-posts index that is
  *not* fanout-dependent for celebrity authors; merge reads it directly (the Twitter analog: query
  expansion against a search index — publicly documented in 02.twitter sources).
- Ranker outage → full feed freeze. Mitigation: reverse-chron over the pool is served instead;
  user-visible but sane. Cross-ref the two-stage ranking architecture in
  ../15.ml-ranking-system/04-deep-dives.md.
- Unfollow must remove the author's items from the candidate pool (or filter at merge) — pool
  hygiene on graph changes is a real job (batch rewrite of the affected pools; a heavy unfollow of a
  prolific author needs filtering at read time until the pool catches up).

## 4. Explore / Reels: ranking hooks on the *global* graph

The home feed's candidate universe is "accounts you follow" — small, fanout-able. Explore/Reels is
"anything on the platform" — no fanout is possible, ever, because the candidate universe is
billions of items and the audience for any given item is unknown. This is a different retrieval
problem wearing the same UI.

**Option A — brute-force scoring** (score every item for every viewer at request time):
- wins: perfect recall; no index maintenance.
- loses: arithmetic kills it — billions of items × ~56K QPS of feed/explore opens is beyond any
  fleet; latency dies too (scoring is only cheap when the candidate set is ~1–2K, which is exactly
  what A cannot restrict).

**Option B — prebuilt candidate feeds** (a materialized "explore for user X" recomputed offline):
- wins: read is same shape as the home pool, simple API.
- loses: recency and personalization die together — offline recompute hours old misses the
  freshest viral content (Explore's whole value is "new right now"); materializing billions of
  per-user explore feeds is storage-heavy and useless for users who never open Explore.

**Option C — two-stage retrieval + rank, with hooks shared with the home feed**:
- wins: stage 1 (cheap, approximate retrieval — creator-graph expansion, embedding similarity,
  trending/engagement indexes) narrows billions to ~1–2K per viewer in milliseconds; stage 2 reuses
  the *same* ranker, feature store and engagement-signal infrastructure as the home feed, which is
  the whole point: like/save/dwell events feed one feature store, both feeds consume it.
- loses: recall quality depends on retrieval design (bad candidate generation = bad explore, no
  ranker can save it); the retrieval index itself is a hard piece of infra (embedding index +
  engagement aggregates + creator graph), with its own freshness and consistency requirements;
  and because candidates come from the global index, *content-level* moderation/eligibility must
  gate retrieval (a private or policy-violating item must be invisible at stage 1, not caught later).

**Chosen: C.** The engagement loop is the architecture: engagement events (like, save, share,
dwell/skip) → feature store → next scoring pass → more engagement. The home feed and explore
differ *only in the retrieval stage* — the ranker, features, signals and serving path are shared.
This is also why Instagram's ranking work is publicly described as a system of many retrieval
sources + multi-stage ranking rather than one model (public engineering write-ups from Instagram's
recommendation teams describe retrieval-then-ranking for Feed/Explore/Reels alike). The deep infra
detail (feature stores, serving SLAs, model rollout) lives in ../15.ml-ranking-system/.

**Failure modes**:
- Retrieval index falls behind real-time (a viral post exists but no candidate path finds it).
  Mitigation: a "trending/fresh" bypass lane ingests high-velocity items directly from the event
  bus with minimal featurization — biased toward freshness when features are thin, since the cheap
  ranker can still use author-follow-rate and initial engagement velocity.
- Engagement-loop poisoning (ranker amplifies a narrow content type → users disengage → features
  degrade). Mitigation: exploration budget (a small % of impressions spent on unproven candidates)
  and diversity guards at the merge step — a systems consequence of an ML choice, worth naming.
- Feature store staleness under event-bus backlog → ranker scores on old counts, feed feels stale
  after a burst. Mitigation: feature freshness SLOs per feature class; degrade to author-level and
  recency features (which are stable) when item-level features lag.
- Cold-start viewer (no engagement history) → retrieval leans on graph-of-follows and global
  trending; the home feed's follow-based pool hides this (explore exposes it) — accept a worse
  explore experience for new users rather than a personalized-but-wrong one.

## 5. Media delivery: renditions, URLs, and the 80:1 read ratio

From 02: media GET:upload ratio is ~80:1 and the CDN must absorb the overwhelming share
(../09.cdn for mechanics). The Instagram-specific deltas are **rendition policy** (what to
precompute) and **cacheable URL design** (how to make a social object *never need invalidation*).

**Rendition policy**:
- Photos: precompute a small fixed ladder (~4–6 resolutions, modern-format variants). The
  trade-off: precomputing wins cacheable immutable URL keys and near-zero origin CPU —
  transcoding at request time would shift render CPU to the origin fleet at 1.2M GET-QPS and
  destroy hit rates with per-request keys. It loses ~1.5–2× rendition bytes and wasted encodings
  for formats a client never pulls; the client-side hint below trims the waste.
- Video: ABR ladder (~4–5 renditions) + thumbnail. Per 02, video renditions ≈ 300TB/day of
  precompute — expensive, but the alternative (per-request transcode) is worse by orders of
  magnitude.
- Client-side hint: the client sends viewport/format hints *with the upload*, so the rendition
  ladder is sized once at post time rather than serving a ladder nobody needs.

**Cacheable URL design**:
- URLs are immutable and content-addressed per rendition (`/hash/w=1080/…`). Editing never
  mutates an object — an edited photo becomes a new object with new renditions and a new URL.
- wins: edge hit rates can be very high and stale-while-revalidate is trivially safe (immutable
  content can't be stale-wrong); takedowns are enumerated invalidations from the metadata row.
- loses: every edit/re-upload costs storage again (deduped only by exact hash reuse); URL scheme
  is a permanent public contract — changing rendition policy means *new* URLs, requiring a
  client-version rollout, since the client caches and prefetches URLs from feed responses.

**Failure modes**:
- Origin stampede for a viral item not yet cached (the same thundering-herd shape as any viral
  spike) → mitigate with origin shields + single-flight at the shield tier (../09.cdn).
- Deleted-but-cached rendition at an edge → invalidation plus short default TTLs for small renditions
  as a backstop; public content takedowns (CSAM/legal) need a fast global purge path separate from
  the normal CDN TTL regime.
- Rendition-generation bug ships a broken ladder → clients must fall back to the next available
  rendition (the ladder is ordered by preference, not exclusive).
