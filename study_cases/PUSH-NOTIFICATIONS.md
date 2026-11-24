# Push notifications across the case catalog

Where the offline notification plane - APNs/FCM wake-up, background delivery, badges and unread
counts, device-push fanout pipelines - appears in the 50 study cases. Companion to
[CONCEPTS.md](CONCEPTS.md), which has **no row for this plane at all** (its "fanout" row,
11/43, counts feed fanout; notifications appear there only as an example phrase). This doc is
the catalog's first pass on the topic.

Scope split, up front: [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md) owns delivery when the
device *is* connected - sockets, push over a live connection, park-and-pull on reconnect. Here
we own the opposite case: the process is backgrounded, suspended or dead, only the OS vendor's
push channel can reach it. REALTIME-TRANSPORT names the boundary itself; 24's sentence joins
the two planes: **push is for waking, socket is for transferring**.

Headline: **the catalog designs this plane only at its two edges and treats the middle as
vendor plumbing.** At the client edge, every case that touches the plane converges on two
contracts - the payload is a *pointer, not content*, and the push is a *wake-up, not a
delivery* (a dropped push costs badge latency only; sync-on-open is always the real recovery).
At the server edge, notification fanout is a dedup'd/throttled *derived consumer* of an event
stream. The vendor middle - APNs priority classes, push TTL, collapse/silent-notification keys,
OS vendor rate limits - is designed by nobody; the only vendor-adjacent controls named anywhere
are a per-user rate limit on mention pushes (05) and suppression lists (42).

- Cases mentioning the plane substantively (device push, wake-up, or badge design): **12 / 50**
- Full case - the socket/push boundary policy is the subject: **1** (24)
- Core component - device-push pipeline or the client-visible wake contract: **3** (21, 05, 19)
- Moderate - the badge/unread side and notification services: **3** (30, 42, 46)
- Brief / adjacent-plane mention: **4** (43, 29, 16, 49)
- APNs/FCM named by name: **3** (24, 21, 05); 39.e-commerce matched only on "BFCM"; **GCM
  appears nowhere**
- Badge/unread-count design appears in: **4** (30, 19, 05, 24)
- Declared out of scope in the case's own requirements: **5** (02, 20, 25, 33; 19 declares only
  the infrastructure out)
- Cases that design vendor-plane semantics (priority, TTL, OS rate-limit budgets, collapse
  keys): **0**

False-positive filtering (counted nowhere above): `"FCM"` matched "**BFCM**" (Black Friday
Cyber Monday) in [39.e-commerce](39.e-commerce/README.md) - no notification content in the
case; its "badges" are stock/price/ATP UI labels, as are 23's and 42's degree badge.
`"notification"` matched *internal* system notifications: ZooKeeper watches (11), Bigtable
tablet->master metadata notification (12), OCPP `StatusNotification` messages (27), the
deployment outbox (04), Alertmanager alert routing (49 - adjacent plane). `"push"` matched
socket-tier push delivery (03's park-and-push, 28's trip offers, 22's map cadence, 32.spotify's
cross-device sync pushes - owned by [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md) or [SYNC-AND-CRDT.md](SYNC-AND-CRDT.md)), push verbs, and 27's "push token lists" (EV charging tokens).
`"park"` matched parking lots/drivers (27, 36, 28); `"wake"` matched battery-budget wake-ups
(24, 22, 43) rather than the push wake-up itself.

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [24.mobile-chat-app](24.mobile-chat-app/README.md) | Three-state client model: foreground socket / background push-handoff / dead | Full case |
| [21.frontend-calendar](21.frontend-calendar/README.md) | Server-decided reminder pipeline; ~333K pushes/s bursts; cross-device dedup | Core component |
| [05.discord](05.discord/README.md) | APNs/FCM fires only on mention matches; unread path decoupled from delivery | Core component |
| [19.frontend-chat-app](19.frontend-chat-app/README.md) | Client-visible wake contract; badge server-computed at push time | Core component |
| [30.facebook-messenger](30.facebook-messenger/README.md) | Badge/unread counters as a materialized-counter design problem | Moderate |
| [42.linkedin](42.linkedin/README.md) | Notification service: email/push digests, dedup'd, throttled, suppression lists | Moderate |
| [46.google-docs](46.google-docs/README.md) | Notifications as a dedup'd derived consumer of the doc event stream | Moderate |
| [29.uber-eats](29.uber-eats/README.md) | Notification & push service on the bus; soft-recovery push | Brief |
| [43.zoom](43.zoom/README.md) | Control-plane notification svc: invites, "meeting starting" push | Brief |
| [16.dalle](16.dalle/README.md) | Webhooks + poll endpoints: the server-to-server wake-up analog | Brief |
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | Alertmanager collapse of duplicate alerts - same dedup shape, ops plane | Brief (adjacent) |
| [03.whatsapp](03.whatsapp/README.md) | Boundary: park queues are socket-tier; no push service is designed there | Boundary |

