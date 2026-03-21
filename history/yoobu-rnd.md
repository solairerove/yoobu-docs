# Yoobu — Product & Technical Design Document

## What is Yoobu

SaaS platform for small and medium businesses that communicate with clients via Telegram.
Telegram acts as the superapp — the client is already there, no app download needed.
The business gets a Telegram Mini App and a simple admin panel.
The goal: eliminate "so what's the status?" messages and manual order-taking in personal chats.

**One line:** Yoobu connects small businesses with their customers — without the chat noise.

---

## The Problem

Small business owners (barbers, home bakers, tutors, cleaners, flower shops) accept orders and bookings manually in Telegram personal chats or group chats. This means:

- Orders get lost in message threads
- No overview of what's coming tomorrow
- Owner spends time on repetitive replies ("yes, 3pm is available")
- Client has no confirmation, no status update, writes again to check
- No history, no analytics, nothing

Existing solutions (Yclients, Booksy, Poster) are expensive, complex, and not Telegram-native.

---

## The Solution

Each business gets:
- Their own Telegram bot (registered by owner via BotFather)
- A Telegram Mini App — the storefront/booking UI for clients
- An admin panel — incoming orders/bookings, status management, daily summary
- Telegram notifications — confirmation to client, alert to owner on new booking

Owner pays a monthly subscription. Client pays nothing, needs nothing except Telegram.

---

## Business Model

**Model:** SaaS, monthly subscription per tenant

**Pricing (target):** $15–25/month per business

**5 paying tenants = $75–125/month** — covers Railway costs and generates first revenue

**Upsell later:** analytics, multiple staff accounts, custom domain, SMS fallback

---

## Target Audience

Primary: small businesses in Telegram-heavy communities
- Russian-speaking expat communities (Vietnam, Thailand, Georgia, etc.)
- Local Vietnamese small business owners
- Any market where Telegram is primary communication channel

Business types covered:
- Food & delivery (home kitchen, dark kitchen, meal prep)
- Appointment services (barbershop, nail salon, massage, tutor)
- Request-based services (repair, cleaning, consulting)

---

## Business Flow Types

Three tenant types, each maps to a different client UI flow and data model:

### FOOD_ORDER
Client selects products → adds to cart → confirms with name/phone/note → gets delivery date.
Examples: dark kitchen, home bakery, flower delivery, meal kits.

**Implementation status: DONE.** Full create/list/detail/cancel flow with validation, cutoff date logic, admin management.

### APPOINTMENT
Client selects service → selects staff (optional) → selects available time slot → confirms.
Examples: barbershop, nail salon, massage, tutoring session.

**Implementation status: NOT STARTED.** Schema designed, tables not yet created.

### CATALOG_REQUEST
Client selects service from list → leaves name/phone/comment → owner contacts them.
Examples: repair shop, cleaning service, legal consultation.
No cart, no slots — just a structured inquiry form.

**Implementation status: NOT STARTED.** Schema designed, tables not yet created.

### Booking Validation Rules by Type

Backend enforces mandatory fields per `BookingType` at creation time. Requests missing required fields are rejected with 400. Non-FOOD_ORDER tenants are rejected with 400 before field validation.

| Field | FOOD_ORDER (ORDER) | APPOINTMENT | CATALOG_REQUEST (REQUEST) |
|-------|-------------------|-------------|--------------------------|
| `customer_name` | required (`@NotBlank`) | not implemented | not implemented |
| `customer_phone` | required (`@NotBlank`) | not implemented | not implemented |
| `delivery_date` | required (`@NotNull`), validated against tenant-local earliest allowed date | not implemented | not implemented |
| `items[]` (BookingItem) | required (`@NotEmpty`), each `@Valid` | not implemented | not implemented |
| `items[].serviceId` | required (`@NotNull`), must exist with `status = ACTIVE` | not implemented | not implemented |
| `items[].quantity` | required, `@Min(1)` | not implemented | not implemented |
| `slot_id` | ignored; not in DTO | not implemented | not implemented |
| `service_id` | ignored; not in DTO | not implemented | not implemented |
| `note` | optional | not implemented | not implemented |

`total_price` on `booking` is computed server-side from `sum(booking_item.unit_price * quantity)` for FOOD_ORDER. For APPOINTMENT — will be copied from `service.price`. For REQUEST — null.

Validation source: `BookingController#createBooking`, `BookingService#createFoodOrder`, `BookingService#requireFoodOrderTenant`, `BookingService#validateDeliveryDate`.

---

## Architecture

### Stack
- **Backend:** Java 21, Spring Boot 3.x, Spring WebMVC, PostgreSQL, Flyway
- **Frontend:** Angular (separate repo `yoobu-web`), hosted on Railway
- **Hosting:** Railway — one Spring Boot service, one PostgreSQL instance
- **Telegram:** Bot API via plain HTTP (no SDK), Mini App via Telegram JS SDK from CDN

