# yoobu-api — Actual API State
Generated from source code on: 2026-03-30

---

## Endpoints

### Client API (`/t/{slug}/...`)

| Method | Path | Request Body | Response Body | Auth | Notes |
|--------|------|-------------|---------------|------|-------|
| GET | `/t/{slug}/config` | — | `TenantConfigResponse` | none | Returns branding, hints, cutoff, earliest delivery date |
| GET | `/t/{slug}/services` | — | `List<ServiceResponse>` | none | Returns ACTIVE services only |
| POST | `/t/{slug}/bookings` | `CreateBookingRequest` | `BookingResponse` | Telegram initData | FOOD_ORDER tenants only |
| GET | `/t/{slug}/bookings/my` | — | `List<BookingResponse>` | Telegram initData | Filtered by telegramUserId |
| GET | `/t/{slug}/bookings/{bookingId}` | — | `BookingResponse` | Telegram initData | 404 if not owned by user |
| POST | `/t/{slug}/bookings/{bookingId}/cancel` | — | `BookingResponse` | Telegram initData | Customer-initiated cancel |
| POST | `/t/{slug}/bookings/{bookingId}/confirm-payment` | — | `BookingResponse` | Telegram initData | Triggers PaymentConfirmedEvent |

### Admin API (`/admin/{slug}/...`)

#### REST API

| Method | Path | Request Body | Response Body | Auth | Notes |
|--------|------|-------------|---------------|------|-------|
| GET | `/admin/{slug}/bookings` | — | `List<BookingResponse>` | BasicAuth | Filter by `?status=` and `?deliveryDate=` |
| GET | `/admin/{slug}/bookings/{bookingId}` | — | `BookingResponse` | BasicAuth | |
| PUT | `/admin/{slug}/bookings/{bookingId}/status` | `UpdateStatusRequest` | `BookingResponse` | BasicAuth | State machine enforced |
| GET | `/admin/{slug}/services` | — | `List<ServiceResponse>` | BasicAuth | All statuses |
| POST | `/admin/{slug}/services` | `AdminUpsertServiceRequest` | `ServiceResponse` | BasicAuth | 201 Created |
| PUT | `/admin/{slug}/services/{serviceId}` | `AdminUpsertServiceRequest` | `ServiceResponse` | BasicAuth | |
| POST | `/admin/{slug}/services/{serviceId}/image` | `multipart/form-data` (`file`) | `ServiceResponse` | BasicAuth | Delegates to ImageServiceClient |
| DELETE | `/admin/{slug}/services/{serviceId}` | — | — | BasicAuth | 204 No Content; soft-delete |

#### HTML Panel (Thymeleaf)

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | `/admin/{slug}/panel` | BasicAuth | Dashboard / home redirect |
| GET | `/admin/{slug}/panel/bookings` | BasicAuth | Paginated list; filter by status, deliveryDate |
| GET | `/admin/{slug}/panel/bookings/{bookingId}` | BasicAuth | Booking detail view |
| POST | `/admin/{slug}/panel/bookings/{bookingId}/status` | BasicAuth | Form-based status update |
| GET | `/admin/{slug}/panel/services` | BasicAuth | Paginated service list; search by query |
| GET | `/admin/{slug}/panel/services/new` | BasicAuth | New service form |
| POST | `/admin/{slug}/panel/services` | BasicAuth | Create via HTML form |
| GET | `/admin/{slug}/panel/services/{serviceId}/edit` | BasicAuth | Edit form |
| POST | `/admin/{slug}/panel/services/{serviceId}` | BasicAuth | Update via HTML form |
| POST | `/admin/{slug}/panel/services/{serviceId}/delete` | BasicAuth | Requires `confirmName` param |
| POST | `/admin/{slug}/panel/services/{serviceId}/status` | BasicAuth | Toggle ACTIVE/INACTIVE |

### Superadmin API (`/superadmin/...`)

#### REST API