## Per-case notes

### The boundary-policy case

**[24.mobile-chat-app](24.mobile-chat-app/README.md)** - the only case that designs the *hand-off* between the two planes. State model ([04-deep-dives.md](24.mobile-chat-app/04-deep-dives.md)): **FOREGROUND: real socket**; **BACKGROUND: push-handoff** - entering background,
the client sends "park me", closes the socket, and the server delivers "new-message
notifications via APNs/FCM"; on wake or tap the process reconnects and sync catches up by
cursor: *"push is for waking, socket is for transferring"*. **SUSPENDED/DEAD: nothing**. The
alternatives are a battery trade-off: socket-alive loses "a wakeup every ~4 min x hundreds of
millions of devices"; push-only wins OS compliance, loses push-service latency ("seconds,
occasionally minutes when push itself is queued on the device's doze state") - chosen, because
"a backgrounded user does not need the message in 300 ms". Payload rule: *"Push carries only
conversation-ID/sender metadata (no plaintext payload, keeping E2E intact)"*; F7: the badge
renders "without waking the app to full execution" ([01-requirements.md](24.mobile-chat-app/01-requirements.md)). Failure mode: the push service *silently dropping* notifications
(absence, nothing signals it) is survivable only because foreground entry reconciles *always* -
"missed pushes cost only badge latency, never message loss"; token rotation by re-registering
on every launch. Economics: ~20 pushes/day/device at ~100 B = 1 TB/day fleet-wide; the fixed-interval keepalive alternative costs "2,880 wakeups/day/device vs our foreground-only model's
~70" ([02-scale-estimates.md](24.mobile-chat-app/02-scale-estimates.md)). One inconsistency:
the scope-out list says the push service is "given. Designed in
[03.whatsapp](03.whatsapp/README.md)" - but case 03 has no push-service design (zero APNs/FCM
mentions; socket-tier park queues).

### Device-push pipelines: mention-only and calendar-reminder fanout

**[21.frontend-calendar](21.frontend-calendar/README.md)** - the catalog's only device-push
*burst arithmetic*: ~5 reminders/user/day x 10M users = 50M pushes/day = ~580/s average (50M /
86,400 s), but meetings start on :00/:30 boundaries, so 30-50% land in ~60 s bursts - "a 40%
burst: 20M in 60 s = ~333K pushes/s during bursts, a spike of ~576x the average"; payload
consequence: *"reminder payloads must be small (id + summary + fire time)"*, delivery "best-effort within the minute" ([02-scale-estimates.md](21.frontend-calendar/02-scale-estimates.md)). The scheduling dive ([04-deep-dives.md](21.frontend-calendar/04-deep-dives.md))
weighs client-scheduled local notifications vs server-decided pushes: A wins offline firing,
loses multi-device duplication, stale schedules, OS-evicted alarms; B (chosen as authority)
wins "cross-device dedup by reminder ID", loses connectivity at fire time and platform
delays - hedged with bounded <=24 h local alarms reconciled on reconnect. Late delivery: render "now",
never "late by 3 min"; drop a reminder >1 h late if the event already started. Sync rides the
push channel as pure pointer: FCM/APNs/WS carries "invalidate, pull now" hints, never payloads
- "a broken/pushed-missed hint costs one delayed pull, never lost data"
([03-architecture.md](21.frontend-calendar/03-architecture.md)).

**[05.discord](05.discord/README.md)** - pushes as a *filter*, not a delivery channel
([04-deep-dives.md](05.discord/04-deep-dives.md)): delivery goes to connected sockets
("membership != delivery subscription"); offline attention takes two decoupled paths - unread
counters, and *"push notifications (APNs/FCM) fire only on mention/ping/mention-role
matches... they resolve mentions eagerly and are rate-limited per user"*. The sharpest push-by-economics argument in the catalog: fanout-to-members would make push volume = membership (their
estimate: hundreds of millions of member-deliveries/s), so pushes ride an event-rate filter;
fanout/notification workers (unread-count increment, mention resolution -> push) are stateless
async consumers ([03-architecture.md](05.discord/03-architecture.md)). Trade-off explicit:
they "lose exact real-time unread counts for offline users... a briefly-wrong unread badge is
recoverable state, not lost data - the same must-not-lose/must-not-cost-anything split" as case
03.

