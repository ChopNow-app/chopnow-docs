# Vendor surface

Everything a vendor sees, does, and triggers in ChopNow. The vendor is the cook — the human who receives an order, prepares the food, hands it to the rider. They are the operational bottleneck of every transaction and the experience for them must minimise lost orders, missed acceptance windows, and "I didn't know" friction.

This section documents the technical contract end-to-end:

- **[Onboarding & KYC](onboarding.md)** — three vendor types (informal, semi-formal, restaurant), what each submits, how admin validates, who can sign forms before RCCM.
- **[Self-service surfaces](self-service.md)** — `/vendor` dashboard, `/vendor/menu`, `/vendor/hours`, `/vendor/profile`, `/vendor/commande/[id]`, `/vendor/preparation/[id]`.
- **[Order lifecycle from the vendor POV](order-lifecycle.md)** — status machine, race-safe transitions, payment-gated visibility, the 60s acceptance countdown timing.
- **[Notifications](notifications.md)** — Web Push primary, WhatsApp fallback, service-worker `postMessage` for in-app chime, why we don't use SSE.
- **[Concurrency model](concurrency.md)** — the four updateMany status guards, OTP and refresh in-flight locks, the Campay webhook race guard, item-availability flip telemetry.
- **[Data model](data-model.md)** — `Vendor`, `Item`, `MenuCategory`, `VendorOpeningHours`, `Order` from the vendor's columns.
- **[API reference](api-reference.md)** — every vendor-facing route, request/response shape, error codes.
- **[Operational telemetry](operational.md)** — structured event codes emitted for vendor events; how to filter them in Kibana / Loki / Datadog.

## Mental model

The vendor sits in three states at any moment:

1. **Idle** — open or closed flag flipped, no active orders.
2. **Decision** — order arrived, 60 seconds to accept or refuse.
3. **Preparation** — order accepted, items being marked-prepared, rider en route.

Every backend choice traces back to one of these:

- **Idle → Decision** transitions are the highest-trust signal. The push notification + 60s countdown have to be reliable, audible, and visible across device states (PWA backgrounded, PWA foregrounded, screen off).
- **Decision → Preparation** is where races bite. A vendor tapping accept at the 60s deadline boundary, a Campay webhook retry landing twice, a consumer cancelling mid-tap — all need guarded transitions. See [concurrency](concurrency.md).
- **Preparation → handover** is mechanical. The pickup code is what gates rider handover; the delivery code is what gates consumer handover. Both are 4-digit, generated server-side at order creation, owner-scoped on read.

## Surface diagram

```mermaid
flowchart LR
  subgraph PWA["chopnow-app PWA (Next.js 16)"]
    Vendor[/vendor — dashboard/]
    Menu[/vendor/menu — items, categories, stock/]
    Hours[/vendor/hours — opening hours editor/]
    Profile[/vendor/profile — name, photos, MoMo/]
    Decide[/vendor/commande/[id] — accept/refuse countdown/]
    Prep[/vendor/preparation/[id] — item-by-item checklist/]
  end

  subgraph API["chopnow-api (NestJS)"]
    VC[VendorController]
    OC[OrdersController]
    AC[AuthController]
    NC[NotificationsController]
  end

  subgraph Events["DomainEvents"]
    OCR[order.created]
    OAC[order.accepted]
    OREF[order.refused]
    OREADY[order.ready]
  end

  Vendor -->|polling 10s + SW push| OC
  Menu --> VC
  Hours --> VC
  Profile --> VC
  Decide -->|accept/refuse| OC
  Prep -->|mark item prepared| OC
  Vendor -.subscribe.-> NC
  OC -.emit.-> OCR
  OC -.emit.-> OAC
  OC -.emit.-> OREF
  OC -.emit.-> OREADY
  OCR -.listener.-> Push[(Web Push + WhatsApp fallback)]
  Push --> Vendor
```

## Document versions

- **2026-05-19** — initial version. Reflects platform state post-PR #186 (structured pino logging): payment-gated visibility (#178), countdown timing fix (#179), cash code removal (#177), Campay webhook race guard (#172), vendor-side race guards on accept/refuse/markReady (#169), Web Push primary with WhatsApp fallback (#170, frontend #132), OTP idempotency lock (#171), refresh-token in-flight lock (#175), cancel race guard (#174), item-flip telemetry (#173). The race-condition audit is fully closed except for #176 which turned out to already be implemented.
