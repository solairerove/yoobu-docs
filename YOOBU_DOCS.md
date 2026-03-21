# YOOBU_DOCS.md — Canonical Platform Documentation

Generated from `RND_API_STATE.md` (backend) and `RND_WEB_STATE.md` (frontend) source-derived snapshots.

---

## 1. System Overview

Yoobu is a multi-tenant platform for Telegram Mini App–based food ordering. A tenant operator configures a storefront (menu, branding, checkout hints, payment QR). End users open the Mini App inside Telegram, browse a catalog of services/items, build a cart, submit delivery orders, track booking status, and confirm payment. Tenant admins manage catalog and bookings via a server-rendered Thymeleaf panel. A superadmin manages tenants, credentials, and audit logs via a separate Thymeleaf panel and REST API.

### Stack

| Layer | Technology |
|---|---|
| Backend | Java, Spring Boot, Spring Security (custom filter chains), Spring MVC (REST + Thymeleaf), JPA/Hibernate, Flyway |
| Frontend | Angular 19+ (standalone components, signals), RxJS, custom CSS, Telegram Web App SDK |
| Database | PostgreSQL 17 |
| Hosting | Railway (Docker); nginx reverse proxy for frontend |
| Auth (client) | Telegram `initData` HMAC validation (per-tenant bot token) |
| Auth (admin) | HTTP Basic per tenant (BCrypt password in `tenant_config`) |
| Auth (superadmin) | HTTP Basic (env var credentials) |

### Repository Map

| Repository | Contents |
|---|---|
| `yoobu-api` (inferred from `spring.application.name: yoobu-api`) | Backend: REST API, admin/superadmin Thymeleaf panels, database migrations, security, business logic |
| `yoobu-web` | Frontend: Angular SPA, Telegram Mini App client, food-order feature |

---

## 2. Architecture

### 2.1 Multi-Tenancy Model

Tenant isolation is enforced at every layer:

1. **URL routing**: All client endpoints use `/t/{slug}/...`, all admin endpoints use `/admin/{slug}/...`. The slug is the tenant discriminator.
2. **TenantContextFilter** (Spring Security filter): Parses slug from URL path. Loads tenant via `TenantRepository.findBySlugAndActiveTrue`. Stores tenant in `TenantContext` (static `ThreadLocal<Tenant>`). Returns `400` if slug missing, `404` if tenant not found or inactive.
3. **Service layer**: All repository queries are scoped by `tenantId` obtained from `TenantContext.getRequiredTenantId()`.
4. **Database**: No row-level security. Isolation relies entirely on application-layer tenant ID filtering in JPA queries.
5. **Frontend**: `TenantShellComponent` reads `slug` from route param, passes it to all API calls. The frontend never stores or accesses data across tenants.

### 2.2 Request Lifecycle

```
User opens Telegram Mini App
  → Bot sends Mini App URL with slug in path (e.g., /t/demo)
  → Angular loads, router matches t/:slug
  → TenantShellComponent lazy-loads
  → TelegramService.init() called (tg.ready(), tg.expand())
  → TenantApiService.getConfig(slug) → GET /api/t/{slug}/config
  → Frontend proxy rewrites /api/t → /t (dev: proxy.conf.json; prod: nginx)
  → Backend: TenantContextFilter resolves slug → TenantContext
  → TenantConfigService reads tenant + tenant_config rows → TenantConfigResponse
  → Response returns to frontend
  → TenantShellComponent applies theme (--yoobu-primary CSS var)
  → resolveFeatureComponent selects component by TenantType
  → FOOD_ORDER → FoodOrderHomeComponent loaded
  → Facade loads services, renders menu
```

For authenticated requests (bookings):

```
Frontend interceptor:
  → Reads Telegram initData (or dev fallback X-Telegram-User-Id)
  → Attaches X-Telegram-Init-Data header (polls up to 1500ms for data availability)
Backend:
  → TelegramUserArgumentResolver reads header
  → TelegramInitDataValidator validates HMAC using tenant.botToken
  → Extracts TelegramUser (id, firstName, lastName, username)
  → Injects into controller via @TelegramPrincipal
```

### 2.3 Security Model

#### Filter Chains (`SecurityConfig`)

| Order | Bean | Matcher | Auth | Custom Filters |
|---|---|---|---|---|
| 1 | `superAdminSecurityFilterChain` | `/superadmin/**` | `authenticated()` | `SuperAdminBasicAuthenticationFilter` before `BasicAuthenticationFilter` |
| 2 | `adminSecurityFilterChain` | `/admin/*/**` | `authenticated()` | `TenantContextFilter` before `BasicAuthenticationFilter`; `TenantBasicAuthenticationFilter` after `TenantContextFilter` |
| 3 | `tenantPublicSecurityFilterChain` | `/t/*/**` | `permitAll()` | `TenantContextFilter` before `AnonymousAuthenticationFilter` |
| 4 | `defaultSecurityFilterChain` | fallback | `permitAll()` | none |

All chains: CSRF disabled, form login disabled, logout disabled, framework HTTP Basic disabled.

#### Custom Filter Behavior

| Filter | Credential Source | Validation | Failure |
|---|---|---|---|
| `SuperAdminBasicAuthenticationFilter` | `Authorization: Basic` | Exact match against `SecurityProperties.superadmin` (env vars) | `401` + realm `Yoobu Super Admin` |
| `TenantBasicAuthenticationFilter` | `Authorization: Basic` | Requires `TenantContext`. Loads `admin_username`/`admin_password` from `tenant_config`. BCrypt match. Also accepts superadmin credentials as bypass. | `401` + realm `Yoobu Tenant Admin: {slug}` |
| `TenantContextFilter` | URL path segment | Parses slug from URI after `t` or `admin` segment. Loads active tenant. | `400` slug missing; `404` tenant not found |

#### Telegram Auth

| Component | Behavior |
|---|---|
| `TelegramInitDataValidator.validate` | Parses init data as query string, removes `hash`, sorts params, computes HMAC-SHA256 (`secretKey = HMAC("WebAppData", botToken)`), constant-time comparison. Extracts `user` JSON, requires non-null `id`. |
| `TelegramUserArgumentResolver` | Supports `@TelegramPrincipal TelegramUser`. Reads `X-Telegram-Init-Data`; in `dev` profile, falls back to `X-Telegram-User-Id` (synthetic user). |
| Frontend interceptor | For `/bookings` URLs: polls for `initData` up to 1500ms/50ms interval. Attaches `X-Telegram-Init-Data` or dev fallback `X-Telegram-User-Id`. |

### 2.4 Data Flow: Layer Responsibilities

| Concern | Layer |
|---|---|
| Request validation (field presence, size, format) | Controller (`@Valid` + Bean Validation annotations) |
| Business validation (delivery date cutoff, status transitions, tenant type gating) | Service layer |
| Computed fields (total price, currency, delivery date validation) | Service layer |
| Audit logging | Service layer (calls `AuditLogService`) |
| Default values (`unit="шт"`, `sortOrder=0`, `status=ACTIVE`, `currency=USD`) | Service layer on write; DB defaults as fallback |
| Optimistic locking | JPA `@Version` on `Booking.version` |

---

## 3. API Contract Reference

### 3.1 Client API (`/t/{slug}/...`)

