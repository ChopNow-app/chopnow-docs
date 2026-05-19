# Concurrency model

The vendor side has been audited end-to-end for race conditions. Every state-mutating call uses one of three patterns:

1. **`updateMany` + status guard** — for `Order.status` transitions where two actors could write at once.
2. **Redis `SET NX` in-flight lock** — for endpoints where two requests carrying the same identity must be serialised.
3. **Conditional emission** — events fire only when the underlying DB write actually changed a row, not just because the request succeeded.

This page documents every guard. The pattern was first introduced for PR #169 (vendor accept/refuse/markReady) and has been applied symmetrically wherever a race could land.

## Pattern 1 — status-guarded `updateMany`

```ts
// Pre-check stays for the common "stale URL" case → clean error message.
const order = await this.requireVendorOrder(orderId, userId);
if (!VENDOR_CAN_DECIDE.has(order.status)) {
  throw new ConflictException({ code: 'order_not_pending', ... });
}

// Conditional update — the WHERE clause is the locking primitive.
const res = await this.prisma.order.updateMany({
  where: { id: order.id, status: { in: [PENDING, CONFIRMED] } },
  data: { status: OrderStatus.ACCEPTED, acceptedAt: now },
});

// count === 0 means another writer flipped the status between our read
// and our write. Critical: throw + DO NOT emit downstream events.
if (res.count === 0) {
  throw new ConflictException({ code: 'order_state_changed', ... });
}

this.events.emit(DomainEvents.ORDER_ACCEPTED, { ... });
```

Why the pre-check stays even with the guard: it gives the *common* failure (vendor double-taps, revisits a stale URL) a clean error message (`order_not_pending`). The `count===0` branch fires only for the rare sub-100ms cron-vs-human race.

### Where applied

| Method | File | Guard | `count===0` error code |
|---|---|---|---|
| `OrdersService.acceptOrder` | `src/modules/orders/orders.service.ts` | `status IN [PENDING, CONFIRMED]` | `order_state_changed` |
| `OrdersService.refuseOrder` | same | `status IN [PENDING, CONFIRMED]` | `order_state_changed` |
| `OrdersService.markOrderReady` | same | `status IN [ACCEPTED, IN_PREP]` | `order_state_changed` |
| `OrdersService.cancelOrder` (consumer) | same | `status IN [PENDING, CONFIRMED]` | `order_state_changed` |
| `OrdersService.onPaymentSucceeded` | same | `paymentStatus IN [PENDING, PROCESSING]` | (no error — silent no-op + no event) |
| `OrdersExpiryService.sweepExpired` | `src/modules/orders/orders-expiry.service.ts` | `status IN [PENDING, CONFIRMED]` | (silent skip + no event) |
| `DispatchService.dispatchOrder` | `src/modules/dispatch/dispatch.service.ts` | `riderId: null` | (silent skip + no event, plus structured log) |

In total: **seven places** where a status-guarded `updateMany` replaces a bare `update`.

### Race scenarios closed

| Issue | Scenario | Fix landed in |
|---|---|---|
| #169 | Vendor decision vs auto-refuse cron | PR #169 |
| #172 | Campay webhook double-fire | PR #182 (folded in with #178) |
| #174 | Consumer cancel vs vendor accept | PR #183 |
| #176 | Two riders accepting same order | Already present (dispatch was server-side) |

## Pattern 2 — Redis SET NX in-flight lock

For endpoints where the unique constraint is "the same caller doing the same thing within milliseconds" — typically auth flows where two devices/tabs both hit refresh.

### `requestOtp` — sub-second double-tap (PR #171)

```ts
const acquired = await this.redis.setNX(
  `otp:inflight:${phone}`,
  String(Date.now()),
  30,  // OTP_INFLIGHT_LOCK_SECONDS
);
if (!acquired) {
  // Original code is already on its way + still valid for ~5 min.
  return { ok: true, expiresInSeconds: OTP_TTL_MINUTES * 60 };
}
```

Returning the success-shape (instead of 429) is deliberate: the user just double-tapped the "Send code" button. The OTP is already on the way.

Lock released:

- On the **failed delivery** path (`event:otp_delivery_failed`) → legit retry within seconds is unblocked.
- On **successful verify** (`AuthService.verifyOtp`) → re-signing in within 30s works.

The lock naturally expires after 30s. The IP-level + per-phone rate limiters (5/15min each via `@Throttle` and `PhoneRateLimitGuard`) sit above this and bound overall abuse.

