# 04 — Deep dives

## 1. Socket lifecycle under OS constraints: push vs socket handoff (THE dive)

The server (case 03) keeps 100M sockets because *it can*; the client cannot. The OS
owns the process and the radio:

- **Doze / App Standby (Android)** and **iOS background suspension**: a backgrounded
  app gets no network within minutes, no timers (timers coalesce to maintenance
  windows), and eventually a process kill. Any design that assumes "my keepalive
  keeps my socket up" is wrong by construction.
- **Carrier NAT / stateful firewalls**: idle TCP mappings are dropped in minutes
  anyway (network-side, OS-independent) — a second reason app-level keepalives can't
  hold the socket.

Design — a three-state client model:

- **FOREGROUND: real socket.** Live delivery, typing, receipts, presence. The socket
  is a foreground feature.
- **BACKGROUND: push-handoff.** On `onStop`/`applicationDidEnterBackground`, the
  client sends "park me" and closes the socket. The server delivers new-message
  notifications via APNs/FCM. When the user opens the app (or taps the notification),
  the process reconnects and the sync engine catches up via cursors — the *handoff
  pattern*: push is for waking, socket is for transferring.
- **SUSPENDED/DEAD: nothing.** No background state, no background battery. All
  recovery logic lives at next launch.

- **Option A — keep the socket alive in background** (foreground-service + aggressive
  keepalive): wins sub-second message latency while backgrounded. Loses the battery
  budget (a wakeup every ~4 min × hundreds of millions of devices, see 02), fights
  the OS (Android will still doze; iOS will kill you), and gets the app flagged by
  OS battery dashboards — a store-review and user-trust problem.
