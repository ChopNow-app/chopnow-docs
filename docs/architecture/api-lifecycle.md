# API lifecycle

How ChopNow's HTTP API is versioned, supported, deprecated, and removed.
Written before any external integration to avoid the "what's the
contract?" conversation arriving in panic.

!!! info "Status — 2026-05-24"
    **v1 is the only version live.** Webhooks are version-neutral. No
    deprecations active.

## Versioning scheme

URI-based, single integer:

```
https://api-staging.tchopnow.app/api/v1/orders
https://api-staging.tchopnow.app/api/v1/auth/request-otp
```

Wired in `src/main.ts` via NestJS `VersioningType.URI` with
`defaultVersion: '1'`. Every controller method automatically picks up
`/api/v1/...` unless it opts out (see Webhooks below).

A future v2 lands per-controller via `@Version('2')` — v1 and v2 can
co-exist on the same controller. No big-bang migration.

## What gets versioned vs not

| Surface | Versioned? | Why |
|---|---|---|
| Consumer + livreur + vendeur + admin REST endpoints | ✅ `/api/v1/*` | App clients are ours; we control the rollout window |
| Webhooks from Campay, Twilio, voice-proxy (Twilio Voice) | ❌ unversioned (`@Version(VERSION_NEUTRAL)`) | External parties register a fixed URL once; we cannot ask them to update it |
| `/health`, `/ready`, `/metrics`, `/openapi.json` | ❌ no `/api` prefix at all | Infra concerns — Prometheus / probes / spec consumers — not part of the consumer API surface |

The "webhooks never version" rule is load-bearing. Adding `@Version('2')`
to a webhook controller breaks every registered URL silently. If a
breaking change to a webhook is unavoidable, the right move is a NEW
controller at a different path (`/webhooks/campay-v2/...`) and ask the
external party to migrate the registration.

## Support window

| Version | Status | Support starts | Support ends |
|---|---|---|---|
| v1 | 🟢 GA (pilot) | 2026 pilot launch | **Pilot launch + 12 months** (sliding — moves with pilot date) |
| v2 | not yet | — | — |

The 12-month window lets pilot vendors and any early API consumers
plan migrations without churn pressure. It's a floor, not a ceiling —
v1 stays supported beyond that if no v2 has shipped.

## When to bump v1 → v2

A version bump is only justified by a **breaking change** to a
contract. Pick the lighter alternative when possible:

| Change | Bump? |
|---|---|
| Add a new field to a response | No — additive, clients ignore unknown fields |
| Add a new endpoint | No — orthogonal to existing surface |
| Add a new optional query/body param | No — old callers keep working |
| Tighten a validation rule (e.g. `min(1)` → `min(5)`) | Yes if existing valid traffic would start 400'ing; no otherwise |
| Rename a field | Yes — but consider keeping the old field as deprecated for a release first |
| Change a field's type (e.g. `string` → `string[]`) | Yes |
| Remove an endpoint | Yes (after deprecation — see below) |
| Change auth model on a route | Yes |

## Deprecating a v1 endpoint

When v2 ships a replacement for a v1 endpoint, mark the v1 version
deprecated before removing it:

1. Add `@ApiOperation({ deprecated: true, ... })` on the v1 handler
2. Add a response header on the v1 path:
   `Deprecation: true` + `Sunset: <RFC 3339 date>` (per IETF draft-ietf-httpapi-deprecation-header)
3. Log a `warn` with `event: 'deprecated_endpoint_called'` + `userAgent` + `userId` so we can see who's still hitting it
4. Keep deprecated for **≥ 6 months** before removal
5. Email known integration partners with the sunset date

## Removing a deprecated endpoint

After the 6-month deprecation window:

1. Confirm `deprecated_endpoint_called` Loki count is at zero or
   negligible for 30 days
2. Open a PR that deletes the v1 handler
3. The route returns 404; client libraries should already have
   migrated to v2

If usage is still non-trivial at the 6-month mark, extend the window
rather than break a live integration.

## Where to read more

- Live spec: <https://api-staging.tchopnow.app/openapi.json> (production URL once it lands)
- Swagger UI: <https://api-staging.tchopnow.app/api/docs>
- Committed snapshot in the repo: `chopnow-api/openapi.json`
- Versioning code in `chopnow-api`: `src/main.ts` (NestFactory + `enableVersioning`)
- Webhook opt-outs: `src/modules/payments/campay-webhook.controller.ts`, `src/infra/twilio/twilio-webhook.controller.ts`, `src/modules/voice-proxy/voice-proxy.controller.ts`

## Related

- [Architecture overview](overview.md) — high-level system shape
- [Observability stack](observability.md) — how `deprecated_endpoint_called` logs land in Loki