### The client-side wake contract

**[19.frontend-chat-app](19.frontend-chat-app/README.md)** - the client-visible half, with the
infrastructure explicitly out of scope: "Push notification infrastructure and OS notification
plumbing (we specify the client-visible behavior only)"; content rule: "the notification
content is computed server-side; the client just displays and syncs"
([01-requirements.md](19.frontend-chat-app/01-requirements.md)). The badge is the interesting
part ([04-deep-dives.md](19.frontend-chat-app/04-deep-dives.md)): unread is client-computed
from `last_read_seq` markers because under E2E the server "cannot distinguish 'unread for you'
from noise" - yet *"the notification badge on lock screens is server-computed at push time
anyway (approximate by design, self-corrects on open)"*: the lock-screen badge is deliberately
the cheap, wrong-until-open copy, while in-app correctness lives in derived marker arithmetic.
Multi-device: `last_read_seq` is "user state, not device state - reading on desktop must clear
the phone's badge"; "drift until sync" is the accepted failure.

### The badge side

**[30.facebook-messenger](30.facebook-messenger/README.md)** - the server-side badge problem at
web scale ([04-deep-dives.md](30.facebook-messenger/04-deep-dives.md)): ~600M DAU, unread
counts correct across devices. Three designs: count-on-read (always correct, multiplies the hot
path), materialized per-(user, thread) counters ("a drifted badge is a user-visible bug with no
self-heal"), and the chosen hybrid - counter in the same transaction as the mailbox append, re-derived from rows on thread open; "the badge is allowed seconds of staleness", with an async
reconciler because "a thread nobody opens carries the wrong badge for days". Its offline
parking is socket-tier (parked `(thread, seq)` queues) - owned by [SYNC-AND-CRDT.md](SYNC-AND-CRDT.md) and [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md); only the badge layer belongs here.

### Notification services: digests with dedup, throttling and suppression

**[42.linkedin](42.linkedin/README.md)** and **[46.google-docs](46.google-docs/README.md)**
build the same component: a notification service as an async derived consumer, never on the
write path. 42: "invite + WVYP + feed digests via email/push, dedup'd, throttled - templates,
suppression lists" ([03-architecture.md](42.linkedin/03-architecture.md)); fanout rides a state
transition event on the bus (pipeline side -> [KAFKA.md](KAFKA.md)): "notification service down
does not block the invite... the edge never blocks on notifications", and "dedup window wrong
-> notification spam storms, throttled by the notification service's per-viewer/per-target caps
regardless" ([04-deep-dives.md](42.linkedin/04-deep-dives.md)). 46 shares the discipline by
explicit cross-link: dedup/throttle windows per user and per doc (a document that changes
1,000x/hour must never generate 1,000 emails) ([04-deep-dives.md](46.google-docs/04-deep-dives.md)). Both keep email/push as an undifferentiated tail channel - neither case designs
device-level push semantics.

### Brief mentions

- **[43.zoom](43.zoom/README.md)** - one control-plane line: notification service for invites
  and "'meeting starting' push" ([03-architecture.md](43.zoom/03-architecture.md)); no wake
  semantics designed.
- **[29.uber-eats](29.uber-eats/README.md)** - a "NOTIFICATION & PUSH (state->client)" service
  consuming order events on the bus ([03-architecture.md](29.uber-eats/03-architecture.md));
  when the OS backgrounds the courier app, tracking degrades to "courier nearby" + ETA refresh
  and *"a notification asks the courier to reopen the app - a soft recovery path"* (04) - the
  one legal way to ask a suspended app back.
- **[16.dalle](16.dalle/README.md)** - the server-to-server analog: "poll endpoints + webhooks;
  at-least-once events, client-side dedup" ([03-architecture.md](16.dalle/03-architecture.md))
  - a wake hint, with the receiver's poll/sync as the real delivery.
- **[49.monitoring-metrics](49.monitoring-metrics/README.md)** - adjacent plane (ops alerting):
  both HA Prometheus servers fire the same alert and "Alertmanager's group-duplicate...
  collapses them to one notification" ([03-architecture.md](49.monitoring-metrics/03-architecture.md)). Same dedup shape as 42/46, different transport.

### Boundary adjacencies and explicit exclusions

- **[03.whatsapp](03.whatsapp/README.md)** - fenced off deliberately: "if offline, park; if
  parked > N days" and park-and-push on reconnect are *socket-plane* delivery (the receiver is
  woken by its own reconnect, not a vendor push); [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md) owns it; no APNs/FCM mention exists.
