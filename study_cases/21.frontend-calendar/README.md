# Case 21 — Frontend System Design: Calendar Application

**The system**: the *client* of a Google Calendar/Outlook-like calendar — the app itself,
not the event servers behind it. Like [Case 19](../19.frontend-chat-app/README.md), the
server contract is taken as given: REST/push sync API, per-user event store, a notification
pipeline. This case designs what runs inside the app: an offline-first local store, a sync
engine, recurring-event expansion, timezone handling, and fast rendering of dense views.

**Why it's hard** (all client problems):

- **The calendar must work with zero connectivity.** You open your phone in a subway and
  your whole schedule is there, and edits you make offline must survive to merge cleanly
  hours later. This is stricter than chat: chat tolerates "message arrives when it arrives";
  a calendar edit lost is a missed meeting.
- **Events are not rows; some are infinite series.** "Weekly standup" is one stored record
  that must *render* as hundreds of occurrences, with per-occurrence exceptions ("this
  Friday moved to 3pm") — without ever materializing an infinite list.
- **"9am on Tuesday" is ambiguous.** Meetings must stay at 9am through DST changes, in the
  timezone they were created in, even when the user flies to another continent, even when
  governments change timezone rules retroactively. Naive datetime storage fails all of these.
- **Convergence without a meeting.** Two devices edit the same event offline; both must
  end at the same state without a merge UI nobody wants to read. Conflict resolution is
  in scope, not an "assume it goes away" footnote.
- **Dense rendering.** A month view over ten subscribed calendars (team, rooms, shared
  holidays) easily shows 1,000+ occurrences. Grid layout per day column must be cached
  and invalidated per-day, never recomputed globally.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: local-first store + outbox replay, delta pull with sync tokens,
field-level last-writer-wins on structured records, lazy expansion of infinite series,
wall-clock-stable datetime modeling, server-decided notification delivery, per-view-unit
layout caching. The sync/reconciliation ideas are the structured-record twin of chat's
message sync in [Case 03](../03.whatsapp/04-deep-dives.md) and [Case 19](../19.frontend-chat-app/04-deep-dives.md).

**Try it first**: before reading 03/04, answer: (1) An offline device renames an event;
another device reschedules it. What state do both end in, and what metadata made that
decidable? (2) How do you query "everything overlapping next month" when some of those
events are defined by an RRULE with no end date? (3) Why can't the client just schedule its
own local reminders?

**Primary sources**: synthesis case — no vendor-specific claims; IANA timezone-database
behavior (tzdata updates changing past instants) is public and cited as such. All scale
numbers are labeled estimates.
