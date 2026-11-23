# Case 27 — Related cases, ranked by architectural similarity

The 45 other study cases, evaluated by similarity to this case's hard
problems: (1) gateway fleet for ~100K unreliable embedded clients with reconnect
storms, (2) at-least-once ingest → exactly-once money effect via idempotency/ledger,
(3) offline device autonomy + replay/dedup at the resync moment, (4) fanout-on-write
to ~30 external partner orgs with per-partner backpressure, (5) firmware OTA to a
fleet you cannot physically touch, (6) cross-organization stale-data SLA.

Judged on shared hard problems and architectural patterns, not domain surface.
Each case scored 1–10 by one dedicated review agent that read both cases' README,
requirements, architecture and deep dives, anchored at 03.whatsapp=9 and
01.youtube=2; ties broken by how central the shared pattern is to this case's
own design mass (gateway fleet and money ledger weigh heaviest).

| # | Case | Score | Dominant shared pattern |
|---|------|-------|--------------------------|
| 1 | [03.whatsapp](../03.whatsapp/README.md) | 8 | Connection fleet on flaky networks + at-least-once delivery with dedup and offline resync replay — 2 of 6 shared, still the strongest non-identical neighbour |
| 2 | [05.discord](../05.discord/README.md) | 5 | Stateful gateway tier with reconnect-storm admission shaping + at-least-once idempotent fanout; no money, no devices, no partners |
| 3 | [44.payment-system](../44.payment-system/README.md) | 4 | At-least-once ingest → exactly-once money via idempotency keys + insert-if-absent immutable ledger; cross-cited both ways |
| 4 | [36.airbnb](../36.airbnb/README.md) | 4 | Exactly-once money effects: idempotency-keyed ledger + corrections-as-new-entries, plus an intra-org staleness contract echo |
| 5 | [13.kafka](../13.kafka/README.md) | 4 | The plumbing of this case's ingest: durable per-key log absorbing replay bursts, consumer groups as fanout/backpressure |
| 6 | [28.uber](../28.uber/README.md) | 4 | Flaky-client ingest over an event log into an idempotent persist-then-ack money-adjacent state machine |
| 7 | [06.apple-app-store](../06.apple-app-store/README.md) | 4 | Client-minted idempotency keys → exactly-once ledger effect; signed receipts = offline authorization mirror, no connection fleet |
| 8 | [29.uber-eats](../29.uber-eats/README.md) | 4 | Idempotent order transitions feeding an immutable settlement ledger; phone-shaped retries, not device queues |
| 9 | [04.deployment-system](../04.deployment-system/README.md) | 4 | Health-gated wave rollback + at-least-once→harmless-duplicate fencing; the server half of fleet OTA/canary \* |
| 10 | [24.mobile-chat-app](../24.mobile-chat-app/README.md) | 4 | Offline device autonomy from the device seat: durable queue, reconnect-storm math, idempotent resync replay |
| 11 | [32.spotify](../32.spotify/README.md) | 4 | Offline client autonomy reconciled at resync (rights, not money); declines the gateway design it names |
| 12 | [46.google-docs](../46.google-docs/README.md) | 4 | Offline op queue replayed and deduped against a server-sequenced durable log; no money, no storms |
| 13 | [02.twitter](../02.twitter/README.md) | 3 | Fanout-on-write with per-recipient queues and slow-path isolation — the pattern this case re-aims at OCPI partner orgs |
| 14 | [37.ticketmaster](../37.ticketmaster/README.md) | 3 | At-least-once ingest → exactly-once money via idempotency keys + ledger, near line-for-line the same dive |
| 15 | [39.e-commerce](../39.e-commerce/README.md) | 3 | Idempotency-keyed money correctness under client retries; outsources the ledger, no device fleet |
| 16 | [34.reddit](../34.reddit/README.md) | 3 | Keyed idempotent ingest: at-least-once event log → exactly-once effect, with deliberate fuzzing instead of money stakes |
| 17 | [16.dalle](../16.dalle/README.md) | 3 | Durable at-least-once jobs with idempotency keys yielding exactly-once money effects; worker retries, not device replays |
| 18 | [18.sora](../18.sora/README.md) | 3 | At-least-once execution collapsed to effective exactly-once via stable-key idempotent compute; idempotent-compute, not idempotent-money |
| 19 | [08.chatgpt](../08.chatgpt/README.md) | 3 | Persistent-connection gateway tier + canary ladder, both shallow and explicitly borrowed from WhatsApp |
| 20 | [17.elevenlabs](../17.elevenlabs/README.md) | 3 | Long-lived websocket layer with reconnect-resume (echo of the WhatsApp pattern), at friendlier client scale |
| 21 | [43.zoom](../43.zoom/README.md) | 3 | Persistent client sessions surviving network flaps via a server-owned reconnect state machine; no storms, no money |
| 22 | [11.zookeeper](../11.zookeeper/README.md) | 3 | Session/lease liveness, half-open-vs-dead, and reconnect-storm herd handling at tiny scale, with fencing tokens |
| 23 | [38.leetcode](../38.leetcode/README.md) | 3 | Push delivery of async results + reconnect-storm shaping + idempotent event-log read models; browser-shaped only |
| 24 | [30.facebook-messenger](../30.facebook-messenger/README.md) | 3 | Write-fanout with per-recipient parked queues and seq-ordered offline replay; WhatsApp minus the shared part |
| 25 | [41.web-crawler](../41.web-crawler/README.md) | 3 | At-least-once replay with keyed consumer-side dedup over durable recovery state; corpus consistency, not money |
| 26 | [09.cdn](../09.cdn/README.md) | 3 | Control-plane push with bounded-staleness contracts + exactly-once billing from logs (each a paragraph, not a dive) |
| 27 | [26.google-maps](../26.google-maps/README.md) | 3 | Freshness-budgeted telemetry ingest with unknown-beats-stale serving; producer side of the availability loop, thin by design \* |
| 28 | [19.frontend-chat-app](../19.frontend-chat-app/README.md) | 3 | Durable idempotent offline outbox + seq-gap resync, deliberately client-side only (server out of scope) |
| 29 | [21.frontend-calendar](../21.frontend-calendar/README.md) | 3 | Offline outbox: ordered replay, idempotent resync dedup, sequence-not-clock ordering — the same problem without money or scale |
| 30 | [22.frontend-uber](../22.frontend-uber/README.md) | 3 | Client-side offline intents + snapshot-then-fold resync; the consumer twin of this case's device story |
| 31 | [25.mobile-newsfeed](../25.mobile-newsfeed/README.md) | 3 | Durable outbox, last-good state, idempotent replay on reconnect; tap-scale stakes, dedup delegated to the server |
| 32 | [14.ml-production-system](../14.ml-production-system/README.md) | 3 | Staged artifact rollout with health gates + rollback and stream reconciliation; the willing-fleet variant of OTA only |
| 33 | [01.youtube](../01.youtube/README.md) | 2 | Idempotent retryable async ingest; no connection fleet, money, offline, or partner contracts (the low anchor) |
| 34 | [07.blob-store](../07.blob-store/README.md) | 2 | Persist-then-ack ordering + idempotent resumable work units; every hard problem happens offline of this case's |
| 35 | [12.bigtable](../12.bigtable/README.md) | 2 | Durable append-before-ack ingest + staleness-bound reads; a storage engine this case buys, not one it teaches |
| 36 | [10.distributed-cache](../10.distributed-cache/README.md) | 2 | TTL staleness bounds and thundering-herd load shaping only; the session-store component, generalized away |
| 37 | [15.ml-ranking-system](../15.ml-ranking-system/README.md) | 2 | Generic event-log ingest with late data + internal staleness SLA; none of case 27's differentiators |
| 38 | [33.instagram](../33.instagram/README.md) | 2 | Generic async ingest with idempotent-retry tokens; user fanout deferred by reference |
| 39 | [35.tiktok](../35.tiktok/README.md) | 2 | At-least-once ingest with client-minted idempotency keys; the exactly-once money consequence stripped out |
| 40 | [31.netflix](../31.netflix/README.md) | 2 | Push-to-fleet-with-backpressure and degradation ladders as faint structural echoes only |
| 41 | [42.linkedin](../41.linkedin/README.md) | 2 | Idempotent transactional state machines on a shared event bus; generic, no ledger or fleet |
| 42 | [45.typeahead](../45.typeahead/README.md) | 2 | Generic event-log ingest into derived indexes + bounded-staleness gestures; nothing substitutable |
| 43 | [20.frontend-newsfeed](../20.frontend-newsfeed/README.md) | 2 | Anchor-cursor and dedup vocabulary only; near-zero overlap, borders 1 |
| 44 | [23.frontend-video-streaming](../23.frontend-video-streaming/README.md) | 1 | Client-side flaky-network retry resilience as a theme; designs none of the server patterns |
| 45 | [40.tinyurl](../40.tinyurl/README.md) | 1 | KV read hot path with insert-if-absent writes; its side-pipeline drops events, the opposite delivery contract |

