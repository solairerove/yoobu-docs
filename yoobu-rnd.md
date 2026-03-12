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

### APPOINTMENT
Client selects service → selects staff (optional) → selects available time slot → confirms.
Examples: barbershop, nail salon, massage, tutoring session.

### CATALOG_REQUEST
Client selects service from list → leaves name/phone/comment → owner contacts them.
Examples: repair shop, cleaning service, legal consultation.
No cart, no slots — just a structured inquiry form.

---

## Architecture

### Stack
- **Backend:** Java 21, Spring Boot 3.x, Spring WebMVC, PostgreSQL, Flyway
- **Frontend:** Angular (separate repo `yoobu-web`), hosted on Railway
- **Hosting:** Railway — one Spring Boot service, one PostgreSQL instance
- **Telegram:** Bot API via plain HTTP (no SDK), Mini App via Telegram JS SDK from CDN

### Project structure
```
com.yoobu.api
├── config/
│   ├── AppProperties.java
│   ├── WebConfig.java
│   └── TenantContext.java         // ThreadLocal tenant storage
│
├── tenant/
│   ├── Tenant.java                // @Entity
│   ├── TenantConfig.java          // @Entity — key/value config per tenant
│   ├── TenantRepository.java
│   ├── TenantService.java
│   ├── TenantResolver.java        // HandlerInterceptor — resolves tenant from path
│   └── dto/
│       └── TenantConfigResponse.java  // public config for frontend
│
├── service/                       // replaces "product" — universal across tenant types
│   ├── Service.java               // @Entity
│   ├── ServiceRepository.java
│   ├── ServiceService.java
│   ├── ServiceController.java
│   └── dto/
│       └── ServiceResponse.java
│
├── booking/                       // universal — covers ORDER, APPOINTMENT, REQUEST
│   ├── Booking.java               // @Entity
│   ├── BookingItem.java           // @Entity — only for FOOD_ORDER
│   ├── BookingStatus.java         // Enum: NEW, CONFIRMED, DONE, CANCELLED
│   ├── BookingType.java           // Enum: ORDER, APPOINTMENT, REQUEST
│   ├── BookingRepository.java
│   ├── BookingService.java
│   ├── BookingController.java
│   └── dto/
│       ├── CreateBookingRequest.java
│       └── BookingResponse.java
│
├── slot/                          // only for APPOINTMENT tenants
│   ├── Slot.java                  // @Entity
│   ├── Staff.java                 // @Entity
│   ├── SlotRepository.java
│   ├── StaffRepository.java
│   ├── SlotService.java
│   └── SlotController.java
│
├── admin/
│   ├── AdminController.java       // Thymeleaf views
│   ├── SuperAdminController.java  // tenant management
│   └── dto/
│       └── DailySummaryResponse.java
│
├── telegram/
│   ├── TelegramInitDataValidator.java
│   ├── TelegramUser.java          // record
│   ├── TelegramPrincipal.java     // annotation
│   ├── TelegramUserArgumentResolver.java
│   ├── TelegramNotifier.java      // sendMessage via RestClient
│   └── DailySummaryScheduler.java // @Scheduled per tenant
│
└── YoobuApplication.java
```

---

## Multitenancy

### Strategy
Shared database, shared schema. Every table has `tenant_id`. Simple, cheap, scales to hundreds of tenants on one PostgreSQL instance without issues.

### Tenant resolution
Every client request comes to a path prefixed with tenant slug:
```
GET /t/dark-kitchen-dn/services
POST /t/barber-shop/bookings
GET /t/nail-studio/slots
```

`TenantResolver` (HandlerInterceptor) reads slug from path, loads `Tenant` from DB (cached), puts it into `TenantContext` (ThreadLocal). All repositories and services read `tenantId` from context — never from request parameters.

### Security
Every repository method filters by `tenant_id`. This is the most critical correctness requirement — one tenant must never see another tenant's data.

Consider PostgreSQL Row Level Security as an additional safety layer on top of application-level filtering.

