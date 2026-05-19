# Operational telemetry

Every vendor-relevant warning and error emitted by the API is a structured pino log with an `event` field. Once Kibana / Loki / Datadog is wired, these become first-class dashboard inputs — no grok parsing, no regex over `msg`.

This page is the inventory.

## How the logs look

`nestjs-pino` is configured in `app.module.ts`. Every `this.logger.warn({...}, '...')` call produces:

```json
{
  "level": "warn",
  "time": 1747654680000,
  "pid": 1234,
  "hostname": "chopnow-api-prod-1",
  "context": "OrdersService",
  "event": "order_cancelled_while_paid",
  "orderId": "abc-123-...",
  "paymentMethod": "MTN_MOMO",
  "msg": "Order cancelled while PAID — refund needed (Story 3.8 pending wiring)"
}
```

The `event` field is the stable filter handle. Dashboards query `event:order_cancelled_while_paid` directly; alerting filters on `level:warn AND event:dispatch_expired_no_rider`.

## Vendor-related events

### Order lifecycle

| `event` | Level | Fires from | Meaning | Suggested dashboard |
|---|---|---|---|---|
| `order_created_notification_failed` | warn | `OrderNotificationsService.onOrderCreated` | Both Web Push AND WhatsApp threw on order arrival | Per-vendor failure rate; spike = Twilio sandbox window lapsed |
| `order_refusal_whatsapp_failed` | warn | `OrderNotificationsService.onOrderRefused` | Couldn't WhatsApp the consumer after refusal | Spike = Twilio outage |
| `order_cancelled_while_paid` | warn | `OrdersService.cancelOrder` | Consumer cancelled a PAID order — refund needed | Manual refund worklist (until Campay refund API is wired) |
| `order_refused_while_paid` | warn | `OrdersService.refuseOrder` | Vendor refused after the order was PAID — refund needed | Same |
| `order_auto_expired_while_paid` | warn | `OrdersExpiryService.sweepExpired` | Cron auto-refused a PAID order (vendor never responded) — refund needed | Same; also surfaces vendor reliability issue |
| `order_auto_refused` | info | same | Cron auto-refused an unpaid order — normal flow | Volume / vendor metric |
| `order_auto_refuse_failed` | error | same | DB error during the cron flip | Per-row alert |
| `order_low_rating` | warn | `OrdersService.rateOrder` | Consumer gave vendor or rider ≤ 2/5 | Admin review queue |
| `order_item_flip_race` | warn | `OrdersService.checkPostCreateItemAvailability` | Vendor flipped an item to unavailable mid-order creation (telemetry for #173) | Rate gauge — non-zero means we revisit the race fix |
| `order_item_recheck_failed` | warn | same | The telemetry query itself errored (DB blip) | Low-priority |

### Payments / Campay

| `event` | Level | Meaning |
|---|---|---|
| `campay_webhook_missing_reference` | warn | Campay webhook arrived without `reference` field |
| `campay_webhook_duplicate_ignored` | warn | Webhook hit the 5s Redis lock — second delivery dropped |
| `campay_webhook_unknown_reference` | warn | Webhook for a `reference` that doesn't match any Order |
| `campay_collect_failed` | error | Outbound Campay `/collect/` call failed (4xx / 5xx) |
| `campay_collect_missing_reference` | error | Campay responded without a `reference` — initiation broke |

### Dispatch (matters for vendor's prep timeline)

| `event` | Level | Meaning |
|---|---|---|
| `dispatch_no_online_rider` | warn | No rider in range — will retry every 30s |
| `dispatch_already_assigned` | warn | Concurrent dispatch / duplicate event tried to claim an already-assigned order |
| `dispatch_retry_scheduled` | info | One dispatch attempt failed; next scheduled |
| `dispatch_expired_no_rider` | warn | Gave up after 10 retries → order EXPIRED, refund needed (if paid) |
| `dispatch_rider_assigned` | info | Rider successfully claimed an order |

### Notifications

| `event` | Level | Meaning |
|---|---|---|
| `vapid_unconfigured` | warn | Boots without VAPID env vars — Web Push disabled, every send falls back to WhatsApp |
| `web_push_transient_failure` | warn | Push service returned 429 / 5xx / network blip — subscription kept |

### Auth (vendor exercises these during sign-in)

| `event` | Level | Meaning |
|---|---|---|
| `otp_inflight_lock_held` | info | Sub-second double-tap on `request-otp` — no second Twilio send |
| `otp_delivery_failed` | error | Twilio threw on OTP send |
| `otp_dev_stub` | warn | Dev mode — OTP printed to logs |
| `otp_whatsapp_failed_fallback_to_sms` | warn | WhatsApp OTP delivery failed, falling back to SMS |
| `refresh_reuse_detected` | warn | A genuinely-replayed refresh token — token family revoked. **Security signal**, alert on |
| `jwt_user_revoked` | warn | Admin suspended a user account |
| `jwt_user_reactivated` | info | Admin unsuspended |

### Admin actions on vendors

| `event` | Level | Meaning |
|---|---|---|
| `admin_account_locked` | warn | Admin login lockout after 5 failed attempts |
| `admin_notification_failed` | warn | Admin's approval/reject/suspend WhatsApp didn't deliver — vendor wasn't notified |

### Other vendor-touched events

| `event` | Level | Meaning |
|---|---|---|
| `vendor_submission_whatsapp_failed` | warn | "Dossier reçu" WhatsApp didn't deliver after submission |
| `rider_submission_whatsapp_failed` | warn | Same for rider onboarding |
| `voice_proxy_call_started` | info | Rider↔consumer bridged voice call started (Twilio TwiML) |
| `mail_dev_stub` | warn | Dev mode — email printed to logs |
| `twilio_status_callback_malformed` | warn | Status webhook missing `MessageSid` / `MessageStatus` |
| `twilio_status_unknown_sid` | debug | Status webhook for a `MessageSid` we don't recognise |
| `redis_connected` | info | Redis client connected at boot |
| `unhandled_5xx` | error | Catch-all from `AllExceptionsFilter` — uncaught exception returned a 500 |

## Suggested first dashboards

When stand-up Kibana / Loki, three dashboards are highest-value for vendor ops:

### 1. Vendor SLA

- **Per-vendor 60s acceptance compliance** — `event:order_auto_refused` grouped by `vendorId`. Vendors with > 5% auto-refusal in a week need a check-in.
- **Order arrival → vendor notification latency** — derived from the gap between `order_paid` event time and `event:order_created_notification_failed` (failure) or absence (success).
- **Vendor-side push success rate** — ratio of `event:web_push_transient_failure` to total pushes.

### 2. Refund worklist

Until the Campay refund API is wired (Story 3.8), every "refund needed" event is a manual ops task:

- Query: `(event:order_cancelled_while_paid OR event:order_refused_while_paid OR event:order_auto_expired_while_paid OR event:dispatch_expired_no_rider) AND wasPaid:true`
- Render as a table with `orderId`, `paymentMethod`, `time`, and a "refund processed" checkbox state held elsewhere (Notion / Linear).

### 3. Race-condition observability

The race fixes are belt-and-suspenders. The events tell us if they ever fire:

- `event:campay_webhook_duplicate_ignored` — confirms the Redis lock + status guard caught a real duplicate.
- `event:dispatch_already_assigned` — confirms the dispatch race guard fired.
- `event:order_state_changed` (implicit: there's no log when an `updateMany` returns 0 — only the HTTP 409 — so look for HTTP 409 patterns).
- `event:order_item_flip_race` — pilot telemetry for #173. Should be ~zero at pilot volume; a non-zero rate is the signal to escalate.

## Adding new events

When adding new structured logs, the naming convention is:

> `<domain>_<thing>_<state>` — snake_case, present-tense verb where applicable.

Existing prefixes: `order_*`, `dispatch_*`, `campay_*`, `otp_*`, `refresh_*`, `jwt_*`, `admin_*`, `vendor_*`, `rider_*`, `twilio_*`, `voice_proxy_*`, `web_push_*`. Try to reuse a prefix rather than inventing a new domain.

Reserved field names within the structured payload:

| Field | Type | Use |
|---|---|---|
| `event` | string | Required. The event code. |
| `orderId` | string (uuid) | When applicable. |
| `vendorId` | string (uuid) | When applicable. |
| `userId` | string (uuid) | When applicable. |
| `error` | string | The `(err as Error).message` for failure events. |
| `statusCode` | number | HTTP/external status. |

Avoid sticking a phone number, raw token, or password hash in the structured fields. The audit baseline is: no secret should be reconstructible from log output.
