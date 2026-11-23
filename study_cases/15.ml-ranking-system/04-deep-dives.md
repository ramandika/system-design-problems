# 04 — Deep dives

## 1. The funnel: two-stage retrieval+ranking and the latency budget (THE dive)

You cannot score 500M items with a good model at 200K QPS. Arithmetic from
[02-scale-estimates](02-scale-estimates.md): scoring everything would mean 200K QPS × 500M items ≈ 100T scores/s
at peak; scoring the ~2K retrieved set is ~400M — a 250,000× reduction. So relevance must
be earned with a funnel: **retrieval finds plausible items fast (recall), the ranker
spends the machine time only there (precision).**

**Option A — exhaustive scoring** (score everything, maybe with a small model):
- wins: no candidate will ever be missed for capacity reasons; no index to keep fresh;
  simplest possible correctness story.
- loses: 100T+ scores/s at peak is physically out of reach; even a tiny 1-ms-per-score
  model would need 500M × 1 ms = 500,000 s of compute per request — ~1e11 core-seconds/s
  at 200K QPS. Only a ~2 ns/score model (~1 s of compute per request) gets exhaustive
  scoring down to 200K core-seconds/s — hundreds of thousands of machine-cores — and that
  is itself far below
  any real model's cost. And exhaustive scoring forces a *weak* model (latency bound),
  which loses precision against a strong model on a small set anyway.

**Option B — two-stage**: cheap high-recall retrieval (embedding ANN + graph + trending) →
~2K candidates → heavyweight DNN ranks 2K → re-rank → 50.
- wins: puts compute where it pays; the ranker can afford expensive features/cross
  features (user×item interactions) that a corpus-wide model could never run; each stage
  has an auditable recall budget.
- loses: every retrieval miss is forever invisible downstream — the ranker never sees it;
  the funnel adds cross-stage coupling (retrieval trains against ranker's decisions, see
  dive 2), and adds failure modes (stale index, source outages) that exhaustive doesn't have.

**Chosen: two-stage, with an explicit per-stage budget** — retrieval ~10 ms, feature
assembly ~10–15 ms, ranker ~20–40 ms, re-rank ~5–10 ms. Non-negotiables: (a) each stage's
budget is enforced in code (stage times out → serve that stage's partial/degraded output,
never blow the page), (b) retrieval quality is measured as recall@2K against a ranker-
labeled "what would have ranked top-50" offline set, and (c) if capacity pressure grows,
the first lever is a **small pre-ranker** (tiny model, cached features, 2K → ~500) before
the DNN, not shrinking the ranker.

