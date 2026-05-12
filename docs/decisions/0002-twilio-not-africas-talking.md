# ADR-0002 — Twilio for WhatsApp + SMS + Voice, not Africa's Talking direct

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-04-13 (POC-2 + POC-4 validation) |
| **Authors** | Founder + POC results |

## Context

OTP delivery is a hard requirement for Story 1.1 (consumer signup). Cameroon's network reality:

- Most users on **MTN** or **Orange**; SMS deliverability across both carriers is uneven on 3G
- **WhatsApp** has near-universal penetration in Douala — more reliable than SMS in practice
- Voice proxy (livreur ↔ client number masking) is needed for Story 4.17

Originally architecture v1/v2 specified **Meta Cloud API** (direct WhatsApp Business API) + **Africa's Talking** for SMS + **Africa's Talking Voice** for the proxy. POC-2 surfaced two problems:

1. Meta Cloud API requires **15-day Business verification** before you can send your first OTP. That blocks the soft launch by half a sprint.
2. Two providers (Meta for WhatsApp, AT for SMS + Voice) = two SDKs, two billing accounts, two operational dashboards, two outage surfaces.

Twilio offers **WhatsApp Business API as a managed BSP**, plus SMS, plus Voice (TwiML Dial bridge), under one SDK and one account.

## Decision

**Use Twilio as the unified provider for WhatsApp OTP + SMS fallback + Voice proxy.**

- WhatsApp via Twilio (primary OTP channel)
- SMS fallback via Twilio (when WhatsApp send fails or recipient hasn't opted into sandbox)
- Voice proxy via Twilio TwiML Dial bridge

Twilio Sandbox is used in dev (72h opt-in via `join <keyword>`); a production WhatsApp Business sender needs Meta approval (~24-72h via Twilio, **much** less than direct Meta).

## Alternatives considered

| Option | Dealbreaker |
|---|---|
| **Meta Cloud API + Africa's Talking** | 15-day verification gate on Meta blocks launch. Two SDKs + two dashboards = double the operational surface for a solo dev. |
| **Africa's Talking for everything (incl. WhatsApp)** | AT didn't have a clearly-listed Cameroon WhatsApp Business offering at the time (their Cameroon coverage page lists SMS + Voice only). Voice yes, WhatsApp no. |
| **Termii (Nigerian provider, multi-channel)** | Untested in Cameroon at the time. Kept as fall-back-of-fall-back. |
| **Self-hosted Signal or Matrix bot** | Operationally insane for a solo dev. WhatsApp's user base is the moat; we don't fight that. |

## Consequences

**Positive**

- Unified SDK (`@twilio/sdk`) — same shape for WhatsApp, SMS, Voice
- One billing account, one Sentry/error route, one set of credentials
- Per-message status webhooks (`POST /api/twilio/status`) work across all channels with the same shape
- Free trial credits cover all of dev + sandbox
- Twilio fronts the Meta Business verification — faster, less ceremony

**Negative**

- **Cost.** Twilio is more expensive than direct Meta Cloud at scale (~$0.005/WhatsApp message vs. Meta's $0.003 marketing rate). Acceptable through M5–M6; revisit at >10k OTPs/day.
- **Vendor lock for OTP.** Migrating to direct Meta later would be doable (`OtpDeliveryService` is the only consumer), but every existing OtpLog row holds a Twilio SID — historical webhook reconciliation only works against the original vendor.
- **Twilio sandbox 72h opt-in** is a development friction for testing on multiple devices.

**Reversal cost** — small for SMS/Voice (swap the provider in `infra/twilio/twilio.service.ts`); medium for WhatsApp because the Meta direct path needs the business verification done in advance.

## Follow-ups

- todo.md B2 — submit Meta approval via Twilio Sender Registration before launch
- todo.md B3 — buy a Twilio Voice number with Cameroon geographic permissions enabled
- Future ADR: Africa's Talking SMS as bulk-SMS optimisation lever at scale (>20k SMS/month tipping point)
