# 04 — Deep dives

## 1. A gateway fleet for 100K flaky embedded clients

The chargers are not phones or browsers. They are embedded Linux boxes on power-cycled 4G
modems behind carrier NAT, built by five hardware vendors across three OCPP generations,
running firmware we pushed months ago. The gateway's design burden is **disconnect
churn and reconnect storms**, not steady-state throughput.

Design decisions inside this:

- **Keepalive ahead of NAT**: carrier NAT idle-timeout is 2–5 min; OCPP heartbeats run at
  60 s (charger-initiated request/response — the charger proves the link each minute;
  WebSocket ping/pong covers the server→charger direction), with TCP keepalive at the
  gateway as a second net. Getting this wrong
  shows up as thousands of half-open sockets: the server thinks the charger is there, the
  charger's next send fails, the charger reboots its connection with a replay of buffered
  events. The fix is round-trip liveness, not
  TCP state alone.
- **Stateless acceptors + device shadow**: no charger-specific state lives in gateway
  memory — device shadow (config, pending commands, session context) sits in Redis/DB. Any
  node can pick up any reconnect. Trade-off: one extra store lookup per connection vs the
  alternative — sticky gateways — which turns every gateway deploy into a reconnect storm
  we'd create ourselves. We pick the lookup.
- **Admission shaping, not connection throttling**: during a storm (~400 sockets/s for
  2 min after a 10-min carrier outage — [02](02-scale-estimates.md)), the cost is not
  socket memory, it is (a) TLS handshakes amortized by load balancers, (b) the replay
  burst of queued device events hitting the bus, (c) shadow writes. We admit connections
  immediately (the charger is a paying customer mid-trip) but shape *replayed message
  intake* per charger with a token bucket, so the storm's drain burst — [02](02-scale-estimates.md)
  sizes the unshaped replay from ~50K devices at 0.5–1.5M messages over the window, some
  0.6–2.5× the ~5–7K peak budget — stays within the bus's and consumers' per-key headroom.
