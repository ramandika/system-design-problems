# 04 — Deep dives

## 1. Binary storage & delta-update delivery (patching vs full download)

The store delivers ~1.4B binary events/day (02, estimate); at mean 150MB, a full-only
world costs ~210 PB/day and delta delivery cuts the update line ~3x (mean delivered
51MB vs 150MB full, 02 A4). The design
question is not "patch or not" — it's **which patch pairs to precompute, for which
variants, and how to survive a broken chain**.

**Option A — full downloads only.** Wins: zero pipeline complexity, client is a dumb
downloader, signature covers the artifact as served. Loses: payload cost is dominated
by unchanged content; every small bugfix release costs the full 150MB per device; CDN
caches still help, but per-device network (and metered connections) do not.

**Option B — precompute deltas for every (old version → new version) × device variant.**
Wins: fastest possible device path — download patch, apply, verify signature. Loses:
combinatorial explosion — 71K new versions/day × 4 base versions × ~2–3 device variants
× per-locale slices is millions of patch jobs daily; storage of all deltas would dwarf
the ~340 TB steady-state estimate (02) if ever kept unlimited.

**Option C — compute deltas on demand at request time.** Wins: zero combinatorial waste
— only pairs devices actually request are ever built. Loses: first requester after a
release pays pipeline latency (bad for launch-moment spikes when the *whole world*
requests simultaneously); requires compute fleet sized for bursty patch generation.

**Chosen: B with retention discipline + C-shaped fallback.** Generate deltas eagerly
only for *recent* predecessors (last ~4 versions — the ones auto-updaters are plausibly
on), for the universal variant plus the device slices that actually exist; evict old
deltas with their base versions. If a device's installed version is outside the delta
window, the manifest returns a full download — no on-demand patching of ancient chains.
Rationale: eager-for-hot, full-for-cold. The eager window is chosen because prompt
pickups (A3 — ~22% of releases fetched promptly, the rest late or never) are always
against one of the last few installed versions — devices that update near release time
sit inside the N≈4 window — while the long tail of "installed version is 2 years old"
is rare and cheap to serve as full.

**The signature is the correctness oracle.** The client applies the delta to a copy of
its installed app and then verifies the pipeline's distribution signature over the
result. A mismatched patch (device's local copy was modified, OS version drift, patch
bug) fails verification and the client discards the work and downloads the full
variant. This moves delta correctness out of the trust boundary: a broken patch can
waste bandwidth, never install wrong code. It also means deltas must be generated
*against the canonical published bytes*, not against whatever variant shape a client
happens to have — patch format must be robust to benign local diffs (page hash-based
patching: the delta is expressed over content-identified blocks, so a locally-perturbed
file still resolves).

**Failure modes**
- *Delta generator bug ships bad patches at scale*: caught by signature verification —
  full-download fallback rate jumps; alert on fallback %. Mitigation: pipeline runs
  apply-and-verify on every produced patch before publishing it (the generator
  bootstraps its own correctness; a patch that fails in the lab never reaches a device).
- *Base version disappears while deltas still reference it*: kept-version retention
  (N≈4) + manifest service resolves the chain server-side, so a client never chases a
  half-evicted object.
- *Cache invalidation storm on popular apps*: the release-gate event triggers CDN
  pre-warm of the new variant's objects ahead of the phased-rollout percentages
  (deep dive 3's phased release is exactly the throughput-shaping tool here — 1%, 2%,
  5%... rollouts cap the concurrent full-download wave). Same immutable-artifact/CDN
  economics as the blob-store and CDN-pattern cases — compare
  `../07.blob-store/04-deep-dives.md` and `../09.cdn/04-deep-dives.md`.

## 2. Search ranking & index freshness on a daily-refresh catalog

**The embarrassing-but-true starting point: the corpus is tiny.** ~2M documents with
~50 tokens each is a weekend-scale index by web-search standards. The hard parts are
(a) freshness semantics, (b) ranking that can't be gamed, (c) being right per
storefront.

**Option A — per-storefront shards, one index per storefront.** Wins: availability and
price can live *inside* the index (search returns only purchasable results; no
post-filtering that breaks pagination counts); regional ranking is natural. Loses:
~175 index copies of 2M docs — memory multiplies by storefront count for docs that are
90% identical across storefronts.