| Method | Path | Request Body | Response | Auth | Status Codes | Frontend Calls |
|---|---|---|---|---|---|---|
| `GET` | `/t/{slug}/config` | — | `TenantConfigResponse` | none | `200`, `400`, `404` | Yes (`TenantApiService.getConfig`) |
| `GET` | `/t/{slug}/services` | — | `List<ServiceResponse>` | none | `200`, `400`, `404` | Yes (`TenantApiService.getServices`) |
| `POST` | `/t/{slug}/bookings` | `CreateBookingRequest` | `BookingResponse` | Telegram | `200`, `400`, `401`, `404` | Yes (`TenantApiService.createBooking`) |
| `GET` | `/t/{slug}/bookings/my` | — | `List<BookingResponse>` | Telegram | `200`, `401` | Yes (`TenantApiService.getMyBookings`) |
| `GET` | `/t/{slug}/bookings/{bookingId}` | — | `BookingResponse` | Telegram | `200`, `401`, `404` | Yes (`TenantApiService.getBooking`) |
| `POST` | `/t/{slug}/bookings/{bookingId}/cancel` | — | `BookingResponse` | Telegram | `200`, `401`, `404`, `409` | Yes (`TenantApiService.cancelBooking`) |
| `POST` | `/t/{slug}/bookings/{bookingId}/confirm-payment` | — | `BookingResponse` | Telegram | `200`, `401`, `404`, `409` | Yes (`TenantApiService.confirmBookingPayment`) |

Notes:
- `400 Tenant slug is missing` / `404 Tenant not found` from `TenantContextFilter` applies to all endpoints.
- `401 Invalid initData` from `TelegramUserArgumentResolver` on Telegram-auth endpoints.
- `createBooking` additional 400s: `Tenant does not support food ordering`, `Delivery date must be on or after ...`, `Service not found`, `Service price is missing`.
- `cancelBooking` 409: `Completed booking cannot be cancelled`.
- `confirmPayment` 409: `Payment can only be confirmed for booking in NEW status`.

### 3.2 Admin REST API (`/admin/{slug}/...`)

| Method | Path | Query Params | Request Body | Response | Auth | Status Codes | Notes |
|---|---|---|---|---|---|---|---|
| `GET` | `/admin/{slug}/services` | — | — | `List<ServiceResponse>` | HTTP Basic tenant | `200`, `401` | Returns non-DELETED |
| `POST` | `/admin/{slug}/services` | — | `AdminUpsertServiceRequest` | `ServiceResponse` | HTTP Basic tenant | `201`, `400`, `401` | |
| `PUT` | `/admin/{slug}/services/{serviceId}` | — | `AdminUpsertServiceRequest` | `ServiceResponse` | HTTP Basic tenant | `200`, `400`, `401`, `404` | DELETED status rejected |
| `DELETE` | `/admin/{slug}/services/{serviceId}` | — | — | `204` empty | HTTP Basic tenant | `204`, `401`, `404` | Soft delete |
| `GET` | `/admin/{slug}/bookings` | `status?`, `deliveryDate?` | — | `List<BookingResponse>` | HTTP Basic tenant | `200`, `400`, `401` | Non-food → 400 |
| `GET` | `/admin/{slug}/bookings/{bookingId}` | — | — | `BookingResponse` | HTTP Basic tenant | `200`, `401`, `404` | |
| `PUT` | `/admin/{slug}/bookings/{bookingId}/status` | — | `UpdateStatusRequest` | `BookingResponse` | HTTP Basic tenant | `200`, `400`, `401`, `404`, `409` | Optimistic lock → 409 |

Frontend does not call admin REST endpoints. These are backend-only.

### 3.3 Admin Panel (Thymeleaf, `/admin/{slug}/panel/...`)

| Method | Path | Notes |
|---|---|---|
| `GET` | `/admin/{slug}/panel` | Redirect → `/admin/{slug}/panel/bookings` |
| `GET` | `/admin/{slug}/panel/bookings` | Paged booking list with status/date/page filters |
| `GET` | `/admin/{slug}/panel/bookings/{bookingId}` | Booking detail view |
| `POST` | `/admin/{slug}/panel/bookings/{bookingId}/status` | Status update form; `returnTo` param for redirect target |
| `GET` | `/admin/{slug}/panel/services` | Paged service list with query/page filters |
| `GET` | `/admin/{slug}/panel/services/new` | New service form |
| `POST` | `/admin/{slug}/panel/services` | Create service form submit |
| `GET` | `/admin/{slug}/panel/services/{serviceId}/edit` | Edit service form |
| `POST` | `/admin/{slug}/panel/services/{serviceId}` | Update service form submit |
| `POST` | `/admin/{slug}/panel/services/{serviceId}/delete` | Delete with name confirmation |
| `POST` | `/admin/{slug}/panel/services/{serviceId}/status` | Inline status update |

All require HTTP Basic tenant auth. Binding/service errors surfaced as flash messages with redirect.

### 3.4 Superadmin REST API (`/superadmin/...`)

| Method | Path | Query Params | Request Body | Response | Auth | Status Codes |
|---|---|---|---|---|---|---|
| `GET` | `/superadmin/tenants` | — | — | `List<TenantSummaryResponse>` | HTTP Basic superadmin | `200`, `401` |
| `GET` | `/superadmin/tenants/{tenantId}` | — | — | `TenantDetailResponse` | HTTP Basic superadmin | `200`, `401`, `404` |
| `GET` | `/superadmin/tenants/slug-availability` | `slug` required | — | `TenantSlugAvailabilityResponse` | HTTP Basic superadmin | `200`, `401` |
| `POST` | `/superadmin/tenants` | — | `CreateTenantRequest` | `TenantSummaryResponse` | HTTP Basic superadmin | `200`, `400`, `401`, `409` |
| `PUT` | `/superadmin/tenants/{tenantId}` | — | `UpdateTenantRequest` | `TenantSummaryResponse` | HTTP Basic superadmin | `200`, `400`, `401`, `404` |
| `GET` | `/superadmin/audit` | `tenantId?`, `entity?`, `action?`, `actorId?`, `createdFrom?`, `createdTo?`, `page=0`, `size=20` | — | `AuditLogPageResponse` | HTTP Basic superadmin | `200`, `401` |
| `GET` | `/superadmin/audit/export` | same filters; `size=5000` | — | `text/csv` | HTTP Basic superadmin | `200`, `401` |

### 3.5 Superadmin Panel (Thymeleaf, `/superadmin/panel/...`)

| Method | Path | Notes |
|---|---|---|
| `GET` | `/superadmin/panel` | Redirect → `/superadmin/panel/tenants` |
| `GET` | `/superadmin/panel/tenants` | Paged tenant list with query filter |
| `GET` | `/superadmin/panel/tenants/{tenantId}` | Tenant detail view |
| `GET` | `/superadmin/panel/tenants/new` | New tenant form |
| `POST` | `/superadmin/panel/tenants` | Create tenant form submit |
| `GET` | `/superadmin/panel/tenants/{tenantId}/edit` | Edit tenant form |
| `POST` | `/superadmin/panel/tenants/{tenantId}` | Update tenant form submit |
| `GET` | `/superadmin/panel/audit` | Paged audit log with filters |
| `GET` | `/superadmin/panel/audit/export` | CSV export |

All require HTTP Basic superadmin auth.

### 3.6 Unscoped

| Method | Path | Auth | Response |
|---|---|---|---|
| `GET` | `/health` | none | `{ "status": "ok" }` |

---

## 4. Data Models

### 4.1 API Contracts (Frontend ↔ Backend Boundary)

#### `TenantConfigResponse` (backend) ↔ `TenantConfig` (frontend)