### Project structure (actual)
```
com.yoobu.api
├── config/
│   ├── AppProperties.java
│   ├── SecurityConfig.java              // Spring Security filter chains (4 chains)
│   ├── SuperAdminBasicAuthenticationFilter.java
│   ├── TenantBasicAuthenticationFilter.java
│   ├── TenantContextFilter.java         // resolves tenant from path, stores in ThreadLocal
│   ├── TenantContext.java               // static ThreadLocal<Tenant>
│   └── WebConfig.java
│
├── tenant/
│   ├── Tenant.java                      // @Entity
│   ├── TenantConfig.java               // @Entity — key/value config per tenant
│   ├── TenantRepository.java
│   ├── TenantService.java
│   ├── TenantTimeService.java           // tenant-local date and cutoff calculations
│   └── dto/
│       ├── TenantConfigResponse.java    // public config for frontend
│       ├── TenantDetailResponse.java    // superadmin: full tenant info + config map
│       ├── TenantSummaryResponse.java   // superadmin: tenant list item
│       ├── TenantSlugAvailabilityResponse.java
│       ├── CreateTenantRequest.java
│       └── UpdateTenantRequest.java
│
├── service/
│   ├── Service.java                     // @Entity
│   ├── ServiceStatus.java              // Enum: ACTIVE, INACTIVE, DELETED
│   ├── ServiceRepository.java
│   ├── AdminCatalogService.java         // admin service management
│   ├── ServiceController.java           // public: GET /t/{slug}/services
│   ├── AdminCatalogController.java      // admin REST: /admin/{slug}/services
│   └── dto/
│       ├── ServiceResponse.java
│       └── AdminUpsertServiceRequest.java
│
├── booking/
│   ├── Booking.java                     // @Entity
│   ├── BookingItem.java                 // @Entity — only for FOOD_ORDER
│   ├── BookingStatus.java              // Enum: NEW, CONFIRMED, DONE, CANCELLED
│   ├── BookingType.java                // Enum: ORDER, APPOINTMENT, REQUEST
│   ├── BookingRepository.java
│   ├── BookingService.java
│   ├── BookingController.java           // client: /t/{slug}/bookings
│   ├── AdminBookingController.java      // admin REST: /admin/{slug}/bookings
│   └── dto/
│       ├── CreateBookingRequest.java
│       ├── BookingItemRequest.java
│       ├── BookingResponse.java
│       ├── BookingItemResponse.java
│       └── UpdateStatusRequest.java
│
├── admin/
│   ├── AdminPanelController.java        // Thymeleaf: /admin/{slug}/panel/**
│   ├── SuperAdminController.java        // REST: /superadmin/tenants
│   ├── SuperAdminPanelController.java   // Thymeleaf: /superadmin/panel/**
│   └── dto/
│       ├── BookingStatusForm.java       // MVC form model
│       ├── ServiceForm.java             // MVC form model
│       └── SuperAdminTenantForm.java    // MVC form model
│
├── audit/
│   ├── AuditLog.java                    // @Entity
│   ├── AuditLogRepository.java
│   └── AuditService.java
│
├── telegram/
│   ├── TelegramInitDataValidator.java
│   ├── TelegramUser.java               // record
│   ├── TelegramPrincipal.java          // annotation
│   └── TelegramUserArgumentResolver.java
│
└── YoobuApplication.java
```

Classes not yet implemented: `TelegramNotifier`, `DailySummaryScheduler`, `DailySummaryResponse`, slot/staff packages.

---

## Multitenancy

### Strategy
Shared database, shared schema. Every table has `tenant_id`. Simple, cheap, scales to hundreds of tenants on one PostgreSQL instance without issues.

### Tenant resolution
Every client and admin request comes to a path prefixed with tenant slug:
```
GET  /t/dark-kitchen-dn/services
POST /t/barber-shop/bookings
GET  /admin/dark-kitchen-dn/bookings
```

`TenantContextFilter` (OncePerRequestFilter, attached to `/t/*/**` and `/admin/*/**` security chains) parses the slug from URI segments, loads `Tenant` via `TenantRepository.findBySlugAndActiveTrue(slug)`, stores it in `TenantContext` (static `ThreadLocal<Tenant>`).

Failure modes: 400 if slug segment missing, 404 if tenant not found or inactive.

All repositories and services read tenant from `TenantContext` — never from request parameters.

### Security
Every repository method filters by `tenant_id`. This is the most critical correctness requirement — one tenant must never see another tenant's data.

Integration tests (`TenantIsolationIT`) verify cross-tenant isolation for services and admin auth realms.

Consider PostgreSQL Row Level Security as an additional safety layer on top of application-level filtering.

### Bot token per tenant
Each tenant registers their own bot via BotFather and provides the token via superadmin panel. `TelegramInitDataValidator` uses the token from the current tenant context, not a global config.

---

## Database Schema

Current state after migrations V1–V3.