- **Persist-then-ack**: OCPP semantics say the charger deletes a queued record from local
  flash the moment the response arrives — so the gateway responds only *after* the event is
  durably appended to the bus. An "acked in memory, crashed before publish" window in the
  receive→publish path is the exact unbilled-session hole the ingest-durability requirement
  forbids, and gateway crashes there are the most common crash point in the system.
  Trade-off: the durable append sits on the response path (a few ms, plus coupling gateway
  health to the bus's); ack-then-journal re-creates the lost-record window and the journal
  is just a slower bus. We pay the few ms.

**Trade-off**: stateless gateways lose cheap per-connection features (in-memory backpressure,
connection-local command routing) and pay a shadow store on every reconnect path; the win is
deploys, autoscaling, and failure behavior that doesn't itself cause outages — the same
conclusion Discord reached for guild connections ([05.discord](../05.discord/README.md))
and WhatsApp for message links ([03.whatsapp](../03.whatsapp/README.md)).

**Failure modes**: half-open socket inflation exhausting file descriptors (fix: round-trip
liveness + aggressive idle GC); a thundering herd of replayed `StatusNotification`s from old
stale events overwriting newer state (fix: every event carries a sequence established at
ingest — 2.0.1 `TransactionEvent` carries `seqNo`, but the gateway/adapter assigns the
sequence for meter samples and 1.6 lines, since OCPP carries none on `MeterValues`;
consumers drop out-of-order replays — same lesson as Kafka per-key ordering in
[13.kafka](../13.kafka/04-deep-dives.md) §1); reconnect jitter not mandated device-side by
an old firmware generation, producing a second storm at a precise second (fix: never assume
new-client behavior — shape server-side).

## 2. Offline autonomy and the idempotent transaction lifecycle

The requirement: sessions succeed with the backend fully down. The consequence: the backend
receives a *history*, not a stream — records that were produced hours ago, queued on flash
in the charger, replayed after reconnect, possibly more than once.

Design decisions inside this:

- **Local transaction IDs**: on 2.0.1 the charger mints the session ID (UUID) at
  plug-in, offline or not. The backend never assigns identity to something that already
  started. Dedup is
  two-level: *record* identity is `(charger ID, tx ID, device sequence)` — a re-sent record
  is idempotent; the *session* key is `(charger ID, tx ID)`. The split is load-bearing: a
  healthy session legitimately emits several distinct records (Started, MeterValues, Ended)
  under one tx ID, so a record key without the sequence would drop good sessions — the bug
  alarm fires only when the same record key re-arrives with a different payload.
- **Protocol-generation normalization**: the fleet spans OCPP generations (1.6 legacy
  lines alongside 2.0.1), and both are translated into one internal event model at ingest —
  generation differences end at an adapter, never inside the money logic. The dedup key
  shape holds for both, by different means: 2.0.1 mandates the charger mint the
  transaction ID (UUID); 1.6's `transactionId` is assigned by the *backend* in
  `StartTransaction.conf` and therefore does not exist at all for a session whose start
  was offline — so the ingest adapter assigns and persists a stable id namespaced per
  charger (`charger ID, adapter tx id`), and the field audit (below) checks per-vendor
  whether any device-side transaction identifier actually survives reboot; vendors where
  it doesn't are treated as the estimate/reconciliation path, not dedup-safe. What does
  not translate automatically — meter-value payload shapes, reservation and
  authorization-cache features — becomes per-generation adapter work, priced as such.
- **The same state machine, driven late**: session lifecycle on the backend is a pure
  reducer over transaction events (`TransactionEvent` Started/Ended + `MeterValues`; 1.6
  legacy lines' `StartTransaction`/`StopTransaction` map 1:1 onto these) in device-sequence
  order. A
  session that arrives 6 hours late reduces identically to one that arrived live. The
  *billing pipeline does not know or care* when the session happened — this is what makes
  offline correctness a design property instead of an error path.
- **Offline authorization, bounded risk**: plug-in with backend down → charger checks its
  local cache: (a) our own tokens (recently synced allowlist, cached on device after any
  successful auth), (b) EMV contactless cards (issuer can be consulted offline per card
  network rules — the charger accepts a capped offline floor-limit amount, standard EMV
  behavior), (c) roaming tokens — **not** authorized offline; the charger refuses
  unfamiliar eMSP tokens when offline. A stale-allowlist exploit window is accepted and
  bounded by per-token spend limits + cache TTL; "authorize everything offline" would trade
  revenue control for uptime, and the uptime is already handled by the 2 s–online path.
- **Clock distrust**: chargers drift — RTC batteries, NTP through captive 4G, timezone
  config from five vendors. Ordering of meter samples comes from the ingest-assigned
  sequence (above), not from timestamps; the time *used for pricing* is derived from
  backend receive time anchored to device relative deltas, never raw device wall-clock;
  otherwise a day's drift silently misprices every off-peak/peak boundary in the tariff.

**Trade-off**: local authorization caches win autonomy and lose freshness (a revoked token
charges once more on a stale cache until TTL — accepted, compensated by spend caps); local
transaction IDs win replay-safety and lose server-side uniqueness guarantees (two UUID
collisions aside, we give up "the server minted it, so it's unique" and must instead *verify*
uniqueness on ingest — cheap, keyed).

**Failure modes**: double-billing from replay-without-dedup-key (fix above; test by replaying
the exact queue twice in the test rig — this is the one test that must exist); a charger that
restarts mid-transaction and re-mints a transaction ID for the same physical session →
duplicate session on ingest (fix: charger firmware rule — persisted `local tx id` survives
reboot; field audit detects vendors that don't); replay storms of `TransactionEvent(Started)` for
already-settled sessions (fix: dedup key lookup first, late events route to a reconciliation
service, never to a second CDR). The backend does not rely on device-side persistence it
cannot verify: when a new start arrives for a charger that already has an open session on
the same connector, ingest correlates by meter-register continuity and plug-status events —
merge or alarm, never two live sessions on one physical plug. The mirror failure — a lost
*stop* record (flash dies mid-queue, plug-out during an outage) leaves a session open
forever, CDR never created, idle fees accruing unboundedly — is caught by an open-session
watchdog: no events for ~24 h → backend-initiated remote stop; if the charger is
unreachable, an estimate-flagged CDR from the last known register value goes to manual
reconciliation, never silently to the PSP.

## 3. Meter values → tariff engine → CDR: money correctness on an at-least-once bus

Every session ends as real money in two directions (driver pays, partner eMSP is owed a
split, we owe wholesale energy). The raw inputs are an at-least-once, re-orderable, sampled
telemetry stream. There is no "big number of txns/s" excuse here — 200K sessions/day
([02](02-scale-estimates.md)) is roughly two per second — so every design choice pays for itself
**in dispute-avoidance, not throughput**.

Design decisions inside this:

- **Tariff snapshot at session start**: the CDR is priced against the *resolved* tariff as
  of plug-in — resolved including the driver's eMSP contract rate for roaming sessions —
  and the snapshot is persisted with the session's first event, so re-runs (below) consume
  the same inputs they originally did. Price changes mid-session are a
  new tariff for the *next* session. Trade-off: a driver pays the price they saw when they
  plugged in (fair, legally defensible) vs dynamic repricing (possible, but creates the
  dispute class "I plugged in at night rate and got charged day rate" — we pick the
  snapshot).
- **Roaming and displayed prices**: the tariff universe is a function of (site,
  connector, time, and the driver's eMSP contract) — two drivers on the same stall can be
  billed differently, so the CDR sent to a partner must carry the tariff actually applied
  or the partner's settlement diff (deep dive 5) fails. Symmetrically, the price
  *displayed* on the charger screen and published to partner apps must match the priced
  truth: tariff versions are pushed to chargers' displays and partner price endpoints, and
  the propagation window between version and display is a stated contract item, not a
  hidden gap (German-style tariff-transparency rules make displayed ≠ charged a legal
  problem, not a UX one).
- **Card-present capture is its own money flow**: contactless EMV sessions pre-authorize
  an estimate at plug-in and capture the final amount at session end — usually after the
  driver has driven off. A final amount above the pre-auth runs incremental
  authorization; a declined increment (real, especially under offline floor limits) flags
  a bounded shortfall to the PSP's existing collection process. The daily money
  invariants are split by journey: on card-present flows, every EMV capture matches a CDR
  and every card-present CDR resolves to ≥ 1 capture or a flagged shortfall; on invoiced
  flows (the majority), every CDR resolves to a ledger line — capture, flagged shortfall,
  or billed invoice line — so no daily check alarms on journey type. Trade-off:
  declining over-limit captures loses small revenue vs charging without a fresh
  authorization (chargeback exposure, PSP relationship risk) — we accept the shortfall.
- **Energy = calibrated register read, sample = observability**: the billable kWh is the
  MID-certified meter register delta (the `TransactionEvent(Ended)` final register reading
  minus the start reading), per
  connector. Sampled `MeterValue` telemetry (every 10–60 s) is for the live app, load
  management, and dispute forensics — **never** the billing quantity. A sampled average can
  be contradicted by the certified register; a certified register cannot.
- **Exactly-once *effect* from at-least-once inputs**: the tariff engine is a deterministic
  pure function `f(raw event history for session) → CDR`; the ledger takes CDRs keyed by
  `(session ID, ledger shard)` with insert-if-absent semantics. Re-running the engine on the
  raw store (for disputed sessions, or after a consumer bug) produces byte-identical CDRs
  but cannot produce a second ledger entry. This is the same at-least-once → exactly-once
  contract as the outbox pattern in [04.deployment-system](../04.deployment-system/04-deep-dives.md)
  and the purchase ledger in [06.apple-app-store](../06.apple-app-store/README.md) — with
  less friendly clients.
- **Country VAT and partner splits are ledger features, not tariff features**: the tariff
  engine emits a *net* CDR; VAT derivation and roaming splits (per-OCPI rules) are applied
  downstream as separate ledger entries. Mixing them into one calculation makes every
  tariff change a tax-law risk.

**Trade-off**: sampling rate. 1 s sampling gives silky live-app graphs and forensic clarity,
but ~60× the ingest and storage of 60 s sampling (and ~15× of the chosen ~15 s —
[02](02-scale-estimates.md) sized the
middle at ~15 s) and more bus load during reconnect replay — we chose "cheap enough, keeps
the live app under 15 s staleness"; the billing path doesn't care at any setting. If a
future product (per-second idle-fee audit) needs 1 s, it pays for a separate high-rate topic
for opted-in sites, not a fleet-wide default.

**Failure modes**: tariff config error priced 80K sessions before notice (fix: shadow-run
each tariff change against the previous day's sessions and diff the aggregate revenue before
activation — the cheapest possible canary); a meter register reset (firmware, board swap)
mid-session producing negative kWh (fix: detect at CDR, mark session for manual
reconciliation, never auto-bill a negative or absurd register delta); consumer replay after a
bug fix re-deriving CDRs that were already settled with a partner (fix: reconcile mode is
explicit, produces diff reports, and ledger entries are immutable — corrections are new
entries, like any double-entry system).

## 4. Site-level load management: the cloud advises, the site decides

A 12-stall DC hub may share a 400 kW grid connection; peak simultaneous demand is 12 × 200 kW
= 2.4 MW. Something must allocate. The load allocation problem is a *control* problem, and
putting the cloud in the control loop fails two ways: latency and availability. The
latency one is physics, not preference: overload prevention needs a local reaction in the
~100–200 ms class (est.) — a DC session ramping to full power, breaker margin — and one
WAN round trip alone consumes that budget before the cloud has computed anything, so the
cloud cannot be in the overload-prevention path **even with a healthy network**; when the
WAN or the cloud app itself is unavailable, the consequence window is minutes. Availability: the
cloud being down cannot mean the site overdraws its physical fuse.

Design decisions inside this:

- **Local controller owns the constraint**: a PLC or edge box enforces `Σ site power ≤
  limit`, always, including with WAN down. It speaks locally with chargers over a wired
  local bus — a *procurement* requirement: the legacy installed base is reachable only via
  the WAN, where the cloud's `SetChargingProfile` becomes the effective control path, and
  those sites run conservative static allocations and accept poorer efficiency rather than
  pretend the cloud is not in their critical path.
- **Cloud sends schedules, not commands**: price-driven scheduling ("charge fleet at 03:00",
  solar-following, contractual demand-peak smoothing) is computed cloud-side where the data
  (prices, forecasts, site history) lives, delivered as a *plan* the controller may follow
  within its rules. Trade-off: local-only loses global optimization (grid-price arbitrage
  across sites, demand-peak smoothing vs the grid tariff's monthly 15-min peak — a real
  money lever, roughly the margin between good and mediocre years); cloud-primary loses
  physics. We take advice-shaped compromise.
- **Allocation algorithm at the controller**: simple beats clever over a flaky local bus.
  Equal-share with headroom reclaim ("everyone starts at limit/n; a car that finishes early
  donates its share to neighbors within seconds, a car that can't accept the share is idle
  and donated past") behaves predictably, re-converges after any disturbance within one
  renegotiation (~minutes — cars ramp power slowly; OCPP charging-profile renegotiation is
  not instant), and is explainable to a driver. Per-car-optimal ("give the car closest to
  full its fastest rate first") is better in aggregate kWh/hour *when it works* and thrashes
  renegotiation otherwise — we deploy it as an upgrade behind the shared-headroom baseline,
  not as the baseline. On AC sites allocation is a *phase* problem too: single-phase cars
  saturate one phase while Σ-kW reads fine, so the controller allocates per-phase current
  against the site's per-phase limit — the equal-share baseline applied at phase
  granularity, not site-kW.

**Trade-off**: fairness (equal-share) vs throughput-maximizing allocation — equal share gives
up a few percent of site energy throughput (cars with full batteries hold shares until
reclaimed) for predictability, stability, and a support story at every site; the reclaim
mechanism claws most of that back.

**Failure modes**: oscillation (controller A/B/C disagree on limit → power thrash,
renegotiation storms) — fix: one controller per site is *the* writer, others follow (the
single-writer/fencing reasoning of [04.deployment-system](../04.deployment-system/README.md)
applies to a cabinet, not a cluster); grid connection physically violated during controller
failover — fix: failover is to *conservative default* (all stalls capped to a safe fraction
of limit) until a new leader computes; cloud sends a schedule the controller can't safely
follow at current temperature/cable rating — fix: controller-side safety overrides are
non-negotiable and alarm (cloud learns its plan was refused — the advisory model's error is
visible instead of dangerous).

## 5. Availability and OCPI roaming: fanout to companies you can't page

~30 eMSP partners + POI feeds consume our occupancy and prices; ~60% of our sessions are
authorized through them and their drivers' CDRs flow back. This is fanout-on-write
([02.twitter](../02.twitter/README.md)) with three differences that change the design:
recipients are **companies with SLAs in both directions**, payloads are **contract-governed
schemas that move slowly (OCPI 2.2.1)**, and staleness has a **contractual bound** (≤5 min
occupancy) rather than a product instinct.

Design decisions inside this:

- **Per-partner queue, per-partner rate, per-partner retention**: a partner's API being
  slow or down affects only their queue. Trade-off vs one shared firehose: the shared
  version lets the slowest partner set global latency or else enforces drop-tail on
  everyone — unacceptable when partners contractually depend on *our* fanout quality.
  But the queue is *not* a delivery guarantee: OCPI is push-optimistic and tells senders
  they SHOULD NOT queue and retry failed pushes (OCPI 2.2.1 §4.4, "Offline behaviour") — when a
  partner's endpoint stays down we suspend pushing rather than age a multi-day backlog,
  because recovery is theirs, not ours.
- **Pull resync is the spec-mandated recovery path, not a fallback**: after an outage a
  receiver is expected to pull (GET on Locations/Status/Tariffs/CDRs with a since-filter)
  from the sender to get back in sync. This halves the blast radius of our own fanout
  incidents (*"our pusher is broken, their puller still gets data"*) and it is why we
  keep per-module pull endpoints with retention — they carry the delivery guarantee our
  pushes don't. Failure mode to plan for: 30 partners resyncing at once after a shared
  incident is a read stampede against our pull endpoints.
- **EVSE-level truth, station-level summaries**: OCPI status is per-EVSE (per connector)
  and that is what partners contract for — so the normalizer publishes per-EVSE status
  directly and builds station-level aggregates only for our own app, with hysteresis
  against connector-flap. A two-connector hub must not read "available" because its
  station aggregate looks half-open when the exact EVSE the partner's app sent the driver
  to is occupied. Link loss is a status, too: gateway connection-loss events feed the same
  normalizer, and an EVSE whose charger has missed heartbeats for N minutes is published
  as *unknown*, never as last-known-state — a 10-min carrier blip (the design case in
  [02](02-scale-estimates.md)) otherwise leaves "available" hanging for hours, which is
  the over-promise direction that destroys trust.
- **Reservations ride the same pipeline**: a hold is a timer-bearing state on one EVSE
  (booked → released on no-show or TTL expiry), published through the same push infra as
  charging status — not a separate service. The normalizer consumes holds as first-class
  connector states, because a published "available" that is actually a paid hold is a
  double-booking with a refund, not a data error.
- **Authorization hot path across partners**: an eMSP driver's session start needs our
  2 s p95 to include a *cross-company* round trip when the token is a roaming token. OCPI
  token lookups are pushed (partner pushes token lists to us; we cache) first, pulled (we
  ask their API) only on cache miss — the same push-then-cache-then-pull shape as the
  availability data, pointed at the hot path instead. The latency contract splits
  honestly: cached tokens (the recurrent majority) meet the 2 s p95; a *cold* token rides
  the partner's API, where their latency and outage become ours — on cache miss during
  partner degradation we refuse rather than hang. Accept-and-validate-async would trade a
  bounded fraud window for session completion; we use it only where the partner contract
  allows clawback (spend-capped accounts).
- **CDR settlement is batch, not stream**: OCPI carries CDRs as individual POSTs with
  per-CDR ACK (there is no batch push) — the batch is *our side's* aggregation per partner
  for reconciliation. Our pushes are at-most-once (see the OCPI §4.4 rule above); a
  partner that misses CDRs recovers by pulling them (GET with a since-last-CDR filter),
  so CDRs are retained for resync. Dedupe on the **(country_code, party_id, cdr_id)**
  triple — CDR ids are unique only per country/party — because some partner impls retry
  despite the spec's SHOULD NOT. CDRs are immutable: a wrong one is never replaced, only
  offset by a **Credit CDR** (new id, negative amounts, `credit_reference_id` pointing at
  the original). Monthly settlement totals are diffed against acknowledged per-CDR counts.
  Money flows don't ride the same fanout service as occupancy — different retention, retry,
  and audit requirements.

**Failure modes**: a partner schema silently tolerant of our new enum value until it isn't
(fix: OCPI version pinning per partner; adapter-per-partner so a schema bump is a contained
change, and staging accounts with every major partner for contract tests); our fanout
silently degrading to 40-min staleness and the first complaint arriving via a partner's
account manager (fix: fanout service emits per-partner p95 *push latency* — if a number can
breach a contractual SLA, that number must exist before it does); authorization pull-path
overload during a partner's own outage (fix: circuit breaker with cached-token fallback and
explicit "roaming token, backend down → offline refuse" as decided in deep dive 2).

## 6. Firmware OTA to a fleet you cannot physically reach

100K devices across 5 hardware platforms; a bad rollout is not a bad deploy — it is
thousands of trucks and weeks. All the deployment-system reasoning
([04.deployment-system](../04.deployment-system/README.md)) applies, but the rollback story
is worse: the "old version" is a 150 MB image whose download came over the same flaky 4G
uplink that just went down with the rollout.

Design decisions inside this:

- **A/B partitions + a bootloader that cannot be uninstalled**: charger runs image A or B;
  new image writes the inactive slot, bootloader boots it once *marked unconfirmed* and
  auto-reverts unless software confirms health (reachable, session-capable) within a
  window. This is the only mechanism that survives "the device flashed successfully and
  then lost 4G for a week" — which is the normal case, not the tail.
- **Authenticity, not just correctness**: firmware artifacts are signed and the
  bootloader verifies the signature (and ideally secure-boots the chain) before boot —
  the OCPP update message is just a URL, and an unsigned or mis-routed CDN object is a
  fleet-takeover channel across 100K devices. Trade-off: signing adds a key ceremony and
  vendor coordination (the oldest platforms need a bootloader update before they can
  verify anything — staged in their own slowest waves, see failure modes below).
  Separately, releases touching the
  MID-certified measurement chain follow re-verification timelines in several EU
  jurisdictions — those take a regulation-shaped path coordinated with notified bodies,
  not the fleet-fast path; the ≤2-week NFR applies to non-metering components
  ([01](01-requirements.md)).
- **Staged waves gated on real fleet health, not a fixed percentage**: canary on
  self-selected lab + friendly sites, then waves sized to finish in ~10 active rollout
  days (≈ 100K devices ÷ 10K/day with health gates between), holding back the remaining
  days of the two-week budget from [01](01-requirements.md) for automatic halt-and-resume
  cycles.
  Gates compare the new cohort against the fleet baseline on: charger uptime, session
  success rate, CDR completion, reconnect behavior. Regression on any gate = automatic wave
  pause, human gate to resume.
- **CDN-served artifacts with resume**: firmware blobs from object storage/CDN with
  HTTP range/resume (a 150 MB download dying at 90% over 4G restarts at 90%, or DC fast
  sites in remote areas take days to upgrade). Devices download *before* ever rebooting;
  flash-write happens from local flash, not from the network mid-session.
- **Never brick a charging session**: firmware application is constrained to idle windows;
  a charger mid-session defers until the session ends (OCPP firmware update supports
  install-time policies). Trade-off: rollout takes longer (idle windows at busy motorway
  hubs are scarce) vs bricking a paying session — not a real trade-off.

**Trade-off**: fleet heterogeneity. One firmware per platform version × 5 vendors is a
combinatorial test matrix we can't fully cover; per-platform canaries (one site per vendor
per hardware gen, per wave) is the cheapest mechanism that catches vendor-specific breakage.
The alternative — forcing vendors into one maintained firmware line — is a procurement
policy decision years earlier that no architecture can retrofit.

**Failure modes**: auto-revert loop (bad image boots, reverts, retries forever, burning
flash wear cells) — fix: revert counters perma-pin the last-good image after N attempts and
alarm for field service; mass simultaneous reboot ("fleet restart now") — same jittered-wave
logic as the gateway storm in deep dive 1; firmware confirmed healthy by device metrics but
business-broken (CDRs stop completing because the image changed a queue behavior) — fix:
gates include *business* metrics, not just device metrics; **bootloader update failure** —
the one path the A/B auto-revert cannot save (the bootloader is what executes the revert;
a bad bootloader is the true-brick path, truck roll and weeks, on exactly the oldest
hardware generations) — fix: bootloader updates sit *outside* the A/B safety net: their own
slowest wave, an immutably verified vendor-signed fallback kept on flash, and a canary per
hardware generation before any mass wave.