| Method | Path | Request Body | Response Body | Auth | Notes |
|--------|------|-------------|---------------|------|-------|
| GET | `/superadmin/tenants` | — | `List<TenantSummaryResponse>` | BasicAuth (superadmin) | |
| GET | `/superadmin/tenants/{tenantId}` | — | `TenantDetailResponse` | BasicAuth (superadmin) | Includes config map |
| GET | `/superadmin/tenants/slug-availability` | — | `TenantSlugAvailabilityResponse` | BasicAuth (superadmin) | `?slug=` query param |
| POST | `/superadmin/tenants` | `CreateTenantRequest` | `TenantSummaryResponse` | BasicAuth (superadmin) | |
| PUT | `/superadmin/tenants/{tenantId}` | `UpdateTenantRequest` | `TenantSummaryResponse` | BasicAuth (superadmin) | |
| GET | `/superadmin/audit` | — | `AuditLogPageResponse` | BasicAuth (superadmin) | Multiple filter params; default page=0, size=20 |
| GET | `/superadmin/audit/export` | — | `text/csv` bytes | BasicAuth (superadmin) | Same filters; default size=5000 |

#### HTML Panel (Thymeleaf)

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | `/superadmin/panel` | BasicAuth (superadmin) | Dashboard |
| GET | `/superadmin/panel/tenants` | BasicAuth (superadmin) | Paginated; search by query |
| GET | `/superadmin/panel/tenants/{tenantId}` | BasicAuth (superadmin) | Tenant detail |
| GET | `/superadmin/panel/tenants/new` | BasicAuth (superadmin) | Create form |
| POST | `/superadmin/panel/tenants` | BasicAuth (superadmin) | Create via form |
| GET | `/superadmin/panel/tenants/{tenantId}/edit` | BasicAuth (superadmin) | Edit form |
| POST | `/superadmin/panel/tenants/{tenantId}` | BasicAuth (superadmin) | Update via form |
| POST | `/superadmin/panel/tenants/{tenantId}/payment-qr` | BasicAuth (superadmin) | Upload QR image via multipart |
| GET | `/superadmin/panel/audit` | BasicAuth (superadmin) | Audit log table |
| GET | `/superadmin/panel/audit/export` | BasicAuth (superadmin) | CSV export |

### Other

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | `/health` | none | Returns `{"status": "ok"}` |

---

## DTOs

### `CreateBookingRequest`
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| customerName | String | `@NotBlank` | |
| customerPhone | String | `@NotBlank` | |
| deliveryAddress | String | `@NotBlank` | |
| deliveryDate | LocalDate | `@NotNull` | Must be >= earliest allowed date |
| note | String | — | Optional |
| items | `List<BookingItemRequest>` | `@NotEmpty`, each `@Valid` | |

### `BookingItemRequest`
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| serviceId | Long | `@NotNull` | Must be ACTIVE in tenant |
| quantity | int | `@Min(1)` | |

### `UpdateStatusRequest`
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| status | BookingStatus | `@NotNull` | State machine validated in service |
| trackingUrl | String | `@Size(max=2048)` | Optional; validated for valid http/https URL if present |

### `AdminUpsertServiceRequest`
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| name | String | `@NotBlank` | |
| description | String | — | Optional |
| price | BigDecimal | `@NotNull` | |
| unit | String | — | Optional |
| durationMinutes | Integer | — | Optional |
| sortOrder | Integer | — | Optional |
| status | ServiceStatus | — | Optional; defaults to ACTIVE |