| Field | Backend Java Type | Frontend TS Type | Required/Optional | Notes |
|---|---|---|---|---|
| `slug` | `String` | `string` | required | |
| `name` | `String` | `string` | required | |
| `type` | `TenantType` | `TenantType` | required | |
| `currency` | `String` | `string \| null` (optional) | nullable | Frontend marks as optional (`?`). Backend sends from tenant config with `USD` fallback. |
| `paymentQrUrl` | `String` | `string \| null` (optional) | nullable | Frontend marks as optional (`?`). |
| `primaryColor` | `String` | `string \| null` | nullable | |
| `logoUrl` | `String` | `string \| null` | nullable | |
| `welcomeMessage` | `String` | `string \| null` | nullable | |
| `checkoutNameHint` | `String` | `string \| null` (optional) | nullable | Frontend marks as optional (`?`). |
| `checkoutPhoneHint` | `String` | `string \| null` (optional) | nullable | Frontend marks as optional (`?`). |
| `checkoutNoteHint` | `String` | `string \| null` (optional) | nullable | Frontend marks as optional (`?`). |

**Mismatches**: None structural. Frontend `?` (optional property) vs backend always-present nullable is a TypeScript idiom difference, not a runtime mismatch — JSON `null` satisfies both. See Section 12 for full cross-check.

#### `ServiceResponse` (backend) ↔ `ServiceItem` (frontend)

| Field | Backend Java Type | Frontend TS Type | Required/Optional | Notes |
|---|---|---|---|---|
| `id` | `Long` | `number` | required | |
| `name` | `String` | `string` | required | |
| `description` | `String` | `string \| null` | nullable | |
| `price` | `BigDecimal` | `number` | required | **Type note**: Java `BigDecimal` serializes to JSON number. Frontend receives as `number`. Precision loss possible for very large values but not practically relevant at `NUMERIC(10,2)`. |
| `unit` | `String` | `string \| null` | nullable | |
| `durationMinutes` | `Integer` | `number \| null` | nullable | |
| `sortOrder` | `int` | `number` | required | |
| `status` | `ServiceStatus` | `'ACTIVE' \| 'INACTIVE' \| 'DELETED'` | required | Client API only returns `ACTIVE` services; admin API returns non-DELETED. Frontend enum is closed (no `\| string` fallback). |

**Mismatches**: None. Frontend `ServiceItem` `status` is a closed union — if backend adds a new `ServiceStatus` value, frontend will receive it but TypeScript type won't match. Low risk since client API filters to `ACTIVE` only.

#### `CreateBookingRequest` (backend) ↔ `CreateBookingRequest` (frontend)

| Field | Backend Java Type | Frontend TS Type | Required/Optional | Validation | Notes |
|---|---|---|---|---|---|
| `customerName` | `String` | `string` | required | `@NotBlank` | |
| `customerPhone` | `String` | `string` | required | `@NotBlank` | |
| `deliveryAddress` | `String` | `string` | required | `@NotBlank` | |
| `deliveryDate` | `LocalDate` | `string` | required | `@NotNull` | Frontend sends ISO date string; Jackson deserializes to `LocalDate`. |
| `note` | `String` | `string \| null` | nullable | none | |
| `items` | `List<BookingItemRequest>` | `CreateBookingItem[]` | required | `@NotEmpty`, elements `@Valid` | |

**Nested: `BookingItemRequest` ↔ `CreateBookingItem`**

| Field | Backend Java Type | Frontend TS Type | Required/Optional | Validation |
|---|---|---|---|---|
| `serviceId` | `Long` | `number` | required | `@NotNull` |
| `quantity` | `int` | `number` | required | `@Min(1)` |

**Mismatches**: None.

#### `BookingResponse` (backend) ↔ `BookingResponse` (frontend)

| Field | Backend Java Type | Frontend TS Type | Required/Optional | Notes |
|---|---|---|---|---|
| `id` | `Long` | `number` | required | |
| `type` | `BookingType` | `'ORDER' \| 'APPOINTMENT' \| 'REQUEST' \| string` | required | Frontend uses open union with `\| string` fallback. |
| `status` | `BookingStatus` | `BookingStatus` (open union with `\| string`) | required | Frontend uses open union. |
| `customerName` | `String` | `string` | required | |
| `customerPhone` | `String` | `string` | required | |
| `deliveryAddress` | `String` | `string \| null` | nullable | Backend: nullable in DB, required by API for creation. Response can be null for legacy/non-food bookings. |
| `totalPrice` | `BigDecimal` | `number` | required | |
| `currency` | `String` | `string` | required | |
| `deliveryDate` | `LocalDate` | `string` | required | Serialized as ISO date string. |
| `note` | `String` | `string \| null` | nullable | |
| `trackingUrl` | `String` | `string \| null` | nullable | |
| `items` | `List<BookingItemResponse>` | `BookingItem[]` | required | |
| `createdAt` | `OffsetDateTime` | `string` | required | Serialized as ISO datetime string. |

**Nested: `BookingItemResponse` ↔ `BookingItem`**

| Field | Backend Java Type | Frontend TS Type |
|---|---|---|
| `serviceName` | `String` | `string` |
| `quantity` | `int` | `number` |
| `unitPrice` | `BigDecimal` | `number` |
| `currency` | `String` | `string` |

**Mismatches**: None structural. See Section 12 for nuance on `deliveryAddress` nullability.

### 4.2 Backend-Only Models

#### Entities

See Section 6 for full schema. Entity-to-table mapping summary:

| Entity | Table | Key Relationships |
|---|---|---|
| `Tenant` | `tenant` | — |
| `TenantConfig` | `tenant_config` | `@ManyToOne Tenant` |
| `CatalogService` | `service` | `@ManyToOne Tenant` |
| `Booking` | `booking` | `@ManyToOne Tenant`, `@ManyToOne CatalogService` (reserved) |
| `BookingItem` | `booking_item` | `@ManyToOne Booking`, `@ManyToOne CatalogService` |
| `AuditLog` | `audit_log` | none (uses `tenantId` as plain Long) |

#### MVC Form Models (Thymeleaf only)

| Form | Fields | Used By |
|---|---|---|
| `ServiceForm` | `name` (`@NotBlank`), `price` (`@NotNull BigDecimal`), `status` (`@NotNull ServiceStatus`, default `ACTIVE`), `description`, `unit`, `durationMinutes`, `sortOrder` | Admin panel service create/update |
| `ServiceStatusForm` | `status` (`@NotNull ServiceStatus`) | Admin panel inline service status update |
| `BookingStatusForm` | `status` (`@NotNull BookingStatus`), `trackingUrl` (`@Size(max=2048)`) | Admin panel booking status update |
| `SuperAdminTenantForm` | `slug` (`@NotBlank`), `name` (`@NotBlank`), `type` (`@NotNull TenantType`, default `FOOD_ORDER`), `timezone` (default `Asia/Ho_Chi_Minh`), `currency` (default `USD`), `adminUsername` (`@NotBlank`), `adminPassword` (required on create, optional on update), `active` (default `true`), branding/checkout/payment fields | Superadmin panel tenant create/update |

#### Superadmin-only Response DTOs

| DTO | Fields | Used By |
|---|---|---|
| `TenantSummaryResponse` | tenant core fields | Superadmin tenant list, create, update responses |
| `TenantDetailResponse` | tenant core fields + `config` map | Superadmin tenant detail |
| `TenantSlugAvailabilityResponse` | `slug`, `available` | Slug availability check |
| `AuditLogItemResponse` | audit row fields | Audit API and panel |
| `AuditLogPageResponse` | pagination wrapper around `AuditLogItemResponse` | Audit API |

#### Admin-only Request DTOs

| DTO | Fields | Used By |
|---|---|---|
| `AdminUpsertServiceRequest` | `name` (`@NotBlank`), `price` (`@NotNull BigDecimal`), `description`, `unit`, `durationMinutes`, `sortOrder`, `status` | Admin REST service create/update |
| `UpdateStatusRequest` | `status` (`@NotNull BookingStatus`), `trackingUrl` (`@Size(max=2048)`) | Admin REST booking status update |
| `CreateTenantRequest` | `slug`, `name`, `type`, `botToken`, `ownerTelegramId`, `timezone`, `currency`, `adminUsername`, `adminPassword`, branding/checkout/payment fields | Superadmin REST create |
| `UpdateTenantRequest` | `name`, `type`, `adminUsername`, `active`, optional fields (same set minus slug/password-required) | Superadmin REST update |

