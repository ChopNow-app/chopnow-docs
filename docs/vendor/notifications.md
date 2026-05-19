# Vendor notifications

Vendors must learn about a new order within a few seconds, whether the PWA is open, backgrounded, or the phone is locked. The chosen channel is **Web Push (VAPID) primary, WhatsApp fallback** — no SSE, no WebSocket, no Firebase.

## Why this stack

| Option | Why we picked / dropped |
|---|---|
| **Web Push (VAPID)** | ✅ Native PWA notification, works locked-screen, deep-link on tap, deliverable while the app is closed. POC-3 (2026-04-13) proved iPhone iOS 18.7 + Android 10 hold the push for 10 min over an unstable Douala 3G link. |
| **Twilio WhatsApp** | ✅ Fallback for vendors who haven't installed the PWA or denied permission. Every Cameroonian vendor lives in WhatsApp anyway. |
| **SSE (Server-Sent Events)** | ❌ Considered for in-app live updates. Dropped because the service worker can `postMessage` to focused clients on every push event — one channel covers both background and foreground. SSE would have added a long-lived connection layer + Redis pubsub for multi-instance scale-out + parallel auth, for a marginal latency win over push's 1-2s. |
| **Firebase FCM** | ❌ Architecture-level NO. Adds a Google dependency on a PWA that's deliberately Google-free; FCM web push is built on the same VAPID protocol anyway, so direct VAPID is one less moving part. |

## Push-first cascade

`OrderNotificationsService.onOrderCreated` listens for `DomainEvents.ORDER_CREATED` (which now fires from `onPaymentSucceeded`, not order creation — see [order-lifecycle](order-lifecycle.md)):

```ts
@OnEvent(DomainEvents.ORDER_CREATED)
async onOrderCreated(payload) {
  const order = await this.prisma.order.findUnique({ ... });
  const pushPayload = {
    title: `🍲 Nouvelle commande ${order.code}`,
    body: `${itemCount} plat(s) · ${total} · ${paymentLabel}`,
    data: {
      kind: 'ORDER_CREATED',
      orderId: payload.orderId,
      deepLink: `/vendor/commande/${payload.orderId}`,
    },
  };

  // 1. Web Push first
  const pushResult = await this.webPush.sendToUser(order.vendor.userId, pushPayload);
  if (pushResult.sent > 0) return;  // vendor got the native notification

  // 2. WhatsApp fallback — only when push reached zero subscriptions
  if (!order.vendor.whatsappPhone) return;
  await this.twilio.sendWhatsApp(order.vendor.whatsappPhone, whatsappBody);
}
```