```sql
CREATE TABLE tenant (
    id                  BIGSERIAL PRIMARY KEY,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    name                VARCHAR(255) NOT NULL,
    type                VARCHAR(30) NOT NULL,           -- FOOD_ORDER | APPOINTMENT | CATALOG_REQUEST
    bot_token           VARCHAR(255),
    owner_telegram_id   BIGINT,
    timezone            VARCHAR(50) NOT NULL DEFAULT 'Asia/Ho_Chi_Minh',
    active              BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE tenant_config (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   BIGINT NOT NULL REFERENCES tenant(id),
    key         VARCHAR(100) NOT NULL,
    value       TEXT,
    UNIQUE (tenant_id, key)
);

CREATE TABLE service (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT NOT NULL REFERENCES tenant(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    price               NUMERIC(10, 2),
    unit                VARCHAR(50) DEFAULT 'шт',
    duration_minutes    INT,                            -- only for APPOINTMENT
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE | INACTIVE | DELETED
    sort_order          INT NOT NULL DEFAULT 0,
    deleted_at          TIMESTAMP WITH TIME ZONE,       -- set on soft delete
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE booking (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT NOT NULL REFERENCES tenant(id),
    type                VARCHAR(20) NOT NULL,           -- ORDER | APPOINTMENT | REQUEST
    telegram_user_id    BIGINT NOT NULL,
    customer_name       VARCHAR(255) NOT NULL,
    customer_phone      VARCHAR(50),
    status              VARCHAR(20) NOT NULL DEFAULT 'NEW',
    note                TEXT,
    total_price         NUMERIC(10, 2),                 -- computed server-side
    delivery_date       DATE,                           -- FOOD_ORDER only
    slot_id             BIGINT,                         -- APPOINTMENT only (FK added when slot table created)
    service_id          BIGINT REFERENCES service(id),  -- APPOINTMENT and REQUEST
    deleted_at          TIMESTAMP WITH TIME ZONE,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE booking_item (
    id          BIGSERIAL PRIMARY KEY,
    booking_id  BIGINT NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    service_id  BIGINT NOT NULL REFERENCES service(id),
    quantity    INT NOT NULL CHECK (quantity > 0),
    unit_price  NUMERIC(10, 2) NOT NULL
);

CREATE TABLE audit_log (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   BIGINT NOT NULL,
    entity      VARCHAR(50) NOT NULL,
    entity_id   BIGINT NOT NULL,
    action      VARCHAR(50) NOT NULL,       -- CREATE, UPDATE, UPDATE_STATUS, DELETE, CANCEL
    actor_id    VARCHAR(255),               -- changed from BIGINT in V2
    old_value   TEXT,                        -- JSON string
    new_value   TEXT,                        -- JSON string
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_service_tenant ON service(tenant_id, status, sort_order, id) WHERE status <> 'DELETED';
CREATE INDEX idx_booking_tenant ON booking(tenant_id);
CREATE INDEX idx_booking_tenant_date ON booking(tenant_id, delivery_date);
CREATE INDEX idx_booking_tenant_status ON booking(tenant_id, status);
CREATE INDEX idx_booking_telegram_user ON booking(tenant_id, telegram_user_id);
CREATE INDEX idx_audit_tenant_entity ON audit_log(tenant_id, entity, entity_id);
```

### Tables not yet created (planned for APPOINTMENT phase)

```sql
CREATE TABLE staff (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT NOT NULL REFERENCES tenant(id),
    name                VARCHAR(255) NOT NULL,
    telegram_user_id    BIGINT,
    active              BOOLEAN NOT NULL DEFAULT TRUE,
    deleted_at          TIMESTAMP WITH TIME ZONE
);

CREATE TABLE slot (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   BIGINT NOT NULL REFERENCES tenant(id),
    staff_id    BIGINT REFERENCES staff(id),
    starts_at   TIMESTAMP WITH TIME ZONE NOT NULL,
    ends_at     TIMESTAMP WITH TIME ZONE NOT NULL,
    is_booked   BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_slot_tenant_time ON slot(tenant_id, starts_at) WHERE is_booked = FALSE;
```

### Flyway Migration History

| Version | Filename | Summary |
|---------|----------|---------|
| V1 | `V1__baseline_schema.sql` | Creates `tenant`, `tenant_config`, `service`, `booking`, `booking_item`, `audit_log`; adds baseline indexes. |
| V2 | `V2__audit_actor_id_as_string.sql` | Changes `audit_log.actor_id` from `BIGINT` to `VARCHAR(255)`. |
| V3 | `V3__service_status_model.sql` | Adds `service.status`, backfills from `active`/`deleted_at`, adds new partial index, drops `service.active`. |

---

## Enums

| Enum | Values | Notes |
|------|--------|-------|
| `TenantType` | `FOOD_ORDER`, `APPOINTMENT`, `CATALOG_REQUEST` | Stored on `tenant.type` |
| `ServiceStatus` | `ACTIVE`, `INACTIVE`, `DELETED` | Replaced boolean `active` in V3 |
| `BookingStatus` | `NEW`, `CONFIRMED`, `DONE`, `CANCELLED` | |
| `BookingType` | `ORDER`, `APPOINTMENT`, `REQUEST` | Derived from `TenantType`, not sent by client |