### 4.3 Schema vs Entity Drift

| Area | Flyway Schema | Entity Mapping | Difference |
|---|---|---|---|
| `tenant_config` uniqueness | `UNIQUE (tenant_id, key)` | No `@Table(uniqueConstraints=...)` on `TenantConfig` | DB-enforced unique constraint not declared in JPA metadata |
| FK delete behavior | `booking_item.booking_id ON DELETE CASCADE` | `BookingItem.booking` has no cascade remove/orphan config | Cascade exists only at DB layer |
| DB defaults (`tenant`) | `timezone DEFAULT 'Asia/Ho_Chi_Minh'`, `active DEFAULT TRUE`, `created_at DEFAULT NOW()` | No `columnDefinition`/default metadata | Defaults rely on DB; service sets values explicitly on create |
| DB defaults (`service`) | `unit DEFAULT 'шт'`, `status DEFAULT 'ACTIVE'`, `sort_order DEFAULT 0`, timestamps default `NOW()` | No default metadata | Defaults in DB; service also sets values at write time |
| DB defaults (`booking`) | `status DEFAULT 'NEW'`, timestamps default `NOW()`, `version DEFAULT 0` | No default metadata (except `@Version`) | Defaults in DB; service sets most values explicitly |
| DB default (`audit_log.created_at`) | `DEFAULT NOW()` | No default metadata | Timestamp default DB-side; service sets `createdAt` explicitly |
| `audit_log.actor_id` length | `VARCHAR(255)` | `@Column(name="actor_id")` with implicit provider length | Explicit SQL length not mirrored as explicit annotation parameter |

---

## 5. Enums

| Enum | Values | Backend Definition | Backend References | Frontend Definition | Frontend References | Value Mismatches |
|---|---|---|---|---|---|---|
| `BookingStatus` | `NEW`, `PAYMENT_PENDING`, `CONFIRMED`, `DELIVERING`, `DONE`, `CANCELLED` | `BookingStatus` enum | `Booking`, `BookingService` transition matrix, `UpdateStatusRequest`, `BookingStatusForm`, controllers | `BookingStatus` type in `booking.model.ts` (open union with `\| string`) | `BookingResponse`, `FoodOrderBookingsComponent` status badges/grouping, facade cancel/confirm logic | None. Frontend open union is a superset. |
| `BookingType` | `ORDER`, `APPOINTMENT`, `REQUEST` | `BookingType` enum | `Booking.type`, `BookingService.createFoodOrder` (always sets `ORDER`) | Inline union in `BookingResponse.type` (open with `\| string`) | Display only; no branching on type value in frontend | None |
| `ServiceStatus` | `ACTIVE`, `INACTIVE`, `DELETED` | `ServiceStatus` enum | `CatalogService.status`, catalog repos/services/controllers/forms | Inline union in `ServiceItem.status` (closed: no `\| string`) | Frontend receives `ACTIVE` only from client API | None. **Risk**: closed frontend union will cause TS error if new value appears, but client API filters to `ACTIVE`. |
| `TenantType` | `FOOD_ORDER`, `APPOINTMENT`, `CATALOG_REQUEST` | `TenantType` enum | `Tenant.type`, create/update DTOs, food-order gating | `TenantType` type in `tenant-config.model.ts` (closed union) | `TenantShellComponent.resolveFeatureComponent` (only maps `FOOD_ORDER`; others → `UnsupportedFlowComponent`) | None |

---

## 6. Database

### 6.1 Full Schema (Flyway V1–V8 Reconstructed)

```sql
CREATE TABLE tenant (
    id BIGSERIAL PRIMARY KEY,
    slug VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(30) NOT NULL,
    bot_token VARCHAR(255),
    owner_telegram_id BIGINT,
    timezone VARCHAR(50) NOT NULL DEFAULT 'Asia/Ho_Chi_Minh',
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE tenant_config (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenant(id),
    key VARCHAR(100) NOT NULL,
    value TEXT,
    UNIQUE (tenant_id, key)
);

CREATE TABLE service (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price NUMERIC(10, 2),
    unit VARCHAR(50) DEFAULT 'шт',
    duration_minutes INT,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    sort_order INT NOT NULL DEFAULT 0,
    deleted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE booking (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenant(id),
    type VARCHAR(20) NOT NULL,
    telegram_user_id BIGINT NOT NULL,
    customer_name VARCHAR(255) NOT NULL,
    customer_phone VARCHAR(50),
    status VARCHAR(20) NOT NULL DEFAULT 'NEW',
    note TEXT,
    total_price NUMERIC(10, 2),
    delivery_date DATE,
    slot_id BIGINT,
    service_id BIGINT REFERENCES service(id),
    deleted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    version BIGINT NOT NULL DEFAULT 0,
    delivery_address TEXT,
    tracking_url TEXT
);

CREATE TABLE booking_item (
    id BIGSERIAL PRIMARY KEY,
    booking_id BIGINT NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    service_id BIGINT NOT NULL REFERENCES service(id),
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10, 2) NOT NULL,
    currency VARCHAR(10) NOT NULL
);

CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    entity VARCHAR(50) NOT NULL,
    entity_id BIGINT NOT NULL,
    action VARCHAR(50) NOT NULL,
    actor_id VARCHAR(255),
    old_value TEXT,
    new_value TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

### 6.2 Indexes

```sql
CREATE INDEX idx_service_tenant ON service(tenant_id, status, sort_order, id) WHERE status <> 'DELETED';
CREATE INDEX idx_booking_tenant ON booking(tenant_id);
CREATE INDEX idx_booking_tenant_date ON booking(tenant_id, delivery_date);
CREATE INDEX idx_booking_tenant_status ON booking(tenant_id, status);
CREATE INDEX idx_booking_telegram_user ON booking(tenant_id, telegram_user_id);
CREATE INDEX idx_audit_tenant_entity ON audit_log(tenant_id, entity, entity_id);
CREATE INDEX idx_audit_created_at_id ON audit_log(created_at DESC, id DESC);
CREATE INDEX idx_audit_tenant_created_at_id ON audit_log(tenant_id, created_at DESC, id DESC);
CREATE INDEX idx_audit_action_created_at_id ON audit_log(action, created_at DESC, id DESC);
```

### 6.3 Constraints Summary

| Table | Constraint | Type |
|---|---|---|
| `tenant` | `slug UNIQUE` | unique |
| `tenant_config` | `(tenant_id, key) UNIQUE` | composite unique |
| `booking_item` | `quantity > 0` | check |
| `booking_item.booking_id` | `ON DELETE CASCADE` | FK cascade |
| All FKs | `REFERENCES` on `tenant_id`, `service_id`, `booking_id` | referential integrity |

### 6.4 Migration History

| Version | Filename | Summary |
|---|---|---|
| 1 | `V1__baseline_schema.sql` | Initial tables + base indexes |
| 2 | `V2__audit_actor_id_as_string.sql` | `audit_log.actor_id` changed from numeric to `VARCHAR(255)` |
| 3 | `V3__service_status_model.sql` | Added `service.status`, rebuilt index, dropped `active` column |
| 4 | `V4__audit_log_filter_indexes.sql` | Audit feed/filter indexes |
| 5 | `V5__booking_optimistic_lock.sql` | `booking.version` for optimistic locking |
| 6 | `V6__booking_item_currency.sql` | `booking_item.currency`, backfilled from tenant config |
| 7 | `V7__booking_delivery_address.sql` | `booking.delivery_address` |
| 8 | `V8__booking_tracking_url.sql` | `booking.tracking_url` |

### 6.5 Unused Schema Elements

| Column | Table | Status |
|---|---|---|
| `slot_id` | `booking` | Reserved. No controller/service logic reads or writes. |
| `service_id` | `booking` | Reserved. Direct booking→service link unused; itemized model uses `booking_item.service_id`. |

---

## 7. Frontend Architecture

### 7.1 Module Tree and Routing

Angular 19+ standalone bootstrap. No NgModules. Single route tree:

```
''              → redirect → 't/demo'
't/:slug'       → lazy loadComponent → TenantShellComponent
'**'            → redirect → 't/demo'
```

No guards. No resolvers. No child routes.

`TenantShellComponent` dynamically loads feature components via `ngComponentOutlet` based on `TenantConfig.type`:
- `FOOD_ORDER` → `FoodOrderHomeComponent`
- All others → `UnsupportedFlowComponent`

### 7.2 Component Inventory

| Component | Selector | Role | API Calls | Telegram SDK |
|---|---|---|---|---|
| `AppComponent` | `app-root` | Shell; renders `<router-outlet />` | none | none |
| `TenantShellComponent` | `app-tenant-shell` | Tenant resolution, theme application, feature component loading | `getConfig` | `init()`, `setMainButton(null)` |
| `UnsupportedFlowComponent` | `app-unsupported-flow` | Fallback for non-FOOD_ORDER tenants | none | none |
| `FoodOrderHomeComponent` | `app-food-order-home` | Feature root; tab switching (menu/orders), delegates to facade | none (via facade) | none (via facade) |
| `FoodOrderMenuComponent` | `app-food-order-menu` | Service card list with quantity controls | none | none |
| `FoodOrderCartBarComponent` | `app-food-order-cart-bar` | Fixed bottom cart CTA with count/total | none | none |
| `FoodOrderCheckoutComponent` | `app-food-order-checkout` | Order form (local mode card or Telegram bottom sheet) | none | none |
| `FoodOrderBookingsComponent` | `app-food-order-bookings` | Order list grouped by status, detail view, actions | none | none |
| `FoodOrderSuccessCardComponent` | `app-food-order-success-card` | Post-submission snapshot with payment QR and actions | none | none |

### 7.3 State Management

| Store/State Holder | Scope | What It Holds | Lifecycle |
|---|---|---|---|
| `FoodOrderStore` | Feature-scoped (provided in `FoodOrderHomeComponent`) | `slug`, `services[]`, `quantities{}` + derived `selectedItems`, `selectedCount`, `selectedTotal` | Resets on tenant change (`setTenant`) |
| `FoodOrderFlowFacade` | Feature-scoped (provided in `FoodOrderHomeComponent`) | Checkout form, view state, booking selection, submission state, error signals, customer draft, VM signals | Resets on `setConfig` (new tenant) |
| `TelegramService` | Root singleton | `webAppSignal`, `mainButtonHandler` | App lifetime |
| `TenantApiService` | Root singleton | Stateless (HTTP calls only) | App lifetime |

No NgRx, no global store, no localStorage/sessionStorage.

### 7.4 Theming

`TenantConfig.primaryColor` → `--yoobu-primary` CSS variable set on `document.documentElement` and host style binding. Fallback: `#ff6b35`.

