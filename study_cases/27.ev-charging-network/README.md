# Case 27 — EV Charging Network

**The system**: a Charge Point Operator (CPO) — the company that owns and operates tens of
thousands of public EV chargers across Europe (think Ionity, Fastned, Allego). Four surfaces
work together: the **charger fleet** itself (embedded Linux boxes on 4G, speaking the
industry-standard OCPP 2.0.1 protocol), a **driver app** (find a charger, plug in, charge,
pay), the **billing pipeline** (meter values → tariff engine → Charge Detail Records →
settlement), and **roaming + availability publishing** (drivers from 30+ partner eMSPs
charge on our network; our occupancy and prices are pushed to their apps and to map POIs).

**Why it's hard**:

- The clients are **flaky embedded devices**, not browsers or phones: ~100K chargers on
  power-cycled 4G modems, NAT timeouts, firmware from five hardware generations, and a
  reconnect storm the moment a cell carrier blips. The server has no control over the
  client — OCPP fixes the protocol, we fix the rest.
- **Money correctness under at-least-once messaging**: every session is a bill — a driver's
  card gets charged, a roaming partner must be paid, VAT per country applies. The device
  layer guarantees at-least-once delivery of queued records after outages; the billing side
  must turn that into exactly-once *effect* or double-charge real people.
- The system **must degrade toward autonomy**: when our backend is down, drivers still
  deserve a charge. Chargers authorize offline, cache tokens, queue sessions, and replay on
  reconnect — so most of the hard correctness problems happen *at the moment of resync*.
- **Availability is the product**: a stale "this charger is free" that turns out occupied
  costs a driver 10 minutes of detour. Occupancy flows from charger → us → 30 partner apps
  → map POIs, each hop adding latency; we own the first 5 minutes of that SLA.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**:

- A gateway fleet for 100K persistent, unreliable client connections (same physics as the
  Discord gateway in [05.discord](../05.discord/README.md) and WhatsApp's connection layer
  in [03.whatsapp](../03.whatsapp/README.md) — but the clients are 4G toasters, not phones).
- At-least-once ingest → exactly-once billing effect, keyed idempotency (the outbox/ledger
  reasoning of [04.deployment-system](../04.deployment-system/README.md) and the purchase
  ledger in [06.apple-app-store](../06.apple-app-store/README.md)).
- Fanout-on-write to external partners with per-partner backpressure — the Twitter fanout
  pattern ([02.twitter](../02.twitter/README.md)) applied to *companies* instead of users.
- Deploying firmware to a fleet you can't physically touch — fleet-side mirror of
  [04.deployment-system](../04.deployment-system/README.md), with the worst possible
  failure mode: a charger that bricks a paying customer's visit.
- Stale-data SLAs across organizations (the POI/availability loop of
  [26.google-maps](../26.google-maps/README.md) seen from the data *producer* side).

**Try it first**: before reading 04, answer: (1) the cell network drops for 10 minutes; when
it comes back, 50K chargers reconnect at once — what does your gateway need to survive that,
and what queued up on the devices, (2) a charger approved a session offline and queues a
transaction; how does the backend dedupe replayed records so nobody is billed twice, (3) who
decides how much power each car at a 12-stall site with a 400 kW grid connection gets — the
cloud or a box in the cabinet? Why can't the cloud be primary?

**Primary sources**: OCPP 2.0.1 is publicly specified by the Open Charge Alliance; OCPI by
the EVRoaming Foundation; Plug & Charge by ISO 15118; payment card and/or contactless
acceptance is a public requirement on newly installed public charge points ≥50 kW under the
EU Alternative Fuels Infrastructure Regulation (AFIR). The [EU Alternative Fuels
Observatory](https://alternative-fuels-observatory.ec.europa.eu)
publicly reports 1,061,890 public charge points across the EU27 at end-2025 (roughly
~1.2M by late 2026). Our 100K-charger fleet is ~10% of that — and **deliberately
futuristic**: Europe's largest fast-charging CPOs (IONITY ~4.4K charge points, Fastned
~2.7K charge spots) are roughly 20–40× smaller, so the premise forces the hard
problems at a scale no operator has reached; treat it as an estimate-based stress case,
not a description of any existing company — assumptions in [02](02-scale-estimates.md).