### Bot token per tenant
Each tenant registers their own bot via BotFather and provides the token in admin panel. `TelegramInitDataValidator` uses the token from the current tenant context, not a global config.

---

## Database Schema

```sql
CREATE TABLE tenant (
    id                  BIGSERIAL PRIMARY KEY,
    slug                VARCHAR(100) NOT NULL UNIQUE,  -- dark-kitchen-dn
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
    -- keys: cutoff_hour, cutoff_minute, summary_chat_id, summary_cron,
    --       primary_color, logo_url, welcome_message
);

CREATE TABLE service (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT NOT NULL REFERENCES tenant(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    price               NUMERIC(10, 2),
    unit                VARCHAR(50) DEFAULT 'шт',
    duration_minutes    INT,                            -- only for APPOINTMENT
    active              BOOLEAN NOT NULL DEFAULT TRUE,
    sort_order          INT NOT NULL DEFAULT 0,
    deleted_at          TIMESTAMP WITH TIME ZONE,       -- soft delete
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

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

CREATE TABLE booking (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           BIGINT NOT NULL REFERENCES tenant(id),
    type                VARCHAR(20) NOT NULL,           -- ORDER | APPOINTMENT | REQUEST
    telegram_user_id    BIGINT NOT NULL,
    customer_name       VARCHAR(255) NOT NULL,
    customer_phone      VARCHAR(50),
    status              VARCHAR(20) NOT NULL DEFAULT 'NEW',
    note                TEXT,
    delivery_date       DATE,                           -- FOOD_ORDER only
    slot_id             BIGINT REFERENCES slot(id),     -- APPOINTMENT only
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
    action      VARCHAR(50) NOT NULL,
    actor_id    BIGINT,
    old_value   TEXT,
    new_value   TEXT,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_service_tenant ON service(tenant_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_booking_tenant ON booking(tenant_id);
CREATE INDEX idx_booking_tenant_date ON booking(tenant_id, delivery_date);
CREATE INDEX idx_booking_tenant_status ON booking(tenant_id, status);
CREATE INDEX idx_booking_telegram_user ON booking(tenant_id, telegram_user_id);
CREATE INDEX idx_slot_tenant_time ON slot(tenant_id, starts_at) WHERE is_booked = FALSE;
CREATE INDEX idx_audit_tenant_entity ON audit_log(tenant_id, entity, entity_id);
```

---

## API Structure

All client-facing endpoints prefixed with `/t/{slug}/`:

```
GET    /t/{slug}/config                        -- tenant public config for frontend
GET    /t/{slug}/services                      -- active services list
GET    /t/{slug}/slots?date=2026-03-15         -- available slots (APPOINTMENT only)
POST   /t/{slug}/bookings                      -- create booking (all types)
GET    /t/{slug}/bookings/my                   -- my bookings (by telegramUserId)
GET    /t/{slug}/bookings/{id}                 -- booking details
POST   /t/{slug}/bookings/{id}/cancel          -- cancel booking
```

Admin endpoints:
```
GET    /admin/{slug}/bookings                  -- list bookings
GET    /admin/{slug}/bookings/summary          -- daily summary
PUT    /admin/{slug}/bookings/{id}/status      -- change status
GET    /admin/{slug}/services                  -- manage services
POST   /admin/{slug}/services                  -- create service
PUT    /admin/{slug}/services/{id}             -- update service
DELETE /admin/{slug}/services/{id}             -- soft delete

GET    /superadmin/tenants                     -- list all tenants
POST   /superadmin/tenants                     -- create tenant
PUT    /superadmin/tenants/{id}                -- update tenant
```

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
Constant-time comparison via `MessageDigest.isEqual`.

### Dev profile fallback
When Spring profile contains "dev":
- If `X-Telegram-Init-Data` is absent or empty — fall back to `X-Telegram-User-Id` header
- Returns synthetic TelegramUser for local development without real Telegram

