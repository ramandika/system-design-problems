# 04 — Deep dives

## 1. Threaded comment tree: storage and rendering (THE dive)

The comment tree is the defining data structure. A post is the tree root; every comment has
one parent; depth is effectively unbounded (capped ~10–12 rendered levels, deeper levels
open as their own page). A hot thread = 50K nodes, one partition, millions of reads/day
against continuous writes.

### Storage trade-off

**Option A — adjacency list** (`comment(id, parent_id, ...)`):
- wins: writes are trivial (one insert, parent's `child_count` bump); tree integrity is a
  single-column constraint; edits/moves are O(1). This is what a relational system of record
  wants, and it's why Postgres stays the source of truth even after we pick B for reads.
- loses: **reading a subtree is recursive** — "all replies under comment X at depth 3" is a
  self-join repeated per level. On a 50K-node tree the read becomes a multi-query crawl
  (or an expensive recursive CTE), ~O(depth) queries or one big CTE that scans the whole
  post partition. Against the tens of millions of reads/day a hot thread draws (02: ~5M viewers ⇒ tens of millions of tree reads), this read cost is unpayable.

**Option B — materialized path** (each row stores its full root-to-node path, e.g.
`t3_abc,t1_x,t1_y`):
- wins: **subtree fetch = one range scan on a path prefix** — `WHERE path LIKE
  't3_abc,t1_x,%'` (or a compound key `(post_id, path)` with prefix rows contiguous in a
  clustered/Cassandra layout). Every render operation the UI actually performs — "give me
  the children of this node", "give me this branch's first N pages" — is a prefix query.
  Sorting can be pushed into the same scan when path components are ordered IDs.
- loses: a comment's row contains its ancestors' IDs; renaming/moving a node rewrites the
  whole subtree (fine — Reddit doesn't support reparenting); path strings grow with depth
  (bounded by the render depth cap, so it's ~16 B × depth, negligible); the parent pointer
  is denormalized from the path so the DB itself cannot enforce tree integrity — the write
  path must (the Postgres adjacency record remains the integrity source, the path store is
  derived).

**Option C — closure table** (one row per ancestor-descendant pair):
- wins: any-subtree, any-relative queries are pure joins; works well for small-medium trees
  with ad-hoc queries.
- loses: O(nodes × depth) rows — a 50K-node thread averages ~2–3 ancestors per node, so
  ~150K rows per viral post, all written during the hot window; the write cost lands exactly
  on the partition that's already hot. And the query it can't express as cheaply as B is
  the one we need most: paged prefix scans with the tree's own order.

**Chosen: B for the read store, A as the source of truth.** Option A wins write simplicity
and integrity but loses at exactly the read pattern we estimated to dominate (02: hot
threads are read ~100:1 while being written). Option C wins query generality nobody needs
but pays write amplification nobody can afford. Option B wins the dominant read (prefix
scan), costs a derived store that must stay reconciled with the Postgres adjacency truth —
a reconciliation we accept because comment mutation rates are tiny relative to comment
reads. This is essentially what Reddit itself did publicly when it moved comments into
Cassandra keyed by a per-comment `path` of IDs.

### Rendering trade-off

One page cannot (and should not) deliver a 50K-node tree.

- **Default render**: root-level comments in the chosen sort, then children of each in the
  same sort, to a node budget (~200 per page). Beyond the budget, a **"load more comments"
  stub** (count + path prefix) replaces the rest of a branch — one click issues
  `render_thread(post, sort, that_prefix)` and the storage work is one prefix range scan,
  not a crawl.
- **Depth cap** (~12 levels): rendered as a hard truncation; a thread "continues" on a
  dedicated page whose root is the cut node, whose storage key is exactly that node's path.
  Option A wins nothing here (its reads pay the crawl again); B's cost model is uniform at
  every depth, which is the actual reason the cap is cheap.
- **Sorting within the tree**: score-sorted children must be re-ordered when votes change,
  so tree caches are keyed by `(post_id, sort_algo, sort_version)`; a vote burst bumps the
  version and the next read rebuilds that page (deep dive 2/3). Cache miss cost is bounded:
  one prefix scan + in-process sort of ≤ one page's worth of retained nodes.
- **Cache**: per-page Redis entries, TTL hours, explicit invalidation on comment mutation;
  a "score-only" delta does not invalidate the tree version, only the displayed counts
  (which are allowed to be approximate — see 02).

### Failure modes

- **Hot-key partition**: one post → one Cassandra partition receiving every comment write of
  a viral thread. Mitigate with in-memory write batching at the service layer (comments for
  one post flush together) and by keeping the per-write work tiny (one row + one counter).
  Do NOT solve it with per-comment partitioning — that destroys the prefix scan, which was
  the entire point.
- **Cache stampede on version bump**: a front-page vote burst bumps sort-version on a page
  that 1000 QPS is reading; all readers miss and hit Cassandra simultaneously. Mitigate
  with request coalescing (single flight per version) and a short grace window serving the
  previous version's page (scores on the page may be a few minutes stale — allowed).
- **Path corruption** (a comment written with a wrong/cyclic path): tree integrity is
  enforced only at the Postgres layer, so a bug in the derived-path writer can fragment a
  subtree silently. Mitigate: the adjacency record in Postgres is the arbiter; a daily
  reconcile job walks adjacency order and diffs against path order; mismatched threads are
  repaired from Postgres.
- **Deep-thread abuse**: a single comment chain tens of thousands deep is a storage
  amplification (long path strings) and a render abuse vector. Cap path depth at write time
  and refuse/redirect deeper replies.

## 2. Vote ingestion: a firehose with undo semantics

Votes are ~2.2K QPS average, tens of thousands of QPS at peak, each one tiny, each one
potentially **changed or withdrawn later** — and every consumer (counters, ranking, user
profile stats, mod tooling) must see an idempotent stream.

**Requirements the arithmetic forces** (02): accept tens of K QPS bursts for hours; never
double-count; make a vote's visibility eventual in seconds; never let voting block content
writes.

**Option A — synchronous transactional writes** (vote = SQL row per (user, content), update
counter in the same transaction):
- wins: perfectly consistent, trivially undoable (delete the row, decrement the counter).
- loses: at even a few hundred QPS burst on a single hot post (the 20K–45K QPS global vote
  peak is spread across all content; one front-page post concentrates perhaps tens to a few
  hundred votes/sec), the counter row is a single hot key contended by every transaction;
  latency couples voting to content reads; a viral event is a coordinated thundering herd on
  one row. The failure mode we most want to avoid — losing the ability
  to vote during the exactly the moments that matter (brigades, live events) — is built in.

**Option B — event-log ingestion with idempotent consumers**:
- wins: the API acks after publishing; consumers batch 100s of vote events per flush;
  counters are eventually-per-second aggregations (Redis HINCRBY or Cassandra counters),
  not per-event SQL rows; backpressure hits vote *staleness*, never content writes.
- loses: votes appear with seconds of lag; the system must carry dedupe state and a
  reconciliation mechanism; operator debugging means reading an event log instead of a row.

**Chosen: B.** Option A wins transactional simplicity but loses at the burst shape votes
actually come in (front-page events), and it converts an ingest problem into a lock-contention
problem on one row. Option B wins throughput and isolation, costs eventual consistency we
already accept (02: displayed scores are approximations and even deliberately fuzzed for
anti-abuse), and needs explicit idempotency machinery — which is engineering, not a
showstopper.

**Idempotency design**: every vote event carries `(user_id, content_id, direction,
event_ts, client_request_id)`. The consumer keeps the **user vote ledger** — current state
per `(user, content)` plus a `revision` — keyed by user, sharded by user ID (spreads hot
posts across the ledger, which is the key insight: the *counter* is hot-keyed by content
but the *ledger* is hot-keyed by users, and they never collide). A replayed or duplicate
event whose revision is not newer than the applied one is dropped; a direction change is a
delta of `2 × direction` or `-direction` depending on prior state. Counters are thus
*derived from the ledger*, not accumulated blindly — recount jobs rebuild counters from the
ledger when they disagree.

**Counter storage**: per-content counters in a horizontally sharded KV. For a front-page
post, additionally shard the counter by `content_id_hash + N buckets` and sum on read if a
single counter cell still saturates (rare; only truly viral posts).

### Failure modes

- **Brigading** (off-platform coordination inflating one content item): visible as an
  abnormally high vote velocity in a short window — detectable at the consumer via
  per-post per-window unique-voter counts, feeding deep dive 5, not solved at ingest.
- **Vote-bombing abuse of the ledger**: a single user spamming votes across thousands of
  items hammers their own ledger shard (self-limiting) but also the consumer's per-user
  dedupe path — rate limit vote events per user at the edge *before* the log.
- **Counter/ledger drift**: consumer crashes mid-batch → counters applied without ledger
  update or vice versa. Mitigate: ledger write is authoritative and ordered (per-user key),
  counters are rebuilt from ledger diffs; a periodic sample-audit catches drift rate.
- **Delete/withdraw semantics**: retracting a vote must not leave the counter high (vote
  stuffing then withdrawal). Because counters are derived from the ledger, withdrawal
  converges on next flush — a bounded, acceptable lag, not a permanent inconsistency.

## 3. Ranking: hot score, Wilson, and the whole sort family

Ranking needs to be cheap and stateless over stored aggregates, because feeds are assembled
at read time (deep dive 5) and re-ranking on every read must be microseconds, not
milliseconds.

**The four sorts and what each is actually optimizing**:

- **hot** (content ranking): Reddit's publicly long-documented form is roughly
  `log10(max(|score|, 1)) + sign(score) * (seconds since a fixed epoch) / 45000`. Properties
  to take from it:
  (a) the log term means the 100th vote matters far more than the 1st (first votes are
  noise/brigade-prone; log dilutes them), (b) the signed linear age term is what makes
  "newer wins ties" a smooth gradient rather than a cutoff — and note the sign sits on the
  *age* term, not the log term, so fresh downvoted content ranks lowest and old downvoted
  content cannot linger above fresh positive content, (c) 45000 (= 12.5 h) is a scaling
  constant —
  the actual feed freshness is *tunable without recomputing history*, since scores and ages
  are both stored raw. This is why hot is cheap: it's one arithmetic formula over two
  integers, computed at read time — no precomputation, no invalidation, no stale-sort problem.
- **best** (comment sort): Wilson lower bound on the *upvote ratio* at ~95% confidence,
  `z = 1.96`:
  `(p + z²/2n − z·sqrt(p(1−p)/n + z²/4n²)) / (1 + z²/n)` where `p = ups/(ups+downs)`,
  `n = ups+downs`. Why Wilson and not raw score: a comment with 3 up/0 down (p=1.0, tiny n)
  looks perfect by raw ratio but should rank *below* 100/5 (p=0.95) — Wilson's lower bound
  encodes "we're 95% sure this is good *given the sample size*" and shrinks the small-sample
  score toward uncertainty. For comments this matters a lot more than for posts, because
  comment vote counts are small (median comment gets single-digit votes) — exactly where
  naive score lies the most.
- **rising**: score velocity in a recent window — `(score in last T) / T`, not absolute
  score. Needs a sliding-window counter per post (in the rank cache, fed by vote consumers),
  roughly a "what's hot in the last hour" view.
- **top**: raw score over a period (day/week/month/all) — precomputed per window or served
  from counters as score = ups− downs with the window filter applied to the vote ledger
  timestamps at query time (only for smaller community feeds; the global top is
  precomputed).

**Trade-off: stateless formula vs learned model.** Option (stateless formulas) wins
explainability (mods see exactly why a post is hot), tunability in code, zero training
infrastructure, and O(1) recompute per read; loses: no personalization, exploitable by
known behavior (log-dilutes-brigade is public knowledge). Option (ML ranker) wins
personalization and anti-abuse signal integration; loses: requires constant retraining,
serving, and a feedback loop that itself becomes an abuse target, and its recompute per
feed read is no longer free. We pick formulas as the backbone and treat ML as an overlay
(re-rank top-N candidates only), because formulas can meet the latency budget at every
fan-out level in deep dive 5, and ML can't.

### Failure modes

- **Vote-brigading a rising post**: velocity is exactly the signal a brigade wants to game.
  Mitigations: log dilution in hot, per-window unique-voter velocity from the vote
  consumer, and classifier flags from deep dive 5 demoting suspected-brigaded content.
- **Wilson distortion under vote fuzzing**: displayed counts are fuzzed publicly (02) but
  ranking must use *real* internal aggregates, or the ranking absorbs anti-abuse noise it
  never asked for. Keep the rank aggregates separate from the display aggregates.
- **Hot formula drift with traffic growth**: 45000 is a constant; if the site's traffic
  grows 10×, hot content ages "too slowly" relative to newer content. It's a config change
  (constant is tunable), not a recompute — but only because the formula was kept stateless;
  an ML ranker would have absorbed this silently and become undiscoverably stale.
- **Sort-cache stampede after a large vote burst**: same coalescing + last-good-version
  serving as deep dive 1's sorting failure mode.

## 4. Subreddit sharding and per-community moderation

The community is the natural unit for both routing and permission. But a naive "one shard
per community" breaks on both ends of the size distribution.

**Option A — shard everything by community (subreddit_id = shard key)**:
- wins: reads for r/<sub> are single-shard; mod actions and community ACL checks localize;
  a community's outage is contained to itself; mod tooling hits one partition. Conceptually
  clean — moderation state (bans, rules, automod) is *per community by definition*, so
  co-locating that state with the community matches the domain.
- loses: the largest communities have traffic several orders above the median (~50M-subscriber
  flagship subreddits vs a 50-person hobby sub) → one shard is a hotspot; and anything
  global (front page, r/all, user karma, cross-community spam correlation) fans out over
  every shard.

**Option B — shard by content (ignore communities)**:
- wins: perfectly even load; the content plane never has a hot key it can't handle
  (individual posts/comments hash widely).
- loses: every community-scoped operation (the feed, the mod queue, ACL checks) becomes a
  scatter-gather; moderation state must be replicated to wherever the community's content
  lives.

**Chosen: A for state and control-plane (community metadata, mod state, ban lists, rules),
B for the content plane (posts/comments/votes sharded by content ID), with a service-layer
split.** Option A alone wins locality but loses on the size skew (a ~50M-subscriber
community's vote+read burst is a shard-killer). Option B alone wins load balance but loses
the locality that moderation needs. The split costs a lookup (community_id → state service)
per request, paid at the control plane only, not on content reads.

**Routing rule for mega-communities**: the top ~1K communities by traffic are pinned to
dedicated capacity (their own KV partitions for hot caches, their own mod-queue lanes and
rate-limit buckets). This is not elegant, but it converts an unbounded skew problem into a
bounded, monitored one — same shape as celebrity-follower handling in
[../02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md) (hybrid fanout with a
bypass threshold).

### Per-community moderation model

- **Permission layer**: `mod_action(user, subreddit, verb)` enforced against a per-community
  ACL — mods, automods, bots with scoped credentials. Every action writes an immutable
  audit log row (who, what, target, timestamp, reason) — this log is what the mod queue,
  appeals, and admin oversight all read from.
- **Mod actions as events**: removal/lock/ban/distinguish produce mod events on the same
  log as votes; all downstream consumers (feed assembler, ranker, caches, mod queue) apply
  them uniformly. Removed content drops from feeds within seconds; restored content
  reappears the same way. No feed rebuild needed — event-driven invalidation again.
- **Automod engine**: a per-community rule set evaluated synchronously on every content
  event (new post/comment, new vote burst), with action = filter/approve/remove/report.
  Rules live with the community (option A localizes them); evaluation shares the edge's
  bot heuristics (deep dive 5).
- **Site-level trust & safety sits *on top of*, not inside, mod state**: site bans,
  quarantines, and legal/ToS removals are admin actions that cannot be undone by mods —
  a two-layer state machine analogous in shape to app-review state machines in
  [../06.apple-app-store/04-deep-dives.md](../06.apple-app-store/04-deep-dives.md), but
  with thousands of parallel *community-level* reviewers beneath a thin central layer.
  That reviewer multiplicity is Reddit's actual difference from a centralized store:
  it makes moderation volume tractable, at the cost of *inconsistent community rules*
  (this is accepted policy design, not a system bug).

### Failure modes

- **Mod action storm on a viral post**: a mega-community's mod team can remove hundreds of
  comments per second during an event; per-community event volume spikes. Bound per-community
  event throughput with queues + per-community rate limits rather than letting it flood the
  shared log partitions keyed by community.
- **Cross-shard community metadata cache** (e.g., "is this user banned from r/X" is checked
  on every post/comment): ban lists change rarely, are read constantly. Cache in Redis keyed
  `(subreddit, user)` with explicit invalidation on ban/unban events; do not make this a
  synchronous read to the mod service on every content write.
- **Moderator account compromise**: a compromised moderator in a huge community is a
  content takeover for that community. Mitigate at the control plane: rate limits on mod
  action rate per account and per community, action log review tooling, admin-side
  kill-switch per community that freezes all mod actions (rarely used, must exist).
- **Shard rebalancing as community traffic drifts**: a small community growing large sits
  on shared partitions. Rebalancing is by traffic tier, not per community — accept some
  skew within "hot" and "cold" tiers.

## 5. Feed assembly: home, r/all, and the sort family

Home feed (subscribed communities) and r/all (everything public) must merge across all
communities — the arithmetic in 02 says ~12K QPS average, ~60K QPS peak, and each request
must rank and hydrate ~25–50 posts. Twitter contrast: Reddit is **fanout-on-read**, because
the content pool is shared (everyone in a community sees the same posts) rather than
per-author (every follower sees a different timeline).

**Option A — fanout-on-write** (materialize a per-user home timeline on each post, like
Twitter's):
- wins: reads are O(1) and latency-proof.
- loses: a hot post in a ~50M-subscriber community would materialize into ~50M timelines
  within seconds, and it would be re-fetched by a majority of those users; the write
  amplification is O(subscribers) × O(community size distributions) — with hundreds of
  thousands of communities this multiplies badly, unlike Twitter where the follower graph is
  per-user
  and the fan-out is the *whole* cost. Also: no per-user signal (which posts you've already
  seen, hidden, or downvoted) is naturally captured — a materialized list has to be
  post-hoc filtered.
- verdict: with Reddit's community structure (many-to-many, shared pools, high overlap of
  "who reads what"), fanout-on-write's amortized win never appears; its cost is strictly
  worse.

**Option B — fanout-on-read over per-community candidate pools**:
- wins: no per-user write amplification; the candidate pool per community is shared across
  every reader of that community (cache hit ratio scales with users-per-community, which is
  Reddit's actual strength); the read can merge per-user signals (seen/hidden/voted) at
  the last moment, where they're fresh.
- loses: read cost is the sum of fetching per-community top-N candidate lists + merging;
  ~60K QPS peak × merge work is real; must bound the merge pool (see below).

**Chosen: B, with hard bounds.** Option A loses on Reddit's specific distribution (huge
communities multiply fanout cost that Twitter's flat follow graph doesn't incur); B loses
on read cost but that cost is bounded as follows.

**Merging with bounded cost**:
1. Per-community candidate lists: precomputed per sort (new / hot / rising / top-window),
   cached, ~top 100 posts per community per sort. Hot and rising are cheap enough to
   recompute in the cache refresh on a short TTL (minutes); top windows are precomputed.
2. Home feed: for each subscribed community (median a few tens of subscriptions, heavy users
   in the low hundreds — estimate), multi-get the per-sub top lists. A user subscribed to 500
   communities is the worst case — the merge caps pool size at ~1–2K candidates and weights by a
   per-user/sub activity score rather than exhaustively merging all 500.
3. r/all: cannot enumerate 100K communities. Served from a **global hot cache** — the vote
   consumers maintain a real-time global hot list (~top 1K posts across all communities,
   plus per-tier sampled lists for reach); r/popular and r/all are merges of these lists,
   never a cross-community query. This is the escape hatch that keeps fanout-on-read from
   scaling with community count.
4. Hydrate: multi-get the ~25–50 post bodies + a batched vote-score read; comment counts
   come from counters.

**Sort family at the feed level**: the same formulas from deep dive 3 rank the merged
candidates; new is chronological on `created_at` index; rising is the velocity list; hot
recomputes per read (cheap); best is a comment sort (applies inside a thread, not a feed).

### Failure modes

- **Stampede per-community candidate lists** on a breaking-news event: many subs' hot lists
  expire simultaneously. Mitigate: staggered TTLs (jittered), request coalescing, and
  serving last-good list while refreshing (feed content minutes-stale is allowed).
- **Merge blowup** for power users subscribed to thousands of communities: cap pool size,
  prefer communities with recent user activity; degrade gracefully (some subs get sampled
  out) rather than timing out.
- **Global hot list hot-key**: r/all's list is read by everyone and written to by vote
  consumers constantly. Keep it as a small replicated list (a few KV entries), not a
  computed-on-read aggregation.
- **Seen/hidden/voted filters at merge time** must not force a per-user read of a large
  filter set — bound to recent items (last N posts seen in session), accept occasional
  re-showing of a recently-read post rather than building a per-user content index.

## 6. Spam and anti-abuse

Pseudonymous accounts + public votes + community moderation = the abuse surface. The design
goal: **make the economic cost of abuse higher than the value of the abuse**, layer by layer,
because a single clean signal does not exist (bots are cheap, votes are cheap, and every
hard identity check on Reddit would destroy the product).

**Layer 1 — Edge (pre-write)**: per-user, per-IP, per-community rate limits on posts,
comments, and votes separately (votes are far higher-rate and must have a much looser
limit); basic bot heuristics (client fingerprinting, request-shape anomalies); account-age
and karma thresholds gating high-impact actions (posting links in large communities). This
layer is what keeps the ingest firehoses in deep dive 2 from carrying obvious bot volume.

**Layer 2 — Content classifiers (async, on the event log)**: the same content events feeds
feed ML classifiers for spam/scam/botness, plus velocity anomaly detection
(per-community/per-post vote velocity, unique-voter ratios, correlated IPs). Output is a
*signal*, not a verdict: it either auto-moderates (high confidence), flags into the mod
queue (medium), or downweights in ranking (low, e.g. demote in feeds for N minutes while
observing). Async because a write must not block on a model; the cost is that some spam is
visible for seconds — acceptable (bounded blast radius via deep dive 3's log dilution of
early votes).

**Layer 3 — Vote abuse specifically** (this is where the Wilson/hot choices in deep dive 3
come back): vote brigades show up as a spike in vote velocity per content item with a high
ratio of new/low-karma accounts, or as correlated vote sequences. Countermeasures:
- **fuzzed display counts** (publicly stated by Reddit) — removes the tight feedback loop
  abusers use to verify their voting is working;
- **vote weight by account trust** (site karma, age, per-community history rather than just
  identity) so mass-created accounts contribute little;
- ranking demotion and rate limit on the item, plus queue to moderators.
- **Do not** try to make a single global "is this user a spammer" verdict — the actual
  protection is per-context (community) reputation, matching deep dive 4's community-scoped
  moderation model.

**Layer 4 — Human triage**: mod queues ranked by classifier score; admins for site-wide
patterns (botnets across communities, scam networks); appeals flow for false positives
(a mod action log from deep dive 4 is the substrate).

**Trade-off made explicit**: Option "strict gating at write time" (synchronous classifier
verdict before publish) wins zero spam visibility; loses latency on every write, model
availability coupling to content availability, and false-positive blocking of real users
with no human in the loop early. Option "async classify + signals" (chosen) wins write
latency, isolation (model outage degrades quality, not availability), and iteration speed
on models; loses seconds of spam visibility and needs the mod-queue + demotion plumbing to
be real. We pick async because write availability of the content plane is a harder
requirement than spam-immediacy, and because the vote-side effects of spam (ranking
distortion) are already mitigated independently in deep dive 3.

### Failure modes

- **Classifier false positives** — real users' content silently downweighted or removed.
  This is the worst UX failure: mitigate by keeping *removal* (state-changing) behind
  mod/human confirmation for low-confidence cases, using demotion (non-state-changing)
  alone for automated low-confidence decisions; log every classifier action for appeal.
- **Adversarial adaptation**: bots evolve past fingerprints; vote-fuzzing and log dilution
  are designed to be cheap to *re-tune* (they're parameterized, not learned), so the
  defense stays a moving target rather than a fixed gate.
- **Brigading across communities** (coordinated, many small communities to evade per-community
  rate limits): per-community signals miss it; site-wide correlation layer (IP/behavior/
  account graphs) is the only catch. Correlation is the most compute-expensive layer —
  budget it accordingly and accept that it lags.
- **Moderator abuse or incompetence at scale** (a hostile mod team in a big community):
  site-level admins keep an override path (deep dive 4's two-layer model); no fully
  self-governing community exists.

### Cross-references for these patterns

- Shared-pool fanout-on-read vs per-user fanout-on-write:
  [../02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md).
- Event log as the backbone for ingest + ranking + abuse consumers:
  [../13.kafka/04-deep-dives.md](../13.kafka/04-deep-dives.md).
- Counter/rank cache patterns and stampede handling:
  [../10.distributed-cache/04-deep-dives.md](../10.distributed-cache/04-deep-dives.md).
- Content state machines under human review:
  [../06.apple-app-store/04-deep-dives.md](../06.apple-app-store/04-deep-dives.md).
