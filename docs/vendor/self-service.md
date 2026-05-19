# Self-service surfaces

The vendor PWA at `/vendor/*` is a set of focused, mobile-first surfaces, each with a single job. None of them require a desktop computer — the founder's research showed every pilot vendor manages their kitchen from a phone.

All routes are protected by the global JWT guard. The vendor's JWT carries `role=VENDOR`; non-vendor users hitting these routes get a 403.

## `/vendor` — Dashboard

`features/vendor/components/VendorDashboard.tsx`

The home base for an authenticated vendor. Lays out:

1. **Availability section** — single switch toggling `Vendor.isOpen`. Status shown: 🟢 Ouvert / ⚪ Fermé.
2. **Push permission banner** — only renders when `usePushSubscription` state is `prompt`. See [notifications](notifications.md).
3. **À décider** — orders awaiting decision (`status ∈ {PENDING, CONFIRMED}`, filtered by backend to also require `paymentStatus=PAID`). Each card routes to `/vendor/commande/[id]`.
4. **En cours** — orders being prepared or in transit (`status ∈ {ACCEPTED, IN_PREP, READY_PICKUP, PICKED_UP}`). Each card routes to `/vendor/preparation/[id]`.
5. **Quick nav** — Menu, Hours, Profile.

### Data flow

- `useVendorAvailability` — `GET /api/vendors/me/availability`, mutates via `PATCH /api/vendors/me/availability`.
- `useVendorOrders` — polls `GET /api/orders/vendor/me` every 10s. Re-polls immediately on SW push message (`event.data.kind === 'ORDER_CREATED'`).
- `useMenuItems` — used for the "Mon menu" tile preview ("N plats").

### Push message handler

```ts
React.useEffect(() => {
  const handler = (event: MessageEvent) => {
    if (event.data?.source !== 'push') return;
    if (event.data.data?.kind !== 'ORDER_CREATED') return;
    playChime();
    if (orders.status === 'ready') orders.reload();
  };
  navigator.serviceWorker.addEventListener('message', handler);
  return () => navigator.serviceWorker.removeEventListener('message', handler);
}, [orders]);
```

## `/vendor/menu` — Menu management

`features/vendor/components/VendorMenuScreen.tsx`

Tabbed view by category (Tout · Plats · Boissons), driven by `Item.kind` (FOOD / DRINK) and `Item.categoryId` (foreign key to `MenuCategory`).

For each item:

- Photo (uploaded to R2 via `PATCH /api/vendors/me/items/:id/photo`).
- Name, description, price (FCFA, integer, no decimals).
- `isAvailable` toggle — vendor controls whether the item shows in the consumer catalogue.
- `stockLevel` (3-tier: `IN_STOCK` · `LOW_STOCK` · `OUT_OF_STOCK`). `LOW_STOCK` surfaces a "stock faible" warning on the consumer side without hiding the item. `OUT_OF_STOCK` is equivalent to `isInStock=false`.
- Sort order — drag-to-reorder within a category.

Categories are created/renamed/deleted via the `MenuCategory` CRUD endpoints.

### Item-availability flip race

If a vendor flips an item to `isAvailable: false` *while* a consumer is mid-checkout, the order may commit with the now-unavailable item. The `checkPostCreateItemAvailability` telemetry catches this — `event:order_item_flip_race` fires for ops visibility. See [concurrency](concurrency.md).

## `/vendor/hours` — Opening hours editor

`features/vendor/components/VendorHoursScreen.tsx`

Seven rows, one per weekday, each with: open toggle + open time + close time. Persists via `PATCH /api/vendors/me/hours` with the array of seven `VendorOpeningHours` rows.

The hours feed two consumer-side computations:

1. **`isOpenNow`** — combined with `Vendor.isOpen` toggle, surfaces 🟢 Ouvert / ⚪ Fermé. Both must be true for the vendor to appear in the consumer catalogue.
2. **Catalogue filtering** — closed vendors are sorted to the bottom, not hidden entirely (consumers can pre-order for later).