**Option B — one global index; availability/price filtered at query time.** Wins: one
copy, trivially fresh worldwide. Loses: filtering after ranking breaks result counts
and pagination ("page 2 shows only 3 items"); regional quality signals (a game huge in
JP) must be joined in post, which is exactly where relevance bugs live.

**Chosen: hybrid — global immutable doc corpus + per-storefront *tiny* delta layer.**
Documents (name, keywords, descriptions, category, quality features) are global — one
in-memory inverted index in the tens of GB. Storefront-specific bits (availability,
price tier, regional ranking boost) are small per-storefront postings
(app_id → storefront bitset/mask) updated on catalog events. Ranking = text score ×
quality features (crash rates, ratings velocity, engagement, first-week retention —
platform-internal signals no developer can optimize directly against, which is the
actual anti-gaming mechanism) × regional popularity prior.

**Freshness semantics.** The catalog updates continuously (71K/version-releases/day →
metadata edits land asynchronously); index freshness lags by the event-pipeline lag —
seconds. The design stance: an index is allowed to be stale by minutes, never wrong by
structure. What that means concretely: releases and removals propagate via the same
catalog-change event bus everything else subscribes to; a search must never return an
app whose release gate has not flipped (un-reviewed content is not indexable — this is
 enforced by indexing *only* from approved catalog rows, at the indexer, not by
 filtering at query time).

- **Trade-off made explicit**: event-lag staleness wins latency and simplicity and
  loses "publish a metadata edit and see it in search within one second" — acceptable,
  because the edit path is the developer portal (async expectations) and the same
  acceptance criteria (event-bus lag) apply to browse/detail caches. The one place we
  do not accept staleness: removal-for-legal-reasons (takedowns) — those go through a
  synchronous kill path (index + CDN manifest + storefront cache purge for that app).

**Failure modes**
- *Ranking weight change tanks a storefront*: rollout ranking changes with holdback
  cohorts + metric guards (search CTR, zero-result rate); ranker config is versioned
  and rollback is config-only.
- *Indexer falls behind the event bus*: freshness SLO monitored as `now - latest
  indexed event`; alerts at minutes. Catch-up replay is idempotent because events
  carry (app_id, version, field-hash).
- *Query spam / search-ad gaming*: rate-limit + query-abuse classification at the edge;
  ranking features are internal platform metrics, deliberately unavailable to
  developers, so there is nothing to A/B-test your app into. (The abuse-control shape
  here is the same one as the Twitter abuse dive: limits before the expensive path —
  ../02.twitter/04-deep-dives.md.)

## 3. The review workflow state machine

Every submission of every version goes through one gate. Design as an explicit state
machine with an append-only audit log, not as workflow code in an ad-hoc service.

```
                    metadata-or-binary resubmit
   SUBMITTED ──> WAITING_FOR_REVIEW ──> IN_REVIEW ──┬─> APPROVED ──> (signing) ──> RELEASED
       │                │  expedited request          │                (phased rollout %) │
       │                └──> CANCELLED (developer)    ├──> REJECTED ──┐                   │
       │                                              │      (reason code + guideline)    │
       │                                              └──> METADATA_REJECTED (no re-review  │
       │                                                     of binary; fix-and-resubmit)   │
       └── (automated pre-screen can fast-fail: signature/asset/age-rating validation)       │
                                          PAUSED (crash-rate regression in phased rollout)<──┘
```

- **States carry an SLA clock** (publicly reported: large majority of reviews complete
  within ~24h) — the queue is category-routed (games vs finance; finance needs
  credential-handling expertise), and expedited-request slots are a bounded quota, not
  a queue-jump free-for-all.
- **Human verdict is the only writer** that moves `IN_REVIEW → APPROVED/REJECTED`.
  Automated pre-screens can *fast-fail* (objective checks: bad signature, missing
  assets, crash-on-launch in sandbox run) but never fast-pass. Trade-off: pre-screens
  cut reviewer-hours (02: ~28K reviewer-hours/day at median 0.4h — automation shaving
  20% of touch time is ~5,600 hours/day of reviewers) at the cost of trusting the
  automation's checks; we accept automated *rejection* but never automated *approval* —
  the asymmetry is the point: a false reject costs a developer hours; a false approve
  costs the platform its trust root.
