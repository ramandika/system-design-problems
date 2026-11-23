# 04 — Deep dives

## 1. Connection graph: storage model + 2nd-degree computation (THE dive)

**Option A — general-purpose graph database** (native traversals, Cypher/Gremlin-ish):
- wins: dist-2 is a one-line query; arbitrary hops "just work"; no precompute pipeline to
  operate.
- loses: online BFS is O(fanout) per hop — a member with 30K connections means touching
  30K shard-random adjacency lists per hop, per request; ~10K QPS of such lookups ≈
  ~1–10M adjacency list fetches/sec (3 lookups/DAU × 300M × ~100s of lists...), latency
  p99 unpredictable at tail sizes; native graph stores historically hold less throughput
  than a partitioned KV at ~120B endpoints.

**Option B — relational/KV adjacency (partitioned by member) + precomputed dist-2**:
- wins: reads are point lookups (one shard per member's adjacency, cacheable); dist-2
  aggregated offline/streaming where cost is amortized; scales with commodity KV, which is
  where LinkedIn's own stack went (Voldemort/Espresso-lineage stores; Kafka/Samza for the
  rebuild stream — see [13.kafka](../13.kafka/README.md)).
- loses: dist-2 freshness lags by minutes-to-hours; the "path through whom" needs a
  sampled/paths table, not a computed answer; two storage systems instead of one.