### `CreateTenantRequest`
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| slug | String | `@NotBlank` | Unique identifier |
| name | String | `@NotBlank` | |
| type | TenantType | `@NotNull` | FOOD_ORDER / APPOINTMENT / CATALOG_REQUEST |
| botToken | String | — | Optional; required for Telegram notifications |
| ownerTelegramId | Long | — | Optional; required for admin notifications |
| timezone | String | — | Defaults to Asia/Ho_Chi_Minh |
| currency | String | — | Defaults to USD |
| primaryColor | String | — | |
| logoUrl | String | — | |
| welcomeMessage | String | — | |
| checkoutNameHint | String | — | |
| checkoutPhoneHint | String | — | |
| checkoutNoteHint | String | — | |
| checkoutDeliveryHint | String | — | |
| paymentQrUrl | String | — | Static fallback QR |
| paymentBankBin | String | — | For VietQR dynamic generation |
| paymentAccountNumber | String | — | For VietQR dynamic generation |
| adminUsername | String | `@NotBlank` | |
| adminPassword | String | `@NotBlank` | Stored as BCrypt hash |
| cutoffHour | Integer | — | Delivery cutoff hour |
| cutoffMinute | Integer | — | Delivery cutoff minute |

### `UpdateTenantRequest`
Same as `CreateTenantRequest` except:
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| adminPassword | String | — | Optional on update |
| active | Boolean | `@NotNull` | Only on update |

### `BookingResponse`
| Field | Type | Notes |
|-------|------|-------|
| id | Long | |
| type | BookingType | |
| status | BookingStatus | |
| trackingUrl | String | Nullable |
| customerName | String | |
| customerPhone | String | |
| deliveryAddress | String | |
| totalPrice | BigDecimal | |
| currency | String | |
| deliveryDate | LocalDate | |
| note | String | Nullable |
| items | `List<BookingItemResponse>` | |
| createdAt | OffsetDateTime | |
| paymentQrUrl | String | Nullable; VietQR or static |

### `BookingItemResponse`
| Field | Type | Notes |
|-------|------|-------|
| serviceName | String | Snapshot at booking time |
| quantity | int | |
| unitPrice | BigDecimal | Snapshot at booking time |

### `ServiceResponse`
| Field | Type | Notes |
|-------|------|-------|
| id | Long | |
| name | String | |
| description | String | Nullable |
| price | BigDecimal | Nullable |
| unit | String | Nullable |
| durationMinutes | Integer | Nullable |
| sortOrder | int | |
| status | ServiceStatus | |
| imageUrl | String | Nullable |

### `TenantConfigResponse`
| Field | Type | Notes |
|-------|------|-------|
| slug | String | |
| name | String | |
| type | TenantType | |
| currency | String | |
| primaryColor | String | Nullable |
| logoUrl | String | Nullable |
| welcomeMessage | String | Nullable |
| checkoutNameHint | String | Nullable |
| checkoutPhoneHint | String | Nullable |
| checkoutNoteHint | String | Nullable |
| checkoutDeliveryHint | String | Nullable |
| paymentQrUrl | String | Nullable; VietQR or static |
| cutoffHour | Integer | Nullable |
| cutoffMinute | Integer | Nullable |
| earliestDeliveryDate | LocalDate | Computed from cutoff + current time |

### `TenantSummaryResponse`
| Field | Type | Notes |
|-------|------|-------|
| id | Long | |
| slug | String | |
| name | String | |
| type | TenantType | |
| active | boolean | |
| timezone | String | |
| createdAt | OffsetDateTime | |

### `TenantDetailResponse`
| Field | Type | Notes |
|-------|------|-------|
| id | Long | |
| slug | String | |
| name | String | |
| type | TenantType | |
| active | boolean | |
| timezone | String | |
| botToken | String | Nullable |
| ownerTelegramId | Long | Nullable |
| createdAt | OffsetDateTime | |
| config | `Map<String, String>` | All tenant_config key-value pairs |

### `TenantSlugAvailabilityResponse`
| Field | Type | Notes |
|-------|------|-------|
| slug | String | Requested slug |
| available | boolean | True if not taken |

### `AuditLogPageResponse`
| Field | Type | Notes |
|-------|------|-------|
| items | `List<AuditLogItemResponse>` | |
| page | int | |
| size | int | |
| totalElements | long | |
| totalPages | int | |
| hasNext | boolean | |
| hasPrevious | boolean | |