- **Rejection loops must not reset the queue position silently.** A metadata-rejected
  resubmit re-enters at `WAITING_FOR_REVIEW` with its original history (audit log
  shows the whole loop); a new *binary* requires fresh full review because bytes
  changed.
- **Phased rollout as a post-approval state.** `RELEASED` with phased mode progresses
  1%→2%→5%→10%→…→100% per 24h windows, with automatic pause on crash-regression
  thresholds. This is a *delivery-shaping state*, not just a feature — it caps the
  mass-download launch spike from deep dive 1 and bounds blast radius of a bad
  binary. Full-download fallback storms also shape here.
- **Takedown/removal** is a distinct terminal-ish transition `RELEASED → REMOVED` with
  synchronous kill (index purge, CDN manifest kill, storefront cache purge) that does
  not require the review path; it's a legal/safety action, and it is *additive* to the
  audit trail, never a rewrite of it.

**Failure modes**
- *SLA breach queue explosion* (holiday feature-freeze-lift Monday): queue depth alert +
  category routing rebalance; expedited quota shrinks to protect the SLA for everyone.
- *Reviewer variance* (same binary, two reviewers, opposite verdicts): appeal path
  (developer response with a second reviewer), decision-consistency sampling on the
  audit log.
- *Signing and approval race*: the version's signing artifact is created only after
  `APPROVED` — the distribution manifest points exclusively at signed-approved objects,
  so there is no state where a client can fetch bytes that are signed but unapproved or
  approved but unsigned.

## 4. The purchase pipeline: exactly-once money with offline-verifiable receipts

A purchase is a money event under a flaky network. The wire can't be exactly-once; the
system must make the *user-visible outcome* exactly-once (same shape as the delivery
protocol dive in ../03.whatsapp/04-deep-dives.md — at-least-once wire + dedup at the
correct layer — and same as the resumable-upload trick).

**Option A — client-ordered, server-deduped only post-hoc.** Wins: no client state.
Loses: a retried purchase can double-charge before dedup runs; refunds of duplicates
are an ops nightmare.

**Option B — idempotency key end-to-end.** Client generates a UUID per purchase
attempt (key = (account, product, uuid)); the IAP service takes the key into a
transaction table with a uniqueness constraint; retries present the same key and get
the same receipt back. Wins: provable once-and-only-once charge under any retry storm.
Loses: client must hold the key across process restarts (write-ahead in local
storage before the network call — crash before charge? key exists locally, resume
after restart; crash after charge, before ack? server has it, retry returns the same
receipt).

**Chosen: B**, with these mechanics:
- **Single-writer sharded ledger.** Transactions shard by account_id; the ledger row is
  written once with state `AUTHORIZED → CAPTURED → (REFUNDED|SUBSCRIBED_RENEWED)`;
  state transitions are append-only ledger entries, balances are projections. Payment
  processor calls are themselves keyed by the same idempotency key — the PSP is a
  downstream that may also be retried, and the same trick applies one layer down.
- **The receipt is a signed document, not a query.** After capture, the ledger emits a
  receipt: a chain-signed, account-attested block of entitlements (what was bought,
  when, original transaction ID for subscription lineage). The device stores it and
  validates it offline; a server-side "check my entitlements" endpoint refreshes it.
- **Trade-off made explicit**: *server-authoritative ledger* (Option A of storage:
  never trust the client) *plus client-held signed receipts* (eventual, offline
  validity). The two seem in tension and are instead layered: the ledger answers
  "what is true," the receipt answers "what can this device prove without the
  network." We pay for a signature pipeline + receipt revocation/refresh machinery to
  get offline entitlements; a query-only model would make every app launch a
  server-round-trip and entitlements would die with the network — unacceptable.
- **Subscription renewals** are server-initiated on renewal dates: the same ledger
  receives `SUBSCRIBED_RENEWED` entries generated by a scheduled renewal worker per
  shard — the renewal must also be idempotent (renewal_key = (subscription_id,
  period_index)), because the renewal worker retries and payment processors retry.
- **Family sharing / restore** are projections over the same ledger
  (receipt re-signing for a new device), not separate purchases.