\* 04.deployment-system and 26.google-maps rank lower than their cross-links
suggest. This case's README leans on 04 twice (as the outbox/fencing precedent and
the fleet-side OTA mirror) and explicitly hands Maps the consumer end of the
availability loop — but the rescore weighs shared hard problems, and 04 has no
connection-fleet or money content (its "exactly-once" guards state, never billing),
while Maps deliberately scopes away the OCPI partner fanout that fills most of this
case's partner-facing dive. A load-bearing citation is not a shared problem.

## The shape of the answer

Rescoring with one reviewer per case (against the 26-case, five-reviewer panel
that produced the previous version of this ranking) compresses the top and widens
the middle:

- **03.whatsapp still stands alone at the top**, at 8 rather than 9 — it shares the
  connection fleet and the delivery-semantics rigor whole, but its offline receiver
  is a passive mailbox, not an autonomous money-handling device, and 4 of 6 hard
  problems have no WhatsApp counterpart.
- **The money cluster is now the largest band.** 44.payment-system, 36.airbnb,
  29.uber-eats, 37.ticketmaster, 39.e-commerce, 16.dalle and 18.sora all surface as
  genuine partials of this case's heaviest deep dive — idempotency keys → immutable
  ledger → corrections-as-new-entries. 13.kafka takes the ingest substrate and
  21.frontend-calendar / 22.frontend-uber / 19.frontend-chat-app take offline
  outbox replay from the client seat, mostly stripped of money.
- **02.twitter and 05.discord carry the other two named patterns**: fanout-on-write
  with per-recipient backpressure (13) and gateway-tier storm shaping (5).
- **The old 3–6 tier collapsed.** 04.deployment-system, 09.cdn, 26.google-maps,
  20.frontend-newsfeed etc. moved down when judged strictly on the six hard
  problems rather than on shared vocabulary: one shared sub-problem or a shallow
  echo scores 3, a generic pattern scores 2, and 23.frontend-video-streaming and
  40.tinyurl drop to 1 — a theme is not a pattern.

No case shares more than two of the six hard problems. This case's difficulty
lives where the gateway fleet, the money pipeline, the offline replay moment and
the partner contracts intersect, and every other case here sits on one edge of
that intersection at most. Cases above the floor are study material for one deep
dive each; none is a template for the whole.

This file is a similarity analysis, not one of the five canonical case files
(README + 01–04). It sits outside that fixed layout by design.