### `AuditLogItemResponse`
| Field | Type | Notes |
|-------|------|-------|
| id | Long | |
| tenantId | Long | |
| entity | String | e.g. "Booking", "CatalogService" |
| entityId | Long | |
| action | String | e.g. "CREATE", "UPDATE", "ACTION" |
| actorId | String | Telegram user ID or "system" |
| oldValue | Object | JSON snapshot (nullable) |
| newValue | Object | JSON snapshot (nullable) |
| createdAt | OffsetDateTime | |

---

## Entities

### `tenant`
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | BIGSERIAL | no | — | PK |
| slug | VARCHAR(100) | no | — | UNIQUE |
| name | VARCHAR(255) | no | — | |
| type | VARCHAR(30) | no | — | Enum: FOOD_ORDER, APPOINTMENT, CATALOG_REQUEST |
| bot_token | VARCHAR(255) | yes | — | |
| owner_telegram_id | BIGINT | yes | — | |
| timezone | VARCHAR(50) | no | 'Asia/Ho_Chi_Minh' | |
| active | BOOLEAN | no | TRUE | |
| created_at | TIMESTAMPTZ | no | — | |

### `tenant_config`
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | BIGSERIAL | no | — | PK |
| tenant_id | BIGINT | no | — | FK → tenant.id |
| key | VARCHAR(100) | no | — | |
| value | TEXT | yes | — | |

Unique constraint: `(tenant_id, key)`

Known keys: `admin_username`, `admin_password`, `primary_color`, `logo_url`, `welcome_message`, `checkout_name_hint`, `checkout_phone_hint`, `checkout_note_hint`, `checkout_delivery_hint`, `payment_qr_url`, `payment_bank_bin`, `payment_account_number`, `currency`, `cutoff_hour`, `cutoff_minute`

### `service`
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | BIGSERIAL | no | — | PK |
| tenant_id | BIGINT | no | — | FK → tenant.id |
| name | VARCHAR(255) | no | — | |
| description | TEXT | yes | — | |
| price | NUMERIC(10,2) | yes | — | |
| unit | VARCHAR(50) | yes | 'шт' | |
| duration_minutes | INT | yes | — | |
| status | VARCHAR(20) | no | 'ACTIVE' | Enum: ACTIVE, INACTIVE, DELETED |
| sort_order | INT | no | 0 | |
| image_url | TEXT | yes | — | Added in V9 |
| deleted_at | TIMESTAMPTZ | yes | — | Soft delete |
| created_at | TIMESTAMPTZ | no | — | |
| updated_at | TIMESTAMPTZ | no | — | |

Index: `idx_service_tenant` on `(tenant_id, status, sort_order, id)` WHERE `status <> 'DELETED'`

### `booking`
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | BIGSERIAL | no | — | PK |
| tenant_id | BIGINT | no | — | FK → tenant.id |
| type | VARCHAR(20) | no | — | Enum: ORDER, APPOINTMENT, REQUEST |
| telegram_user_id | BIGINT | no | — | |
| customer_name | VARCHAR(255) | no | — | |
| customer_phone | VARCHAR(50) | yes | — | Added in original schema |
| delivery_address | TEXT | yes | — | Added in V7 |
| status | VARCHAR(20) | no | — | Enum: NEW, PAYMENT_PENDING, CONFIRMED, DELIVERING, DONE, CANCELLED |
| tracking_url | TEXT | yes | — | Added in V8 |
| note | TEXT | yes | — | |
| total_price | NUMERIC(10,2) | yes | — | |
| currency | VARCHAR(10) | no | — | Added in V10 |
| payment_qr_url | TEXT | yes | — | Added in V10 |
| delivery_date | DATE | yes | — | |
| slot_id | BIGINT | yes | — | Reserved, unused |
| service_id | BIGINT | yes | — | FK → service.id (deprecated; items in booking_item) |
| deleted_at | TIMESTAMPTZ | yes | — | Soft delete |
| created_at | TIMESTAMPTZ | no | — | |
| updated_at | TIMESTAMPTZ | no | — | |
| version | BIGINT | no | 0 | Optimistic locking (added in V5) |

