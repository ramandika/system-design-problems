# 31.04 — Deep dives

## 1. VOD ingest + the transcoding ladder, at ingest scale

Ingest here is small in bytes (02: ~12 TB/day vs YouTube's ~6.5 PB/day) but demanding in
*correctness and deadline*: a season premiere goes live at 01:00 PT worldwide, with 30+
dub languages, per-region artwork, and a QC gate. One mispackaged audio track is a global
member-visible defect with no "hotfix at runtime" path.

Design: pipeline as **work orders tied to release dates**, each title = a DAG of
(independent per-rendition/per-language jobs) + (join gates). An analysis pass computes the
shot/complexity map that the ladder decision consumes; every (title, rendition, language)
unit is idempotent and retryable; the title flips to "playable" only when the entire base
ladder + required languages pass QC.

The ladder itself is **per-title and shot-aware**: instead of a fixed 1080p/720p/480p
table, encode decisions are made from per-shot complexity (action-heavy scenes, grain,
dark cinematography can need a different bitrate to look equivalent than a static talking
head; this is publicly documented Netflix practice — per-title encoding, then shot-based
encoding — and co-exists with modern codecs like AV1/HEVC to shift the curve further).

- **Trade-off**: a fixed ladder wins *operational simplicity* (one manifest shape, one
  cache-key universe, trivially testable) and loses *quality-equalization* (cheap scenes
  waste bytes, hard scenes look worse at the same rung). We pick per-title/shot-aware
  ladders because the byte-per-quality decision is Netflix's #1 cost lever (N4 in 01: ~1
  EB/day means even a 5% average-bitrate saving is tens of PB/day), and the complexity is
  contained in one pipeline that runs offline.
- **Failure modes**: a straggling 4K HDR rung blocks a premiere (mitigate: publish the
  completed ladder tiers, gate only on the rungs the launch device set needs); a bad QC
  gate escapes (audio desync, wrong-language track) — mitigate with automated A/V-sync and
  loudness checks plus human spot-QC on premieres, and treat every escape as a pipeline
  bug (fixes go back at ingest, not at serving); transcode compute storms on batch days —
  contained because the cluster is sized for the peak release week, not for mean days.
- Compare the *popularity-triggered re-encode* in
  [Case 01 deep dive 1](../01.youtube/04-deep-dives.md): meaningless here — every title's
  expected audience is known (subscriber demand forecast), so we spend effort up front per
  title; there is no viral long tail to chase after the fact.

## 2. Prepositioned own CDN vs on-demand CDN

The core delivery question: 50M concurrent streams at ~175 Tbps peak (02) on appliances
that each hold only tens of TB raw (up to ~120 TB per public OCA specs). Two designs:

- **On-demand caching, third-party CDN** (the [Case 09](../09.cdn/README.md) shape; what
  YouTube does with a demand-driven hot set — see
  [Case 01 deep dive 3](../01.youtube/04-deep-dives.md)): edge POPs pull on miss, cache the
  popular tail, shield layers collapse origin load. Wins adaptivity (no forecasting),
  elastic capacity (vendor POPs scale), no fleet to own. Loses cost control at Netflix
  scale (peak-hour transit is the priciest bandwidth there is), control of the failure
  path, and *guarantees*: a reactive cache can only promise what it happens to hold at 9 PM.