---

## API Structure

### Client API (`/t/{slug}/...`)

| Method | Path | Request Body | Response Body | Auth | Notes |
|--------|------|-------------|---------------|------|-------|
| GET | `/t/{slug}/config` | — | `TenantConfigResponse` | none | 404 if tenant inactive or missing |
| GET | `/t/{slug}/services` | — | `List<ServiceResponse>` | none | Only `status = ACTIVE` |
| POST | `/t/{slug}/bookings` | `CreateBookingRequest` | `BookingResponse` | Telegram | FOOD_ORDER tenants only; 400 for other types |
| GET | `/t/{slug}/bookings/my` | — | `List<BookingResponse>` | Telegram | Filtered by current Telegram user |
| GET | `/t/{slug}/bookings/{bookingId}` | — | `BookingResponse` | Telegram | Only caller's own booking |
| POST | `/t/{slug}/bookings/{bookingId}/cancel` | — | `BookingResponse` | Telegram | 409 if status is DONE |

"Telegram" auth = `@TelegramPrincipal` resolved from `X-Telegram-Init-Data` header. In `dev` profile, falls back to `X-Telegram-User-Id` header.

### Admin REST API (`/admin/{slug}/...`)

| Method | Path | Request Body | Response Body | Auth | Notes |
|--------|------|-------------|---------------|------|-------|
| GET | `/admin/{slug}/services` | — | `List<ServiceResponse>` | HTTP Basic | All non-deleted services |
| POST | `/admin/{slug}/services` | `AdminUpsertServiceRequest` | `ServiceResponse` | HTTP Basic | 201 Created |
| PUT | `/admin/{slug}/services/{serviceId}` | `AdminUpsertServiceRequest` | `ServiceResponse` | HTTP Basic | Rejects `status = DELETED` with 400 |
| DELETE | `/admin/{slug}/services/{serviceId}` | — | — | HTTP Basic | 204; soft delete via `status = DELETED` |
| GET | `/admin/{slug}/bookings` | — | `List<BookingResponse>` | HTTP Basic | Query params: `status`, `deliveryDate` |
| GET | `/admin/{slug}/bookings/{bookingId}` | — | `BookingResponse` | HTTP Basic | |
| PUT | `/admin/{slug}/bookings/{bookingId}/status` | `UpdateStatusRequest` | `BookingResponse` | HTTP Basic | No transition validation |

### Admin Panel (Thymeleaf, `/admin/{slug}/panel/...`)

| Method | Path | Notes |
|--------|------|-------|
| GET | `/admin/{slug}/panel` | Redirects to `/admin/{slug}/panel/bookings` |
| GET | `/admin/{slug}/panel/bookings` | Query params: `status`, `deliveryDate` |
| GET | `/admin/{slug}/panel/bookings/{bookingId}` | Detail + status change form |
| POST | `/admin/{slug}/panel/bookings/{bookingId}/status` | Form submit → status change |
| GET | `/admin/{slug}/panel/services` | Service list |
| GET | `/admin/{slug}/panel/services/new` | Create form |
| POST | `/admin/{slug}/panel/services` | Create submit |
| GET | `/admin/{slug}/panel/services/{serviceId}/edit` | Edit form |
| POST | `/admin/{slug}/panel/services/{serviceId}` | Edit submit |
| POST | `/admin/{slug}/panel/services/{serviceId}/delete` | Soft delete |

### Superadmin REST API (`/superadmin/...`)

| Method | Path | Request Body | Response Body | Auth | Notes |
|--------|------|-------------|---------------|------|-------|
| GET | `/superadmin/tenants` | — | `List<TenantSummaryResponse>` | HTTP Basic | |
| GET | `/superadmin/tenants/{tenantId}` | — | `TenantDetailResponse` | HTTP Basic | Includes config map |
| GET | `/superadmin/tenants/slug-availability?slug=...` | — | `TenantSlugAvailabilityResponse` | HTTP Basic | |
| POST | `/superadmin/tenants` | `CreateTenantRequest` | `TenantSummaryResponse` | HTTP Basic | Creates tenant + config keys |
| PUT | `/superadmin/tenants/{tenantId}` | `UpdateTenantRequest` | `TenantSummaryResponse` | HTTP Basic | Slug immutable |

### Superadmin Panel (Thymeleaf, `/superadmin/panel/...`)

| Method | Path | Notes |
|--------|------|-------|
| GET | `/superadmin/panel` | Redirects to `/superadmin/panel/tenants` |
| GET | `/superadmin/panel/tenants` | Tenant list |
| GET | `/superadmin/panel/tenants/{tenantId}` | Tenant detail |
| GET | `/superadmin/panel/tenants/new` | Create form |
| POST | `/superadmin/panel/tenants` | Create submit; validates duplicate slug, non-blank password |
| GET | `/superadmin/panel/tenants/{tenantId}/edit` | Edit form |
| POST | `/superadmin/panel/tenants/{tenantId}` | Edit submit |