Global styles (`styles.css`): color system CSS variables, base typography, button/utility classes (`.ghost-button`, `.ui-copy`, `.ui-status-card`).

Component styles: inline `styles` arrays in standalone component TS files. No CSS framework (no Tailwind, no Bootstrap, no Material).

Responsive: `@media (max-width: 640px)`, `env(safe-area-inset-bottom)`, `100dvh` for Telegram bottom sheet.

---

## 8. Telegram Mini App Integration

### 8.1 End-to-End Auth Flow

```
1. Telegram client loads Mini App URL
2. Telegram SDK injects window.Telegram.WebApp with initData
3. Angular boots → TelegramService.init() → resolveWebApp() caches WebApp reference
4. tg.ready() and tg.expand() called via effect
5. On HTTP request to /bookings/* URL:
   a. Interceptor calls telegram.getInitData()
   b. If initData present → attaches X-Telegram-Init-Data header
   c. If absent, polls up to 1500ms (50ms interval)
   d. If still absent and dev user id present → attaches X-Telegram-User-Id
6. Backend receives request:
   a. TelegramUserArgumentResolver reads X-Telegram-Init-Data header
   b. TelegramInitDataValidator.validate():
      - Loads tenant bot token from TenantContext
      - Parses init data as query string
      - Sorts params, removes hash, computes HMAC-SHA256 check string
      - secretKey = HMAC-SHA256("WebAppData", botToken)
      - hash = HMAC-SHA256(secretKey, dataCheckString)
      - Constant-time comparison with provided hash
      - Extracts user JSON, requires non-null id
   c. Returns TelegramUser(id, firstName, lastName, username)
7. Controller receives @TelegramPrincipal TelegramUser
```

### 8.2 Dev Mode

- **Frontend**: `TelegramService.isLocalhost()` checks `window.location.hostname`. If true, `getDevTelegramUserId()` reads `tgWebAppStartParam` from URL or returns hardcoded fallback. Interceptor uses `X-Telegram-User-Id` header.
- **Backend**: `TelegramUserArgumentResolver` checks `environment.matchesProfiles("dev")`. If active profile is `dev` and `X-Telegram-Init-Data` absent, reads `X-Telegram-User-Id` header and constructs synthetic `TelegramUser(id, "Dev", "User", null)`.

### 8.3 SDK Methods Used

| Method | Where | Purpose |
|---|---|---|
| `tg.ready()` | `TelegramService` effect | Signals Mini App is ready |
| `tg.expand()` | `TelegramService` effect | Expands to full height |
| `tg.initData` | `TelegramService.getInitData()` | Auth token source |
| `tg.MainButton.setText/show/hide/enable/disable/onClick/offClick` | `TelegramService.setMainButton/onMainButtonClick` → `FoodOrderFlowFacade` effect | Checkout/submit CTA |
| `tg.showAlert` | `TelegramService.alert` (fallback: `window.alert`) | Error/validation feedback |
| `tg.showConfirm` | `TelegramService.confirm` (fallback: `window.confirm`) | Submit/cancel/repeat confirmation |

### 8.4 SDK Methods Not Used

`tg.BackButton`, `tg.close()`, `HapticFeedback`, `themeParams`, `viewportHeight`, `viewportStableHeight`.

---

## 9. User Flows (End-to-End)

### 9.1 App Init

1. User opens URL (e.g., `https://app.example.com/t/demo`).
2. Angular router: `''` redirects to `t/demo`; `t/:slug` lazy-loads `TenantShellComponent`.
3. `TenantShellComponent.vm$` subscribes to `ActivatedRoute.paramMap`, extracts `slug`.
4. Calls `telegram.init()` and `telegram.setMainButton(null)`.
5. Calls `TenantApiService.getConfig(slug)` → `GET /api/t/{slug}/config`.
6. Proxy rewrites `/api/t` → `/t`. Backend: `TenantContextFilter` resolves tenant → `TenantConfigService.getCurrentTenantConfig()` → `TenantConfigResponse`.
7. On success: `applyTheme(primaryColor)` sets `--yoobu-primary` on `document.documentElement`. `resolveFeatureComponent` selects `FoodOrderHomeComponent` for `FOOD_ORDER`.
8. On error: renders tenant unavailable card. No redirect.

Wildcard route `**` redirects to `t/demo`.

