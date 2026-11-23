# Case 20 — Frontend System Design: Newsfeed

**The system**: the *client* of an infinite newsfeed (Twitter/Facebook-style home
timeline) — the browser or app surface where ranking scores, ad injections and media all
arrive as one smooth, scrollable stream. The ranking backend is a *given*: ranking,
candidate selection and fanout are Cases [02](../02.twitter/README.md) and
[15](../15.ml-ranking-system/README.md)'s problem. This case designs what runs in the
browser: the scroll surface, the pagination protocol, the render pipeline, and the
experimentation machinery.

**Why it's hard** (all client problems):

- **The feed never ends, but the device does.** An infinite scroll session walks hundreds
  of items; naïvely mounting them is memory and layout death. The view is a window over a
  growing paged log — the same windowing problem as [Case 19's](../19.frontend-chat-app/README.md)
  100k-message history, with *ranker-churn* added on top.
- **First paint fights first interaction.** The feed's value is the image+text stream,
  which can stream from the server; but the scroll surface, ads and interactions all live
  in JS that must not block it. SSR vs CSR vs streaming + partial hydration is the central
  rendering decision.
- **The list changes under the user.** Ranking improves, new stories go viral, ads inject:
  items appear at the top of a list the user is mid-scroll in. Naïve insertion scroll-jumps
  the viewport; offset-based pagination duplicates or skips items when the ranked list
  reorders between pages.
- **The client is the ranking backend's biggest cost multiplier.** Every predictive
  prefetch, every impression beacon, every A/B exposure is client-originated load at
  hundreds of millions of users. Client choices set the QPS the backend sees ([02](02-scale-estimates.md)).
- **Experiments must invisible.** Half the traffic renders a variant; a variant that
  arrives late flickers, shifts layout, or desyncs from what SSR rendered. Assignment has
  to happen before first paint and stay sticky.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: windowed rendering over a paged log with stable anchor cursors,
streaming SSR + progressive (island) hydration, predictive pagination gated by network
quality, scroll-anchored insertion of ranked changes, main-thread and bundle budget
hygiene, server-assigned experimentation. The client twins of the timeline-contract
patterns in [Case 02](../02.twitter/04-deep-dives.md) (precomputed timeline, time-sortable
cursor IDs, hydration).

**Try it first**: before reading 03/04, answer: (1) what exactly is the pagination
*cursor* of a ranked, mutating list — an offset, a timestamp, or something else? (2) When
3 new stories rank above the user's current viewport, what are all the ways inserting them
can go visibly wrong? (3) Where must the A/B decision be made so the user never sees a
flicker, and what does that decision cost the CDN?

**Primary sources**: synthesis case — numbers are labeled estimates; where a figure
compares to public scale claims (large-feed DAU, Twitter's poll QPS), it's marked
"publicly reported".