**Failure modes**
- *Double charge*: prevented by the idempotency-key uniqueness constraint; a *bug* that
  bypasses it has a second net: daily reconciliation of charges against PSP records
  flags key-collisions or missing captures (ledger and PSP must agree on each
  transaction's idempotency key).
- *Receipt forgery / downgrade attacks*: receipts are chain-signed; device validation
  checks the chain, expiry semantics are "valid until refresh prompt", and the
  revocation path (refund/chargeback) appends a `REVOKED` ledger entry that refresh
  propagates into new receipts — old receipts die when the device next reaches the
  network, not before (that residual window is the accepted trade-off of offline trust).
- *Payment processor outage for a region*: purchases in that storefront degrade to
  queue-and-capture-later (ledger row `AUTHORIZED`, capture retried) — the
  user-visible event is "purchase recorded", money capture is eventual. Purchases must
  never be user-visible failures because a PSP is down; a ledger row without money is
  recoverable, money without a ledger row is not.

## 5. Worldwide multi-region consistency: catalog + entitlements

~175 storefronts, writes land continuously from every time zone, reads come from
everywhere. Two data classes with opposite needs:

**Option A — strongly consistent, globally replicated catalog.** Wins: one worldwide
truth; a release is visible simultaneously everywhere. Loses: every catalog write pays
a worldwide-latency-consensus round trip across regions; and worse, the *write path is
gated by review SLAs measured in hours anyway* — sub-second global consistency buys
nothing where the true bottleneck is a human reviewer's verdict.

**Option B — active-active per region with conflict resolution on merge.** Wins: local
writes never wait for consensus. Loses: catalog is now multi-master — a release
approved in region A but not yet in region B is a *conflict*, and conflicts in a trust
system are unrecoverable (one side's "approved" can't be merged with the other's
"pending": you'd need a global arbiter anyway).

**Chosen: single logical writer (the release gate), async replication, per-region
eventual read models.** All gate-flipping writes (approve/release/remove/price
tier/availability) land in the catalog's authoritative store; every region's read
model (storefront caches, search index delta layer, manifests, CDN pre-warm) subscribes
to the same catalog-change event stream. Regional read models are allowed minutes of
lag; they are keyed so an app version is either fully published in a region or not at
all — no partial-row publication. Takedowns (legal) ride a synchronous kill path
instead of the lagging bus (see deep dives 2 and 3), because that's the one case where
"eventually consistent" is user-hostile.

- Entitlements take the opposite fork on purpose: **strongly consistent writes**
  (single-writer sharded ledger, deep dive 4) and **eventually consistent,
  self-verifying reads** (signed receipts held by the device). A purchase is a global
  truth about one account the moment its ledger row commits; the *distribution* of
  that truth is exactly what the receipt signature was designed to defer.

- **Trade-off made explicit**: async catalog replication wins write latency and simple
  conflict-free semantics (no arbiter needed) and loses cross-region simultaneity — a
  midnight-PST launch sees the app in region A minutes before region B, which for a
  launch-marketing context is an accepted cost (launches coordinate to the slowest
  region when the developer cares). Global consensus would buy simultaneity nobody
  experiences and pay for it in every one of the 71K/day metadata-path writes.

**Failure modes**
- *Replication lag grows past SLO* (event bus backlog): storefront reads fail
  open-but-stale (serving last-known-good regional model), never closed; the kill
  path is independent of the bulk bus, so safety takedowns don't queue behind lag.
- *Regional read model corruption*: rebuilt by replay of the catalog event stream (the
  stream is the source of truth for the read models; models are projections —
  rebuildable by definition).
- *Split-brain of the authoritative catalog itself*: the authoritative store is
  strongly consistent within itself (single region's consensus cluster), and only its
  *replicas* are async — we do not run two authoritative catalogs. The cost to accept:
  the authoritative region's outage blocks releases worldwide (read models keep
  serving last-known-good), which is the right failure direction — nothing publishes
  unsupervised, nothing previously published vanishes (compare the "two traffic tiers,
  two stacks" separation in 03; and the same publish-once/immutable pattern as
  ../01.youtube/04-deep-dives.md's big-pipeline writes).