### 9.2 Browse Catalog

1. `FoodOrderHomeComponent` constructor effect calls `facade.setConfig(config)`.
2. `setConfig` resets per-tenant state: clears cart, resets signals, sets `slug` in store, calls `loadServices`.
3. `loadServices` calls `TenantApiService.getServices(slug)` → `GET /api/t/{slug}/services`.
4. Backend: `CatalogQueryService.getActiveServices()` → returns `ACTIVE` services ordered by `sortOrder, id`.
5. On success: `FoodOrderStore.setServices(services)`, vm updates, `FoodOrderMenuComponent` renders service cards.
6. On error: vm error message displayed.

### 9.3 Add to Cart

1. User taps `+` on `FoodOrderMenuComponent` service card.
2. Component emits `increaseRequested(serviceId)`.
3. `FoodOrderHomeComponent` calls `facade.increase(serviceId)` → `store.increase(serviceId)`.
4. `store.quantitiesSignal` updated → computed `selectedItems`, `selectedCount`, `selectedTotal` recompute.
5. Menu component updates selected state styling and quantity display.
6. `FoodOrderCartBarComponent` shows with count/total.
7. Facade effect: Telegram MainButton shows `Checkout • {formatted total}`.

Decrease: same flow with `decrease`/`decreaseRequested`. Quantity floor is 0 (removes from cart).

### 9.4 Checkout

1. User opens checkout via cart bar tap or Telegram MainButton click → `facade.openCheckout()`.
2. `checkoutOpen=true`. `FoodOrderCheckoutComponent` renders.
   - Telegram mode: bottom sheet with `100dvh` + safe area.
   - Local mode: inline card with local submit button.
3. Form fields: `customerName`, `customerPhone`, `deliveryAddress`, `deliveryDate`, `note`.
   - Customer details pre-populated from draft (hydrated from most recent booking once per tenant).
   - Hints from tenant config (`checkoutNameHint`, `checkoutPhoneHint`, `checkoutNoteHint`).
4. Submit action (local button or Telegram MainButton):
   - Validates required fields and delivery address non-empty.
   - Invalid → `submitError` set, `telegram.alert`.
5. Valid → `telegram.confirm("Place order for {total}?")`.
6. Confirmed → `TenantApiService.createBooking(slug, request)` → `POST /api/t/{slug}/bookings`.
7. Backend: `BookingService.createFoodOrder`:
   - Requires `TenantType.FOOD_ORDER`.
   - Validates delivery date via `TenantTimeService.earliestDeliveryDate` (cutoff logic).
   - Builds booking + items from active services. Computes total. Sets currency from tenant config.
   - Audit log: `logCreate(entity="booking")`.
   - Returns `BookingResponse`.
8. On success: `submittedBooking` set, `selectedBookingId`/`selectedBooking` set, cart cleared, checkout closed, form reset (preserving customer draft), bookings stream refreshed. `FoodOrderSuccessCardComponent` renders.
9. On error: checkout stays open, `submitError` set, `telegram.alert`.

### 9.5 My Bookings

1. User switches to `My orders` tab → `facade.setActiveView('orders')` + `facade.refreshBookings()`.
2. `bookingsVmSignal` pipeline calls `TenantApiService.getMyBookings(slug)` → `GET /api/t/{slug}/bookings/my`.
3. Backend: `BookingService.getMyBookings(telegramUserId)` → tenant + user scoped, excludes soft-deleted.
4. On success: booking list vm updated. Reconciles selected booking. Hydrates customer draft from latest booking (once per tenant).
5. `FoodOrderBookingsComponent` renders: open orders (non-terminal) and history (DONE/CANCELLED) sections.

### 9.6 Booking Detail

1. User clicks booking row → `bookingSelected(id)` emitted → `facade.selectBooking(id)`.
2. Sets `selectedBookingId`, active view, clears errors.
3. `TenantApiService.getBooking(slug, id)` → `GET /api/t/{slug}/bookings/{id}`.
4. Backend: `BookingService.getMyBooking(bookingId, telegramUserId)` → tenant + user scoped.
5. Uses `selectedBookingRequestVersion` to discard stale async responses.
6. On success: `selectedBooking` set. Detail panel renders status timeline, receipt, tracking link, action buttons.
7. On error: `cancelError='Could not load the order details.'`.

### 9.7 Cancel Booking

1. User taps `Cancel order` → `facade.cancelBooking(id)`.
2. `telegram.confirm(...)` prompt.
3. Confirmed → set `cancellingBookingId`, clear `cancelError`.
4. `TenantApiService.cancelBooking(slug, id)` → `POST /api/t/{slug}/bookings/{id}/cancel`.
5. Backend: `BookingService.cancelMyBooking` → disallows from `DONE` (409), sets `CANCELLED`. Audit log.
6. On success: `selectedBooking` updated, possibly `submittedBooking` updated, bookings refreshed.
7. On error: `cancelError` set, `telegram.alert`.
8. Finally: clear `cancellingBookingId`.

### 9.8 Confirm Payment

1. User taps payment confirm action → `facade.confirmPayment(id)`.
2. Sets `confirmingPaymentBookingId`, clears `paymentError`.
3. `TenantApiService.confirmBookingPayment(slug, id)` → `POST /api/t/{slug}/bookings/{id}/confirm-payment`.
4. Backend: `BookingService.confirmMyBookingPayment` → only from `NEW` → sets `PAYMENT_PENDING`. Audit log.
5. On success: `selectedBooking` updated, possibly `submittedBooking` updated, bookings refreshed.
6. On 409: `telegram.alert`, refreshes bookings and reloads booking detail.
7. On other error: `paymentError` set, `telegram.alert`.
8. Finally: clear `confirmingPaymentBookingId`.

### 9.9 Admin Panel Flows (Backend-Only, Thymeleaf)

**List bookings**: `GET /admin/{slug}/panel/bookings` with optional status/date/page filters. Rendered with paging controls and status filter.

**Booking detail**: `GET /admin/{slug}/panel/bookings/{bookingId}`. Shows full booking with status timeline.

**Update booking status**: `POST /admin/{slug}/panel/bookings/{bookingId}/status` with `BookingStatusForm`. Validates transition matrix. Flash success/error messages. Redirect to list or detail based on `returnTo` param.

**List services**: `GET /admin/{slug}/panel/services` with optional query/page. Paged service listing.

**Create/edit service**: GET forms at `.../services/new` and `.../services/{id}/edit`. POST to create/update. Binding errors re-render form. Service errors flash and redirect.

**Delete service**: `POST .../services/{id}/delete` with `confirmName` param. Requires exact name match.

**Inline status update**: `POST .../services/{id}/status` with `ServiceStatusForm`.

### 9.10 Superadmin Flows (Backend-Only, Thymeleaf)

**Tenant list**: `GET /superadmin/panel/tenants` with query/page.

**Tenant detail**: `GET /superadmin/panel/tenants/{tenantId}`.

**Create tenant**: GET `.../tenants/new` for form; POST `.../tenants` to create. Validates slug uniqueness, required password.

**Edit tenant**: GET `.../tenants/{id}/edit`; POST `.../tenants/{id}`. Blank password keeps existing. Currency maintained/seeded.

**Audit log**: `GET /superadmin/panel/audit` with filters. Export: `GET /superadmin/panel/audit/export` (CSV, capped 5000 rows).

---

## 10. Configuration

