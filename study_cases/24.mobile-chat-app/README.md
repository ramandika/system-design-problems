# 24 — Mobile System Design: Chat Application (the native mobile client)

## What this case designs

The **native mobile client** of a WhatsApp-scale chat app: the app process, its local
database, its socket, its key store, and its upload pipeline running on Android/iOS
under real OS constraints — doze mode, background process kills, NAT timeouts, flaky
cell networks, and a battery the user actually cares about.

**The server is a given.** This case assumes the backend already provides: a socket
gateway, a delivery service with per-conversation sequence numbers, a blob store, a
push notification service, and a key-distribution service (all of these are designed in
[case 03](../03.whatsapp/README.md)). Here we design the hard half that backend-focused
interviews skip: **how the client behaves when the OS is actively trying to stop it.**

## Why this is a different design problem

A web chat client loses its process and gets it back instantly; the browser is always
one tab switch away. A mobile client is systematically suspended:

- Backgrounding kills the socket within minutes; doze mode blocks timers and network.
- The OS hands out three competing budgets (CPU wakeups, network, battery) and the app
  ranks below the alarm clock in all of them.
- The client is the *only* thing standing between the user and message loss: the local
  database and its schema live on a device that may not open the app for weeks.
- Every reconnect wakes the radio (expensive in battery) and every keepalive risks
  being counted as a wakeup by the OS.

So the design questions are inverted from server design: instead of "how do we serve
1M QPS", they are "how do we deliver a message after 9 days of dormancy", "how do we
resume a 40 MB video upload through 12 network switches", and "how does a schema
migration run 3 app versions late on a ~110 MB database on a $100 phone".

## The decisions this case defends

1. **Push-first, socket-on-demand** — the socket is a foreground luxury; APNs/FCM is
   the primary delivery path while backgrounded, with the socket as handoff target.
2. **Single local SQLite database with a monotonically-versioned migration chain** —
   the client DB is a durable system of record, not a cache.
3. **Chunked, resumable, idempotent uploads** keyed by content hash — the same principle
   as wire-level at-least-once plus client dedup in case 03, applied to blobs.
4. **Connection state machine driven by OS signals** (connectivity callbacks, push
   receipt timers) rather than app-level timers, with jittered exponential backoff.

## Reading order

1. [01-requirements.md](01-requirements.md) — scope; what is explicitly out of scope
2. [02-scale-estimates.md](02-scale-estimates.md) — per-device arithmetic: traffic,
   storage, reconnect math, cold-start budget
3. [03-architecture.md](03-architecture.md) — client component diagram and data flows
4. [04-deep-dives.md](04-deep-dives.md) — the core: six hard sub-problems with
   trade-offs and failure modes

## Cross-references

- [03.whatsapp/04-deep-dives.md](../03.whatsapp/04-deep-dives.md) — the server side of
  this same system: wire protocol, delivery guarantees, E2E key agreement
- [19.frontend-chat-app/04-deep-dives.md](../19.frontend-chat-app/04-deep-dives.md) —
  the same client logic with different constraints (browser vs OS-owned process)
- [02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md) — rate-limit shapes shared
  at the edge (the reconnect-storm physics is case 03 §2)
