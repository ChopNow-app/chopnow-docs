# API reference (vendor-facing)

Every HTTP endpoint a vendor user calls — directly via the PWA or transitively via background polling.

All routes are `/api/*` prefixed at the global level. All bodies are JSON unless marked `multipart/form-data`. All routes are JWT-protected by default (header `Authorization: Bearer <accessToken>`); the public ones are explicitly marked.

## Onboarding

### `POST /api/vendors` — Submit a vendor application

| Auth | Public (no JWT — the vendor doesn't have an account yet) |
| Throttle | 5 / hour / IP |
| Body | `multipart/form-data` — `SubmitVendorDto` JSON fields + photo fields |

Files: `profilePhoto` (required), `firstItemPhoto` (required), `enseignePhoto` (required only when `type=RESTAURANT`).

```json
{
  "type": "INFORMAL",
  "name": "Chez Maman Mboué",
  "description": "Cuisine maison camerounaise",
  "whatsappPhone": "670000000",
  "momoPhone": "670000000",
  "quartier": "Bonamoussadi",
  "landmark": "Rond-point Total",
  "firstItem": { "name": "Ndolé", "priceXAF": 2000, "kind": "FOOD" },

  // RESTAURANT only:
  "ownerName": "Marie Mboué",
  "rccm": "RC/DLA/2024/B/12345",
  "niu": "M122345678910N"
}
```

**Responses**

| Status | Body |
|---|---|
| 201 | `{ vendorId, status: 'PENDING_REVIEW' }` |
| 400 | Validation failure (missing field, photo too large, etc.) |
| 409 | `{ code: 'vendor_already_submitted' }` or `{ code: 'phone_used_by_other_role' }` |
| 429 | Throttled |

## Profile (authenticated vendor)

### `GET /api/vendors/me` — Read own profile

Returns the vendor's full self-view including hours array. Stripped of admin-only fields.

```json
{
  "id": "...",
  "type": "RESTAURANT",
  "status": "ACTIVE",
  "name": "...",
  "description": "...",
  "whatsappPhone": "+237670000000",
  "momoPhone": "+237670000000",
  "quartier": "Bonamoussadi",
  "rccm": "...",
  "niu": "...",
  "isOpen": true,
  "isOpenNow": true,
  "profilePhotoUrl": "/r2/vendor-photos/...",
  "coverPhotoUrl": "/r2/vendor-photos/...",
  "hours": [{ "dayOfWeek": 0, "isOpen": false, "openTime": null, "closeTime": null }, ...],
  "badge": "Restaurant"
}
```

### `PATCH /api/vendors/me` — Update self-editable fields

```json
{ "name": "...", "description": "...", "momoPhone": "..." }
```

Forbidden fields silently dropped server-side: `type`, `quartier`, `landmark`, `rccm`, `niu`, `enseignePhotoUrl`, `status`. Returns the refreshed profile.

### `PATCH /api/vendors/me/photo` — Replace profile photo

`multipart/form-data` with field `photo`. Max 5 MB. Accepts JPEG / PNG / WebP / HEIC / HEIF. Server resizes + recompresses via `sharp` and uploads to R2 before updating `Vendor.profilePhotoUrl`. Returns `{ photoUrl }`.

### `PATCH /api/vendors/me/cover` — Replace cover photo

Same shape as `/photo` but writes to `Vendor.coverPhotoUrl`.

## Availability

### `GET /api/vendors/me/availability`

```json
{ "isOpen": true, "isOpenNow": true }
```

`isOpen` is the manual switch; `isOpenNow` is the AND of `isOpen` with the current weekday's `VendorOpeningHours` window.

### `PATCH /api/vendors/me/availability`

```json
{ "isOpen": false }
```

Returns the updated availability snapshot.

### `PATCH /api/vendors/me/hours`

Replace the seven `VendorOpeningHours` rows in one shot:

```json
{
  "hours": [
    { "dayOfWeek": 0, "isOpen": false, "openTime": null, "closeTime": null },
    { "dayOfWeek": 1, "isOpen": true,  "openTime": "08:00", "closeTime": "22:00" },
    ...
  ]
}
```

Validation: exactly 7 entries, each `dayOfWeek` distinct in `[0, 6]`. `openTime < closeTime` when `isOpen`.

## Menu

### `GET /api/vendors/me/items`

Returns the vendor's items with categories and stock. Used by `/vendor/menu`.

### `POST /api/vendors/me/items`

Create a menu item.

```json
{
  "name": "Ndolé",
  "description": "Plat traditionnel",
  "priceXAF": 2000,
  "kind": "FOOD",
  "categoryId": "<menucategory-uuid>",
  "preparationMinutes": 15
}
```

### `PATCH /api/vendors/me/items/:itemId`

Partial update. Fields: `name`, `description`, `priceXAF`, `isAvailable`, `stockLevel`, `kind`, `categoryId`, `sortOrder`, `preparationMinutes`.

### `PATCH /api/vendors/me/items/:itemId/photo`

`multipart/form-data` with field `photo`. Replaces item photo in R2. Returns `{ photoUrl }`.

### `DELETE /api/vendors/me/items/:itemId`

Soft-deletes the row from the menu view. `OrderItem` rows retain the snapshot fields (name + price) so historic orders are unaffected.

### Category CRUD

```
GET    /api/vendors/me/categories
POST   /api/vendors/me/categories       { name, sortOrder? }
PATCH  /api/vendors/me/categories/:id   { name?, sortOrder? }
DELETE /api/vendors/me/categories/:id
```

Deleting a category sets `Item.categoryId` to null on its items (`onDelete: SetNull`).

## Orders

### `GET /api/orders/vendor/me`

Returns the vendor's orders, **filtered to `paymentStatus=PAID`**. An order in `PENDING` with `paymentStatus=PENDING` is invisible to the vendor (payment-gated visibility; see [order-lifecycle](order-lifecycle.md)).

The response strips `deliveryCode` (consumer's secret) — vendor only ever sees `pickupCode`.

Limit: 100 most recent orders, ordered by `placedAt DESC`.

### `PATCH /api/orders/:id/accept`

Flips `status: PENDING|CONFIRMED → ACCEPTED`. Status-guarded `updateMany` — see [concurrency](concurrency.md).

| Status | Body code |
|---|---|
| 200 | (returns refreshed order) |
| 404 | order belongs to a different vendor (`order_not_found`) |
| 409 | `order_not_pending` (stale URL — already decided) |
| 409 | `order_state_changed` (sub-100ms race — cron auto-refused first) |

Emits `DomainEvents.ORDER_ACCEPTED` on success.

### `PATCH /api/orders/:id/refuse`

```json
{
  "reason": "POWER_OUTAGE",
  "note": "Pas de courant depuis 30 min"
}
```

`reason` is an enum: `POWER_OUTAGE`, `CLOSED`, `TOO_MANY_ORDERS`, `ITEM_OUT_OF_STOCK`, `OTHER`. `note` is optional free-text — appended to `Order.refusalReason` as `${reason}: ${note}`.

Same status-guarded `updateMany` pattern. Same conflict codes.

Emits `DomainEvents.ORDER_REFUSED` on success → consumer-side `OrderNotificationsService.onOrderRefused` sends a humanised WhatsApp to the consumer.

### `PATCH /api/orders/:id/items/:itemId/prepared`

```json
{ "prepared": true }
```

Sets `OrderItem.preparedAt`. Side effect: if this is the **first** item marked prepared on an order, `Order.status` flips `ACCEPTED → IN_PREP` in the same transaction.

| Status | Body code |
|---|---|
| 200 | (returns updated order line) |
| 404 | order belongs to a different vendor |
| 409 | `order_not_in_prep_phase` (not in `ACCEPTED` or `IN_PREP`) |

### `PATCH /api/orders/:id/ready`

No body. Flips `status: ACCEPTED|IN_PREP → READY_PICKUP`. Status-guarded `updateMany`. Pre-condition: every `OrderItem` for this order has `preparedAt != null`.

| Status | Body code |
|---|---|
| 200 | (returns refreshed order) |
| 409 | `items_not_all_prepared` (at least one line still has `preparedAt: null`) |
| 409 | `order_not_in_prep_phase` (status guard pre-check) |
| 409 | `order_state_changed` (`updateMany` returned 0) |

Emits `DomainEvents.ORDER_READY` on success.

## Notifications

### `POST /api/notifications/push/subscribe`

```json
{
  "endpoint": "https://fcm.googleapis.com/...",
  "keys": { "p256dh": "...", "auth": "..." },
  "deviceFingerprint": "<localStorage UUID>"
}
```

Upserts on `(userId, deviceFingerprint)` — re-granting permission from the same device refreshes the endpoint URL in place.

| Status | |
|---|---|
| 201 | `{ id: <subscription-uuid> }` |
| 429 | Throttled (10/min) |

### `DELETE /api/notifications/push/subscribe`

```json
{ "endpoint": "https://fcm.googleapis.com/..." }
```

Idempotent — 204 even if no row matched.

## Auth (consumed by vendor onboarding + sign-in)

These are the same auth endpoints as consumers. Documented in full in `architecture/security`; called out here because the vendor flow exercises them:

```
POST   /api/auth/request-otp       throttled 5 / 15min / IP + 5 / 15min / phone
POST   /api/auth/verify-otp        creates/upgrades user, returns access + refresh JWTs
POST   /api/auth/refresh           in-flight Redis lock (PR #175)
```

`request-otp` has the 30s in-flight idempotency lock per PR #171; `refresh` has the 5s per-token lock per PR #175. See [concurrency](concurrency.md) for the full picture.
