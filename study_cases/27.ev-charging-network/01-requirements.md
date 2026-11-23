# 01 — Requirements

## Who the actors are

- **CPO (us)**: operates the chargers and the backend. Owns hardware, grid contracts, and
  the operating margin per kWh.
- **Driver (direct)**: uses our app, registers a payment method, charges, pays.
- **eMSP (roaming partner)**: a mobility service provider (Tesla, Enel X, Chargemap,
  Plugsurfing…) whose customer charges on *our* charger using *their* app/token. Roughly
  30 partners. They pay us periodically; drivers never see our app.
- **Operations team**: monitors the fleet, unlocks cables, pushes firmware, dispatches
  field service.
- **Grid / site owner**: caps how much power a site may draw from the building connection.

## Functional requirements

1. **Charging session lifecycle**: authorize → start → meter → stop → bill. Auth by app
   QR/code, RFID card, contactless EMV card (AFIR — Regulation (EU) 2023/1804 Art. 5(1) —
   requires ad hoc electronic payment at all publicly accessible charge points deployed
   since April 2024: at ≥50 kW at least one of a card reader or a contactless device is
   mandatory — the QR/internet option is only permitted below 50 kW; from 1 Jan 2027 the
   same requirement extends retroactively to pre-2024 ≥50 kW points on the TEN-T network
   and safe-and-secure parking areas), and Plug & Charge (ISO 15118, certificate-based, in-car).
2. **Charger backend communication**: OCPP 2.0.1 over WebSocket — transaction events,
   meter values, status notifications, remote commands (start/stop/unlock/reset), firmware
   push, diagnostics, reservation support. (Legacy fleet lines still speak OCPP 1.6
   JSON — see the naming note in [03](03-architecture.md).)
3. **Driver app**: find chargers on a map with live availability and prices, start/stop a
   session on any stall, show live kWh/power/cost during a session, invoice history,
   receipts with country-correct VAT.
4. **Billing**: tariff engine (per-kWh, per-minute, per-session fee, time-of-day rates,
   idle fees after charging ends), Charge Detail Record (CDR) per session, monthly
   invoicing, ledger of every money movement.
5. **Roaming both ways (OCPI 2.2.1)**:
   - *Outbound roam*: eMSP customers charge on our network; we authenticate their tokens
     via the partner and send them CDRs. CDRs are immutable once sent — errors are
     corrected via Credit CDRs, and CDR ids are unique per (country_code, party_id), not
     globally.
   - *Inbound*: roaming CDRs from other CPOs arrive for our drivers; we validate and
     pass through to our own billing.
6. **Availability & price publishing**: real-time occupancy and current tariffs pushed to
   all eMSP partners, our app, and POI feeds (our charger data reaches Google Maps and
   similar POI layers via partner aggregator platforms).
7. **Fleet operations**: health monitoring, remote diagnostics/log fetch, connector unlock,
   remote reset, charger configuration changes, staged firmware rollout across hardware
   generations.
8. **Smart & load-managed charging**: per-site power allocation under a grid connection
   limit, charge scheduling (price-solar signals) where the site controller supports it.

## Scope decisions

- We are a **CPO, not an eMSP** (in product terms — on the wire we hold both OCPI roles):
  we do not build the partner-side apps; the OCPI contract
  fixes what they see from us and vice versa.
- Home/private chargers are out of scope: selling consumer hardware is a different business
  (resale margin vs selling metered kWh) and a different problem class (one charger behind a
  user's router vs 100K on flaky 4G, with money and legal duty only in the public case).
- Battery/grid-scale storage and energy trading are out of scope — grid interaction is
  limited to the site-level load limit.
- Hardware fleet management (CPMS: charge point management system) is in scope; no
  off-the-shelf platform gets adopted as-is, we deliberately own the CPMS: at this fleet
  size the hardware heterogeneity makes the CPMS itself the product, not a commodity we can
  outsource.

## Non-functional requirements

| Property | Target | Why |
|---|---|---|
| Backend availability | ≥ 99.95% | Roaming contracts carry availability SLAs with penalties |
| **Charger autonomy** | Sessions succeed with the backend fully down for locally authorizable tokens (our own tokens via the on-device allowlist, contactless EMV); unfamiliar roaming tokens are refused offline by decision (deep dive 2) — that covers the direct-driver majority of revenue per session (roaming drivers still get accepted when the backend is up) | Charging is not allowed to fail because our control plane did; offline auth + queued billing is a *requirement*, not a fallback |
| Authorization latency | p95 < 2 s for cached tokens; cold roaming tokens ride the partner API and may wait or be refused (deep dive 5) | Drivers stand at the connector; >5–10 s reads as broken and generates support tickets |
| Availability staleness (our hop) | ≤ 5 min for occupancy, ≤ 15 min for prices | Contractual with major eMSPs; stale data erodes the partner's user trust in *their* app |
| Money correctness | Zero double-billing, zero lost sessions; every automatically billed CDR reproducible from raw meter data (estimate-flagged CDRs go to manual reconciliation, deep dive 2) | Regulatory exposure and direct driver/PSP disputes |
| Metering legality | Charging energy measured by a **MID-certified** meter (EU Measuring Instruments Directive) | kWh is the thing we sell; it must be a calibrated register read, not a computed estimate |
| Ingest durability | No OCPP message lost, even across gateway crashes | A dropped CDR is unpaid revenue; a dropped status event is stale availability |
| Firmware rollout | Non-metering firmware: full fleet ≤ 2 weeks, staged, health-gated, automatic halt | Releases touching the MID chain follow re-verification timelines, a regulation-shaped path (deep dive 6) |
| Data retention | Raw meter stream ≥ 18 months, CDRs ≥ 10 years (invoice law varies by country) | Disputes arrive months later; tax law is not negotiable |
| Privacy | GDPR; location history of drivers is personal data | EU-wide regulation; roaming partners share minimal identity data by design |

## The one requirement that shapes everything

Charger autonomy. A phone app can refuse to work when its backend is down; a €100K DC
charger with a driver standing in the rain cannot. Every subsystem in
[03](03-architecture.md) is designed so the *edge* keeps working — authorization caching,
local transaction lifecycle, site-local load control — and the backend is a reconciler of
events the devices already produced, not a required middleware in the driver's critical
path. The scope of "keeps working" is deliberate and bounded (deep dive 2): our own tokens
and contactless EMV charge offline; unfamiliar roaming tokens are refused offline rather
than opened to an unbounded fraud window. Deep dives 2 and 4 are entirely about the
consequences.