- **Trade-off made explicit**: we accept an entire retrieval subsystem + recall-eval
  discipline (Option B's cost) because exhaustiveness costs orders of magnitude more
  hardware, and even exhaustive scoring would have to be a weak model — Option B wins on
  both latency *and* precision.

**Failure modes**: (1) *Recall starvation* — a source silently degrades (e.g., user
embedding drifts after a retrain) and the funnel happily ranks garbage with high
confidence; countermeasure: recall@K dashboards per source, alert on drop, not just on
score distributions. (2) *Ranker overload* — a campaign or incident multiplies candidates
per request and every request goes p99-terrible at once; countermeasure: hard candidate
cap per source, load shedding by dropping to the pre-ranker path. (3) *Budget leakage* —
one stage's tail eats the next stage's budget; stages timeout independently and degrade
individually, the page never fails as a whole.

## 2. ANN index: HNSW vs two-tower sharded brute-force

Retrieval must find the right ~2,000 out of 500M in ~10 ms. Options for the
embedding ANN:

**Option A — HNSW (graph index)**:
- wins: very low query latency and high recall at practical memory sizes; incremental
  point insertion (new items join the graph without a full rebuild).
- loses: memory-hungry (graph links on top of vectors, plus fp32 unless you quantize);
  recall depends on an EF-search parameter you must tune and re-tune as data shifts;
  deletions and retraining-induced drift quietly rot the graph (edges were learned for the
  old distribution); shard rebalancing copies links, not just vectors.

**Option B — sharded brute-force / IVF sweep** (two-tower embeddings; shard the 500M table
across ~dozens of RAM nodes — ~128 GB of fp16 vectors total per
[02](02-scale-estimates.md) — and do exact distance or IVF-clustered search per shard):
- wins: exact/controllable recall, dead-simple semantics (no graph to rot), trivially
  re-derivable from raw vectors whenever embeddings retrain — no incremental-structure
  debt; scaling = adding RAM nodes.
- loses: more memory bandwidth per query than HNSW (you scan more), so per-query cost is
  higher at the same recall; IVF clustering has posting-list staleness between rebuilds.

**Chosen: two-tower + sharded IVF/quantized brute-force as the primary, with the fresh-item
source outside the ANN** (dive 3), and HNSW admitted at the edge cases where incremental
insert dominates (per-user "more like this item" rails). The deciding argument: our
embedding table re-trains daily, and drift-rotted graph topologies fail *silently*
(recall drops for hours before anyone notices), while a re-sharded exact/IVF sweep has a
known freshness clock and cheap rollback. We pay more memory bandwidth for a system whose
staleness is a scheduled rebuild, not a slow rot.

- **Trade-off made explicit**: exhaustive-in-a-shard wins staleness and auditability and
  loses per-query efficiency; HNSW wins efficiency and loses maintenance. Pick B because
  in recommendation, *quietly stale recall* is worse than *2× more RAM*: it corrupts the
  training loop's view of what users could see.

The training side couples to this (the "retriever trained on the ranker's labels" idea,
from the YouTube two-stage paper — Covington et al., RecSys 2016): train the retrieval
towers so that positives are items the *ranker would rank top-50* (sample the label pool
from ranker outcomes), not merely items watched. Otherwise retrieval optimizes a
different objective than the funnel ends up serving.

**Failure modes**: embedding retrain changes the vector space → in-flight mismatch
between user-tower version and item-tower version; ship item/user towers as one atomic
version and warm new shards behind old (same rollout discipline as
[case 14](../14.ml-production-system/04-deep-dives.md)). Hot shard collapse (one popular
item queried by all via "related" rails) — cache neighbor lists per hot item at the edge.

## 3. Feature freshness: realtime counters vs precomputed features

"Views in the last hour," "completion rate this morning," "is this trending right now" —
features that justify their own existence by being *young*. The split:

- **Realtime tier** (streaming counters off the event log): item-level recent windows
  (views/watch-rate last 5 min–1 h), trending flags, live engagement velocity. Seconds of
  staleness; small (a few KB per hot item); written ~6M×/s on average, ~23M×/s at peak.
- **Batch tier** (daily/hourly precompute): long-horizon item stats (30-day completion
  rate), the user profile row (embedding, top-category affinities), creator-level stats.
  Hours-to-day staleness; big state (~50 TB user tier per 02).

**Option A — realtime everything**: push every feature through the streaming layer.
- wins: maximum freshness everywhere, uniform code path.
- loses: you'd be streaming 500M users' continuous state and paying to keep cold-state
  hot; outages in the stream layer now corrupt *all* features; and most long-horizon
  features gain nothing from being 10 minutes younger (one hour of new data is 1/720 of a
  30-day window, so a long-horizon rate moves by at most ~0.14%/hour even when the new
  hour is atypical).

**Option B — batch everything**: precompute daily, serve from the profile store.
- wins: cheapest possible serving, single slow-moving source of truth, trivially consistent
  with the training-time feature generation (no online/offline skew).
- loses: the system is blind to *the product's most important events*: a video going viral
  at 10:00 would look average until 09:00 tomorrow; the ranker would keep surfacing
  yesterday's winners and miss every burst. Freshness isn't a garnish here, it's the
  difference between ranking as a live system and ranking as yesterday's report.

**Chosen: two tiers on purpose — realtime only for features whose value ratio depends on
it, batch for state that moves slowly; and every realtime feature must be
training-visible too** (the trainer reads the same counter store's snapshots, or joins the
same stream at training-example time), otherwise you build a classic offline/online skew
bug: the model served on a feature it never trained on.

- **Trade-off made explicit**: two tiers win freshness where it pays and cost where it's
  cheap, at the price of running a streaming-counter layer with hot-key problems and
  training-time counter replication. We pick this because feed products live or die on the
  5-minute signal; a single-tier design is either ruinously expensive (all-realtime) or
  product-broken (all-batch).

**Fresh item bootstrapping (the "not in the ANN index yet" problem)**: the ANN reflects
last night's corpus. New items (2M/day) reach retrieval through a separate **fresh source**
— a tiny recency-ordered stream-backed index holding minutes-to-hours-old items, blended
into the candidate merge with a freshness prior. Once old enough (hours), items are in the
rebuild window, gain batch stats, and let go of the prior. Cold-start for a new creator is
handled by exploration slots (dive 5) targeting their first hours of traffic.

**Failure modes**: viral hot key — one item's counter absorbs 50K writes/s (single-key
stress); shard sub-counters per key and aggregate at read. Counter store outage → ranker
must be able to run batch-only with zeroed realtime features (and models should be robust
to feature-missing masking, trained exactly for that). Stream lag piling up under peak →
alert on counter-staleness SLO (e.g., p99 event-to-feature < 60 s), since silent 10-minute
lag looks like "model got worse" and triggers the wrong debugging — the same "feature
freshness is a serving SLA" discipline as [case 14](../14.ml-production-system/04-deep-dives.md).