- **Prepositioning into Open Connect**: appliances live *inside ISP networks and IXPs*;
  a nightly fill process pushes the forecast-popular catalog out during off-peak hours so
  evening demand serves from local disks, and steering picks the appliance per session from
  live throughput/capacity probes. Wins: transit-cost flattening (fill rides the ISP's
  idle hours; the CDN economics become a settlement/planning problem, not a peak-rate
  problem), deterministic peak capacity (fleet size is a forecastable function of member
  growth, not a viral-event panic), and deep ISP integration (a public engineering
  pattern, publicly documented: thousands of appliance sites in ~175+ countries, each
  appliance holding tens of TB raw (a 2U chassis with high-capacity disks; up to ~120 TB
  per public OCA specs). Loses:
  inventory planning becomes the core problem (you must *predict* Saturday's viewing to
  load Thursday night's disks), hardware becomes Netflix's opex and logistics burden, and
  cold-tail titles (rarely-viewed, not prepositioned) still need a reachable source.

We pick prepositioning because of the read:write skew (~80,000:1, 02), the
*size-of-day demand* (945 PB on ~1 EB/day — a reactive fleet this specialized would still
re-deliver the same popular catalog every night), and the peak math: the cost difference
at this volume is enough to fund the fleet many times over — and YouTube doesn't get this
free ride, it *buys* adaptivity with the same money because its catalog has a viral tail
Netflix's does not.

- **How sizing works in practice**: forecast popularity per title per region (from the
  same demand models that rank rows and decide production spend); appliances fill what
  their *local* audience will watch; the steering pool's global set must always contain
  every title a member can reach — so the tail is covered by the aggregate inventory, plus
  enough redundancy that any single appliance loss shaves capacity, not availability.
- **Failure modes**: forecast miss (a surprise hit eats local capacity — mitigation:
  over-provision headroom on the top of the demand curve and let steering fall back the
  session to a still-holding appliance or an origin-backed tier mid-playback); appliance
  loss or ISP maintenance window (steering must learn in seconds — it probes continuously,
  it does not trust DNS); fill storms saturating an ISP path that members use at night —
  fill is rate-limited per link and scheduled to the local idle window, because the fill
  system burning the members' evening bandwidth is a worse outage than a cache miss.
- Contrast with rented multi-CDN: Netflix *did* run behind multiple third-party CDNs
  historically before consolidating onto Open Connect — with per-session quality-based
  selection, the same steering idea later applied to its own fleet. The lesson is about
  control, not patriotism: when you have ~1 EB/day of near-static content, owning the edge
  is the cheap option (deep dive 6 covers the resilience consequences).

## 3. ABR and playback quality tracking

Delivery per session: ladder at the appliance + client ABR ([Case 23](../23.frontend-video-streaming/README.md)
covers the client machinery; here: the system around it). Design: client-side ABR with a
ladder-wide throughput/window model, plus **server-side QoE telemetry aggregated per
session** — startup delay, rebuffer ratio, play vs exit, session length — fed back both to
on-call and to offline experimentation.

Why the loop matters more than the algorithm: a 1-second improvement in startup time and
a few-point drop in rebuffer ratio are directly measurable in member retention and
session length (Netflix has long publicly attributed major product decisions to QoE-A/B
evidence), so the measurement system outranks any particular ABR heuristic.

- **Trade-off**: measurement wins insight but pays at 500M sessions/day in event volume
  and in client complexity (the player now embeds a mini-observability product);
  server-controlled quality (steering + session QoS from the device's manifest
  negotiation) wins control and simplifies clients but needs per-session state and
  round-trips at 50M concurrency. We pick client ABR + asynchronous telemetry because the
  client has the freshest link knowledge and no round-trip to pay, and the telemetry is
  batchable; the server keeps only *coarse* influence (which appliance, which codec, which
  ladder floor to hand out per device class).
- **Failure modes**: telemetry loss looks identical to "nobody watched" (mitigate with
  client-side counters that survive to be flushed later, and with delivery-quality
  monitoring that doesn't depend on the same path as the data); ABR thrash on a
  steered-to-a-capacity-edge appliance (mitigation: the steering decision itself carries
  expected-capacity info the ABR can seed its model with, instead of cold-starting — a
  concrete example of control plane and client cooperating); over-experimentation with
  QoE (a metric-gaming ranker that keeps sessions alive longer but at lower quality is
  *worse* and must be guarded by guardrail metrics: exit-during-buffer, exit-to-cancel).
- Where the two cases meet: 23 owns the state machine; 31 owns the *fleet-level* consequence
  — the sum of 50M independent ABR controllers must not oscillate the appliances they
  share [case 09's cache-hit dynamics](../09.cdn/04-deep-dives.md) apply, since a session
  that downshifts mid-playback changes which chunks it requests.

## 4. Recommendation / personalization surface, over a small catalog

The contrast with [Case 15](../15.ml-ranking-system/README.md) is the whole point: there,
the funnel exists because you cannot score 500M items; here, with a few-thousand-title
regional catalog (02), retrieval is cheap and the hard problems move to the *surface*: which rows, in
which order, with which artwork, for this household, tonight.

Design: candidates (the whole plausible catalog for this profile) → rankers for
p30n-style row composition and title scoring → re-rank layer enforcing diversity,
business rules, and freshness on rows like "Trending" and "Continue Watching"; plus
**artwork personalization** (per-member per-title image selection, publicly-documented
Netflix practice) and per-profile row ordering. Session-level state (play position,
profile context) is cheap to serve because the catalog smallness makes per-member
precompute feasible.

- **Trade-off**: deep heavy rankers win long-term engagement maximization and lose
  interpretability and latency headroom (and, notably, need much more traffic per arm to
  be sure); small/light models win p99 latency and easy experimentation and lose
  long-tail precision for niche profiles. We pick the deep-but-not-broad variant: the
  candidate pool is small enough that a strong ranker is affordable at every request —
  what you *can't* afford is heavy exploration on a paying subscriber's home screen,
  which is why Netflix-style systems run personalization under an experimentation
  platform with long horizons rather than online bandit churn in-session.
- **Failure modes and their fixes**: cold start on new titles (a brand-new show has no
  per-title signal — mitigate with content features + launch-phase exploration slots +
  artwork variants that double as the exploration); filter bubbles (a member stuck in one
  genre row — mitigate with explicit diversity terms in the objective, not just ranking
  score); popularity feedback loops amplifying hits (mitigate with position- and
  popularity-debiasing in training); personalization outage should regress to a
  **generic-but-non-empty** fallback, never an empty screen (N3 in 01) — a personalization
  service that 500s must be *skipped*, not retried on the critical path.
- Cross-reference: the infra lessons of 15 (feature freshness, training loop, latency
  budget) transfer directly; what differs is the candidate scale (1–2K vs 500M) and that
  "discovery" here means *re-surfacers* for known titles — a catalog of 100K hours is
  deep but finite, so a re-watcher is a valid target per session.

## 5. Entitlements and DRM

Every session must prove: subscriber is valid, plan permits this stream (count + quality
tier + ad-tier status), geo/age rating permits this title, and the device can actually
enforce playback via a CDM. Design: session-scoped authorization (fast, cached lookup on
subscription state, per-device stream registration with per-account concurrency limits) →
multi-DRM license issuance (Widevine/PlayReady/FairPlay/Hardware L1) with token/key
rotation, packaging keys separated from license issuance, license TTLs aligned to typical
session length.

- **Trade-off**: strict enforcement (short license TTL, hard concurrency caps, frequent
  device re-authentication) wins revenue preservation and loses household friction —
  and *within a household* the member is paying to be trusted, so heavy friction punishes
  exactly the paying customer (N5 in 01: convenience inside the household). Loose
  enforcement (long-lived device tokens, no concurrency check) wins UX and loses revenue
  to credential sharing and to a single $10 plan driving a 12-screen black market. We
  pick session-scoped authorization with soft concurrency (limit + allowlist + grace),
  short TTLs for the *key*, long-lived convenience for the *device* — because key leakage
  is where money is lost and device friction is where subscribers are lost.
- **DRM multi-sourcing is a support burden, not a strategy question**: every major device
  class carries its own CDM, so you package once per DRM scheme and let the playback
  manifest negotiation decide (F9/03 step 2). What actually varies is L1 vs L3 hardware
  trust: high-trust levels unlock 4K/HDR (a *plan* feature, F8), low-trust caps quality
  — meaning the entitlement system and the DRM level *together* decide the ceiling, and
  they must agree at issuance time, not just at the client.
- **Failure modes**: drift between billing state and entitlement cache (a canceled but
  cached-active member plays for a week — mitigate with short cache TTLs and a
  subscription-state change feed, trading cache churn for revenue protection; or an
  expired-but-cached-canceled member gets locked out mid-binge — mitigate with a grace
  read path keyed on the renewal date); license-service outage = total playback outage
  (mitigate with per-region license serving + stale-serve of recently-valid licenses for
  already-trusted devices over a degraded window — the member-visible failure then shifts
  from "can't play" to "can't start *new* devices"); device farm abuse (mitigate via
  device fingerprint anomalies and concurrency caps — imperfect, but the economics are
  asymmetric: a household is worth 10 years of churn-avoidance, a pirate farm is worth
  detection).
- Note what DRM is *not*: it cannot stop a determined recorder (analog hole), and Netflix
  does not pretend it can — its purpose is scale-limiting casual redistribution (an
  economic objective, not a mathematical one). Confusing the two leads to over-built
  client security and under-built license infrastructure, which is the wrong ratio.

## 6. Multi-region control plane + chaos-engineering resilience

Publicly reported practice (Chaos Monkey onward, 2011–present-day tooling at Netflix):
deliberately inject failure into **production** to make degradation behavior the tested
path, not the emergency path. The design that makes chaos survivable is the real
substance: a **cellular, per-region control plane** with stateless services that can
regress along a defined ladder (03) and an independent data plane (Open Connect) whose
serving does *not* need the control plane once a session has its manifest and license.

Design: per-region cells (compute + regional state for playback progress/profiles) +
asynchronous cross-region state replication (progress/My List are eventually consistent);
regional load shedding with a *degradation ladder* — drop personalization depth → drop
row diversity/similarity freshness → fall back to a generic-but-non-empty row set → (worst
case) cached catalog with a "something went wrong" header; chaos experiments scheduled as
routine, sized by a blast-radius budget, and gated behind automatic rollback if member QoE
degrades beyond thresholds.

- **Trade-off**: chaos-in-production wins *proof*, not just intent — you discover that a
  region can lose its ranker service and playback still works, before a real outage does —
  and it loses short-term risk (experiments cause real incidents) and engineering cost
  (every service written chaos-first: timeouts, fallbacks, idempotency, are not free).
  Staging-only testing wins safety and loses validity: it cannot reproduce
  member-visible failure modes at true concurrency scale (the 50M-peak session dynamics
  in 02 do not exist in any test environment). We pick production experiments, because
  the alternative is finding out during a member-visible outage that "our fallbacks did
  not fire" — which is the only alternative actually available to anyone (a fallback you
  never fire in testing does not fire when you need it either).
- **Chaos target selection**: kill the components whose failure *should* be invisible
  first (personalization, recommendations) — if an experiment there *does* hurt members,
  the dependency structure is wrong and you just found the bug. Then escalate to the
  harder ones (regional license-serve failover, steering-pool partition), each of which
  must show a graceful degradation per the ladder in N3/01.
- **Failure modes of chaos itself** (the meta-failure): a bad experiment takes real
  members down (mitigate: blast-radius budget — a fixed cap on concurrent experiments and
  % of fleet affected, automatic rollback on QoE guardrail breach, never during peak
  block); experiments teach you only what you already suspected (mitigate: each experiment
  needs a stated hypothesis and a *metric that should have survived* — without a
  counterfactual expectation, you are just breaking things); chaos as ritual (the
  practice decays into theater once the real remediation work isn't done — the metric is
  whether *unplanned* outage minutes drop, not whether you ran experiments).
- Cross-reference: the CDN-side consequence of chaos thinking appears in deep dive 2 —
  steering must *measure*, not trust, exactly because appliance failure is normal enough
  to design around; and [Case 09](../09.cdn/04-deep-dives.md)'s POP failure story is the
  generic version of the same lesson.