Indexes:
- `idx_booking_tenant` on `(tenant_id)`
- `idx_booking_tenant_date` on `(tenant_id, delivery_date)`
- `idx_booking_tenant_status` on `(tenant_id, status)`
- `idx_booking_telegram_user` on `(tenant_id, telegram_user_id)`

### `booking_item`
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | BIGSERIAL | no | — | PK |
| booking_id | BIGINT | no | — | FK → booking.id ON DELETE CASCADE |
| service_id | BIGINT | no | — | FK → service.id |
| quantity | INT | no | — | CHECK (quantity > 0) |
| unit_price | NUMERIC(10,2) | no | — | Snapshot at booking time |

### `audit_log`
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | BIGSERIAL | no | — | PK |
| tenant_id | BIGINT | no | — | Not FK; denormalized |
| entity | VARCHAR(50) | no | — | e.g. "Booking" |
| entity_id | BIGINT | no | — | |
| action | VARCHAR(50) | no | — | e.g. "CREATE", "UPDATE", "ACTION" |
| actor_id | VARCHAR(255) | yes | — | Telegram user ID string; altered VARCHAR in V2 |
| old_value | TEXT | yes | — | JSON snapshot |
| new_value | TEXT | yes | — | JSON snapshot |
| created_at | TIMESTAMPTZ | no | — | |

Indexes:
- `idx_audit_tenant_entity` on `(tenant_id, entity, entity_id)`
- `idx_audit_created_at_id` on `(created_at DESC, id DESC)`
- `idx_audit_tenant_created_at_id` on `(tenant_id, created_at DESC, id DESC)`
- `idx_audit_action_created_at_id` on `(action, created_at DESC, id DESC)`

---

## Enums

### `BookingType`
Values: `ORDER`, `APPOINTMENT`, `REQUEST`

Note: Only `ORDER` is actively used. `APPOINTMENT` and `REQUEST` are defined but all booking endpoints reject non-FOOD_ORDER tenants (checked in `BookingService.requireFoodOrderTenant()`).

### `BookingStatus`
Values: `NEW`, `PAYMENT_PENDING`, `CONFIRMED`, `DELIVERING`, `DONE`, `CANCELLED`

### `ServiceStatus`
Values: `ACTIVE`, `INACTIVE`, `DELETED`

`DELETED` is used for soft-delete (sets `deleted_at` column). Only `ACTIVE` services appear in the public catalog.

### `TenantType`
Values: `FOOD_ORDER`, `APPOINTMENT`, `CATALOG_REQUEST`

---

## Booking Validation Matrix

All booking creation is currently restricted to `FOOD_ORDER` tenants. The `CreateBookingRequest` applies the same validation regardless of any sub-type.

| Field | Validation | Notes |
|-------|-----------|-------|
| customerName | Required (`@NotBlank`) | Always |
| customerPhone | Required (`@NotBlank`) | Always |
| deliveryAddress | Required (`@NotBlank`) | Always |
| deliveryDate | Required (`@NotNull`); must be >= earliest allowed | Earliest date is computed from tenant cutoff time |
| note | Optional | |
| items | Required, non-empty (`@NotEmpty`) | Each item: serviceId not null, quantity >= 1 |
| Each serviceId | Must exist and be ACTIVE in tenant | Validated in `BookingService` |
| Each service price | Must be non-null | Validated in `BookingService` |

Source: `CreateBookingRequest` (Bean Validation) + `BookingService.createBooking()` (business rules)

**Status transition rules** (enforced in `BookingService.getAllowedAdminStatuses()`):