## 4. Negative sampling & label delay in training

Two structural problems in the training data:

**(a) Labels are delayed and implicit.** An impression at 10:00:00 may be watched (or
skipped — but the watch-fraction arrives only when the watch *ends*) minutes later. If you
train directly off "events in the last hour," you systematically see: impressions (all of
them), watches (the fast ones), skips-in-progress. The fix is a **label-joiner job keyed on
impressions**: emit the training example when the label *closes* (watch ended, or a
session-scoped timeout expires the impression to "unseen/ignored"). Impression-first labeling
— not event-first — is what makes the positive/negative ratio honest.

- **Trade-off**: impression-first joining wins label correctness and loses freshness
  (training examples lag minutes-to-hours); event-first wins freshness and loses the
  negatives (most impressions produce no event at all — dropping them makes the model
  learn "everything shown succeeds"). Pick impression-first with bounded delay; an hour-delta
  in training data costs almost nothing in a daily/hourly retrain, a wrong negative ratio
  costs every request.

**(b) Softmax / candidate sampling.** Retrieval tower training in principle needs a
softmax over the whole 500M corpus per example. Impossible; you sample.
- Full softmax over vocabulary × batch: exact gradients, impossible at 500M items.
- **Sampled negatives** (in-batch + N sampled uniform/popularity negatives): cheap, but
  uniform negatives are too easy (random items are never plausible confusables), and
  popular-item negatives are over-penalized (false negatives: the sampled negative could
  in reality be a fine item for this user).
- **Hard negatives** (retrieved-but-not-ranked, ranker-rejected): teach the retriever the
  real decision boundary it must make — but mined from the *current* serving distribution,
  so they can amplify present bias (you train retrieval to fix retrieval's own blind spots,
  and it never sees what it never retrieves).

**Chosen: in-batch negatives + a modest fraction of ranker-adjacent hard negatives plus
popularity-log-sampled standard negatives, with known-false-negative filtering.** The mix
is tuned by the same recall@2K offline metric that retrieval serving is judged on (dive 2) —
training and serving share one number, that's the discipline that keeps sampling choices honest.

- **Trade-off made explicit**: mixed sampling wins boundary learnability and loses purity
  (some "negatives" are real positives); pure uniform wins purity and loses everything it
  teaches. We deliberately accept impure negatives — with a false-negative filter — because
  the retriever's actual job (dive 1) is to discriminate *plausible* from *implausible*,
  and uniform negatives don't represent that task. An extra failure of sampled training:
  the label log inherits position and selection bias — the model learns "what we showed
  was ranked high was watched" — mitigated further in dive 5 (exploration) and dive 6
  (interleaving/unbiased measurement), not by training tricks alone.

**Failure modes**: label delay spike (slow watchers, binge sessions) → example age
histograms must be monitored or the trainer trains on an accidentally skewed slice; the
sampling distribution changing without the serving behavior changing is invisible unless
recall@K is tracked over time; a feedback loop where hard-negative mining from serving
traffic slowly narrows retrieval's scope — countermeasure: hold out a fixed fraction of
uniform negatives in every training round.

## 5. Exploration vs exploitation without wrecking UX

The model can only learn what the system dares to show. A pure exploiter shows each user
the argmax of current predictions and never discovers anything better; a pure explorer
burns user attention on random items. Budget honestly:
[02](02-scale-estimates.md) says each user has ~400 impressions/day — every exploration
slot is paid in that currency.

**Option A — ε-greedy slot injection**: some % of slots serve a uniformly-random (or
fresh-prior) candidate.
- wins: trivially correct, trivially logged, unbiased-ish data for what you inject.
- loses: random exploration is *low-information* (learn nothing about *near-decisions*) and
  UX-random: a 50-item page with 3 random items reads as "the feed is broken."

**Option B — Thompson sampling / bandit re-ranking** over the candidate set: for uncertain
items (few impressions, high posterior variance), sample from the posterior instead of
argmax; exploration effort concentrates on items that *might* be good.
- wins: explores near the decision boundary where the information value is; variance
  shrinks naturally as data arrives; fits cleanly inside the existing ranker (uncertainty
  is just another output).