### initData expiry
Check `auth_date` field — reject if older than 24 hours. Not in MVP, add after first working tenant.

---

## Notifications

### On booking created
- To client: "Your booking #42 is confirmed. Delivery: March 15" (via sendMessage to telegram_user_id)
- To owner: "New booking from Alexey — Haircut, March 15 at 14:00" (via sendMessage to summary_chat_id)

### On status change (CONFIRMED / CANCELLED)
- To client: status update message

### Daily summary (cron)
- Per tenant, configurable cron from tenant_config
- Format same as dark kitchen implementation
- Sent to summary_chat_id from tenant_config

All via `TelegramNotifier.sendMessage(botToken, chatId, text)` — plain HTTP POST to Bot API.

---

## Admin Panel (Thymeleaf)

Two levels:

**Tenant admin** (`/admin/{slug}/`)
- Login via Telegram (owner_telegram_id check) or simple form login with credentials from tenant_config
- Dashboard: today's bookings count, upcoming slots
- Bookings list: filter by date, status
- Daily summary view
- Service management: create, edit, deactivate
- Slot management (APPOINTMENT only): open/close time slots

**Super admin** (`/superadmin/`)
- Protected by separate credentials from environment variables
- Tenant list: create, edit, activate/deactivate
- Per-tenant config editor

---

## Configuration

### application.yml
```yaml
app:
  timezone: Asia/Ho_Chi_Minh
  superadmin:
    username: ${SUPERADMIN_USER}
    password: ${SUPERADMIN_PASS}

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/yoobu
    username: ${DB_USER:yoobu}
    password: ${DB_PASS:yoobu}
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    locations: classpath:db/migration
```

### Tenant-level config keys (tenant_config table)
```
cutoff_hour          -- for FOOD_ORDER: orders after this hour → day after tomorrow
cutoff_minute
summary_chat_id      -- Telegram chat to send daily summary
summary_cron         -- cron expression for daily summary
primary_color        -- hex color for frontend theme
logo_url             -- public image URL
welcome_message      -- shown on Mini App open
```

---

## Non-functional Requirements

**Scale target:** 100+ tenants, 10k bookings/month — single Spring Boot instance handles this comfortably. No queues, no caches, no microservices needed.

**Observability:** tenant_id in every log statement. When a tenant reports a bug, find their logs in seconds.

**Data safety:** soft delete everywhere (deleted_at). Audit log for status changes and price edits. No hard deletes in production.

**Migrations:** Flyway from day one. Every schema change is versioned.

**Connection pool:** HikariCP (default in Spring Boot), set pool size explicitly. Default 10 is fine for MVP, revisit at 50+ tenants.

---

## Out of Scope (not in MVP)

- Online payments
- Delivery zones and routing
- Client-facing reviews or ratings
- Multi-language UI
- Horizontal scaling / multiple instances
- Separate DB schema per tenant
- Mobile app
- Webhooks for external integrations

---

## Implementation Order

| Step | What | Depends on |
|------|------|------------|
| 1 | Project setup, DB, Flyway V1 (tenant + service + booking + slot + audit) | — |
| 2 | Tenant resolution — TenantContext, TenantResolver interceptor | 1 |
| 3 | Service entity + REST (replaces product) | 2 |
| 4 | Booking entity + REST — FOOD_ORDER flow first | 2, 3 |
| 5 | Telegram initData validation + TelegramPrincipal | 2 |
| 6 | Admin panel Thymeleaf — bookings list, status change | 4 |
| 7 | Super admin — tenant CRUD | 2 |
| 8 | Telegram notifications — on booking, on status change | 5 |
| 9 | Slot + Staff entities + REST — APPOINTMENT flow | 4 |
| 10 | Angular frontend (yoobu-web) — FoodOrderModule first | 4, 5 |
| 11 | Angular AppointmentModule | 9, 10 |
| 12 | Angular CatalogRequestModule | 10 |
| 13 | Daily summary scheduler | 8 |
| 14 | auth_date expiry check | 5 |

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