The two channels are **mutually exclusive** — a vendor with active push subscriptions never gets a duplicate WhatsApp. The fallback fires only when `sendToUser` returns `{sent: 0}` (no subscriptions OR every subscription's endpoint returned a `4xx`).

Labels distinguish providers: `Payé via MTN MoMo` vs `Payé via Orange Money`. The legacy `Cash à la livraison` label was removed in PR #177 — there is no COD product.

## Push subscription lifecycle

### Client side (chopnow-app)

The hook at `features/vendor/hooks/usePushSubscription.ts` walks one of these states on mount:

| State | Cause | UI |
|---|---|---|
| `idle` | First render, detection still pending | banner not rendered |
| `unsupported` | No `serviceWorker`/`Notification`/`PushManager` in the runtime (older iOS, WebView, etc.) | banner not rendered |
| `denied` | Browser-level permission already denied | banner not rendered (re-prompt budget conserved) |
| `prompt` | Permission default, no existing subscription | red banner "Active les notifications" |
| `subscribing` | User tapped Activer; permission prompt + POST in flight | spinner |
| `subscribed` | Subscription registered with backend | banner hidden |
| `error` | VAPID misconfigured server-side, or backend POST failed | error message in banner |

On `request()`:

1. `Notification.requestPermission()` → if grants, continue; if denies, flip to `denied`.
2. `registerServiceWorker()` (idempotent — `/sw.js` is registered at app shell mount via `<RegisterServiceWorker />`).
3. `pushManager.subscribe({ applicationServerKey: VAPID_PUBLIC_KEY })`.
4. POST to `/api/notifications/push/subscribe` with `{endpoint, keys: {p256dh, auth}, deviceFingerprint}`.

The `deviceFingerprint` is a `crypto.randomUUID()` minted on first call and persisted in `localStorage` under `chopnow.deviceFingerprint`. Not a real fingerprint — just a stable per-device dedupe key. A vendor logged in on two phones gets two rows; re-granting permission on the same device updates the row in place.

### Server side (chopnow-api)

`NotificationsModule` (which was empty scaffolding pre-PR #170) now hosts:

```
POST   /api/notifications/push/subscribe       PushSubscriptionsController
DELETE /api/notifications/push/subscribe
```

The `PushSubscription` Prisma model has been in the schema since Story 1.11 — `(userId, deviceFingerprint)` is a unique constraint, the upsert reuses the row when the same device re-grants permission.

`WebPushService.sendToUser(userId, payload)` returns `{sent, deactivated}`:

- `sent` — number of `webpush.sendNotification` calls that resolved.
- `deactivated` — number of rows deleted because the push service returned **404** (Not Found) or **410** (Gone). Browsers retire endpoints periodically; cleaning them up keeps the table from accumulating dead rows.
- Transient errors (429, 5xx, network blip) leave the row in place and emit `event:web_push_transient_failure`.
- VAPID unconfigured at boot → emits `event:vapid_unconfigured` and `sendToUser` always returns `{sent: 0}` → caller's WhatsApp fallback fires. Boot does not crash.

## Service worker handling

`public/sw.js` (v0.3.0+) has two responsibilities on every push event:

```js
self.addEventListener('push', (event) => {
  const data = event.data.json();
  event.waitUntil(Promise.all([
    // 1. OS notification — Chrome requires userVisibleOnly so this is non-optional.
    //    On Android the system auto-suppresses the banner if the PWA is foregrounded.
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icons/icon-192.png',
      badge: '/icons/icon-192.png',
      data: data.data,
      tag: data.data?.orderId,  // collapses duplicate retries
    }),
    // 2. postMessage focused clients so in-app code can react before the user
    //    even sees the notification — chime, dashboard refresh, etc.
    self.clients.matchAll({ type: 'window', includeUncontrolled: true })
      .then((clients) => clients.forEach((c) => c.postMessage({ source: 'push', data }))),
  ]));
});
```

Tap handling deep-links via `data.deepLink` (which the backend sets to `/vendor/commande/<id>`), focusing an existing tab when one is open on that URL instead of opening a duplicate.

## In-app chime + auto-refresh

`features/vendor/components/VendorDashboard.tsx` listens for SW messages:

```ts
React.useEffect(() => {
  const handler = (event: MessageEvent) => {
    if (event.data?.source !== 'push') return;
    if (event.data.data?.kind !== 'ORDER_CREATED') return;
    playChime();                          // two-tone Web Audio synth
    if (orders.status === 'ready') orders.reload();  // skip the 10s poll
  };
  navigator.serviceWorker.addEventListener('message', handler);
  return () => navigator.serviceWorker.removeEventListener('message', handler);
}, [orders]);
```

The chime uses Web Audio API (two sine oscillators with envelope) rather than a static MP3. Tradeoff: a slightly less polished sound, swappable later by pointing the play call at `/sounds/new-order.mp3`. Reasoning: avoids shipping an asset, no 404 risk, autoplay policies are forgiving because the vendor is by definition interacting with the dashboard.

## End-to-end timing

For a vendor with active subscription:

```
T+0ms     Consumer's Campay payment SUCCESSFUL webhook hits API
T+15ms    PaymentsService.handleWebhook acquires Redis lock, emits PAYMENT_SUCCEEDED
T+30ms    OrdersService.onPaymentSucceeded flips status, sets deadline, emits ORDER_CREATED
T+50ms    OrderNotificationsService.onOrderCreated reads vendor.userId
T+90ms    WebPushService dispatches to Mozilla / FCM push gateway
T+800ms~  Vendor's phone receives the push (Mozilla ~600ms, FCM ~1.2s typical)
T+850ms~  PWA SW push handler runs — OS notification + postMessage to focused clients
T+900ms~  If dashboard foregrounded: chime + orders.reload() → new card appears
```

For a vendor without active subscription (WhatsApp fallback):

```
T+90ms    WebPushService returns {sent: 0}
T+150ms   Twilio sendWhatsApp call dispatched
T+800ms~  Twilio messaging gateway processes
T+2-5s    WhatsApp delivery to the vendor's phone (sandbox 24h-window required)
```

In both paths, by the time the 60-second `acceptanceDeadlineAt` countdown is showing under 55s, the vendor has been pinged.

## Pilot constraint — Twilio sandbox

Production WhatsApp requires Meta-approved templates. The pilot uses the Twilio sandbox (`whatsapp:+14155238886`) which only delivers to phones that joined within the last 24h. Pilot vendors authenticate via OTP frequently enough that the 24h window stays warm.

**Production switchover** (post-RCCM): Meta template submission is on the launch runway in `chopnow-api` issue #181. Template approval takes 3-10 days; the template body has to match what `OrderNotificationsService` actually sends. When the template lands, the only code change is one env var (`TWILIO_WHATSAPP_FROM`) — the message body stays identical.