### Endpoints not yet implemented

| Endpoint | Phase |
|----------|-------|
| `GET /t/{slug}/slots?date=...` | APPOINTMENT |
| `GET /admin/{slug}/bookings/summary` | Daily summary |
| `POST /admin/{slug}/slots` | APPOINTMENT |
| `DELETE /admin/{slug}/slots/{id}` | APPOINTMENT |
| `GET /admin/{slug}/staff` | APPOINTMENT |
| `POST /admin/{slug}/staff` | APPOINTMENT |
| `PUT /admin/{slug}/staff/{id}` | APPOINTMENT |
| `DELETE /admin/{slug}/staff/{id}` | APPOINTMENT |

---

## DTO Contracts

### Client-facing DTOs

**TenantConfigResponse** — `GET /t/{slug}/config`
```json
{
  "slug": "dark-kitchen-dn",
  "name": "Dark Kitchen Da Nang",
  "type": "FOOD_ORDER",
  "primaryColor": "#FF6B35",
  "logoUrl": "https://...",
  "welcomeMessage": "Welcome! Order by 14:00 for same-day delivery."
}
```

**ServiceResponse** — `GET /t/{slug}/services`
```json
{
  "id": 1,
  "name": "Борщ 0.5л",
  "description": "Классический, со сметаной",
  "price": 85000.00,
  "unit": "шт",
  "durationMinutes": null,
  "sortOrder": 0,
  "status": "ACTIVE"
}
```
`durationMinutes` present only for APPOINTMENT tenants. `unit` present only for FOOD_ORDER tenants. `status` is `ServiceStatus` enum. Frontend ignores irrelevant fields based on tenant type.

**CreateBookingRequest** — `POST /t/{slug}/bookings`

Currently only FOOD_ORDER is implemented. Other tenant types are rejected with 400 before field validation.

```json
{
  "customerName": "Алексей",
  "customerPhone": "+84901234567",
  "deliveryDate": "2026-03-15",
  "note": "без лука",
  "items": [
    { "serviceId": 1, "quantity": 2 },
    { "serviceId": 3, "quantity": 1 }
  ]
}
```

| Field | Type | Validation |
|-------|------|-----------|
| `customerName` | `String` | `@NotBlank` |
| `customerPhone` | `String` | `@NotBlank` |
| `deliveryDate` | `LocalDate` | `@NotNull`; also validated against tenant-local earliest allowed date |
| `note` | `String` | optional |
| `items` | `List<BookingItemRequest>` | `@NotEmpty`, each `@Valid` |
| `items[].serviceId` | `Long` | `@NotNull`; must exist with `status = ACTIVE` in current tenant |
| `items[].quantity` | `int` | `@Min(1)` |

`type` is NOT sent by client — derived from `tenant.type` on backend. Always `ORDER` for FOOD_ORDER tenants.

**BookingResponse** — `GET /t/{slug}/bookings/{id}`, items in `/bookings/my` list
```json
{
  "id": 42,
  "type": "ORDER",
  "status": "CONFIRMED",
  "customerName": "Алексей",
  "totalPrice": 255000.00,
  "deliveryDate": "2026-03-15",
  "note": "без лука",
  "items": [
    { "serviceName": "Борщ 0.5л", "quantity": 2, "unitPrice": 85000.00 },
    { "serviceName": "Хлеб", "quantity": 1, "unitPrice": 25000.00 }
  ],
  "createdAt": "2026-03-14T10:30:00+07:00"
}
```
Fields `slotStartsAt`, `slotEndsAt`, `serviceName`, `staffName` are not yet present — will be added when APPOINTMENT flow is implemented.

### Admin DTOs

**AdminUpsertServiceRequest** — `POST /admin/{slug}/services`, `PUT /admin/{slug}/services/{serviceId}`
```json
{
  "name": "Борщ 0.5л",
  "description": "Классический, со сметаной",
  "price": 85000.00,
  "unit": "шт",
  "durationMinutes": null,
  "sortOrder": 0,
  "status": "ACTIVE"
}
```

| Field | Type | Validation | Default |
|-------|------|-----------|---------|
| `name` | `String` | `@NotBlank` | — |
| `description` | `String` | optional | — |
| `price` | `BigDecimal` | `@NotNull` | — |
| `unit` | `String` | optional | `"шт"` if null/blank |
| `durationMinutes` | `Integer` | optional | — |
| `sortOrder` | `Integer` | optional | `0` if null |
| `status` | `ServiceStatus` | optional | `ACTIVE` if null; `DELETED` rejected with 400 |

**UpdateStatusRequest** — `PUT /admin/{slug}/bookings/{id}/status`
```json
{
  "status": "CONFIRMED"
}
```

### Superadmin DTOs