### `refresh` — concurrent rotation (PR #175)

Two tabs both hitting `/auth/refresh` with the same refresh token used to race on the `revokedAt` write — the first transaction revoked the row, the second's `findMany` saw `revokedAt != null` and **tripped reuse detection**, revoking the entire token family and logging the user out everywhere.

```ts
const tokenFp = createHash('sha256').update(rawRefreshToken).digest('hex').slice(0, 32);
const acquired = await this.redis.setNX(`refresh:inflight:${tokenFp}`, '1', 5);
if (!acquired) {
  throw new UnauthorizedException({
    code: 'refresh_in_flight',
    message: 'Session refresh in progress, please retry.',
  });
}
```

Key design choices:

- **Hash-only Redis key** — raw refresh token never appears in Redis logs or keyspace dumps.
- **Per-token scope** — a user refreshing on two different devices in parallel each get their own slot.
- **`finally` releases on every exit** — success, `refresh_invalid`, or genuine `refresh_reuse_detected`. The 5s TTL is the crash-safety net.

A genuine token replay (attacker captures token + replays it >5s after legitimate rotation) still trips reuse detection legitimately. The lock only suppresses the **race** failure mode.

### Webhook idempotency

`PaymentsService.handleWebhook` (Campay) acquires a 5s `lock:payment:${reference}` Redis key before processing. Combined with the `updateMany` status guard inside `onPaymentSucceeded`, double-webhooks are belt-and-suspenders idempotent. Logs `event:campay_webhook_duplicate_ignored` when the second hits the held lock.

## Pattern 3 — conditional event emission

If a write didn't happen (status guard didn't match, lock not acquired, idempotent no-op), **no event fires**. This rule is enforced everywhere — `events.emit` calls live inside the `count > 0` branch, never before.

Why this matters concretely: if `onPaymentSucceeded` fired `ORDER_CREATED` regardless of whether the row actually flipped, a duplicate Campay webhook would emit two `ORDER_CREATED` events → two vendor pushes → vendor sees a duplicate "🍲 Nouvelle commande TC-A23F4" notification. The status guard prevents this.

## Telemetry-only race observation

Issue #173 (vendor flips item to `isAvailable: false` mid-order) was deliberately left without a `SERIALIZABLE` fix. At pilot volume (~50 orders/day, ~1 every 30 min) the race window is statistically negligible — the cost of bumping the create transaction to `SERIALIZABLE` isolation isn't justified speculatively.

Instead: telemetry. After every `order.create` commits, `checkPostCreateItemAvailability` re-reads the items. If any are now `isAvailable: false` (or `isInStock: false`) with `updatedAt` in the last 10s, the structured log `event:order_item_flip_race` fires with the order ID and the flipped items. Production data tells us whether to escalate.

```ts
this.logger.warn(
  {
    event: 'order_item_flip_race',
    orderId,
    flippedCount: flippedRecently.length,
    flippedItems: flippedRecently.map((i) => ({
      id: i.id, name: i.name, isAvailable: i.isAvailable, isInStock: i.isInStock,
    })),
  },
  'Order created during item-availability flip race — vendor may refuse on accept screen',
);
```

## What's deliberately NOT guarded

Some operations are inherently last-write-wins and intentionally so:

- **`updateVendorAvailability` (vendor toggles open/closed)** — a vendor rapidly flipping their own status is fine; last write wins.
- **`updateVendorProfile` (name / description / momoPhone)** — same.
- **Profile photo upload** — R2 upload + Vendor row update. Two parallel uploads from the same vendor produce two R2 objects; the later DB write wins on which one is referenced. Orphan in R2 is acceptable (lifecycle cleanup).
- **Rider heartbeat** — last-write-wins on `lastLocation`/`lastSeenAt`.

These are not race-safety concerns because the actor is single-threaded (one vendor) and the outcome is monotonic in the user's intent.

## Verification

Every guard has unit-test coverage that exercises the `count===0` / lock-held path explicitly. Search the spec files for `'order_state_changed'` or `'refresh_in_flight'` or `mockResolvedValueOnce({ count: 0 })` to find them.

Production: the event codes (`event:order_state_changed_lost_race` is implicit in the absence of follow-up events; `event:campay_webhook_duplicate_ignored` is direct) will show up in dashboards once Kibana/Loki is wired. Any non-zero rate is a signal to revisit.