## `/vendor/profile` — Profile editor

`features/vendor/components/VendorProfileScreen.tsx`

What the vendor can self-edit:

- `name` — restaurant name (max 80 chars).
- `description` — optional, 500 char max ("cuisine maison camerounaise").
- `momoPhone` — MoMo number for payouts (cannot match `whatsappPhone`).
- Profile photo (the card avatar in the catalogue).
- Cover photo (the hero image at the top of the consumer-side vendor detail page).

Saves trigger up to 3 API calls in sequence (not parallel — order matters for cache invalidation downstream):

1. `PATCH /api/vendors/me` — name + description + momoPhone (only if `isDirty`).
2. `PATCH /api/vendors/me/photo` — only if profile photo changed.
3. `PATCH /api/vendors/me/cover` — only if cover photo changed.

What the vendor **cannot** edit here:

- `type`, `quartier`, `landmark`, `rccm`, `niu`, `enseignePhoto` — admin-controlled (see [onboarding](onboarding.md)).
- `whatsappPhone` — tied to the authentication identity; changing it requires a support handoff.

## `/vendor/commande/[orderId]` — Decision screen

`features/vendor/components/OrderAcceptanceScreen.tsx`

Full-screen mobile takeover (no bottom nav, no app shell) so the vendor's eyes stay on the 60s countdown timer. Three subviews:

- **`DecisionView`** — large red 💚 Accept + outlined ❌ Refuse buttons. Countdown to `acceptanceDeadlineAt` shown as `mm:ss` over a thinning progress bar. Refuse opens a reason picker (`POWER_OUTAGE`, `CLOSED`, `TOO_MANY_ORDERS`, `ITEM_OUT_OF_STOCK`, `OTHER` + free-text note).
- **`AcceptedView`** — flashes briefly with the order code + delivery address before routing to `/vendor/preparation/[id]`.
- **`TerminalView`** — if the order is no longer decidable (already accepted, refused, expired), shows what happened and routes back.

The countdown uses **absolute** time (`new Date(order.acceptanceDeadlineAt).getTime() - Date.now()`) not a from-now decrement. A tab reload preserves the remaining seconds.

Error handling for the accept/refuse mutations: any 409 body's `code` field is surfaced verbatim in the error banner. The three codes the vendor might see: `order_not_pending`, `order_state_changed`, `wrong_pickup_code` (latter doesn't happen on this screen).

## `/vendor/preparation/[orderId]` — Prep checklist

`features/vendor/components/PreparationScreen.tsx`

Per-line-item: dish name, quantity, an "X préparé(s)" toggle that hits `PATCH /api/orders/:id/items/:itemId/prepared`. The first item flipped triggers status `ACCEPTED → IN_PREP` server-side.

Above the list: the pickup code (4-digit) in big monospace numerals — what the vendor shows the rider on handover.

Bottom of the screen: a **Commande prête** button enabled once every line is checked off. Hits `PATCH /api/orders/:id/ready`, status flips `IN_PREP → READY_PICKUP`, emits `DomainEvents.ORDER_READY`, dispatch's rider-search kicks off (Story 4.14) if not already assigned.

## Browser support

The PWA is tested on:

- Chrome / Edge (Android + Desktop)
- Safari iOS 16.4+ (PWA must be Added to Home Screen for Web Push)
- Firefox (Android + Desktop)

Older iOS (< 16.4) falls back to the WhatsApp notification path. The `usePushSubscription` hook reports `unsupported` and the banner stays hidden.

## Frontend hook conventions

All vendor surfaces use the same hook shape:

```ts
type HookState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'unauthenticated' }
  | { status: 'not_found' }
  | { status: 'error'; message: string }
  | { status: 'ready'; data: T; reload: () => void };
```

This makes loading / 401 / 404 / error / ready states uniform across components and matches the React 19 lint rules (no `useEffect` ping-pong that the new `react-hooks/set-state-in-effect` rule flags).
