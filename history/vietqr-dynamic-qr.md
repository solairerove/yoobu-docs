# VietQR Dynamic QR — Tech Spec

## Overview

Replace static `payment_qr_url` tenant config with per-booking dynamic QR URLs constructed from VietQR's public CDN. No API calls, no storage, no CDN uploads. Backend constructs the URL at response time; frontend renders it as `<img>`.

URL format:
```
https://img.vietqr.io/image/{bank_bin}-{account_number}-compact2.png
  ?amount={totalPrice}
  &addInfo=ORDER-{bookingId}
```

---

## Changes

### Backend

#### 1. New `tenant_config` keys

| Key | Type | Required | Notes |
|---|---|---|---|
| `payment_bank_bin` | `String` | required for dynamic QR | e.g. `970436` |
| `payment_account_number` | `String` | required for dynamic QR | e.g. `1059202107` |

Existing `payment_qr_url` key is retained as fallback for tenants not configured with bank details.

#### 2. `TenantSettings`

Add `payment_bank_bin` and `payment_account_number` to `payment()` settings slice alongside existing `payment_qr_url`.

#### 3. `BookingResponse` DTO

Add field:
```java
String paymentQrUrl  // nullable
```

Constructed in the booking → response mapping step:
- If `payment_bank_bin` and `payment_account_number` are present in tenant settings → build VietQR URL with `amount` and `addInfo=ORDER-{id}`.
- Else if `payment_qr_url` is present → use as-is.
- Else → `null`.

`totalPrice` is already on `Booking` at response time. No additional queries required.

#### 4. Superadmin panel — tenant form

Add two fields to `SuperAdminTenantForm`:
- `paymentBankBin` (optional)
- `paymentAccountNumber` (optional)

Existing `paymentQrUrl` field remains. Superadmin can configure either: static QR URL or bank details for dynamic generation. Both can coexist; dynamic takes priority in the mapping logic above.

#### 5. Admin panel — booking detail view

Show QR in booking detail (`admin/panel/booking-detail`) when `paymentQrUrl` is resolvable for that booking. Use the same mapping logic: instantiate via settings + booking fields directly in the controller/service method that builds the view model.

No new endpoint. Read settings from `TenantContext`, build URL inline.

#### 6. No Flyway migration needed

`payment_bank_bin` and `payment_account_number` are stored in existing `tenant_config` table as new key-value rows. Schema unchanged.

---

### Frontend

#### `BookingResponse` model

```typescript
export interface BookingResponse {
  // existing fields ...
  paymentQrUrl: string | null;  // add
}
```

#### QR display priority

Wherever `paymentQrUrl` is currently read from `TenantConfig`, switch to:
1. `booking.paymentQrUrl` (per-booking, dynamic)
2. `tenantConfig.paymentQrUrl` (static fallback)

Affected components: `FoodOrderSuccessCardComponent`, `FoodOrderBookingsComponent`.

`TenantConfig.paymentQrUrl` remains in the model and is still fetched. It serves only as fallback now.

---

### Rust service

No changes.

---

## Admin reconciliation flow (post-MVP)

With dynamic QR:
1. Customer scans → bank app pre-fills amount and `ORDER-{id}` as transfer comment.
2. Customer completes transfer.
3. Admin receives bot notification (existing) that booking moved to `PAYMENT_PENDING`.
4. Admin checks bank statement — `ORDER-{id}` in memo matches booking ID.
5. Admin sets status to `CONFIRMED` in panel.

Manual but deterministic. No payment provider integration required.

---

## Out of scope

- Webhook-based auto-confirmation (MoMo, ZaloPay, VNPay)
- Telegram Payments
- Any server-side HTTP call to `api.vietqr.io`