**CreateTenantRequest** — `POST /superadmin/tenants`
```json
{
  "slug": "dark-kitchen-dn",
  "name": "Dark Kitchen Da Nang",
  "type": "FOOD_ORDER",
  "botToken": "123456:ABC-DEF...",
  "ownerTelegramId": 123456789,
  "timezone": "Asia/Ho_Chi_Minh",
  "primaryColor": "#FF6B35",
  "logoUrl": "https://...",
  "welcomeMessage": "Welcome!",
  "adminUsername": "admin",
  "adminPassword": "secret"
}
```

| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `slug` | `String` | `@NotBlank` | Uniqueness checked in service |
| `name` | `String` | `@NotBlank` | |
| `type` | `TenantType` | `@NotNull` | |
| `botToken` | `String` | optional | Blank trimmed to null |
| `ownerTelegramId` | `Long` | optional | |
| `timezone` | `String` | optional | Defaults to `Asia/Ho_Chi_Minh` |
| `primaryColor` | `String` | optional | Config key `primary_color` |
| `logoUrl` | `String` | optional | Config key `logo_url` |
| `welcomeMessage` | `String` | optional | Config key `welcome_message` |
| `adminUsername` | `String` | `@NotBlank` | Config key `admin_username` |
| `adminPassword` | `String` | `@NotBlank` | BCrypt-hashed → config key `admin_password` |

**UpdateTenantRequest** — `PUT /superadmin/tenants/{tenantId}`

Same fields as `CreateTenantRequest` except:
- `slug` is not present (immutable)
- `adminPassword` is optional — blank keeps previous hash
- `active` (`Boolean`, `@NotNull`) — `false` makes tenant inaccessible via `TenantContextFilter`
- Blank config values (primaryColor, logoUrl, welcomeMessage) remove the config key

**TenantSummaryResponse** — list item in `GET /superadmin/tenants`
```json
{
  "id": 1,
  "slug": "dark-kitchen-dn",
  "name": "Dark Kitchen Da Nang",
  "type": "FOOD_ORDER",
  "active": true,
  "timezone": "Asia/Ho_Chi_Minh",
  "createdAt": "2026-03-10T12:00:00+07:00"
}
```

**TenantDetailResponse** — `GET /superadmin/tenants/{tenantId}`
```json
{
  "id": 1,
  "slug": "dark-kitchen-dn",
  "name": "Dark Kitchen Da Nang",
  "type": "FOOD_ORDER",
  "active": true,
  "timezone": "Asia/Ho_Chi_Minh",
  "botToken": "123456:ABC-DEF...",
  "ownerTelegramId": 123456789,
  "createdAt": "2026-03-10T12:00:00+07:00",
  "config": {
    "admin_username": "admin",
    "primary_color": "#FF6B35",
    "logo_url": "https://...",
    "welcome_message": "Welcome!"
  }
}
```
Note: `admin_password` hash is included in config map. Consider filtering in a future iteration.

**TenantSlugAvailabilityResponse** — `GET /superadmin/tenants/slug-availability?slug=...`
```json
{
  "slug": "dark-kitchen-dn",
  "available": false
}
```

### MVC Form Models (Thymeleaf only, not JSON)

**BookingStatusForm** — `POST /admin/{slug}/panel/bookings/{bookingId}/status`

| Field | Type | Validation |
|-------|------|-----------|
| `status` | `BookingStatus` | `@NotNull` |

**ServiceForm** — `POST /admin/{slug}/panel/services`, `POST /admin/{slug}/panel/services/{serviceId}`

| Field | Type | Validation | Default |
|-------|------|-----------|---------|
| `name` | `String` | `@NotBlank` | — |
| `description` | `String` | optional | — |
| `price` | `BigDecimal` | `@NotNull` | — |
| `unit` | `String` | optional | — |
| `durationMinutes` | `Integer` | optional | — |
| `sortOrder` | `Integer` | optional | `0` |
| `status` | `ServiceStatus` | `@NotNull` | `ACTIVE` |

**SuperAdminTenantForm** — `POST /superadmin/panel/tenants`, `POST /superadmin/panel/tenants/{tenantId}`

| Field | Type | Validation | Default |
|-------|------|-----------|---------|
| `slug` | `String` | `@NotBlank` | — |
| `name` | `String` | `@NotBlank` | — |
| `type` | `TenantType` | `@NotNull` | `FOOD_ORDER` |
| `botToken` | `String` | optional | — |
| `ownerTelegramId` | `Long` | optional | — |
| `timezone` | `String` | optional | `Asia/Ho_Chi_Minh` |
| `primaryColor` | `String` | optional | — |
| `logoUrl` | `String` | optional | — |
| `welcomeMessage` | `String` | optional | — |
| `adminUsername` | `String` | `@NotBlank` | — |
| `adminPassword` | `String` | optional | Controller requires non-blank on create |
| `active` | `boolean` | — | `true` |

### DTOs not yet implemented

| DTO | Phase |
|-----|-------|
| `DailySummaryResponse` | Daily summary feature |
| `SlotResponse` | APPOINTMENT |
| `CreateSlotRequest` | APPOINTMENT |