| From \ To | NEW | PAYMENT_PENDING | CONFIRMED | DELIVERING | DONE | CANCELLED |
|-----------|-----|-----------------|-----------|------------|------|-----------|
| NEW | ✓ | — | — | — | — | ✓ |
| PAYMENT_PENDING | — | ✓ | ✓ | — | — | ✓ |
| CONFIRMED | — | — | ✓ | ✓ | — | ✓ |
| DELIVERING | — | — | — | ✓ | ✓ | ✓ |
| DONE | — | — | — | — | ✓ | ✓ |
| CANCELLED | — | — | — | — | — | ✓ |

---

## Security

### Filter Chains

| Order | Matcher | Mechanism | Notes |
|-------|---------|-----------|-------|
| 1 | `/superadmin/**` | `SuperAdminBasicAuthenticationFilter` | Plain-text comparison against `app.superadmin.username/password` |
| 2 | `/admin/*/**` | `TenantContextFilter` + `TenantBasicAuthenticationFilter` | BCrypt-verified against `tenant_config`; superadmin creds also accepted |
| 3 | `/t/*/**` | `TenantContextFilter` only | All requests permitted; Telegram auth is handled per-endpoint |
| 4 | `/**` (default) | None | All requests permitted; covers `/health` and static paths |

CSRF: disabled on all chains. CORS: enabled on all chains (see Configuration section).

### Telegram Auth (per-endpoint, not filter-chain level)

- Header: `X-Telegram-Init-Data`
- Validation: HMAC-SHA256 using `tenant.bot_token` as secret key (Telegram WebApp algorithm)
- Handled by: `TelegramUserArgumentResolver` resolving `@TelegramPrincipal TelegramUser`
- Returns 401 if header is missing or signature invalid

### Dev Profile Fallback

- Header: `X-Telegram-User-Id` (plain user ID, no signature)
- Only accepted when Spring profile `dev` is active
- Bypasses HMAC verification entirely

### Tenant Resolution

- Class: `TenantContextFilter`
- Matches patterns: `/t/{slug}/...` and `/admin/{slug}/...`
- Extracts slug from path segment index 2 (1-based)
- Loads `Tenant` entity from DB; throws 404 if not found or not active
- Storage: `TenantContext` (ThreadLocal)
- Cleanup: `TenantContext.clear()` in `finally` block

---

## Flyway Migrations

| Version | Filename | Summary |
|---------|----------|---------|
| V1 | `V1__baseline_schema.sql` | Creates `tenant`, `tenant_config`, `service`, `booking`, `booking_item`, `audit_log`; initial indexes |
| V2 | `V2__audit_actor_id_as_string.sql` | Alters `audit_log.actor_id` from BIGINT → VARCHAR(255) |
| V3 | `V3__service_status_model.sql` | Adds `service.status` VARCHAR(20); backfills from `active`/`deleted_at`; drops `service.active` column; updates index |
| V4 | `V4__audit_log_filter_indexes.sql` | Adds `idx_audit_created_at_id`, `idx_audit_tenant_created_at_id`, `idx_audit_action_created_at_id` |
| V5 | `V5__booking_optimistic_lock.sql` | Adds `booking.version` BIGINT DEFAULT 0 |
| V6 | `V6__booking_item_currency.sql` | Adds `booking_item.currency` VARCHAR(10); backfills from `tenant_config` |
| V7 | `V7__booking_delivery_address.sql` | Adds `booking.delivery_address` TEXT |
| V8 | `V8__booking_tracking_url.sql` | Adds `booking.tracking_url` TEXT |
| V9 | `V9__service_image_url.sql` | Adds `service.image_url` TEXT |
| V10 | `V10__booking_currency_and_payment_qr_url.sql` | Adds `booking.currency` VARCHAR(10) (backfilled from `booking_item.currency` or 'USD'); adds `booking.payment_qr_url` TEXT; makes `booking_item.currency` nullable |

---

## Schema vs Entity Drift