### 10.1 Backend: `application.yml`

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
    allowed-origin-patterns: ${CORS_ALLOWED_ORIGIN_PATTERNS:http://localhost:*,http://127.0.0.1:*,https://localhost:*,https://127.0.0.1:*,https://*.up.railway.app}
  superadmin:
    username: ${SUPERADMIN_USER:superadmin}
    password: ${SUPERADMIN_PASS:superadmin}

server:
  port: ${PORT:8080}
  error:
    include-message: always

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} %-5level [%thread] %logger - %msg%n"
```

No `application-dev.yml` or profile-specific YAML files exist. The `dev` profile is activated only via test infrastructure (`@ActiveProfiles("dev")`).

### 10.2 Backend: Configuration Properties Binding

| Class | Prefix | Fields | Consumed By |
|---|---|---|---|
| `SecurityProperties` | `app` | `superadmin.username`, `superadmin.password` | `SuperAdminBasicAuthenticationFilter`, `TenantBasicAuthenticationFilter` (superadmin bypass) |
| `CorsProperties` | `app.cors` | `allowedOriginPatterns`, `allowedMethods`, `allowedHeaders`, `exposedHeaders`, `allowCredentials`, `maxAge` | `CorsConfig.corsConfigurationSource` |

`AppProperties` class does not exist. All `app.*` keys are bound through the above specific classes.

### 10.3 Backend: `tenant_config` Keys

| Key | Written By | Read By |
|---|---|---|
| `admin_username` | `TenantManagementService` create/update | `TenantSettings.admin()`, `TenantBasicAuthenticationFilter`, superadmin detail/panel |
| `admin_password` | create/update (BCrypt hash) | `TenantSettings.admin()`, `TenantBasicAuthenticationFilter` |
| `currency` | create/update (default `USD`) | `TenantSettings.pricing()`, `BookingService` currency assignment, panel money formatting |
| `primary_color` | create/update | `TenantSettings.branding()`, `/t/{slug}/config`, panel |
| `logo_url` | create/update | same |
| `welcome_message` | create/update | same |
| `checkout_name_hint` | create/update | `TenantSettings.checkout()`, `/t/{slug}/config` |
| `checkout_phone_hint` | create/update | same |
| `checkout_note_hint` | create/update | same |
| `payment_qr_url` | create/update (normalized by `PaymentQrUrlValidator`) | `TenantSettings.payment()`, `/t/{slug}/config`, panel |
| `cutoff_hour` | **not written** by current flows | `TenantSettings.delivery()`, `TenantTimeService` |
| `cutoff_minute` | **not written** by current flows | `TenantSettings.delivery()`, `TenantTimeService` |

### 10.4 Frontend: Environment and Build

**Environment files**: `src/environments/environment.ts` and `environment.prod.ts` do not exist. No environment-specific configuration.

**`angular.json`**: Builder `@angular-devkit/build-angular:application`. Output: `dist/yoobu-web`. Production budgets: initial warning 500kB, error 1MB. Development: no optimization, source maps enabled.

**Proxy config** (`proxy.conf.json`, dev only):

| Frontend Path | Backend Target | Rewrite |
|---|---|---|
| `/api/t` | `http://localhost:8080` | `/api/t` → `/t` |
| `/api/admin` | `http://localhost:8080` | `/api/admin` → `/admin` |
| `/api/superadmin` | `http://localhost:8080` | `/api/superadmin` → `/superadmin` |

**Dependencies** (non-Angular-core):

| Package | Version |
|---|---|
| `rxjs` | `^7.8.0` |
| `zone.js` | `^0.15.0` |
| `typescript` | `^5.6.0` |
| `eslint` | `^8.57.1` |
| `karma` | `^6.4.4` |

No CSS framework. No third-party UI library.

### 10.5 Deployment

**Backend**: Inferred Railway deployment. Env vars: `DB_URL`, `DB_USER`, `DB_PASS`, `CORS_ALLOWED_ORIGIN_PATTERNS`, `SUPERADMIN_USER`, `SUPERADMIN_PASS`, `PORT`.

**Frontend** (Railway Docker):
1. `railway.toml` triggers Dockerfile build.
2. Stage 1: `npm ci` + `npm run build` (Angular production build).
3. Stage 2: `nginx:1.27-alpine` serves static files on `PORT` (default 8080).
4. Entry script (`40-validate-backend-url.sh`) requires `BACKEND_URL` env var, forbids trailing slash.
5. Nginx `default.conf.template`: proxies `/api/*` to `${BACKEND_URL}`, serves SPA fallback (`try_files ... /index.html`).

---

## 11. Test Coverage

### 11.1 Backend Test Infrastructure

| Class | Details |
|---|---|
| `IntegrationTestSupport` | `@SpringBootTest`, `@AutoConfigureMockMvc`, `@ActiveProfiles("dev")`. Shared `PostgreSQLContainer("postgres:17-alpine")`. Dynamic datasource/superadmin props. DB reset before each test: `TRUNCATE ... RESTART IDENTITY CASCADE`. Helpers for superadmin/tenant basic auth and Telegram dev header. |
| Web MVC tests | `@WebMvcTest(..., addFilters=false)` + stubbed services + custom `TelegramInitDataValidator` bean. |
| Unit tests | Proxy stubs/test doubles for repos/services. No DB container. |

### 11.2 Backend Test Classes

| Test Class | Coverage |
|---|---|
| `CorsConfigurationIT` | CORS headers on tenant public endpoint; admin preflight without auth challenge |
| `TelegramUserArgumentResolverTest` | Resolver param support, initData path, dev fallback, invalid/no header |
| `AuditLogServiceTest` | Search page-size caps, export cap, principal extraction, enum/temporal serialization, nested JSON parsing |
| `TenantContextTest` | `TenantContext` success/failure semantics |
| `TenantSettingsServiceTest` | `getSettings`/`getCurrentTenantSettings` wiring |
| `TenantSettingsTest` | Duplicate-key override, read-only map, domain slices, default currency fallback |
| `TenantTimeServiceTest` | Timezone-aware dates, cutoff behavior, edge at exact cutoff, no cutoff |
| `PaymentQrUrlValidatorTest` | Valid/invalid URL normalization and error reasons |
| `BookingControllerWebMvcTest` | Client booking controller binding/validation for all endpoints |
| `AdminBookingControllerWebMvcTest` | Admin status update binding, `trackingUrl` validation |
| `BookingServiceTest` | Total/currency computation, transition matrix, tracking URL normalization, page-size normalization, allowed-status matrix |
| `TenantAdminAccessIT` | Tenant basic auth success/failure, superadmin bypass, missing credentials |
| `TenantIsolationIT` | Cross-tenant credential isolation, service visibility separation |
| `TenantFoodOrderConstraintsIT` | Tenant type gating, unknown/deleted service rejection, cross-tenant denial |
| `TenantAdminCatalogAndBookingIT` | End-to-end admin service create + customer booking + admin listing/filtering |
| `ServiceManagementAndValidationIT` | Config output, payment QR update/clear, service update/delete, booking status filter, payload validation, delivery-date validation by timezone |
| `BookingLifecycleIT` | Full lifecycle, ownership constraints, payment confirm, transition matrix, tracking URL, audit entries, currency immutability after tenant change |
| `BookingOptimisticLockingIT` | JPA optimistic locking conflict on stale update |
| `SuperAdminTenantControllerIT` | Auth/challenge, tenant CRUD, slug availability, duplicate slug, payment QR validation, password keep/rotate, deactivation |
| `SuperAdminAuditControllerIT` | Audit filtering/pagination/caps/auth/CSV export |
| `SuperAdminPanelIT` | Thymeleaf panel tenant CRUD forms, credential rotation, duplicate slug error, audit rendering/filter/export |
| `AdminPanelIT` | Thymeleaf panel rendering, booking/service form mutations, inline status, delete confirmation, tracking URL UI, flash messaging |
| `AuditLogIndexPlanIT` | `EXPLAIN` index usage assertions for audit queries |

