# 01 — Requirements

## Functional requirements

The client is one process on one mobile device, talking to a given server fleet.

- **F1. Send/receive 1:1 and group messages** (text, emoji, voice notes, photos, videos,
  documents) through the server's delivery service. Client responsibility: reliable
  submission, dedup on receive, per-conversation ordered render even across reconnects.
- **F2. Offline-first local store.** All conversation history the user has viewed (and
  what arrives while the app is closed) is readable from the local DB with zero network.
  Unsent messages persist across process death and device reboot.
- **F3. Media upload resumability.** A photo/video upload interrupted mid-flight
  resumes from the last acknowledged chunk without re-sending bytes (flaky networks
  make this a weekly event, not an edge case).
- **F4. Delivery receipts, read receipts, typing indicators** — high-churn, low-value
  signals; the client sends them best-effort and coalesces them.
- **F5. Multi-device account.** A phone (primary) plus up to ~4 linked devices, each
  holding its own sessions against every peer, per the Signal-style design in case 03.
- **F6. E2E encryption.** Client holds the identity key, signed prekey stock, and
  double-ratchet session state; the server never sees plaintext or (usefully) keys.
- **F7. Push notification handling.** Client-visible notification when the app is
  backgrounded; the push carries enough metadata to render a badge without waking
  the app to full execution.
- **F8. Backup/restore.** Optional encrypted end-of-chain backup (text history, not
  media) so a device replacement restores conversations. Out of the deep-dive path.

## Non-functional requirements

- **N1. Message durability: MUST-NOT-LOSE.** An accepted send survives app kill, OS
  reboot, network loss of arbitrary duration. (Same guarantee class as case 03's wire
  protocol; here the guarantee lives in the local DB.)
- **N2. Cold start to interactive conversation list: ≤ 2 s** on a mid-range device
  (estimate; the budget is broken down in 02-scale-estimates.md).
- **N3. Battery: minimize radio wakeups and CPU wakeups.** No app-level polling timers;
  every wake is justified by user-visible work. We treat the OS wakeup budget as a
  hard input, not an optimization target.
- **N4. Local storage cap.** Media cache bounded (default ~2 GB, user-adjustable);
  text DB grows but stays in the tens–hundreds of MB over years (estimate, see 02).
- **N5. App version skew.** A client updated through N app versions must migrate its
  local DB without data loss, downgrade-safe to "the older app refuses to open rather
  than silently corrupt" (a deliberate policy, defended in 04).
- **N6. Crash/reconnect rate.** Reconnect logic must not contribute to a thundering
  herd: jittered backoff, honor server-provided retry hints.

## Scope decisions — explicitly OUT

- **The server side** (gateway tier, connection registry, delivery store, key server,
  push service): given. Designed in [03.whatsapp](../03.whatsapp/README.md).
- **The web/desktop client**: see [19.frontend-chat-app](../19.frontend-chat-app/README.md);
  same protocol, wildly different constraints (no OS kill, but no persistent either).
- **Video/audio calling**: a separate connection stack (WebRTC-class); out.
- **Status/stories/stories-media delivery**: out; nothing here depends on it.
- **Custom themes, reactions beyond a fixed set, chat migration between platforms**:
  out — product breadth is not the design problem.
- **Ephemeral/disappearing messages**: noted as an N5 migration case (a schema
  addition) but the enforcement logic itself is out.
- **Spam/abuse**: server-side (and hard under E2E, see case 03 §5); the client only
  surfaces report/block.

## What the constraints actually decide

The hard requirements are the OS ones. N1 × "the process can be killed at any instant"
drives the local DB from "cache" to "system of record" (04 §3). N3 × F1 drives the
push-first connection model (04 §1). F3 × mobile networks drive chunked upload (04 §2).
Everything else is downstream plumbing.