- **Option B — push-only background**: wins battery and OS compliance; latency is
  push-service latency (seconds, occasionally minutes when push itself is queued on
  the device's doze state).
- **Chosen: B.** For a chat app, a backgrounded user does not need the message in
  300 ms — they need the *notification* in seconds and the content available
  instantly on open. Every downstream component already assumes at-least-once +
  cursor-sync, so the socket being closed is invisible to correctness, only to the
  badge latency. Push carries only conversation-ID/sender metadata (no plaintext
  payload, keeping E2E intact, see case 03 §5); decrypt-and-store-if-small is an
  optional later optimization that costs key access while backgrounded — we skip it
  initially (shortcut: notification preview requires an unlock; acceptable at v1,
  upgrade when users demand it).

**Failure modes**: push service silently dropping notifications (MitM of nothing, just
absence) → user-visible "message arrived only when I opened the app" → mitigation:
on foreground entry the sync engine reconciles *always*, so missed pushes cost only
badge latency, never message loss; push token rotations after app updates → handled
by re-registering token on every launch (idempotent).

## 2. Resumable photo/video upload on flaky mobile networks

A 40 MB video over cellular reliably experiences at least one network break
(estimate, 15%+ of uploads affected — see 02). Re-sending from zero is the naive
failure. Design:

- **Chunked, content-hash-addressed upload**: file split into 1–4 MB chunks
  (bigger chunks = fewer round trips, smaller = less re-send per break; pick ~1–4 MB
  so a single break wastes at most ~1–3 s of radio time — a 1–4 MB chunk at a
  10 Mbps uplink (02's math) is 0.8–3.2 s of radio-on time), each upload carries
  `(upload_session_id, chunk_index, content_hash)`.
- **Idempotent, resumable protocol** (same principle as case 03 §1's wire protocol,
  applied to bytes instead of messages: at-least-once chunks + server-side dedup +
  client ledger → user-level exactly-once): before uploading, `HEAD` the
  blob-by-hash endpoint — if the content already exists (user re-uploading the same
  media, or a linked device already sent it), skip bytes entirely.
- **Resume ledger persisted in the local store** (not memory): upload state survives
  process death mid-upload; next launch resumes from the last ACKed chunk.
- **Chunk ACKs are per-chunk**, not per-upload: a break at 80% means 20% re-sent.
- **Uploads ride plain HTTPS, not the chat socket** (decoupling: an upload failing
  must not block or corrupt message delivery; different retry policy, different
  congestion behavior, and background upload can run under OS background-transfer
  APIs where available).

- **Option A — single streaming PUT with Range resume**: wins simplicity (one
  request). Loses: needs the server to buffer partial state keyed by connection; a
  mid-flight protocol change or proxy retry is unrecoverable; can't verify
  individual chunks.
- **Option B — chunked with per-chunk ACK + ledger (chosen)**: costs an extra
  protocol and a small client table; buys resume after process death, per-chunk
  integrity, and fleet-level dedup by hash. We pick B because the failure mode it
  prevents (re-upload of PB-scale content, see 02's ~150–260 TB/day waste estimate) is
  the expensive one.
- **Bandwidth-aware chunk pacing**: on cellular, send chunks only while the app is
  foreground or under OS-deferred transfer; video uploads queued to WiFi when
  possible (user-configurable). Trade-off: upload latency for metered-network
  kindness — pick kindness by default.

**Failure modes**: hash collision practically impossible (SHA-256-class); partial
chunk corruption despite TLS → per-chunk integrity check at commit; server GC of
incomplete uploads after a TTL (say 24h) → client ledger expires stale upload IDs
gracefully and restarts the file.

## 3. Local DB schema evolution across app versions

The client DB is the only thing the user's history lives on locally, and upgrades
arrive with arbitrary gaps: a device may jump 3 versions in one update, or the
update itself fails, or the user restores a DB backup from a year-old version.

Design — a **monotone versioned migration chain**:

- Every schema change is `N → N+1`, transactionally, with the version stored in the
  DB itself (SQLite `PRAGMA user_version` or a metadata table). No "recreate from
  code", no ORM-auto-migrate.
- **Downgrade policy is explicit: refuse to open.** An older app reading a newer
  schema must not guess. Trade-off: a reverted update bricks the app until the user
  re-upgrades, vs silently losing columns/indexes it doesn't know about. Pick the
  brick — data integrity outranks a transient version revert, and app-store rollbacks
  of native clients are rare enough that support messaging suffices.
- **Copy-of-table migrations, not ALTER-in-place**, for destructive ones: SQLite
  ALTER is weak (`DROP COLUMN` only in newer versions, no type change). Pattern:
  create `messages_new` → copy rows in batches → swap in a transaction. Costs more
  time and 2× space transiently; buys atomicity (crash mid-migration = DB still at
  version N on the old table) — we pick this because the failure it prevents is
  unrecoverable user data loss.
- **Batched long migrations** for huge DBs: a ~100 MB DB copying ~219K messages
  (02's math, sent + received per F2) must chunk per-batch, so cold start doesn't
  blow N2; if a migration can
  pause/resume, it also must not run a giant transaction that starves the OS's
  "app unresponsive" watchdog.

- **Trade-off stated**: migration chains accumulate complexity (v1→v12 runs 11
  steps); an alternative is periodic "re-migrate to canonical schema" squashes. We
  pay the accumulation because a squash is a one-time expensive operation timed
  arbitrarily by the release calendar, while incremental steps amortize into every
  launch's normal path (also 04 §6).

**Failure modes**: migration crashes mid-copy → SQLite transaction rolls back to
version N, old app must still be *able* to run (keep the preceding binary's schema
readable for one release); a migration bug that writes garbage → per-release backup
copy of the DB (or at least the WAL) before starting, so release engineering has an
undo path. Ephemeral/disappearing-message features arrive as schema additions — the
migration chain is how product features ship, which is why it must be monotone and
safe rather than an afterthought.

## 4. Battery-aware reconnection and backoff

Every reconnect = radio wake + TLS handshake + sync cursor check. 02's estimate:
**~20 reconnects/device/day → 116K/s fleet-wide**, and reconnect behavior is the
client's main contribution to edge-tier load (same storm physics as case 03 §2 and
the rate-limit shape in [case 02](../02.twitter/04-deep-dives.md)).

Design — the connection manager schedules reconnects from **OS signals, not timers**:

- React to `ConnectivityManager.NetworkCallback` / `NWPathMonitor` path changes, app
  lifecycle (foreground entry), and push receipts. Never schedule a blind timer.
- **Jittered exponential backoff on failure**: base 1 s → 2× → cap 5 min, jitter by
  ±20% to de-correlate a fleet of devices recovering from the same base-station
  outage (a herd synchronizing on 5-min boundaries would itself be a spike).
- **Battery-state gating**: on low battery (OS BatteryManager / `batteryLevel`),
  lengthen backoff cap and suppress non-message-driven reconnects (typing/presence
  can wait; message delivery still arrives via push — consistent with 04 §1's
  split).
- **Network-type awareness**: metered/expensive networks → chunk-upload pacing
  (04 §2) and reduced receipt fan-out (coalesce read receipts up to ~2 s windows —
  a per-device cost so small it only matters multiplied by 500M, see 02).
- **Server retry hints honored**: if the given server responds with a backoff hint
  (overloaded gateway), the client treats it as authoritative over its own policy.

- **Option A — fixed-interval keepalive + reconnect** ("every 30 s"): wins sub-second
  notification latency and simpler code. Loses battery (2,880 wakeups/day/device vs
  our foreground-only model's ~70: 02's 30 keepalives + ~20 pushes + ~20 reconnects), and fleet-wide it's a per-device tax that
  doesn't scale.
- **Option B — event-driven + jittered backoff (chosen)**: costs a slightly more
  complex state machine; buys battery compliance and storm de-correlation. We pick
  B because the cost of A scales linearly with our user count and the OS
  increasingly *forbids* A anyway (Android doze, iOS background restrictions).
- **State retention across kills**: the backoff state (last attempt time, backoff
  window) persists in the store so a crash-loop of the process doesn't restart the
  backoff curve at 1 s forever.

**Failure modes**: OS delivers spurious network callbacks (VPNs, captive portals) →
dedupe callbacks within a window; captive portal "connected but no DNS" → treat as
disconnected after one failed probe; server outage → backoff cap plus push fallback
keeps the user functional (notifications still arrive — the two delivery paths
independent-fail, which is the design intent).

## 5. Multi-device and session keys: client-side secure key storage

The server side is settled (case 03 §5: prekey bundles, X3DH, double ratchet, one
key server). The client design problem is **where keys live on a phone** and **how a
linked device gets sessions**:

- **Identity (long-term) private key**: generated and bound inside the OS secure
  enclave / Android Keystore where backed by hardware; the key never leaves protected
  storage — signing operations are requested through the OS API. Trade-off: hardware
  binding means no backup export for that key (lose device → new identity key pair
  and re-publish; peers re-verify) vs software keystore (exportable, but the crown
  jewel sits in app-readable files). Pick hardware-bound: identity compromise is the
  one failure no other component can undo.
- **Prekey stock and per-peer ratchet state**: app DB (SQLCipher-class
  file-encryption or app-private storage), not the enclave: these churn constantly
  (one-time prekeys deplete per session, per case 03 §5), and DB-encrypted storage
  is the only practical balance between security and per-message cost. Trade-off:
  encrypted-file storage is OS-complexity-dependent (rooted/jailbroken devices can
  weaken it) vs enclave-per-access cost; for session state, we pick the DB since a
  leaked *session* key compromises one conversation's forward secrecy window, not
  the identity.
- **Linking a new device** (case 03's multi-device model, client side): the existing
  primary device performs the provisioning (QR/short-code pairing over a
  server-relayed channel), transfers its prekey-publishing role and shares a
  per-device session strategy — each device holds **independent sessions with each
  peer** (never shares ratchet state devices-to-devices, which would break forward
  secrecy and multi-device delivery order). Trade-off: more sessions = more key
  material and more storage (02's ~0.4 MB estimate, trivial) vs any shared-state
  shortcut that would make one device a decryption oracle for all of them.
- **Safety numbering / identity verification across devices**: safety-number
  recomputation includes all devices' identity keys, so a rogue linked device is
  visible at verify-time.

**Failure modes**: enclave denial (OS keystore corrupted after OS update) → detect,
prompt user for re-registration (degrade to new identity keys, not to unencrypted);
prekey stock exhaustion while offline (all devices offline → peers can't initiate)
→ graceful 3-DH fallback per case 03 §5 and prekey top-up on first launch; backup
restore that lacks device-linked sessions → sessions renegotiated lazily on demand
(messages encrypted under old session keys still decrypt with ratchet state restored
from chain-backup, otherwise re-fetch decrypted history from the backup channel).

## 6. Startup state restore and the cold-start budget

Cold start is the one user-perceived latency the client fully owns (N2, ≤ 2 s). The
failure to avoid: **a server round-trip before the conversation list renders.**

Design — the local DB is the source of truth for launch:

- **First frame**: process init → open DB → read pinned/recent conversations and the
  last N messages of the top conversation *from local rows* → render. The sync
  engine runs concurrently and reconciles after first paint.
- **DB open on the critical path**: WAL mode keeps a 100 MB DB's open cheap (no
  full-checkpoint read); indexes already exist from prior migrations (04 §3 chains:
  migrations must NOT be allowed to pile up at launch — the migration runner
  executes pending ones *during* launch, and a version gap of >1 triggers the
  batched-copy path from 04 §3, which is exactly why that design keeps cold-start
  acceptable after long app-version gaps).
- **Key store unlock**: session resumption materials fetched from OS keystore
  (~50 ms estimate); the socket connect, if any, happens after first paint — the
  crypto layer never blocks launch on network.
- **Post-launch reconciliation** (sync engine): compare per-conversation sequence
  cursors with the server; fetch gaps. Server is at-least-once (case 03 §1), so
  client dedup on message UUID closes the loop.
- **What must never be on the launch critical path**: network fetches, full-DB
  migrations (batched), key agreement for new sessions (lazy per-open-conversation),
  media (lazy by content hash from blob store).

- **Option A — fetch-on-launch** (web-client pattern, see
  [case 19](../19.frontend-chat-app/04-deep-dives.md) where the browser guarantees
  nothing and server fetch is the only truth): wins simplicity and always-fresh.
  Loses offline functionality entirely and puts launch latency at the mercy of the
  network — unacceptable when the OS may have killed the app 3 days ago.
- **Option B — local-first, reconcile-after-paint (chosen)**: costs local storage
  (02's ~100 MB estimate) and a reconciliation protocol; buys cold start bounded by
  device I/O only, and full offline reads — which chat users expect as the base
  behavior. Chosen because N2 is a hard requirement and A cannot meet it on a
  disconnected device.
- **Restore-after-reinstall** (user wipes app, keeps account): same code path as
  launch, but DB is empty → restore from cloud backup (F8, encrypted, out of
  deep-dive scope) or start empty + catch-up sync. The cold-start design must not
  assume a non-empty DB: every read path is guarded for the fresh-install case.

**Failure modes**: DB open blocked by an fs-level lock from a previous crashed
process → SQLite WAL recoverable by design; stale cursor (server compacted history
beyond cursor TTL) → client detects sequence mismatch and falls back to a bounded
"recent messages" fetch, accepting a history gap with a visible notice rather than
silently pretending nothing is missing; battery-saver OS mode delaying process
warmup → cold-start budget assumes mid-range device, degrade gracefully (spinner
after first frame, never a white screen).