- `booking.service_id` — present in DB migration (V1) and entity (`@ManyToOne CatalogService service`), but not used in any booking creation or response logic. Items are stored in `booking_item`. This column appears to be a leftover from an earlier design.
- `booking_item.currency` — added in V6, made nullable in V10, and is no longer the source of truth for currency (replaced by `booking.currency`). The entity field may still be present.

No other drift detected.

---

## Telegram Integration State

- **InitData validation:** Implemented — HMAC-SHA256 using `tenant.bot_token`; `TelegramInitDataValidator`
- **Notifications on booking create:** Yes — sends to `tenant.owner_telegram_id` (admin) with order summary including items, phone, address, delivery date
- **Notifications on payment confirmed:** Yes — sends to `tenant.owner_telegram_id` (admin): "💰 Payment confirmed"
- **Notifications on status change:** Yes — sends to `booking.telegram_user_id` (customer) for: CONFIRMED, DELIVERING (+ tracking URL if set), DONE, CANCELLED
- **Daily summary:** Not implemented
- **Dev fallback:** `X-Telegram-User-Id` header accepted when Spring profile `dev` is active; bypasses HMAC check; resolved in `TelegramUserArgumentResolver`
- **Delivery mechanism:** `TelegramBotApiClient` — POSTs to `https://api.telegram.org/bot{token}/sendMessage`; 3 retries with exponential backoff (1s → 3s → 9s); handles 429 rate limiting with `retry_after`; HTML parse mode
- **Async execution:** Notifications run in `@Async("notificationExecutor")` pool; triggered by `@TransactionalEventListener(phase = AFTER_COMMIT)` to avoid notifying on rollbacks

---

## Payment QR Logic

Computed in `BookingService.buildPaymentQrUrl()` and `TenantConfigResponse`:

1. If `payment_bank_bin` AND `payment_account_number` are both configured → generate dynamic VietQR URL:
   `https://img.vietqr.io/image/{bankBin}-{accountNumber}-compact2.png?amount={totalPrice}&addInfo=NUM-{bookingId}`
2. Else if `payment_qr_url` is configured → use static QR URL as-is
3. Else → `null`

---

## Configuration Shape

```yaml
spring:
  application:
    name: yoobu-api
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/yoobu}
    username: ${DB_USER:yoobu}
    password: ${DB_PASS:yoobu}
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    properties:
      hibernate:
        jdbc:
          time_zone: UTC
  flyway:
    locations: classpath:db/migration

app:
  cors:
    allowed-origin-patterns: ${CORS_ALLOWED_ORIGIN_PATTERNS:http://localhost:*,...,https://*.up.railway.app}
  superadmin:
    username: ${SUPERADMIN_USER:superadmin}
    password: ${SUPERADMIN_PASS:superadmin}
  image-service:
    url: ${IMAGE_SERVICE_URL:http://localhost:3000}
    api-key: ${INTERNAL_API_KEY:dev-internal-key-change-in-prod}
    cdn-base-url: ${CDN_BASE_URL:http://localhost:9000}

server:
  port: ${PORT:8080}
  error:
    include-message: always
```

### Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `DB_URL` | PostgreSQL JDBC URL | `jdbc:postgresql://localhost:5432/yoobu` |
| `DB_USER` | DB username | `yoobu` |
| `DB_PASS` | DB password | `yoobu` |
| `CORS_ALLOWED_ORIGIN_PATTERNS` | Comma-separated origin patterns | localhost + Railway wildcard |
| `SUPERADMIN_USER` | Superadmin BasicAuth username | `superadmin` |
| `SUPERADMIN_PASS` | Superadmin BasicAuth password | `superadmin` |
| `IMAGE_SERVICE_URL` | External image upload service base URL | `http://localhost:3000` |
| `INTERNAL_API_KEY` | API key for image service | `dev-internal-key-change-in-prod` |
| `CDN_BASE_URL` | CDN base URL for serving images | `http://localhost:9000` |
| `PORT` | Server port | `8080` |