### 11.3 Frontend Test Classes

| Spec File | Coverage | Telegram Mock | HTTP Mock |
|---|---|---|---|
| `telegram-init-data.interceptor.spec.ts` | Header attachment for init data, dev fallback, no-header | `SpyObj<TelegramService>` | `HttpTestingController` |
| `telegram.service.spec.ts` | `ready/expand`, initData extraction, launch param parsing, MainButton wiring, popup fallback/version | Manual fake `window.Telegram.WebApp` | none |
| `tenant-shell.component.spec.ts` | Slug change, error fallback vm, theme application | `SpyObj<TelegramService>` | `SpyObj<TenantApiService>` |
| `food-order-flow.facade.spec.ts` | Service loading, stale detail protection, repeat-order, submit validation/success, booking refresh, cancel/payment flows | `SpyObj<TelegramService>` | `SpyObj<TenantApiService>` |
| `food-order.store.spec.ts` | Selected totals/count, catalog pruning, snapshot, tenant reset | N/A | N/A |
| `food-order-home.component.spec.ts` | Facade wiring, menu controls, tab switch, bookings visibility, repeat routing, cart bar | Facade stub | none |
| `food-order-menu.component.spec.ts` | Quantity button emissions, disabled decrease, selected styling | N/A | N/A |
| `food-order-cart-bar.component.spec.ts` | Click emission, label variants | N/A | N/A |
| `food-order-checkout.component.spec.ts` | Submit/close emissions, submitting disabled, first-order hints, repeat banner | N/A | N/A |
| `food-order-bookings.component.spec.ts` | Refresh/select/repeat/cancel/payment events, grouping/ordering, address fallback, tracking link | N/A | N/A |
| `food-order-success-card.component.spec.ts` | Rendering, action events, payment confirm, tracking link | N/A | N/A |

### 11.4 Coverage Gaps

| Area | Gap |
|---|---|
| Frontend E2E | No Cypress/Playwright/Protractor tests |
| Frontend `UnsupportedFlowComponent` | No spec file |
| Frontend `AppComponent` | No spec file (trivial: renders router-outlet only) |
| Backend Telegram initData validation (prod path) | Covered via IT with dev header; no test with real HMAC computation against a real bot token |
| Offline/network error recovery | No tests for retry or offline scenarios |

---

## 12. Contract Mismatches

| Endpoint/DTO | Field/Aspect | Backend State | Frontend State | Verdict |
|---|---|---|---|---|
| `TenantConfigResponse` / `TenantConfig` | `currency`, `paymentQrUrl`, `checkoutNameHint`, `checkoutPhoneHint`, `checkoutNoteHint` | Always present in JSON (nullable) | Marked as optional (`?`) TS properties | **DRIFT** — cosmetic. JSON `null` satisfies both. No runtime issue. |
| `ServiceItem.status` | Closed vs open union | Backend `ServiceStatus` enum could evolve | Frontend: `'ACTIVE' \| 'INACTIVE' \| 'DELETED'` (closed, no `\| string`) | **DRIFT** — client API filters to `ACTIVE` only. New enum values won't reach frontend via client endpoints. Risk only if admin API is consumed by frontend in future. |
| `BookingResponse.type` | Open union | Backend `BookingType` enum: `ORDER`, `APPOINTMENT`, `REQUEST` | Frontend: open union with `\| string` | **FRONTEND_IGNORES** — acceptable. Frontend displays raw value for unknown types. |
| `BookingResponse.status` | Open union | Backend `BookingStatus` enum: 6 values | Frontend: open union with `\| string` | **FRONTEND_IGNORES** — acceptable. Frontend normalizes known values, displays raw for unknown. |
| `BookingResponse.deliveryAddress` | Nullability | Backend: nullable in DB, `@NotBlank` on creation request. Response field is nullable (legacy bookings could lack it). | Frontend: `string \| null` | **FRONTEND_IGNORES** — acceptable. Frontend handles null with fallback display. |
| `BookingResponse.currency` | Source | Backend: computed from `booking_item.currency` (first item's currency) in `BookingResponse` mapping | Frontend: `string`, also reads `TenantConfig.currency` as `fallbackCurrency` | **DRIFT** — frontend uses response `currency` when present, falls back to tenant config. Consistent for current flow since item currency is seeded from tenant config at booking time. |
| `CreateBookingRequest.deliveryDate` | Type | Backend: `LocalDate` (deserialized from ISO string) | Frontend: `string` (ISO date string) | **DRIFT** — cosmetic. Jackson handles `string` → `LocalDate`. |
| `TenantApiService` base URL | Path prefix | Backend endpoints: `/t/{slug}/...` | Frontend: `/api/t/{slug}/...` (proxy/nginx rewrites `/api/t` → `/t`) | **FRONTEND_IGNORES** — by design. Proxy layer handles rewrite. |
| `confirmBookingPayment` request body | Empty body | Backend: no `@RequestBody` annotation (empty POST) | Frontend sends `{}` as body | **DRIFT** — cosmetic. Backend ignores body. `{}` is valid empty JSON. |
| `cancelBooking` request body | Empty body | Backend: no `@RequestBody` annotation | Frontend sends `{}` | **DRIFT** — same as above. |
| Interceptor polling | Timing | Backend: no timeout on header presence; returns 401 immediately if absent | Frontend: polls up to 1500ms for init data availability before sending | **FRONTEND_IGNORES** — acceptable. Frontend polling is for SDK initialization delay; if it times out, request goes without auth and backend returns 401. |

---

## 13. Not Yet Implemented

### Backend

| Item | Location | Status |
|---|---|---|
| Appointment/request booking types | `BookingType.APPOINTMENT`, `BookingType.REQUEST` | Enum values exist; no controller/service path creates them. Current flow always sets `ORDER`. |
| Booking slot linkage | `booking.slot_id`, `Booking.slotId` | Column exists; no service/controller logic reads or writes it. |
| Direct booking→service link | `booking.service_id`, `Booking.service` | Column/relation exists; unused. Itemized model uses `booking_item.service_id`. |
| Delivery cutoff config write path | `TenantConfigKeys.CUTOFF_HOUR`, `CUTOFF_MINUTE` | Read by `TenantTimeService`; no create/update endpoint/form writes these keys. |
| `TenantConfigRepository.findByTenantIdAndKey` | Repository method | Declared but unused by current production code. |

### Frontend

| Item | Evidence | Status |
|---|---|---|
| `APPOINTMENT` tenant type UI | `TenantType` union includes it; `resolveFeatureComponent` falls through to `UnsupportedFlowComponent` | Typed but not implemented |
| `CATALOG_REQUEST` tenant type UI | Same as above | Typed but not implemented |
| Non-food route trees | Only `t/:slug` shell route exists | Not implemented |
| `tg.BackButton` | No code references | Not implemented |
| `tg.close()` | No code references | Not implemented |
| `HapticFeedback` | No code references | Not implemented |
| `themeParams` | No code references | Not implemented |
| `viewportHeight`/`viewportStableHeight` | No code references | Not implemented |
| Environment files | `src/environments/` empty | Not present |
| Route guards/resolvers | No guard/resolver classes or route config refs | Not present |

### Cross-Cutting

| Feature | Backend Status | Frontend Status |
|---|---|---|
| Telegram bot notifications (order status push) | No notification service or bot API integration | N/A |
| Appointment booking flow | Entity/enum ready; no service logic | `UnsupportedFlowComponent` fallback |
| Catalog request flow | Entity/enum ready; no service logic | `UnsupportedFlowComponent` fallback |
| Delivery cutoff configuration (write) | Read path exists in `TenantTimeService`; no write path in any endpoint/form | N/A (backend-only config) |
