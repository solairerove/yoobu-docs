# YOOBU_DOCS.md — Platform Documentation

Generated: 2026-03-30
Sources: `RND_API_STATE.md`, `RND_WEB_STATE.md`, `RND_UPLOAD_STATE.md` (all extracted from source code)

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [API Contract Reference](#3-api-contract-reference)
4. [Data Models](#4-data-models)
5. [Enums](#5-enums)
6. [Database](#6-database)
7. [Frontend Architecture](#7-frontend-architecture)
8. [Upload Service Architecture](#8-upload-service-architecture)
9. [Telegram Mini App Integration](#9-telegram-mini-app-integration)
10. [User Flows (End-to-End)](#10-user-flows-end-to-end)
11. [Configuration](#11-configuration)
12. [Inter-Service Communication](#12-inter-service-communication)
13. [Test Coverage](#13-test-coverage)
14. [Contract Mismatches](#14-contract-mismatches)
15. [Not Yet Implemented](#15-not-yet-implemented)

---

## 1. System Overview

Yoobu is a multi-tenant ordering platform that runs as a Telegram Mini App. Tenants (currently food delivery businesses) configure a storefront with a service catalog, and their customers browse, order, pay (via VietQR), and track orders — all inside Telegram. An admin panel (server-rendered HTML) lets tenant owners manage bookings and services. A superadmin panel manages tenants themselves.

### Stack

| Layer | Technology |
|---|---|
| Frontend | Angular 19 (standalone components), served via Caddy, runs inside Telegram WebView |
| Backend API | Java (Spring Boot), JPA/Hibernate, Flyway, Thymeleaf admin panels |
| Upload Service | Rust (axum), image processing (WebP conversion), Cloudflare R2 / MinIO storage |
| Database | PostgreSQL (single instance, shared by all tenants) |
| Hosting | Railway (all three services) |
| Auth (customers) | Telegram `initData` HMAC-SHA256 |
| Auth (admin) | HTTP Basic (per-tenant credentials in `tenant_config`) |
| Auth (superadmin) | HTTP Basic (env vars) |
| Auth (inter-service) | Bearer token (shared `INTERNAL_API_KEY`) |

### Service Map

| Service | Repository | Purpose |
|---|---|---|
| `yoobu-api` | Java backend | REST API, admin/superadmin HTML panels, DB owner, Telegram notifications |
| `yoobu-web` | Angular frontend | Telegram Mini App client, served as static files via Caddy |
| `yoobu-media` | Rust upload service | Image upload, processing (resize + WebP conversion), R2 storage |

---

## 2. Architecture

### 2.1 Service Topology

```
┌─────────────────────┐
│  Telegram WebView    │
│  (yoobu-web)         │
│  Angular 19 + Caddy  │
└──────────┬───────────┘
           │ HTTP (via Caddy /api/* reverse proxy)
           ▼
┌─────────────────────┐       HTTP (Railway private network)       ┌──────────────────┐
│  yoobu-api           │ ────────────────────────────────────────► │  yoobu-media      │
│  Spring Boot         │  POST /upload, DELETE /object              │  Rust (axum)      │
│  PostgreSQL          │  Bearer auth (INTERNAL_API_KEY)            │  Cloudflare R2    │
└─────────────────────┘                                            └──────────────────┘
```

- Frontend → Java backend: all API calls go through Caddy reverse proxy (`/api/*` → strips prefix → forwards to `yoobu-api`)
- Java backend → Rust upload service: internal HTTP on Railway private network (`yoobu-media.railway.internal:3000`)
- Frontend → Rust upload service: **no direct communication**; frontend never calls the upload service
- Database: owned exclusively by `yoobu-api`; `yoobu-media` has no database
- Object storage (R2): owned by `yoobu-media`; Java backend receives URLs back and persists them in PostgreSQL

### 2.2 Multi-Tenancy Model

Tenant isolation is enforced at every layer:

| Layer | Mechanism |
|---|---|
| URL routing | All tenant-scoped paths include `{slug}`: `/t/{slug}/...`, `/admin/{slug}/...` |
| Filter chain | `TenantContextFilter` extracts slug from path, loads `Tenant` entity, stores in `TenantContext` (ThreadLocal) |
| Repository | All queries filter by `tenant_id` from `TenantContext.get()` |
| Database | Single schema, `tenant_id` column on `service`, `booking`, `booking_item`, `tenant_config` |
| Upload service | `X-Tenant-Id` header from Java backend becomes the top-level prefix in object key; no cross-tenant enforcement in Rust service (trusts the caller) |
| Admin credentials | Stored per-tenant in `tenant_config` (`admin_username`, `admin_password` as BCrypt hash) |

### 2.3 Security Model

**Four filter chains (Java backend, ordered):**

| Order | Matcher | Mechanism |
|---|---|---|
| 1 | `/superadmin/**` | `SuperAdminBasicAuthenticationFilter` — plain-text comparison against `app.superadmin.username/password` env vars |
| 2 | `/admin/*/**` | `TenantContextFilter` + `TenantBasicAuthenticationFilter` — BCrypt-verified against `tenant_config`; superadmin creds also accepted |
| 3 | `/t/*/**` | `TenantContextFilter` only — all requests permitted; Telegram auth handled per-endpoint |
| 4 | `/**` (default) | None — covers `/health` and static paths |

CSRF: disabled on all chains. CORS: enabled on all chains.

**Telegram auth (per-endpoint, not filter-chain level):**

- Header: `X-Telegram-Init-Data`
- Validation: HMAC-SHA256 using `tenant.bot_token` as secret key (Telegram WebApp algorithm)
- Handled by: `TelegramUserArgumentResolver` resolving `@TelegramPrincipal TelegramUser`
- Returns 401 if header is missing or signature invalid
- Dev profile fallback: `X-Telegram-User-Id` header (plain user ID, no signature) accepted when Spring profile `dev` is active

**Inter-service auth (Java → Rust):**

- `Authorization: Bearer {INTERNAL_API_KEY}` — constant-time compared in Rust service via `subtle` crate

### 2.4 Data Flow

| Concern | Owner |
|---|---|
| Tenant CRUD | Java backend (superadmin API + panel) |
| Service catalog CRUD | Java backend (admin API + panel) |
| Booking lifecycle | Java backend (client API for create/cancel/confirm-payment; admin API for status transitions) |
| Image storage | Rust upload service (receives file from Java backend, stores in R2, returns URL) |
| Image URL persistence | Java backend (stores URL from upload response in `service.image_url` or `tenant_config.payment_qr_url`) |
| Audit logging | Java backend (`audit_log` table, written by service layer) |
| Telegram notifications | Java backend (async, after commit, via `TelegramBotApiClient`) |
| Theme/branding | Java backend stores config; frontend reads via `/t/{slug}/config` and applies CSS |

---

## 3. API Contract Reference

### 3.1 Client API — Java Backend (`/t/{slug}/...`)

| Method | Path | Request Body | Response Body | Auth | Called By | Notes |
|---|---|---|---|---|---|---|
| GET | `/t/{slug}/config` | — | `TenantConfigResponse` | none | Frontend | Branding, hints, cutoff, earliest delivery date |
| GET | `/t/{slug}/services` | — | `List<ServiceResponse>` | none | Frontend | ACTIVE services only |
| POST | `/t/{slug}/bookings` | `CreateBookingRequest` | `BookingResponse` | Telegram initData | Frontend | FOOD_ORDER tenants only |
| GET | `/t/{slug}/bookings/my` | — | `List<BookingResponse>` | Telegram initData | Frontend | Filtered by telegramUserId |
| GET | `/t/{slug}/bookings/{bookingId}` | — | `BookingResponse` | Telegram initData | Frontend | 404 if not owned by user |
| POST | `/t/{slug}/bookings/{bookingId}/cancel` | — | `BookingResponse` | Telegram initData | Frontend | Customer-initiated cancel |
| POST | `/t/{slug}/bookings/{bookingId}/confirm-payment` | — | `BookingResponse` | Telegram initData | Frontend | Triggers PaymentConfirmedEvent |

### 3.2 Admin API — Java Backend (`/admin/{slug}/...`)

#### REST

| Method | Path | Request Body | Response Body | Auth | Notes |
|---|---|---|---|---|---|
| GET | `/admin/{slug}/bookings` | — | `List<BookingResponse>` | BasicAuth | Filter: `?status=`, `?deliveryDate=` |
| GET | `/admin/{slug}/bookings/{bookingId}` | — | `BookingResponse` | BasicAuth | |
| PUT | `/admin/{slug}/bookings/{bookingId}/status` | `UpdateStatusRequest` | `BookingResponse` | BasicAuth | State machine enforced |
| GET | `/admin/{slug}/services` | — | `List<ServiceResponse>` | BasicAuth | All statuses |
| POST | `/admin/{slug}/services` | `AdminUpsertServiceRequest` | `ServiceResponse` | BasicAuth | 201 Created |
| PUT | `/admin/{slug}/services/{serviceId}` | `AdminUpsertServiceRequest` | `ServiceResponse` | BasicAuth | |
| POST | `/admin/{slug}/services/{serviceId}/image` | `multipart/form-data` (`file`) | `ServiceResponse` | BasicAuth | Delegates to `ImageServiceClient` → Rust upload service |
| DELETE | `/admin/{slug}/services/{serviceId}` | — | — | BasicAuth | 204; soft-delete |

#### HTML Panel (Thymeleaf)

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/admin/{slug}/panel` | BasicAuth | Dashboard / home redirect |
| GET | `/admin/{slug}/panel/bookings` | BasicAuth | Paginated list; filter by status, deliveryDate |
| GET | `/admin/{slug}/panel/bookings/{bookingId}` | BasicAuth | Booking detail |
| POST | `/admin/{slug}/panel/bookings/{bookingId}/status` | BasicAuth | Form-based status update |
| GET | `/admin/{slug}/panel/services` | BasicAuth | Paginated service list; search by query |
| GET | `/admin/{slug}/panel/services/new` | BasicAuth | New service form |
| POST | `/admin/{slug}/panel/services` | BasicAuth | Create via HTML form |
| GET | `/admin/{slug}/panel/services/{serviceId}/edit` | BasicAuth | Edit form |
| POST | `/admin/{slug}/panel/services/{serviceId}` | BasicAuth | Update via HTML form |
| POST | `/admin/{slug}/panel/services/{serviceId}/delete` | BasicAuth | Requires `confirmName` param |
| POST | `/admin/{slug}/panel/services/{serviceId}/status` | BasicAuth | Toggle ACTIVE/INACTIVE |

### 3.3 Superadmin API — Java Backend (`/superadmin/...`)

#### REST

| Method | Path | Request Body | Response Body | Auth | Notes |
|---|---|---|---|---|---|
| GET | `/superadmin/tenants` | — | `List<TenantSummaryResponse>` | BasicAuth (superadmin) | |
| GET | `/superadmin/tenants/{tenantId}` | — | `TenantDetailResponse` | BasicAuth (superadmin) | Includes config map |
| GET | `/superadmin/tenants/slug-availability` | — | `TenantSlugAvailabilityResponse` | BasicAuth (superadmin) | `?slug=` query param |
| POST | `/superadmin/tenants` | `CreateTenantRequest` | `TenantSummaryResponse` | BasicAuth (superadmin) | |
| PUT | `/superadmin/tenants/{tenantId}` | `UpdateTenantRequest` | `TenantSummaryResponse` | BasicAuth (superadmin) | |
| GET | `/superadmin/audit` | — | `AuditLogPageResponse` | BasicAuth (superadmin) | Multiple filter params; default page=0, size=20 |
| GET | `/superadmin/audit/export` | — | `text/csv` bytes | BasicAuth (superadmin) | Same filters; default size=5000 |

#### HTML Panel (Thymeleaf)

| Method | Path | Auth | Notes |
|---|---|---|---|
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

### 3.4 Upload API — Rust Service (`yoobu-media`)

| Method | Path | Auth | Request | Response | Called By |
|---|---|---|---|---|---|
| GET | `/health` | none | — | `{"status":"ok"}` | Railway healthcheck |
| POST | `/upload` | Bearer (`INTERNAL_API_KEY`) | `multipart/form-data` (`file`), headers: `X-Tenant-Id`, `X-Upload-Path` | `{"url":"<cdn_url>"}` (200) | Java backend (`ImageServiceClient`) |
| DELETE | `/object` | Bearer (`INTERNAL_API_KEY`) | header: `X-Object-Key` | 204 empty | Java backend |

### 3.5 Health Endpoints

| Service | Path | Response |
|---|---|---|
| Java backend | `/health` | `{"status":"ok"}` |
| Rust upload service | `/health` | `{"status":"ok"}` |

---

## 4. Data Models

### 4.1 API Contracts — Frontend ↔ Java Backend

#### `TenantConfigResponse` / `TenantConfig`

| Field | Java Type | TypeScript Type | Required (Java) | Required (TS) | Notes |
|---|---|---|---|---|---|
| slug | String | string | yes | yes | |
| name | String | string | yes | yes | |
| type | TenantType | TenantType | yes | yes | |
| currency | String | string \| null | yes | optional (`?`) | |
| primaryColor | String | string \| null | nullable | nullable | Default `#ff6b35` in frontend |
| logoUrl | String | string \| null | nullable | nullable | Loaded but not rendered in any template |
| welcomeMessage | String | string \| null | nullable | nullable | |
| checkoutNameHint | String | string \| null | nullable | optional (`?`) | |
| checkoutPhoneHint | String | string \| null | nullable | optional (`?`) | |
| checkoutNoteHint | String | string \| null | nullable | optional (`?`) | |
| checkoutDeliveryHint | String | string \| null | nullable | optional (`?`) | |
| paymentQrUrl | String | string \| null | nullable | optional (`?`) | VietQR or static |
| cutoffHour | Integer | number \| null | nullable | optional (`?`) | |
| cutoffMinute | Integer | number \| null | nullable | optional (`?`) | |
| earliestDeliveryDate | LocalDate | string \| null | computed | optional (`?`) | Java: `LocalDate`; TS: ISO date string |

#### `ServiceResponse` / `ServiceItem`

| Field | Java Type | TypeScript Type | Required (Java) | Required (TS) | Notes |
|---|---|---|---|---|---|
| id | Long | number | yes | yes | |
| name | String | string | yes | yes | |
| description | String | string \| null | nullable | nullable | |
| price | BigDecimal | number | nullable | yes | **MISMATCH**: Java nullable, TS required |
| unit | String | string \| null | nullable | nullable | |
| durationMinutes | Integer | number \| null | nullable | nullable | Loaded but not displayed |
| sortOrder | int | number | yes | yes | |
| status | ServiceStatus | string literal | yes | yes | |
| imageUrl | String | string \| null | nullable | nullable | |

#### `CreateBookingRequest`

| Field | Java Type | TypeScript Type | Validation (Java) | Notes |
|---|---|---|---|---|
| customerName | String | string | `@NotBlank` | |
| customerPhone | String | string | `@NotBlank` | |
| deliveryAddress | String | string | `@NotBlank` | |
| deliveryDate | LocalDate | string | `@NotNull`, >= earliest | Java: `LocalDate`; TS: ISO date string |
| note | String | string \| null | optional | |
| items | `List<BookingItemRequest>` | `CreateBookingItem[]` | `@NotEmpty`, each `@Valid` | |

#### `BookingItemRequest` / `CreateBookingItem`

| Field | Java Type | TypeScript Type | Validation (Java) | Notes |
|---|---|---|---|---|
| serviceId | Long | number | `@NotNull` | |
| quantity | int | number | `@Min(1)` | |

#### `BookingResponse`

| Field | Java Type | TypeScript Type | Required (Java) | Required (TS) | Notes |
|---|---|---|---|---|---|
| id | Long | number | yes | yes | |
| type | BookingType | string | yes | yes | TS allows unknown string values |
| status | BookingStatus | BookingStatus | yes | yes | |
| trackingUrl | String | string \| null | nullable | nullable | |
| customerName | String | string | yes | yes | |
| customerPhone | String | string | yes | yes | |
| deliveryAddress | String | string \| null | nullable | nullable | |
| totalPrice | BigDecimal | number | nullable | yes | **MISMATCH**: Java nullable, TS required |
| currency | String | string | yes | yes | |
| deliveryDate | LocalDate | string | nullable | yes | Java: `LocalDate`; TS: ISO date string |
| note | String | string \| null | nullable | nullable | |
| items | `List<BookingItemResponse>` | `BookingItem[]` | yes | yes | |
| createdAt | OffsetDateTime | string | yes | yes | Java: `OffsetDateTime`; TS: ISO datetime string |
| paymentQrUrl | String | string \| null | nullable | nullable | |

#### `BookingItemResponse` / `BookingItem`

| Field | Java Type | TypeScript Type | Notes |
|---|---|---|---|
| serviceName | String | string | Snapshot at booking time |
| quantity | int | number | |
| unitPrice | BigDecimal | number | Snapshot at booking time |

Note: The TypeScript `BookingItem` interface includes a `currency` field that is **not present** in Java `BookingItemResponse`. See §14 Contract Mismatches.

#### `UpdateStatusRequest` (Admin only — no frontend equivalent)

| Field | Java Type | Validation | Notes |
|---|---|---|---|
| status | BookingStatus | `@NotNull` | State machine validated in service |
| trackingUrl | String | `@Size(max=2048)` | Optional; validated for valid http/https URL |

### 4.2 Java Backend → Rust Upload Service

Communication is via HTTP headers and multipart form data, not JSON DTOs.

**Upload request (Java → Rust):**

| Aspect | Java sends | Rust expects |
|---|---|---|
| Auth | `Authorization: Bearer {INTERNAL_API_KEY}` | `Authorization` header, constant-time compared |
| Tenant | `X-Tenant-Id: {tenant.id}` | `X-Tenant-Id` header (required) |
| Path hint | `X-Upload-Path: services/{serviceId}` or `payment/qr` | `X-Upload-Path` header (required) |
| File | `multipart/form-data`, field name `file` | Multipart field `file` |

**Upload response (Rust → Java):**

| Status | Body | Java handling |
|---|---|---|
| 200 | `{"url":"<cdn_url>"}` | Persists URL to entity field (`service.image_url` or `tenant_config`) |
| 4xx/5xx | `{"error":"<message>"}` | Propagated as error to caller |

**Delete request (Java → Rust):**

| Aspect | Java sends | Rust expects |
|---|---|---|
| Auth | `Authorization: Bearer {INTERNAL_API_KEY}` | `Authorization` header |
| Key | `X-Object-Key: {object_key}` | `X-Object-Key` header (required) |

### 4.3 Backend-Only Models (Java)

#### `AdminUpsertServiceRequest`

| Field | Type | Validation | Notes |
|---|---|---|---|
| name | String | `@NotBlank` | |
| description | String | — | Optional |
| price | BigDecimal | `@NotNull` | |
| unit | String | — | Optional |
| durationMinutes | Integer | — | Optional |
| sortOrder | Integer | — | Optional |
| status | ServiceStatus | — | Defaults to ACTIVE |

#### `CreateTenantRequest`

| Field | Type | Validation | Notes |
|---|---|---|---|
| slug | String | `@NotBlank` | Unique |
| name | String | `@NotBlank` | |
| type | TenantType | `@NotNull` | |
| botToken | String | — | Optional; required for Telegram notifications |
| ownerTelegramId | Long | — | Optional; required for admin notifications |
| timezone | String | — | Defaults to `Asia/Ho_Chi_Minh` |
| currency | String | — | Defaults to `USD` |
| primaryColor | String | — | |
| logoUrl | String | — | |
| welcomeMessage | String | — | |
| checkoutNameHint | String | — | |
| checkoutPhoneHint | String | — | |
| checkoutNoteHint | String | — | |
| checkoutDeliveryHint | String | — | |
| paymentQrUrl | String | — | Static fallback QR |
| paymentBankBin | String | — | VietQR dynamic generation |
| paymentAccountNumber | String | — | VietQR dynamic generation |
| adminUsername | String | `@NotBlank` | |
| adminPassword | String | `@NotBlank` | Stored as BCrypt |
| cutoffHour | Integer | — | |
| cutoffMinute | Integer | — | |

#### `UpdateTenantRequest`

Same as `CreateTenantRequest` except: `adminPassword` is optional on update; `active` (Boolean, `@NotNull`) is added.

#### `TenantSummaryResponse`

| Field | Type | Notes |
|---|---|---|
| id | Long | |
| slug | String | |
| name | String | |
| type | TenantType | |
| active | boolean | |
| timezone | String | |
| createdAt | OffsetDateTime | |

#### `TenantDetailResponse`

| Field | Type | Notes |
|---|---|---|
| id | Long | |
| slug | String | |
| name | String | |
| type | TenantType | |
| active | boolean | |
| timezone | String | |
| botToken | String | Nullable |
| ownerTelegramId | Long | Nullable |
| createdAt | OffsetDateTime | |
| config | `Map<String, String>` | All `tenant_config` key-value pairs |

#### `TenantSlugAvailabilityResponse`

| Field | Type | Notes |
|---|---|---|
| slug | String | Requested slug |
| available | boolean | |

#### `AuditLogPageResponse`

| Field | Type |
|---|---|
| items | `List<AuditLogItemResponse>` |
| page | int |
| size | int |
| totalElements | long |
| totalPages | int |
| hasNext | boolean |
| hasPrevious | boolean |

#### `AuditLogItemResponse`

| Field | Type | Notes |
|---|---|---|
| id | Long | |
| tenantId | Long | |
| entity | String | e.g. "Booking", "CatalogService" |
| entityId | Long | |
| action | String | e.g. "CREATE", "UPDATE", "ACTION" |
| actorId | String | Telegram user ID or "system" |
| oldValue | Object | JSON snapshot (nullable) |
| newValue | Object | JSON snapshot (nullable) |
| createdAt | OffsetDateTime | |

### 4.4 Upload Service Models (Rust)

No named request/response structs. All responses are inline `serde_json::json!` macro:

```rust
Json(json!({ "url": url }))        // POST /upload 200
Json(json!({ "status": "ok" }))    // GET /health 200
Json(json!({ "error": message }))  // all error responses
```

Error type: `AppError` enum (`Unauthorized`, `BadRequest(String)`, `StorageError`, `ProcessingError(String)`).

### 4.5 Schema vs Entity Drift (Java)

| Issue | Details |
|---|---|
| `booking.service_id` | Present in DB (V1 migration) and JPA entity (`@ManyToOne CatalogService service`), but unused in any booking creation or response logic. Items are in `booking_item`. Leftover from earlier design. |
| `booking_item.currency` | Added in V6, made nullable in V10. No longer source of truth — replaced by `booking.currency`. Entity field may still be present. |

No other drift detected.

---

## 5. Enums

| Enum | Values | Backend (Java) | Frontend (TypeScript) | Upload Service (Rust) | Mismatches |
|---|---|---|---|---|---|
| `TenantType` | `FOOD_ORDER`, `APPOINTMENT`, `CATALOG_REQUEST` | Entity + DTO | `TenantType` type alias | — | None |
| `BookingType` | `ORDER`, `APPOINTMENT`, `REQUEST` | Entity + DTO | `BookingResponse.type` (string union + catch-all `string`) | — | None; only `ORDER` actively used |
| `BookingStatus` | `NEW`, `PAYMENT_PENDING`, `CONFIRMED`, `DELIVERING`, `DONE`, `CANCELLED` | Entity + DTO + state machine | `BookingStatus` type alias; hardcoded in component logic | — | None |
| `ServiceStatus` | `ACTIVE`, `INACTIVE`, `DELETED` | Entity + DTO | `ServiceItem.status` (string literal union) | — | None |

**Booking status transitions** (enforced in `BookingService.getAllowedAdminStatuses()`):

| From | Allowed To |
|---|---|
| NEW | NEW, CANCELLED |
| PAYMENT_PENDING | PAYMENT_PENDING, CONFIRMED, CANCELLED |
| CONFIRMED | CONFIRMED, DELIVERING, CANCELLED |
| DELIVERING | DELIVERING, DONE, CANCELLED |
| DONE | DONE, CANCELLED |
| CANCELLED | CANCELLED |

---

## 6. Database

### 6.1 Schema (reconstructed from Flyway V1–V10)

#### `tenant`

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| id | BIGSERIAL | no | — | PK |
| slug | VARCHAR(100) | no | — | UNIQUE |
| name | VARCHAR(255) | no | — | |
| type | VARCHAR(30) | no | — | FOOD_ORDER, APPOINTMENT, CATALOG_REQUEST |
| bot_token | VARCHAR(255) | yes | — | |
| owner_telegram_id | BIGINT | yes | — | |
| timezone | VARCHAR(50) | no | `'Asia/Ho_Chi_Minh'` | |
| active | BOOLEAN | no | TRUE | |
| created_at | TIMESTAMPTZ | no | — | |

#### `tenant_config`

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| id | BIGSERIAL | no | — | PK |
| tenant_id | BIGINT | no | — | FK → tenant.id |
| key | VARCHAR(100) | no | — | |
| value | TEXT | yes | — | |

Unique constraint: `(tenant_id, key)`

Known keys: `admin_username`, `admin_password`, `primary_color`, `logo_url`, `welcome_message`, `checkout_name_hint`, `checkout_phone_hint`, `checkout_note_hint`, `checkout_delivery_hint`, `payment_qr_url`, `payment_bank_bin`, `payment_account_number`, `currency`, `cutoff_hour`, `cutoff_minute`

#### `service`

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| id | BIGSERIAL | no | — | PK |
| tenant_id | BIGINT | no | — | FK → tenant.id |
| name | VARCHAR(255) | no | — | |
| description | TEXT | yes | — | |
| price | NUMERIC(10,2) | yes | — | |
| unit | VARCHAR(50) | yes | `'шт'` | |
| duration_minutes | INT | yes | — | |
| status | VARCHAR(20) | no | `'ACTIVE'` | ACTIVE, INACTIVE, DELETED |
| sort_order | INT | no | 0 | |
| image_url | TEXT | yes | — | Added V9 |
| deleted_at | TIMESTAMPTZ | yes | — | Soft delete |
| created_at | TIMESTAMPTZ | no | — | |
| updated_at | TIMESTAMPTZ | no | — | |

Index: `idx_service_tenant` on `(tenant_id, status, sort_order, id)` WHERE `status <> 'DELETED'`

#### `booking`

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| id | BIGSERIAL | no | — | PK |
| tenant_id | BIGINT | no | — | FK → tenant.id |
| type | VARCHAR(20) | no | — | ORDER, APPOINTMENT, REQUEST |
| telegram_user_id | BIGINT | no | — | |
| customer_name | VARCHAR(255) | no | — | |
| customer_phone | VARCHAR(50) | yes | — | |
| delivery_address | TEXT | yes | — | Added V7 |
| status | VARCHAR(20) | no | — | NEW, PAYMENT_PENDING, CONFIRMED, DELIVERING, DONE, CANCELLED |
| tracking_url | TEXT | yes | — | Added V8 |
| note | TEXT | yes | — | |
| total_price | NUMERIC(10,2) | yes | — | |
| currency | VARCHAR(10) | no | — | Added V10 |
| payment_qr_url | TEXT | yes | — | Added V10 |
| delivery_date | DATE | yes | — | |
| slot_id | BIGINT | yes | — | Reserved, unused |
| service_id | BIGINT | yes | — | FK → service.id; deprecated, see drift notes |
| deleted_at | TIMESTAMPTZ | yes | — | |
| created_at | TIMESTAMPTZ | no | — | |
| updated_at | TIMESTAMPTZ | no | — | |
| version | BIGINT | no | 0 | Optimistic locking (V5) |

Indexes: `idx_booking_tenant`, `idx_booking_tenant_date`, `idx_booking_tenant_status`, `idx_booking_telegram_user`

#### `booking_item`

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| id | BIGSERIAL | no | — | PK |
| booking_id | BIGINT | no | — | FK → booking.id ON DELETE CASCADE |
| service_id | BIGINT | no | — | FK → service.id |
| quantity | INT | no | — | CHECK (quantity > 0) |
| unit_price | NUMERIC(10,2) | no | — | Snapshot at booking time |

#### `audit_log`

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| id | BIGSERIAL | no | — | PK |
| tenant_id | BIGINT | no | — | Not FK; denormalized |
| entity | VARCHAR(50) | no | — | |
| entity_id | BIGINT | no | — | |
| action | VARCHAR(50) | no | — | |
| actor_id | VARCHAR(255) | yes | — | Altered VARCHAR in V2 |
| old_value | TEXT | yes | — | JSON snapshot |
| new_value | TEXT | yes | — | JSON snapshot |
| created_at | TIMESTAMPTZ | no | — | |

Indexes: `idx_audit_tenant_entity`, `idx_audit_created_at_id`, `idx_audit_tenant_created_at_id`, `idx_audit_action_created_at_id`

### 6.2 Migration History

| Version | Filename | Summary |
|---|---|---|
| V1 | `V1__baseline_schema.sql` | Creates all tables and initial indexes |
| V2 | `V2__audit_actor_id_as_string.sql` | `audit_log.actor_id` BIGINT → VARCHAR(255) |
| V3 | `V3__service_status_model.sql` | Adds `service.status`; drops `service.active`; updates index |
| V4 | `V4__audit_log_filter_indexes.sql` | Adds audit log filter indexes |
| V5 | `V5__booking_optimistic_lock.sql` | Adds `booking.version` |
| V6 | `V6__booking_item_currency.sql` | Adds `booking_item.currency`; backfills from tenant_config |
| V7 | `V7__booking_delivery_address.sql` | Adds `booking.delivery_address` |
| V8 | `V8__booking_tracking_url.sql` | Adds `booking.tracking_url` |
| V9 | `V9__service_image_url.sql` | Adds `service.image_url` |
| V10 | `V10__booking_currency_and_payment_qr_url.sql` | Adds `booking.currency`, `booking.payment_qr_url`; makes `booking_item.currency` nullable |

### 6.3 Unused Schema Elements

| Element | Status |
|---|---|
| `booking.slot_id` | Reserved, never written or read |
| `booking.service_id` | Deprecated; items stored in `booking_item` |
| `booking_item.currency` | Superseded by `booking.currency` (V10) |

### 6.4 Upload Service Database

None. `yoobu-media` has no database dependency.

---

## 7. Frontend Architecture

### 7.1 Framework and Bootstrap

Angular 19 standalone components. No NgModules. Providers registered in `app.config.ts`:

```typescript
providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(appRoutes),
    provideHttpClient(withInterceptors([telegramInitDataInterceptor]))
]
```

### 7.2 Route Table

| Path | Target | Notes |
|---|---|---|
| `` (empty) | redirect → `t/demo` | |
| `t/:slug` | `TenantShellComponent` (lazy) | Entry point for all tenant traffic |
| `**` | redirect → `t/demo` | |

No child routes. `TenantShellComponent` uses `NgComponentOutlet` to dynamically load:

| Condition | Loaded Component |
|---|---|
| `config.type === 'FOOD_ORDER'` | `FoodOrderHomeComponent` |
| Any other type | `UnsupportedFlowComponent` |

### 7.3 Component Inventory

| Component | Selector | Change Detection | Purpose |
|---|---|---|---|
| `AppComponent` | `app-root` | Default | Router outlet |
| `TenantShellComponent` | `app-tenant-shell` | Default | Tenant resolution, theming, feature routing |
| `FoodOrderHomeComponent` | `app-food-order-home` | Default | Orchestrator; provides `FoodOrderFlowFacade` |
| `FoodOrderMenuComponent` | `app-food-order-menu` | OnPush | 2-column service grid with quantity steppers |
| `FoodOrderCartBarComponent` | `app-food-order-cart-bar` | OnPush | Fixed bottom bar (local/dev mode only) |
| `FoodOrderCheckoutComponent` | `app-food-order-checkout` | OnPush | Checkout form + order review (bottom sheet in Telegram, card in local mode) |
| `FoodOrderBookingsComponent` | `app-food-order-bookings` | OnPush | My orders list + detail panel with timeline |
| `FoodOrderSuccessCardComponent` | `app-food-order-success-card` | OnPush | Post-order confirmation with QR and actions |
| `UnsupportedFlowComponent` | `app-unsupported-flow` | Default | Static "not available" placeholder |

### 7.4 Services

#### `TenantApiService` (root)

Stateless HTTP service. All calls use `withRetry` (timeout 10s, 2 retries, exponential backoff 0/2s/4s).

| Method | HTTP | Path |
|---|---|---|
| `getConfig(slug)` | GET | `/api/t/:slug/config` |
| `getServices(slug)` | GET | `/api/t/:slug/services` |
| `createBooking(slug, request)` | POST | `/api/t/:slug/bookings` |
| `getMyBookings(slug)` | GET | `/api/t/:slug/bookings/my` |
| `getBooking(slug, bookingId)` | GET | `/api/t/:slug/bookings/:bookingId` |
| `confirmBookingPayment(slug, bookingId)` | POST | `/api/t/:slug/bookings/:bookingId/confirm-payment` |
| `cancelBooking(slug, bookingId)` | POST | `/api/t/:slug/bookings/:bookingId/cancel` |

#### `TelegramService` (root)

Wraps `window.Telegram.WebApp`. See §9 for full integration details.

#### `FoodOrderStore` (root)

Signal-based cart state. `MAX_ITEM_QUANTITY` = 9.

| Signal | Type | Description |
|---|---|---|
| `services` | `ServiceItem[]` | Current catalog |
| `quantities` | `Record<number, number>` | Cart quantities by serviceId |
| `selectedItems` | `Array<{service, quantity}>` | Services with quantity > 0 |
| `selectedCount` | `number` | Sum of quantities |
| `selectedTotal` | `number` | Sum of price × quantity |

#### `FoodOrderFlowFacade` (scoped to `FoodOrderHomeComponent`)

Orchestrates all business logic: checkout form, order submission, booking management, repeat order, Telegram MainButton state. See §10 User Flows for detailed behavior.

### 7.5 HTTP Interceptor

`telegramInitDataInterceptor`:

1. Localhost? → add `X-Telegram-User-Id: '101'`
2. URL contains `/bookings`? → poll for `initData` every 50ms up to 1500ms → add `X-Telegram-Init-Data` or throw error
3. Otherwise → try `getInitData()` once → add header if found, pass through if not

### 7.6 Theming

`TenantShellComponent.applyTheme()` sets `--yoobu-primary` CSS custom property on `:root` from `config.primaryColor` (default `#ff6b35`). All components consume it via `var(--yoobu-primary)`.

No CSS framework. Hand-written CSS. Font: `"SF Pro Display", "Avenir Next", "Segoe UI", sans-serif`. Max-width: 720px. Responsive breakpoints at 640px and 430px.

Telegram theme params (`tg.themeParams`) are **not used**. Theming is entirely tenant-driven.

### 7.7 Image Upload in Frontend

The frontend does not upload images. It **consumes** image URLs returned in `ServiceResponse.imageUrl` (rendered in `FoodOrderMenuComponent` as `<img>`) and `BookingResponse.paymentQrUrl` (rendered in checkout/bookings as QR code image). All uploads are initiated through the Java admin panel.

---

## 8. Upload Service Architecture

### 8.1 Image Processing Pipeline

```
Input (JPEG/PNG/WebP, ≤2MB)
  → Magic-byte format detection
  → image::load_from_memory (decode)
  → resize_if_needed (max 1200px on either axis, Lanczos3, no upscale)
  → webp::Encoder::from_image → encode(quality) (lossy WebP)
  → S3 PUT to R2/MinIO
  → Return CDN URL
```

| Property | Value |
|---|---|
| Output format | Always WebP |
| Quality | `WEBP_QUALITY` env (default 80) |
| Max dimension | `MAX_IMAGE_DIMENSION` env (default 1200px) |
| Max input size | `MAX_FILE_SIZE` env (default 2MB) |
| Allowed input | JPEG, PNG, WebP (by magic bytes) |
| Thumbnails | Not generated |
| Lossless path | Not used |

### 8.2 File Lifecycle

1. Admin uploads image via Thymeleaf panel (Java backend)
2. Java backend's `ImageServiceClient` forwards multipart to `POST /upload` on Rust service
3. Rust service processes image, stores WebP in R2, returns `{"url":"<cdn_url>"}`
4. Java backend persists URL in entity field (`service.image_url`)
5. Frontend fetches service catalog, renders `imageUrl` as `<img src>`
6. Image served directly from CDN (Cloudflare R2 public URL)

### 8.3 Object Key Format

```
{tenantId}/{uploadPath}-{unix_epoch_ms}.webp
```

Examples:
- Service image: `42/services/17-1719312000000.webp`
- Payment QR: `42/payment/qr-1719312000000.webp`

### 8.4 Storage

| Property | Value |
|---|---|
| Production | Cloudflare R2 |
| Local dev | MinIO (docker-compose) |
| SDK | `aws-sdk-s3` v1, `force_path_style(true)` |
| Content-Type | `image/webp` |
| Cache-Control | `public, max-age=31536000, immutable` |
| Retention | None (no TTL, no cleanup) |
| Collision handling | None (same ms + same tenant/path → silent overwrite) |

---

## 9. Telegram Mini App Integration

### 9.1 SDK Loading

`index.html`: `<script src="https://telegram.org/js/telegram-web-app.js" defer></script>`

### 9.2 Auth Flow (End-to-End)

```
Telegram client opens Mini App
  → WebView loads Angular app
  → TelegramService.init() reads window.Telegram.WebApp
  → tg.ready(), tg.expand() called
  → tg.initData available (signed by Telegram)

Angular makes API call (e.g., POST /bookings)
  → telegramInitDataInterceptor:
      - Localhost? → X-Telegram-User-Id: '101'
      - Production? → polls for initData → X-Telegram-Init-Data: {initData}

Java backend receives request
  → TelegramUserArgumentResolver reads X-Telegram-Init-Data header
  → TelegramInitDataValidator: HMAC-SHA256 using tenant.bot_token
  → Extracts TelegramUser (userId, etc.)
  → 401 if invalid

Dev mode:
  → Frontend (localhost): sends X-Telegram-User-Id: '101'
  → Backend (Spring profile 'dev'): accepts plain user ID, bypasses HMAC
```

### 9.3 InitData Resolution Chain

1. `webApp.initData` (trimmed)
2. URL params: `tgWebAppData` from query string or hash
3. localStorage cache (`tg_init_data_cache`)

### 9.4 SDK Methods Used

| Method | Where | Trigger |
|---|---|---|
| `ready()` | `TelegramService.init()` (via effect) | Each slug navigation |
| `expand()` | `TelegramService.init()` (via effect) | Each slug navigation |
| `MainButton.setText/show/hide/enable/disable` | `FoodOrderFlowFacade` effect | Cart/checkout state changes |
| `MainButton.onClick/offClick` | `TelegramService` effect | Handler changes |
| `showAlert(msg)` | Facade: submit error, payment error, repeat order warning | Error display |
| `showConfirm(msg)` | Facade: before submit, before cancel | User confirmation |
| `HapticFeedback.impactOccurred('light')` | `Facade.increase()` | Add to cart |
| `HapticFeedback.selectionChanged()` | `Facade.decrease()` | Remove from cart |

**Not used:** `BackButton`, `close()`, `themeParams`, `viewportHeight`, `viewportStableHeight`.

### 9.5 MainButton State Logic

| State | Label | Enabled | Handler |
|---|---|---|---|
| Cart empty, checkout closed | (hidden) | — | — |
| Cart has items, checkout closed | "Checkout (N items)" | yes | `openCheckout` |
| Checkout open, not submitting | "Place Order" | yes | `submitOrder` |
| Checkout open, submitting | "Placing order…" | no | — |

### 9.6 Telegram Notifications (Backend → User)

Sent by Java backend via `TelegramBotApiClient` (POST to `api.telegram.org`). Async after commit. 3 retries with exponential backoff (1s → 3s → 9s). Handles 429 with `retry_after`. HTML parse mode.

| Event | Recipient | Content |
|---|---|---|
| Booking created | `tenant.owner_telegram_id` (admin) | Order summary: items, phone, address, delivery date |
| Payment confirmed | `tenant.owner_telegram_id` (admin) | "Payment confirmed" |
| Status → CONFIRMED | `booking.telegram_user_id` (customer) | Status update |
| Status → DELIVERING | `booking.telegram_user_id` (customer) | Status update + tracking URL if set |
| Status → DONE | `booking.telegram_user_id` (customer) | Status update |
| Status → CANCELLED | `booking.telegram_user_id` (customer) | Status update |

---

## 10. User Flows (End-to-End)

### 10.1 App Init / Tenant Resolution

```
User opens Telegram Mini App (URL: /t/{slug})
  → Angular router: t/:slug → lazy-loads TenantShellComponent
  → TelegramService.init(): sets webAppSignal, calls ready() + expand()
  → TelegramService.setMainButton(null): hides main button
  → TenantApiService.getConfig(slug): GET /api/t/{slug}/config
    → Backend: TenantContextFilter resolves slug → loads Tenant → 404 if not found/inactive
    → Backend: returns TenantConfigResponse (branding, hints, cutoff, computed earliestDeliveryDate)
  → Frontend: applyTheme() → sets --yoobu-primary CSS var
  → resolveFeatureComponent(config.type):
    → FOOD_ORDER → lazy-loads FoodOrderHomeComponent
    → else → UnsupportedFlowComponent
  → Renders via NgComponentOutlet
```

### 10.2 Browse Catalog

```
FoodOrderHomeComponent created
  → effect: facade.setConfig(config)
  → store.setTenant(slug): clears cart if slug changed
  → TenantApiService.getServices(slug): GET /api/t/{slug}/services
    → Backend: returns ACTIVE services only, ordered by sort_order
  → store.setServices(services)
  → FoodOrderMenuComponent renders 2-column grid of service cards
    → Each card: image (or initial placeholder), name, description, unit, price, quantity stepper
```

### 10.3 Add to Cart

```
User taps "+" on service card
  → FoodOrderMenuComponent emits increaseRequested(serviceId)
  → facade.increase(serviceId)
    → store.increase(serviceId): quantity++ (capped at 9)
    → telegram.hapticLight(): HapticFeedback.impactOccurred('light')
  → MainButton effect: setMainButton("Checkout (N items)", enabled)
  → onMainButtonClick → openCheckout
```

### 10.4 Checkout + Submit Order

```
User taps Checkout (MainButton or local cart bar)
  → facade.openCheckout()
  → MainButton: "Place Order", enabled, onClick → submitOrder
  → FoodOrderCheckoutComponent renders:
    → Form: customerName, customerPhone, deliveryAddress, deliveryDate (≥ earliestDeliveryDate), note
    → Order review card: selected items with qty × unitPrice, total
    → Cutoff hint if earliestDeliveryDate > today

User taps Place Order (MainButton or local submit button)
  → facade.submitOrder()
  → Validates: required fields, phone pattern, deliveryDate ≥ earliest
  → telegram.confirm("Confirm your order?")
  → MainButton: "Placing order…", disabled
  → TenantApiService.createBooking(slug, request): POST /api/t/{slug}/bookings
    → Interceptor: polls for initData (up to 1500ms) → X-Telegram-Init-Data header
    → Backend: validates Telegram auth, validates all fields, checks services exist + ACTIVE + priced
    → Backend: creates booking + booking_items, computes totalPrice, builds paymentQrUrl
    → Backend: fires BookingCreatedEvent → async Telegram notification to admin
    → Backend: returns BookingResponse
  → Frontend: submittedBooking = response, clearCart, hide MainButton
  → FoodOrderSuccessCardComponent renders: order ID, QR code, status, action buttons
```

### 10.5 Payment QR Logic

Computed in `BookingService.buildPaymentQrUrl()`:

1. If `payment_bank_bin` AND `payment_account_number` configured → dynamic VietQR: `https://img.vietqr.io/image/{bankBin}-{accountNumber}-compact2.png?amount={totalPrice}&addInfo=NUM-{bookingId}`
2. Else if `payment_qr_url` configured → static QR URL
3. Else → `null`

### 10.6 Confirm Payment

```
User taps "I paid" on success card or booking detail
  → facade.confirmPayment(bookingId)
  → TenantApiService.confirmBookingPayment: POST /api/t/{slug}/bookings/{id}/confirm-payment
    → Backend: triggers PaymentConfirmedEvent → async Telegram notification to admin
    → Backend: returns updated BookingResponse
  → Frontend: updates local state; handles 409 (already paid)
```

### 10.7 View My Bookings

```
User taps "My orders" tab
  → facade.setActiveView('orders')
  → facade.refreshBookings()
  → TenantApiService.getMyBookings(slug): GET /api/t/{slug}/bookings/my
    → Backend: filters by telegram_user_id from auth
  → FoodOrderBookingsComponent renders:
    → Current bookings (NEW, PAYMENT_PENDING, CONFIRMED, DELIVERING)
    → Previous bookings (DONE, CANCELLED)
    → First booking auto-selected as detail view
```

### 10.8 Booking Detail

```
User taps a booking card
  → facade.selectBooking(bookingId)
  → TenantApiService.getBooking(slug, bookingId): GET /api/t/{slug}/bookings/{id}
    → Backend: 404 if not owned by user
  → Detail panel renders:
    → Status timeline (Order placed → Payment → Confirmed → Delivering → Delivered)
    → Payment QR (if NEW or PAYMENT_PENDING and QR URL present)
    → Action buttons: Track delivery, Repeat, I paid, Cancel
    → Receipt: delivery date, contact, address, note, item list with prices
```

### 10.9 Cancel Booking

```
User taps "Cancel"
  → telegram.confirm("Cancel this order?")
  → TenantApiService.cancelBooking: POST /api/t/{slug}/bookings/{id}/cancel
    → Backend: customer-initiated cancel; returns updated BookingResponse
  → Frontend: updates local state, refreshes bookings list
```

### 10.10 Repeat Order

```
User taps "Repeat" on a previous booking
  → facade.repeatBooking(bookingId)
  → Loads full booking detail
  → Matches each booking item to current catalog by normalized name (lowercase, trim)
  → store.setQuantities(matchedQuantities)
  → checkoutForm.patchValue(customerName, customerPhone, deliveryAddress)
  → deliveryDate reset to earliest allowed
  → If some items not found: telegram.alert("Some items are no longer available...")
  → Opens checkout view
```

### 10.11 Image Upload (Admin Panel)

```
Admin opens Thymeleaf service edit form
  → Selects image file
  → POST /admin/{slug}/services/{serviceId}/image (multipart)
  → Java backend ImageServiceClient:
    → POST /upload to yoobu-media (internal network)
    → Headers: Authorization: Bearer {INTERNAL_API_KEY}, X-Tenant-Id: {tenantId}, X-Upload-Path: services/{serviceId}
    → Body: multipart file
  → Rust service:
    → Validates auth, headers
    → Reads file, validates magic bytes (JPEG/PNG/WebP)
    → Decodes → resizes (≤1200px) → encodes lossy WebP (quality 80)
    → PUT to R2: key = {tenantId}/services/{serviceId}-{epoch_ms}.webp
    → Returns {"url":"https://media.yoobu.app/{key}"}
  → Java backend:
    → Updates service.image_url = returned URL
    → Returns ServiceResponse
  → Frontend: next catalog fetch shows image via <img src="{imageUrl}">
```

### 10.12 Admin Panel Flows (Thymeleaf, Backend-Only)

**Manage bookings:** List bookings (filter by status, delivery date) → view detail → change status (state machine enforced) → Telegram notification sent to customer on CONFIRMED/DELIVERING/DONE/CANCELLED.

**Manage services:** List services (search by query) → create/edit (name, description, price, unit, sort order, status) → upload image → toggle active/inactive → soft-delete (requires name confirmation).

### 10.13 Superadmin Flows (Thymeleaf, Backend-Only)

**Manage tenants:** List tenants (search) → create (slug, name, type, credentials, config) → edit → upload payment QR → toggle active.

**Audit log:** Paginated list (filter by tenant, entity, action, date range) → CSV export.

---

## 11. Configuration

### 11.1 Java Backend

#### `application.yml`

```yaml
spring:
  application.name: yoobu-api
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/yoobu}
    username: ${DB_USER:yoobu}
    password: ${DB_PASS:yoobu}
  jpa:
    hibernate.ddl-auto: validate
    open-in-view: false
    properties.hibernate.jdbc.time_zone: UTC
  flyway.locations: classpath:db/migration

app:
  cors.allowed-origin-patterns: ${CORS_ALLOWED_ORIGIN_PATTERNS:http://localhost:*,...,https://*.up.railway.app}
  superadmin:
    username: ${SUPERADMIN_USER:superadmin}
    password: ${SUPERADMIN_PASS:superadmin}
  image-service:
    url: ${IMAGE_SERVICE_URL:http://localhost:3000}
    api-key: ${INTERNAL_API_KEY:dev-internal-key-change-in-prod}
    cdn-base-url: ${CDN_BASE_URL:http://localhost:9000}

server:
  port: ${PORT:8080}
  error.include-message: always
```

#### Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `DB_URL` | PostgreSQL JDBC URL | `jdbc:postgresql://localhost:5432/yoobu` |
| `DB_USER` | DB username | `yoobu` |
| `DB_PASS` | DB password | `yoobu` |
| `CORS_ALLOWED_ORIGIN_PATTERNS` | Comma-separated origin patterns | localhost + Railway wildcard |
| `SUPERADMIN_USER` | Superadmin BasicAuth username | `superadmin` |
| `SUPERADMIN_PASS` | Superadmin BasicAuth password | `superadmin` |
| `IMAGE_SERVICE_URL` | Rust upload service base URL | `http://localhost:3000` |
| `INTERNAL_API_KEY` | Shared secret for inter-service auth | `dev-internal-key-change-in-prod` |
| `CDN_BASE_URL` | CDN base URL for image URLs | `http://localhost:9000` |
| `PORT` | Server port | `8080` |

#### `tenant_config` Keys

| Key | Purpose | Read by |
|---|---|---|
| `admin_username` | Admin panel BasicAuth username | `TenantBasicAuthenticationFilter` |
| `admin_password` | Admin panel BasicAuth password (BCrypt) | `TenantBasicAuthenticationFilter` |
| `primary_color` | UI accent color | `TenantConfigResponse` → frontend |
| `logo_url` | Tenant logo URL | `TenantConfigResponse` → frontend (not rendered) |
| `welcome_message` | Storefront welcome text | `TenantConfigResponse` → frontend |
| `checkout_name_hint` | Placeholder for name field | `TenantConfigResponse` → frontend |
| `checkout_phone_hint` | Placeholder for phone field | `TenantConfigResponse` → frontend |
| `checkout_note_hint` | Placeholder for note field | `TenantConfigResponse` → frontend |
| `checkout_delivery_hint` | Placeholder for address field | `TenantConfigResponse` → frontend |
| `payment_qr_url` | Static payment QR image URL | `BookingService.buildPaymentQrUrl()` |
| `payment_bank_bin` | Bank BIN for VietQR dynamic generation | `BookingService.buildPaymentQrUrl()` |
| `payment_account_number` | Account number for VietQR | `BookingService.buildPaymentQrUrl()` |
| `currency` | Tenant currency code | `TenantConfigResponse`, `BookingService` |
| `cutoff_hour` | Delivery cutoff hour | `TenantConfigResponse` (earliestDeliveryDate computation) |
| `cutoff_minute` | Delivery cutoff minute | `TenantConfigResponse` (earliestDeliveryDate computation) |

### 11.2 Rust Upload Service

| Env var | Type | Default | Required |
|---|---|---|---|
| `INTERNAL_API_KEY` | String | — | Yes |
| `R2_ENDPOINT` | String | — | Yes |
| `R2_ACCESS_KEY` | String | — | Yes |
| `R2_SECRET_KEY` | String | — | Yes |
| `R2_BUCKET` | String | — | Yes |
| `R2_REGION` | String | `"auto"` | No |
| `CDN_BASE_URL` | String | — | Yes |
| `PORT` | u16 | `3000` | No |
| `MAX_FILE_SIZE` | usize | `2097152` (2MB) | No |
| `MAX_IMAGE_DIMENSION` | u32 | `1200` | No |
| `WEBP_QUALITY` | f32 | `80` | No |
| `RUST_LOG` | String | `image_service=debug,tower_http=debug` | No |

### 11.3 Frontend

No `environment.ts` files. API base URL hardcoded as `/api/t` in `TenantApiService`.

**`proxy.conf.json` (local dev):**

| Proxy path | Target | Path rewrite |
|---|---|---|
| `/api/t` | `http://localhost:8080` | `/api/t` → `/t` |
| `/api/admin` | `http://localhost:8080` | `/api/admin` → `/admin` |
| `/api/superadmin` | `http://localhost:8080` | `/api/superadmin` → `/superadmin` |

**Production (Caddyfile):**

```
:{$PORT}

handle /api/* {
    uri strip_prefix /api
    reverse_proxy {$BACKEND_URL}
}

handle {
    root * /srv
    try_files {path} /index.html
    file_server
}
```

### 11.4 Deployment (Railway)

| Service | Port | Healthcheck | Image |
|---|---|---|---|
| `yoobu-api` | `$PORT` (8080) | `/health` | Java (Spring Boot jar) |
| `yoobu-web` | `$PORT` (8080) | — | Caddy serving Angular build |
| `yoobu-media` | `$PORT` (3000) | `/health` | Debian slim running Rust binary |

---

## 12. Inter-Service Communication

### 12.1 HTTP Calls Between Services

| Caller | Callee | Method | Path | Auth | Timeout | Retry | Purpose |
|---|---|---|---|---|---|---|---|
| Java backend | Rust upload | POST | `/upload` | Bearer `INTERNAL_API_KEY` | Not explicitly configured in source | Not explicitly configured in source | Upload service/payment images |
| Java backend | Rust upload | DELETE | `/object` | Bearer `INTERNAL_API_KEY` | Not explicitly configured in source | Not explicitly configured in source | Delete old images |
| Java backend | Telegram API | POST | `/bot{token}/sendMessage` | Token in URL | Not specified | 3 retries, exponential backoff (1s→3s→9s), handles 429 | Notifications |

### 12.2 Shared Resources

| Resource | Shared Between | Isolation |
|---|---|---|
| PostgreSQL | Java backend only | N/A |
| Cloudflare R2 | Rust upload (writes), CDN (reads), Frontend (reads via CDN URL) | Tenant ID prefix in object key |
| `INTERNAL_API_KEY` | Java backend, Rust upload | Railway project-level env var |

### 12.3 Failure Modes

| Failure | Impact |
|---|---|
| Rust upload service down | Admin image upload fails; service returns error to admin panel. No impact on customer-facing flows (existing image URLs still served from CDN). |
| Rust upload service slow | Java backend blocks on HTTP call to upload service; admin panel request times out. |
| R2/CDN down | Images don't load in frontend (broken `<img>` tags). Ordering flow still works. |
| PostgreSQL down | Java backend returns 500 on all data-dependent endpoints. Frontend shows error states. |
| Telegram API down | Notifications fail silently (async, after commit). Customer ordering flow unaffected. Admin doesn't receive notification. |

---

## 13. Test Coverage

### 13.1 Frontend Tests

| Spec File | Covers | Telegram Mock | HTTP Mock |
|---|---|---|---|
| `telegram.service.spec.ts` | WebApp init, initData resolution chain, localStorage caching, MainButton state, click handlers, alert/confirm with version gating | `DOCUMENT` token + plain object with spies | N/A |
| `telegram-init-data.interceptor.spec.ts` | Header attachment (initData + dev fallback), pass-through, bookings timeout | `TelegramService` spy object | `HttpTestingController` |
| `food-order.store.spec.ts` | selectedCount/Total, increase/decrease/capping at 9, setQuantities, setServices cleanup, setTenant reset | N/A | N/A |
| `food-order-flow.facade.spec.ts` | earliestDeliveryDate computation (partial) | Spy object | `TenantApiService` spy |

**Gaps:** No component template tests. No end-to-end tests. Facade coverage is partial.

### 13.2 Backend Tests

Not enumerated in source document. Coverage details would require re-extraction.

### 13.3 Rust Upload Service Tests

| Location | Test | Type | Covers |
|---|---|---|---|
| `src/auth.rs` | `valid_token_passes` | Unit | Correct token |
| `src/auth.rs` | `wrong_token_fails` | Unit | Wrong token |
| `src/auth.rs` | `empty_token_fails` | Unit | Empty token |
| `src/auth.rs` | `different_length_fails` | Unit | Length mismatch |
| `src/auth.rs` | `prefix_match_fails` | Unit | Prefix-only match |
| `src/processing.rs` | `detects_jpeg` | Unit | JPEG magic bytes |
| `src/processing.rs` | `detects_png` | Unit | PNG magic bytes |
| `src/processing.rs` | `detects_webp` | Unit | WebP magic bytes |
| `src/processing.rs` | `rejects_unknown_format` | Unit | Unknown format |
| `src/processing.rs` | `rejects_too_small` | Unit | <12 byte input |
| `src/processing.rs` | `resize_not_triggered_when_fits` | Unit | No resize needed |
| `src/processing.rs` | `resize_scales_down_wide_image` | Unit | Downscale |
| `src/processing.rs` | `process_image_produces_webp_output` | Integration | Full pipeline |

**Gaps:** No handler tests, no auth extraction from HTTP, no storage client tests, no config parsing tests, no error-to-HTTP mapping tests.

---

## 14. Contract Mismatches

### 14.1 Frontend ↔ Java Backend

| Endpoint/DTO | Field/Aspect | Backend | Frontend | Verdict |
|---|---|---|---|---|
| `ServiceResponse` / `ServiceItem` | `price` nullability | `BigDecimal`, nullable | `number`, required | **MISMATCH** — frontend assumes non-null; backend allows null for non-priced services. `BookingService` validates non-null price at booking time, but catalog display could show `null`. |
| `BookingResponse` / `BookingResponse` | `totalPrice` nullability | `BigDecimal`, nullable | `number`, required | **MISMATCH** — same pattern. Backend column is nullable. |
| `BookingResponse` / `BookingResponse` | `deliveryDate` nullability | `LocalDate`, nullable in DB | `string`, required in TS | **MISMATCH** — DB column nullable; TS interface marks as required. All FOOD_ORDER bookings require it at creation, so unlikely to be null in practice. |
| `BookingItemResponse` / `BookingItem` | `currency` field | Not present in Java DTO | Present in TS interface | **MISMATCH** — frontend defines `currency: string` on `BookingItem`; backend `BookingItemResponse` has only `serviceName`, `quantity`, `unitPrice`. Field will be `undefined` at runtime. |
| `BookingResponse` | `deliveryAddress` nullability | nullable (Java) | `string \| null` (TS) | MATCH |
| `TenantConfigResponse` / `TenantConfig` | `currency` optionality | Always present (Java) | Optional `?` (TS) | DRIFT — works but TS unnecessarily guards against absence |
| Date/time format | `LocalDate` serialization | ISO-8601 (`yyyy-MM-dd`) | Parsed as string | MATCH |
| Date/time format | `OffsetDateTime` serialization | ISO-8601 with offset | Parsed as string | MATCH |
| Price type | `BigDecimal` serialization | JSON number (Jackson default) | `number` | MATCH |

### 14.2 Java Backend ↔ Rust Upload Service

| Aspect | Java Backend | Rust Service | Verdict |
|---|---|---|---|
| Auth header | `Authorization: Bearer {INTERNAL_API_KEY}` | Expects `Authorization` header, constant-time compare | MATCH |
| Tenant header | `X-Tenant-Id: {tenant.id}` | Required, used as key prefix | MATCH |
| Upload path header | `X-Upload-Path: services/{serviceId}` | Required, used in key | MATCH |
| Response body | Expects `{"url":"..."}` | Returns `{"url":"..."}` | MATCH |
| Error body | Expects `{"error":"..."}` | Returns `{"error":"..."}` | MATCH |
| Delete key header | `X-Object-Key` | Required | MATCH |
| `INTERNAL_API_KEY` env var | Shared Railway project variable | Shared Railway project variable | MATCH |

### 14.3 Frontend → Rust Upload Service

No direct communication. Frontend only consumes CDN URLs. No contract to check.

---

## 15. Not Yet Implemented

### 15.1 Backend

| Item | Notes |
|---|---|
| APPOINTMENT tenant type | Defined in enum; all booking endpoints reject non-FOOD_ORDER tenants |
| CATALOG_REQUEST tenant type | Defined in enum; same rejection |
| `booking.slot_id` column | Reserved, never read or written |
| `booking.service_id` column | Deprecated; items in `booking_item` |
| Daily summary notification | Not implemented |

### 15.2 Frontend

| Item | Notes |
|---|---|
| APPOINTMENT flow | Falls through to `UnsupportedFlowComponent` |
| CATALOG_REQUEST flow | Falls through to `UnsupportedFlowComponent` |
| `TenantConfig.logoUrl` rendering | Loaded but not displayed in any template |
| `ServiceItem.durationMinutes` display | Loaded but not rendered |
| `BookingResponse.type` display/branching | Loaded; not used |
| Offline handling | No service worker, no `navigator.onLine` |
| `tg.BackButton` | Not used |
| `tg.close()` | Not called |
| 404 page | Invalid slug shows generic error in `TenantShellComponent` |
| Admin/superadmin frontend | Proxy config exists; no components built |

### 15.3 Upload Service

| Item | Notes |
|---|---|
| Rate limiting | No middleware or per-IP throttling |
| Request ID / correlation ID | No `X-Request-Id` propagation |
| Content-type validation of multipart field | Relies on magic bytes only |
| Object key collision handling | Same ms + same tenant/path = silent overwrite |
| Upload size streaming enforcement | Full bytes in memory before size check |
| Lossless WebP path | Not used |
| `RUST_LOG` default module mismatch | Default filter `image_service=debug` doesn't match crate name `yoobu-media` |
| Handler tests | Auth extraction, multipart parsing, header validation untested |
| Config validation | `INTERNAL_API_KEY` minimum length not enforced |

### 15.4 Cross-Cutting

| Item | Notes |
|---|---|
| Image deletion lifecycle | Java backend can call `DELETE /object` on Rust service, but no code triggers deletion when a service image is replaced or a service is soft-deleted. Old images remain in R2 indefinitely. |
| Telegram `tg.close()` after order | Not called; user must manually close Mini App |

---

*End of YOOBU_DOCS.md*