---

## Security

### Filter chains (SecurityConfig)

Four filter chains, ordered by specificity:

1. **`/superadmin/**`** — `SuperAdminBasicAuthenticationFilter`; credentials from `app.superadmin.username` / `app.superadmin.password`. CSRF disabled, form login disabled, anonymous disabled.

2. **`/admin/*/**`** — `TenantContextFilter` → `TenantBasicAuthenticationFilter`; checks credentials against tenant config keys `admin_username` / `admin_password` (BCrypt). Superadmin credentials also accepted. CSRF disabled, form login disabled, anonymous disabled.

3. **`/t/*/**`** — `TenantContextFilter`; all requests permitted at Spring Security level. Telegram auth enforced only on controller parameters with `@TelegramPrincipal`.

4. **`/**`** (default) — permit all, CSRF disabled, anonymous enabled.

Authentication is performed inside custom `OncePerRequestFilter` classes, not via Spring `AuthenticationProvider`.

CORS: Not implemented.

---

## Frontend (yoobu-web, Angular)

Separate repository. Hosted on Railway as a separate service.

### Tenant resolution
App reads slug from URL path: `yoobu.io/t/dark-kitchen-dn`
On init: `GET /t/{slug}/config` — loads tenant name, type, colors, logo.
Based on `type`, Angular lazy-loads the correct feature module.

### Feature modules
```
AppModule
├── TenantShellModule          -- resolves tenant config, sets theme
├── FoodOrderModule            -- FOOD_ORDER flow: catalog, cart, checkout
├── AppointmentModule          -- APPOINTMENT flow: services, staff, slots, confirm
└── CatalogRequestModule       -- CATALOG_REQUEST flow: services, request form
```

Shared across all modules: confirmation screen, my bookings, order detail.

### Telegram integration
```javascript
const tg = window.Telegram.WebApp;
tg.ready();
tg.expand();
// initData sent as X-Telegram-Init-Data header on every API request
// tg.MainButton used as primary action button
// tg.showAlert / tg.showConfirm for dialogs
// tg.close() after successful booking
```

---

## Telegram Auth

### Validation (backend)
`TelegramInitDataValidator` — HMAC-SHA256, no external libraries.
Uses bot token from current tenant context (not global config).
Validates `hash` and `user.id` from init data. Returns 401 on any failure.

### Dev profile fallback
When Spring profile contains "dev":
- If `X-Telegram-Init-Data` is absent or empty — fall back to `X-Telegram-User-Id` header
- Returns synthetic `TelegramUser(id, "Dev", "User", null)` for local development

### initData expiry
Check `auth_date` field — reject if older than 24 hours. Not implemented, add after first working tenant.

---

## Notifications

**Implementation status: NOT STARTED.** `TelegramNotifier` and `DailySummaryScheduler` classes do not exist yet.

### Planned: On booking created
- To client: "Your booking #42 is confirmed. Delivery: March 15" (via sendMessage to telegram_user_id)
- To owner: "New booking from Alexey — 2× Борщ, March 15" (via sendMessage to summary_chat_id)

### Planned: On status change (CONFIRMED / CANCELLED)
- To client: status update message

### Planned: Daily summary (cron)
- Per tenant, configurable cron from tenant_config
- Sent to summary_chat_id from tenant_config

All via `TelegramNotifier.sendMessage(botToken, chatId, text)` — plain HTTP POST to Bot API. Synchronous, no retry, no outbox.

---

## Admin Panel (Thymeleaf)

Two levels, both implemented:

**Tenant admin** (`/admin/{slug}/panel/`)
- Auth: custom `TenantBasicAuthenticationFilter`. Credentials from `tenant_config` keys `admin_username` and `admin_password` (bcrypt hash). Superadmin credentials also work.
- Bookings list: filter by date, status
- Booking detail: view + status change form
- Service management: create, edit, soft delete
- No dashboard/summary view yet

**Super admin** (`/superadmin/panel/`)
- Auth: custom `SuperAdminBasicAuthenticationFilter`. Credentials from `app.superadmin.username` / `app.superadmin.password`.
- Tenant list
- Tenant detail view
- Tenant create/edit forms with slug availability check
- Does not yet have read-only view into tenant data (bookings, services)

---

## Configuration

### application.yml (actual)
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
  superadmin:
    username: ${SUPERADMIN_USER:superadmin}
    password: ${SUPERADMIN_PASS:superadmin}