- loses: exploration is now *biased toward already-plausible items* (posterior sampling is
  still model-driven, so garbage-unknown items are rarely sampled — the true cold-start is
  underserved); engineering cost: you need calibrated uncertainty, not just a score.

**Chosen: B, bounded — a small fixed exploration budget per page (low single-digit % of
slots), spent by posterior sampling among ranker-borderline candidates, plus a deterministic
freshness prior for brand-new creators; never at the top 2–3 positions** (top slots are
where user trust concentrates and where a bad experience costs the most — exploring with the
user's attention is a tax, so put the tax where attention is cheapest, usually page-middle
or tail). Every slot served and what it served under (model version, exploration flag) is
logged — this is what later lets us measure honestly (dive 6).

- **Trade-off made explicit**: posterior-bandit exploration wins information-per-slot and
  loses coverage of unknown-unknowns (cold items still need the separate fresh-source prior
  from dive 3); ε-random wins coverage and loses both UX and information efficiency. We
  pick B + fresh-prior because a feed's exploration question is almost never "is this
  random item OK" and almost always "is this borderline-good item actually good."

**Failure modes**: exploration budget drift under A/B pressure (teams quietly ratcheting it
up for their own metric) — the budget is product-owned and audited per version like any
ranker feature; calibration rot — posterior "uncertainty" that stopped being calibrated
turns exploration into noise without anyone noticing, so calibration is an offline metric
with an alert, not an assumption; feedback-loop poisoning persists in the *unexplored*
region and is only fixable by exploration — which is the argument for treating the slot
budget as a first-class product parameter, not an ML afterthought.

## 6. Interleaving for unbiased A/B

Normal A/B compares whole-user experiences: user group A gets ranker v1, group B gets v2,
you compare 30-day watch time. Problems: (a) **power** — between-user variance
(different people, different hunger to watch) is enormous, so real ranker gains are easily
inside the noise; (b) **confounded data reuse** — each arm's model trains on its own arm's
traffic, so a better-feed-makes-more-data-makes-better-model flywheel (the "rich get
richer" problem of long-horizon A/Bs) means the *longer* you run the trial the less it
compares ranker quality in isolation.

**Interleaving** attacks (a): in one user's session, merge two rankers' candidate lists into
a single page, interleaving by an unbiased scheme (team-draft: each ranker "drafts" one
item per pair, order randomized), and score **which ranker's picks the user watched** —
per-item, per-user comparisons remove between-user variance. It usually needs roughly an
order of magnitude (or more) less traffic than A/B to detect the same effect because
it turns the noisy unit (user) into the quiet unit (pair of items, same user, same session).

**Option A — team-draft interleaving**:
- wins: high power, cheap in traffic, simple randomization you can prove unbiased for
  clicks-when-shown; widely used by search ranking teams (the technique was originally
  popularized for web search).
- loses: **team blindness** for *downstream* metrics — each arm only learns from its own
  picks, so you cannot use interleaved traffic as training data for either model (you'd
  contaminate both with the other's selections); positional/length biases must be handled
  by the scheme, not by the analyst; it measures *user preference between two rankings on
  the same session*, not the multi-session product metric the business actually optimizes
  (retention, watch time per user over a month).

**Option B — plain interleaving of metrics + A/B for the rest**: use interleaving as a
*fast gate* on ranker quality (minutes-hours of traffic, pairs as the unit), then confirm
the wins in a slower standard A/B for product-level metrics and to give each arm clean
training traffic.

**Chosen: B — two-stage evaluation.** Interleave to *decide fast and without between-user
noise*, A/B to *confirm the product claim and feed cleanly-labeled training traffic back
into each arm* (needed because of dive 4's selection-bias problem: you want each model
trained on outcomes of its own decisions, not a merged session).

- **Trade-off made explicit**: interleaving-first wins measurement power and speed and
  loses training-data usability and long-horizon validity; A/B-first loses weeks per
  comparison in noise. Run both, in that order, because they answer different questions —
  ranker-vs-ranker (pair unit) and product-vs-product (user unit).

**Failure modes**: interleaving changes the experience itself (two rankers' picks merge —
pages less coherent than either alone), so interleave sessions can degrade retention if run
too broadly — bound the interleave traffic fraction and monitor session-level outcomes even
inside interleave arms; scheme bugs (unbalanced drafting, position bias sneaking past the
scheme) produce confident wrong conclusions — the scheme should be validated by replaying
logged traffic against a known-null pair of rankers (identical rankings must measure zero);
and finally the experiment inventory itself becomes the bottleneck with many ranker
candidates — prioritize interleave-first so cheap traffic kills weak candidates before they
consume A/B space.