**Chosen: Option B.** The product never needs arbitrary-depth traversal (bounded 1/2/3 by
requirements), and the heavy-tail math (30K-cap members → millions of dedup'd 2nd-degree
nodes) kills hot BFS regardless of which store evaluates it. Precompute wins because *what*
to precompute is known and bounded: per-member **dist-2 aggregates** (counts, company/school
rollups for PYMK/search) plus **sampled hop paths** for "how you're connected" UI (a bounded
k paths per pair of interest, expanded from source on demand — source-side BFS of depth 2
over ~400 lists is fine when driven by one user's click, and rate-limited).

- Sub-decision — store shape: adjacency as **KV blob per member** (list of member IDs +
  attributes, one write per edge change) vs **relational edge rows** (one row per edge,
  indexed both directions). Relational rows win transactional invites and "find pending"
  queries; blob wins read throughput and cache locality. **We pick: invite state in a
  relational table (small, transactional), accepted edges in KV blobs (hot, big) — the
  state machine lives where it's small, the graph lives where it's hot.**

**Failure modes**: dist-2 job stalls → PYMK/search badges go stale (degrade: badge shows
"2nd degree" as of last build; hide path names if paths sample is older than threshold).
Shard hot-spotting on a celebrity profile's adjacency → cache the blob aggressively;
celebrity adjacency is read-mostly anyway.

## 2. Connection-request state machine (pending/accept) + notification fanout

States: `SENT → PENDING → {ACCEPTED | DECLINED | WITHDRAWN | IGNORED | EXPIRED}`;
an accept is the only transition that mutates the accepted-edge store.

**Option A — treat invite like follow: apply immediately, let the target "undo"**:
- wins: one write per event; instant.
- loses: semantically wrong product (edges mean "we know each other"); A learns of B's
  employer/location by edge presence *before* consent — a privacy violation, not a UX
  inconvenience; edge store polluted with unaccepted half-edges.

**Option B — pending state machine, accept commits both-side edge atomically**:
- wins: consent is modeled in data; edge store stays clean; audit/history per invite.
- loses: multi-row transaction (invite row + two adjacency blobs); needs idempotency and
  concurrent-request handling.

**Chosen: Option B.** Two races matter and are handled at the state machine, not the client:

1. *Mutual invites*: A→B and B→A pending at once — on the second write, detect the pair's
   pending row in the same transaction and auto-ACCEPT (product behavior: "you two
   connected"), creating one edge and cancelling one notification.
2. *Simultaneous accept + withdraw*: transition guard is a compare-and-swap on state —
   `UPDATE invites SET state='ACCEPTED' WHERE pair=? AND state='PENDING'`, affected-rows
   gate the edge commit; the loser's write matches zero rows and no-ops. The unique
   constraint on (member_a, member_b) normalized pair ordering backs both.

Notifications are fanout from a *state transition event* (→
[13.kafka](../13.kafka/README.md)): invite-sent notifies B (email + push, throttled,
dedup'd; a member with thousands of pending invites cannot trigger thousands of emails/
session). Accept notifies A and downstream systems. Failure mode: notification service
down does not block the invite (async, retried to completion — same discipline as
Twitter's fanout-to-completion, see [02.twitter/01-requirements.md](../02.twitter/01-requirements.md));
the inverse is never true — the *edge* never blocks on notifications.

## 3. Profile service read path: ACL without per-viewer caching

Read:write ≈ 150:1 (estimates), ~35K QPS on profile, peak 100K+. Every read has a
permissions overlay: what this viewer may see depends on degree (1st/2nd/3rd+/non-member),
Private mode, blocked list, and per-section settings ("connections visible to connections
only", "no email/headline for 3rd-degree").

**Option A — cache the *rendered, permissioned* profile per (profile_id, viewer_degree_bucket)**:
- wins: ACL applied once at write-to-cache time; viewer service does zero ACL work.
- loses: key cardinality forks (degree × Private-mode × blocked-list membership × section
  settings) — one member's profile may need several cache entries that must all be
  invalidated on *any* edit or settings change; a blocked-list change must invalidate
  entries in buckets the blocker appears in — easy to get silently wrong, and a wrong
  entry is a **privacy leak**, not a stale badge.

**Option B — cache only the canonical, viewer-independent profile doc; evaluate ACL per
request at the service from cheap per-request inputs (degree from graph service, mode from
session, blocks from a small per-viewer set)**:
- wins: one cache entry per profile (max hit ratio, ~95% goal —
  [10.distributed-cache](../10.distributed-cache/README.md)); invalidation stays simple
  (edit → purge one key); ACL logic bugs are *fail-closed and visible* at service level,
  testable in isolation; degree lookup is a point read on membership in a small set.
- loses: every request pays ACL evaluation — mitigated by the degree answer being a
  handful of integer set-membership checks, not a traversal (thanks to dive 1's precompute);
  heavy members' docs must be split hot/cold (positions vs endorsements) so the cached blob
  stays in the ~10 KB range.

**Chosen: Option B.** The per-viewer cache is the design that couples *correctness* to
cache-coherence bugs; we refuse to put privacy in the cache. The ACL evaluation cost is
bounded to the degree-check (precomputed) and per-viewer block set (small, itself cacheable
per *viewer*, which is safe: it's the viewer's own data).

**Failure modes**: ACL service unavailable → fail **closed** (render public-only skeleton,
never fail open); cache stampede after a viral profile → request coalescing/ttl jitter;
degree data stale by minutes → worst case wrong *annotation* (badge), never wrong
*visibility*, because non-degree ACLs (Private mode, blocks) are read at service time from
authoritative small sets.

## 4. Professional feed: the ranking surface

The feed is *not* Twitter's hybrid-fanout timeline
([02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md) §1): content is mixed-source
(posts, jobs, shared articles, groups, sponsored), ordering is relevance, and the
professional signals are the delta:

- **Job change**: "X started a new position" — an event-weighted signal (high engagement),
  which makes *profile edits* feed-producing events. Profile edit → event bus → feed
  candidates. Same pipeline as PYMK: one event bus, many derived stores.
- **Endorsements**: skill endorsements from 1st-degree connections act as affinity +
  credibility features; their write rate is low (estimates: single-digit per member/month)
  so they're cheap features but strong ones.
- **Comment/like probability, dwell time, network distance, recency, company/school
  affinity** — the standard feature families; the generic funnel (retrieval → light rank →
  heavy rank → policy/blend) is owned by [15.ml-ranking-system](../15.ml-ranking-system/README.md).

**Option A — one global ranker over a static candidate pool (Twitter-style: fetch the
last N items from your feed store, score)**:
- wins: dead simple; no retrieval layer.
- loses: wrong shape — professional feed candidates come from *heterogeneous sources*
  (connection posts are few and fresh; job recs are large; sponsored must be slotted; PYMK
  is a different system), and the score-virality profile differs entirely from
  broadcast-media (a long-tail recruiter's post matters more here than a celebrity's).

**Option B — multi-source retrieval, per-source quotas, then staged ranking**:
- wins: each source is an independent system (own SLO, own freshness); the funnel bounds
  compute (estimates: ~35–85M candidate scores/sec → staged reduction by ~10× per stage
  makes the heavy model tractable); sponsored/organic blend is a *policy* layer, testable
  without retraining.
- loses: a blending/joining layer must exist (candidate ID → content hydration, dedup,
  interleave); source-service failures degrade feed *variety* (a dead source must be
  skippable, not fatal).

**Chosen: Option B.** The sources already exist as separate systems (graph, jobs, ads);
forcing them through a single materialized feed recreates Twitter's timeline without
needing its property (single fresh author stream). Storage: no per-user materialized feed
— the feed is *assembled at request time* from ranked candidates; assembly results are
cached very briefly (seconds) to absorb re-opens, unlike Twitter's deep timeline cache
([02.twitter](../02.twitter/02-scale-estimates.md)) because re-ranking on each open is the
product.

**Failure modes**: heavy ranker latency spike → circuit-break to light ranker (feed
renders, quality drops, p99 holds); candidate-source outage → quota redistributed to
remaining sources; ranker model stale/broken → last-good-model fallback (see
[15.ml-ranking-system/04-deep-dives.md](../15.ml-ranking-system/04-deep-dives.md)).

## 5. Degree-2 privacy + Who's Viewed Your Profile (WVYP)

The ACL model by degree (product-visible):

| Degree | Profile body | Contact info | Connections list | In-network path shown |
|---|---|---|---|---|
| 1st | full | full | visible | n/a |
| 2nd | full | restricted (no email/phone) | not listed | yes, via 1 or 2 sampled paths |
| 3rd+ | restricted by settings | not shown | not listed | no |
| Private-mode viewer | appears as anonymous | excluded from WVYP listing | — | — |

WVYP mechanics: every profile read emits a view event (profile_id, viewer_id, timestamp,
degree, mode) onto the bus → real-time OLAP (Pinot-lineage) answers "who viewed you" over
the recent detail window (~5–30 days, the store's retention in 02-scale-estimates; weekly
rolls cover longer history); Private-mode *records the view for counts but never
exposes the identity*; the profile owner's notification is dedup'd per viewer per period.

**Option A — enforce degree rules in the profile-service ACL only (dive 3) and treat WVYP
as pure analytics downstream**:
- wins: single enforcement point — ACL can't drift between two systems.
- loses: not literally zero — the *viewer's* privacy choice (Private mode) must still be
  enforced at event-capture time under either option (analytics-side anonymization is
  retroactive scrubbing if capture is wrong), and the consumer-side identity filter is a
  second copy of a rule that can rot silently; A minimizes it, it doesn't delete it. The one
  hard rule remains: the identity-exposure decision must happen at capture time, in the
  pipeline that creates the event, not at consumer render time.

**Option B — enforce visibility in each consumer (profile service, search, WVYP)**
independently: fails by drift — two implementations of "what may a 2nd-degree viewer see"
*will* diverge; audit failure = leak. **Chosen: Option A**, with one shared policy library;
consumers stay consumers. Cross-link: the same "authoritative store + derived consumers"
split as the invite state machine (dive 2), and the same reason event-capture-time decisions
are privileged over consumer-time ones.

**Failure modes**: view event loss → WVYP undercounts (acceptable: it's an engagement
feature, not billing); private-mode honored at capture but a bug exposes identity →
this is the incident class to design tests for (capture-time filtering assertions in the
event pipeline, not at UI); dedup window wrong → notification spam storms, throttled by
the notification service's per-viewer/per-target caps regardless.