server:
  error:
    include-message: always

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} %-5level [%thread] %logger - %msg%n"
```

### Tenant-level config keys (tenant_config table)
```
admin_username       -- HTTP Basic Auth username for /admin/{slug}/**
admin_password       -- HTTP Basic Auth password (bcrypt hash)
primary_color        -- hex color for frontend theme
logo_url             -- public image URL
welcome_message      -- shown on Mini App open
payment_qr_url       -- QR image URL shown to client after order (planned)
cutoff_hour          -- for FOOD_ORDER: orders after this hour → next available date
cutoff_minute
summary_chat_id      -- Telegram chat to send daily summary (planned)
summary_cron         -- cron expression for daily summary (planned)
```

---

## Non-functional Requirements

**Scale target:** 100+ tenants, 10k bookings/month — single Spring Boot instance handles this comfortably. No queues, no caches, no microservices needed.

**Observability:** tenant_id in every log statement. When a tenant reports a bug, find their logs in seconds.

**Data safety:** soft delete via `ServiceStatus.DELETED` + `deleted_at` for services. Audit log for status changes and CRUD operations. No hard deletes in production.

**Migrations:** Flyway from day one. Every schema change is versioned. Currently at V3.

**Connection pool:** HikariCP (default in Spring Boot), set pool size explicitly. Default 10 is fine for MVP, revisit at 50+ tenants.

**Hibernate:** `open-in-view: false`, time zone set to UTC. All dates stored in UTC, converted to tenant timezone in application layer via `TenantTimeService`.

---

## Test Coverage

| Test class | Covers |
|------------|--------|
| `SuperAdminTenantControllerIT` | Superadmin auth, tenant CRUD, slug availability, duplicate slug, optional field clearing, password retention, inactive tenant |
| `TenantAdminAccessIT` | Tenant admin auth success/failure, missing credentials, superadmin access to tenant admin endpoints |
| `TenantIsolationIT` | Cross-tenant isolation for services and admin auth realms |
| `TenantAdminCatalogAndBookingIT` | Happy path: service creation → public catalog → booking |
| `TenantFoodOrderConstraintsIT` | FOOD_ORDER-only restriction, deleted service behavior, unknown service rejection |
| `BookingLifecycleIT` | Create/list/read/cancel, admin read/update, audit log, cancel conflict on DONE |
| `ServiceManagementAndValidationIT` | Public config, service update/delete, booking filtering, bean validation, delivery-date cutoff |
| `AdminPanelIT` | Tenant admin Thymeleaf routes and form submissions |
| `SuperAdminPanelIT` | Superadmin Thymeleaf routes and form submissions |
| `TenantTimeServiceTest` | Tenant-local date and cutoff calculations |

All integration tests run against PostgreSQL via Testcontainers.

---

## Out of Scope (not in MVP)

- Online payments (payment_qr_url for manual flow only)
- Delivery zones and routing
- Client-facing reviews or ratings
- Multi-language UI
- Horizontal scaling / multiple instances
- Separate DB schema per tenant
- Mobile app
- Webhooks for external integrations
- CORS configuration

---

## Implementation Status

| Step | What | Status |
|------|------|--------|
| 1 | Project setup, DB, Flyway V1–V3 | DONE |
| 2 | Tenant resolution — TenantContextFilter, TenantContext | DONE |
| 3 | Service entity + REST + admin CRUD | DONE |
| 4 | Booking entity + REST — FOOD_ORDER flow | DONE |
| 5 | Telegram initData validation + TelegramPrincipal | DONE |
| 6 | Admin panel Thymeleaf — bookings, services | DONE |
| 7 | Super admin — tenant CRUD + panel | DONE |
| 8 | Telegram notifications — on booking, on status change | NOT STARTED |
| 9 | Slot + Staff entities + REST — APPOINTMENT flow | NOT STARTED |
| 10 | Angular frontend (yoobu-web) — FoodOrderModule first | NEXT |
| 11 | Angular AppointmentModule | BLOCKED by 9 |
| 12 | Angular CatalogRequestModule | BLOCKED by 10 |
| 13 | Daily summary scheduler | BLOCKED by 8 |
| 14 | auth_date expiry check | DEFERRED |

---

## Schema vs Entity Drift

Known differences between Flyway SQL and JPA entity definitions:

- `tenant.name`: SQL `VARCHAR(255)`, entity does not declare length
- `tenant.timezone`, `tenant.active`, `tenant.created_at`: SQL defines defaults, entity does not
- `tenant_config`: SQL has unique (`tenant_id`, `key`), entity does not declare constraint
- `service.unit`, `service.sort_order`, `service.status`, `service.created_at`, `service.updated_at`: SQL defines defaults, entity does not
- `booking.customer_phone`: SQL allows null, DTO requires non-blank in FOOD_ORDER flow
- `booking.slot_id`, `booking.service_id`: present in SQL and entity, unused by implemented code
- `booking_item.quantity`: SQL has check constraint, entity does not
- `booking_item.booking_id`: SQL has `ON DELETE CASCADE`, entity does not model cascade

None of these cause runtime issues — SQL constraints are the safety net. Entity-side annotations can be aligned later.

---

## Repository Structure

```
yoobu-api      — Java 21 + Spring Boot 3.x (this document)
yoobu-web      — Angular frontend
yoobu-docs     — architecture decisions, ERD, this document
yoobu-kt-spring — R&D: Kotlin + Spring Boot port
yoobu-kt-ktor   — R&D: Kotlin + Ktor port
```

License: BSD 2-Clause