- **[25.mobile-newsfeed](25.mobile-newsfeed/README.md)** - the most complete exclusion: "Push
  notifications, badges, background sync scheduling - treated as OS-platform plumbing; only the
  client's *policy* on receiving them is in scope" ([01-requirements.md](25.mobile-newsfeed/01-requirements.md)).
- **[02.twitter](02.twitter/README.md)**, **[20.frontend-newsfeed](20.frontend-newsfeed/README.md)** and **[33.instagram](33.instagram/README.md)** - in-app notification
  requirements exist, no plane designed (02: "notifications' full design" out of scope; 33:
  engagement notifications "out of scope for the write path here", and "No feed fanout for a
  like except notification" ([03-architecture.md](33.instagram/03-architecture.md))).
- **[22.frontend-uber](22.frontend-uber/README.md)** - excluded as a verb false positive: its
  push/poll cadence dispute is socket-plane map telemetry ([REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md) counts it as Core); only its wakeup-budget framing reaches this plane.

## The portable pattern

Cross-links: [24.mobile-chat-app](24.mobile-chat-app/README.md) as the reference, [21.frontend-calendar](21.frontend-calendar/README.md) for the pipeline side.

1. **The payload is a pointer, not content.** Two forces converge: E2E encryption (24: metadata
   only, "keeping E2E intact"; 19: content computed server-side, which E2E then disqualifies)
   and burst economics (21: id + summary + fire time). The push says *where to look*, never
   *what you missed*.
2. **The push is a wake-up, not a delivery.** Every case treats the push as lossy and
   unacknowledged: 24 reconciles *always* on foreground entry, so "missed pushes cost only
   badge latency, never message loss"; 21's hints are "invalidate, pull now" only. The
   durability guarantee lives in the store + cursor sync - the transport doc's rule for
   sockets, unchanged.
3. **The badge is a deliberately approximate copy.** Computed server-side at push time (19),
   seconds of staleness with lazy repair (30), eventually consistent reconciled on foreground
   (05), renderable without waking the app (24); correct unread state is derived from markers
   (19) or transactional counters (30).
4. **Dedup, throttling and suppression live at the notification service, off the write path.**
   Per-user rate limits on mention pushes (05), per-viewer/per-target caps and suppression
   lists (42), per-user/per-doc dedup windows (46), fingerprint collapse (49), cross-device
   reminder-ID dedup with dismissal acks (21). The pipeline side is the event backbone's
   derived-consumer pattern (-> [KAFKA.md](KAFKA.md); cases 42, 46, 29).
5. **The socket/push boundary is a policy, decided once.** 24's three-state model is the
   template - foreground socket, background push-handoff ("park me"), nothing in dead - plus
   the recovery ask (29) and the authority decision (21: server decides, bounded local hedge).
   05 adds the third input: push volume is an event-rate question, not a membership question.

### Where the shape ends

- **Nobody designs the vendor middle.** No case specifies APNs/FCM priority classes, push TTL,
  collapse/silent-notification keys or OS-vendor rate-limit budgets; 24 marks the push service
  GIVEN, 25 refuses it as "OS-platform plumbing": the plane is treated as purchased.
- **Push latency is never claimed real-time.** The honest numbers are 24's "seconds,
  occasionally minutes when push itself is queued on the device's doze state" and 21's at-or-after-fire-time arrival; 05's argument for live delivery on sockets applies unchanged to
  pushes.
- **One dangling reference, and a catalog gap.** 24's scope-out says the push service is
  "Designed in [03.whatsapp](03.whatsapp/README.md)"; case 03 contains no such design, so no
  server-side push-service case exists in the catalog - the gap this doc documents.
- **In-app notification products are under-designed relative to their ubiquity.** 02, 20, 33
  and 25 all exclude them; only 42 and 46 take an in-app notification event to a dedup'd
  email/push channel, and neither does device-level design.

---

*Counting methodology*: mention-based, per case, from case-insensitive greps of `apns|fcm|gcm`,
`notification`, `badge|unread`, `push/wake/park/offline`, every hit verified in context. Counts
are **not** comparable to [CONCEPTS.md](CONCEPTS.md)'s union-concept counts (different counting
basis; CONCEPTS.md has no row for this plane); 12/50 includes 49's adjacent plane and excludes
39.e-commerce ("BFCM" false positive). Tiers: Full = the boundary policy is the subject; Core =
the plane is a designed component with stated trade-offs; Moderate/Brief = one-component or
one-dive mentions.

