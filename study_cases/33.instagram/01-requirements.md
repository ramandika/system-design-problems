# 01 — Requirements

## Functional

**Media sharing (the core)**
- Upload a photo or video from a mobile client; apply client-side filters/edits before the
  bytes leave the device.
- Post to followers (permanent post) or to stories (expires 24h after posting).
- The platform must serve each media item in multiple renditions: several image resolutions,
  a thumbnail, and for video an adaptive-bitrate ladder of encodings (different resolutions
  and codecs so a client on Wi-Fi pulls 1080p, a client on 3G pulls 480p).
- View a single media item in a detail page; media can be deleted or archived by the author
  (takedown must propagate to all renditions, the feed, and caches).

**Follow graph**
- Follow/unfollow; the graph is the input to both the feed and the audience-of-a-story.
- Private accounts: a follow request must be approved; media/story visibility respects this.

**Feed (ranked, hybrid sources)**
- Home feed: personalized, ranked combination of (a) posts from accounts you follow and
  (b) recommended posts from accounts you don't follow — the recommendation share is a
  product decision, not a systems one, but the architecture must allow both.
- Feed is not reverse-chron: order changes between sessions. This is a requirement change
  vs case 02.twitter with architecture consequences (04-deep-dives #3).

**Stories (ephemeral)**
- Story items (photo or short video) visible for 24h from posting.
- Tray shows accounts with active stories, ordered roughly by recency + relationship; a story
  is marked seen per-viewer, and view/seen state is private to the viewer relationship.
- Author sees who viewed (this is an Instagram-specific product decision that forces per-viewer
  state to be stored, not just counted — contrast Twitter views, which are aggregates).
- Optional archive/highlights: an expired story *may* be retained for the author.

**Explore / recommendations**
- Explore grid and Reels (short video) surface content from the whole graph, not just follows.
- Ranking consumes engagement signals: likes, comments, saves, shares, dwell/skip.

**Engagement**
- Like, comment, save, share; counts are eventually consistent and may be cached aggregates.
- Notifications for engagement on your content (out of scope for the write path here).

## Non-functional

- **Latency**: feed open → first screen of ranked media in ~1s on a mid-range phone on 4G;
  media bytes streamed from CDN (<~300ms to first image tile). Upload *ack* ("posted") within
  seconds; heavy transcoding happens after ack.
- **Scale**: hundreds of millions of DAU, ~2B MAU (publicly reported ~2B for Instagram);
  media traffic dominated by reads (orders of magnitude more media GETs than uploads).
- **Storage**: hundreds of TB per day of new renditions; stories add a churn band of ~1.5PB-ish
  of 24h-live media. Author-archived content accrues like permanent posts.
- **Durability asymmetry**: a permanent post's media must be effectively lossless (user
  content = the product); a story's *expiry* is a correctness requirement (shown after 24h =
  bug), while its *durability* within the 24h window is high but its post-window retention is
  optional (archive is best-effort unless the user opted in).
- **Consistency**: counts and ranking features eventually consistent (seconds); feed ordering
  eventually consistent across devices (minutes); story expiry must be enforced to within
  minutes of 24h at most. Private-account permission checks must be strongly consistent at
  edge-serving time.
- **Availability**: reads >> writes, so degrade reads, not writes. If ranking is down, fall
  back to reverse-chron recency; if the CDN is healthy, the feed can render even when ranking
  features are stale.

## Scope decisions

- In: media upload/renditions pipeline, hybrid fanout under ranking, stories TTL mechanics,
  explore/reels ranking hooks, engagement counts.
- Out: DMs (covered by ../03.whatsapp and similar WhatsApp-style messaging
  systems), ads delivery, image *generation*
  (contrast with ../18.sora: our pipeline moves and re-encodes uploaded pixels, it does not
  create them), the client UI itself (../20.frontend-newsfeed, ../25.mobile-newsfeed),
  generic ML production concerns (../15.ml-ranking-system owns feature stores and serving).
