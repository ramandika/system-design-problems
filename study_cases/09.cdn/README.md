# 09 — Content Delivery Network (CDN)

A global content delivery network in the shape of CloudFront/Cloudflare: hundreds of edge
POPs announcing the same IP space via anycast, terminating TLS close to the user, serving
cacheable static content at 95–98%+ hit rates, proxying non-cacheable dynamic content to
origins, and — because the same pipes carry everything — absorbing DDoS attacks before
they reach customer origins.

## What the system does

- **Cache + reverse proxy at the edge**: customers point their domains at the CDN; every
  request for those domains arrives at the network edge. Cacheable responses are served
  from POP storage; everything else is proxied to the customer's origin.
- **TLS at the edge**: certificate management, termination, and modern protocol negotiation
  (HTTP/2, HTTP/3/QUIC) live at every POP.
- **Control plane**: customer config, cache rules, instant/soft purges, logging and billing.
- **Shield**: a DDoS and abuse absorption layer — the CDN's capacity is the customer's.

## Why it's hard

- The core product is a *cost* product: the entire business is moving bytes more cheaply
  and closer to users than origin operators could themselves. Every percent of hit rate
  and every watt is margin. (See [YouTube's take on the same lever](../01.youtube/04-deep-dives.md) —
  bitrate economics; a CDN's lever is hit-rate economics.)
- Cache correctness and cache performance pull against each other on *every* feature:
  richer cache keys raise correctness and crash hit rates; long TTLs raise hit rates and
  break invalidation; shielding raises hit rates and adds a hop on misses.
- You cannot pre-provision for the worst case anywhere except in the worst case: one
  customer's viral image or one DDoS flood lands on the same hardware that serves
  everything else.

## Reading order

1. `01-requirements.md` — scope, functional/non-functional requirements; what is out of scope.
2. `02-scale-estimates.md` — back-of-envelope sizing: request rate, bandwidth, cache working set.
3. `03-architecture.md` — the data path (edge → shield → origin) and control plane.
4. `04-deep-dives.md` — the core: anycast vs unicast, cache key economics, tiering/shielding,
   invalidation, the hot-object problem, DDoS absorption.

Cross-references: this case is the general form of the cache hierarchy YouTube builds for
video ([01.youtube/04-deep-dives.md](../01.youtube/04-deep-dives.md)), and the
request-coalescing / hot-key techniques reappear in Twitter's celebrity fanout problem
([02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md)).

All numbers in this case are rough estimates or explicitly labeled public figures; a public
"~X" is a vendor-reported value, not an independent measurement.
