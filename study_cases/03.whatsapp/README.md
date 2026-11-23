# Case 03 — WhatsApp

**The system**: private 1:1 and group messaging. Billions of users, a persistent connection
from each, messages that MUST arrive — once, in order, mostly instantly — even across days
of offline time on the receiver's side.

**Why it's hard**:

- Unlike YouTube/Twitter (pull models), messaging is push: you must hold tens of millions
  of **live TCP/WebSocket connections per data center**, and pushes are latency-sensitive.
- Delivery guarantees across flaky mobile networks require an ACK/retry protocol that is
  exactly-once *as the user experiences it* (deduplicated), even when the transport is
  at-least-once underneath.
- Offline users, multi-day gaps, ordering across device changes, group fanout — all interact
  with every message that is ever sent.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: connection-based systems (vs the request/response in
[Twitter](../02.twitter/README.md) and [YouTube](../01.youtube/README.md)), delivery
guarantees/ACK protocol, offline message queues, group fanout at the other end of the
spectrum from Twitter's celebrity fanout, E2E encryption.

**Try it first**: before reading 03, answer: (1) how does a message go from sender to
receiver when the receiver is offline for 3 days, (2) what does "delivered at least once"
mean at the client side and how do you dedupe, (3) why is each user holding a connection
necessary at all — why not poll?

**Primary sources** (all self-reported, 2012–2014 era unless noted — see the caveat in
[02](02-scale-estimates.md)): WhatsApp blog ["1 million is so 2011"](https://blog.whatsapp.com/1-million-is-so-2011)
(2.28M sockets on one box); Rick Reed, Erlang Factory SF 2012 ([slides](http://www.erlang-factory.com/upload/presentations/558/efsf2012-whatsapp-scaling.pdf))
and 2014 [slides](https://www.erlang-factory.com/static/upload/media/1394350183453526efsf2014whatsappscaling.pdf)/[InfoQ talk](https://www.infoq.com/presentations/whatsapp-scalability/)
(fleet + scale numbers); [Signal X3DH spec](https://signal.org/docs/specifications/x3dh/)
(key agreement); Albrecht–Dowling–Jones, [EUROCRYPT 2025](https://eprint.iacr.org/2025/794)
(formal analysis of the reverse-engineered multi-device subset); Discord blogs on its
[message store](https://discord.com/blog/how-discord-stores-trillions-of-messages) and
[Elixir session layer](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)
(documented analogues for parts WhatsApp has never published).
