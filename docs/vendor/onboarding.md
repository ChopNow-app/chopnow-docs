# Onboarding & KYC

ChopNow segments vendors into three types. The type drives what we ask at submission, what admin reviews, and which features are unlocked.

## Vendor types

| Type | Profile | Required at submission | RCCM/NIU |
|---|---|---|---|
| `INFORMAL` | "Tantine Belle, cuisine à domicile" — solo cook, no shop | Profile photo + first item photo + cuisine, ID document of the cook | Not required |
| `SEMI_FORMAL` | Small kitchen with hired hands, no storefront | Same as informal + capacity declaration (orders/hour) | Not required |
| `RESTAURANT` | Formal establishment with storefront, employees, registered | Same as semi-formal + `enseignePhoto` (storefront photo), `rccm` (RCCM number), `niu` (NIU tax ID), `ownerName` | **Required** before activation |

The enum lives at `prisma/schema.prisma` (`VendorType`). Type is set at submission and is not user-editable later — admin can change it via the validation queue if a vendor's footprint outgrows their declared tier.

## Submission flow

Public route, no JWT (the whole point is the vendor doesn't have an account yet):

```
POST /api/vendors           — VendorController.submit (vendor.controller.ts:54)
  Content-Type: multipart/form-data
  Body:        SubmitVendorDto (JSON fields + file fields)
  Files:       profilePhoto, firstItemPhoto, enseignePhoto (RESTAURANT only)
  Throttle:    5 submissions per hour per IP
```

Inside a single transaction:

1. **User row** — created if `phone` is new, role set to `VENDOR`. If the phone is already a `CONSUMER`, upgrade to `VENDOR` keeping the row id (consumers who decide to onboard as cooks don't lose their order history).
2. **Vendor row** — created with `status = PENDING_REVIEW`, location seeded to Douala center until landmark resolution lands (Story 2.15).
3. **First Item row** — created with `isAvailable: true`, `isInStock: true`. Vendor adds more from `/vendor/menu` after activation.
4. **Photos** — uploaded to R2 **before** the transaction. Orphans on failure are acceptable; an R2 lifecycle rule sweeps unreferenced keys.

WhatsApp confirmation fires fire-and-forget after commit ("dossier reçu, validation sous 24h"). Delivery failure emits `event:vendor_submission_whatsapp_failed` and does **not** roll back the submission.

### Conflict codes

| Code | Meaning |
|---|---|
| `vendor_already_submitted` | The phone has an open application (`PENDING_REVIEW` or `CORRECTION_REQUESTED`) |
| `phone_used_by_other_role` | Phone belongs to a `RIDER` or admin user — vendors can't share with operational roles |

## Admin validation queue

Admin views the queue at `/admin/validation` (frontend) backed by:

```
GET    /api/admin/vendors/pending          AdminValidationService.listPendingVendors
POST   /api/admin/vendors/:id/approve      → VendorStatus.ACTIVE + WhatsApp green-light
POST   /api/admin/vendors/:id/reject       → VendorStatus.REJECTED + reason
POST   /api/admin/vendors/:id/suspend      → VendorStatus.SUSPENDED + reason (JWT revoked)
POST   /api/admin/vendors/:id/unsuspend
```

Notification fan-out is best-effort (`event:admin_notification_failed` if WhatsApp drops). Admin action succeeds regardless.

## Self-service updates post-activation

After approval, the vendor can edit a tightly-scoped subset of their own profile via `PATCH /api/vendors/me`:

- ✅ `name`, `description`, `momoPhone`
- ✅ Profile photo (`PATCH /api/vendors/me/photo`)
- ✅ Cover photo (`PATCH /api/vendors/me/cover`)
- ❌ `type`, `quartier`, `landmark`, `rccm`, `niu` — admin-controlled. A vendor wanting these changed contacts support.

The reasoning: KYC-critical fields stay admin-controlled so a vendor cannot quietly downgrade their declared type to dodge tier-specific requirements (e.g., a `RESTAURANT` blanking its RCCM).

## Pilot caveat — TchopNow trading name

The legal entity is in process. Until RCCM lands, vendor agreements are signed in the **TchopNow** trading name + the founder's personal ID. This matters for `RESTAURANT`-type vendors only — informal and semi-formal types don't require their own RCCM to onboard.

When the platform's RCCM is issued, the `Vendor.rccm` field on restaurant rows captures **their** RCCM, not the platform's. The platform's registration affects the Campay Go-Live application, not the vendor's onboarding.

## Data model snapshot

The Prisma enum + key Vendor fields. Full schema lives in [`data-model`](data-model.md).

```prisma
enum VendorType {
  INFORMAL
  SEMI_FORMAL
  RESTAURANT
}

enum VendorStatus {
  PENDING_REVIEW
  ACTIVE
  REJECTED
  CORRECTION_REQUESTED
  SUSPENDED
}

model Vendor {
  id             String       @id @default(uuid())
  userId         String       @unique
  type           VendorType
  status         VendorStatus @default(PENDING_REVIEW)
  name           String
  description    String?
  whatsappPhone  String       // notifications
  momoPhone      String       // payouts
  ownerName      String?
  quartier       String
  landmark       String?
  rccm           String?      // RESTAURANT only
  niu            String?      // RESTAURANT only
  enseignePhotoUrl String?    // RESTAURANT only
  // ... + photos, location, hours, etc.
}
```
