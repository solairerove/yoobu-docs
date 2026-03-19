# Yoobu API MVP Scope

## Product focus

First target tenant:

- Russian-speaking home kitchen in Da Nang
- Sells next-day delivery food through Telegram
- Example products: syrniki, smetana, cottage cheese

Core v1 success criteria:

- A client places an order in Telegram without manual chat with the owner
- The owner sees the order in the admin panel
- The owner changes order status in the admin panel
- The client receives Telegram confirmation and status updates

Out of scope for v1:

- Online payments
- Multiple admins per tenant
- Appointment flow delivery
- Catalog request flow delivery
- PostgreSQL Row Level Security

## Product shape

Yoobu remains a single-instance multi-tenant SaaS for SMEs.

For MVP delivery, implementation starts with `FOOD_ORDER` only, but the backend structure must preserve the ability to add:

- `APPOINTMENT`
- `CATALOG_REQUEST`

Tenant model assumption for now:

- One tenant supports one flow type
- One tenant has one admin account

## Architecture decisions locked now

- Shared PostgreSQL database and shared schema
- Every business row is scoped by `tenant_id`
- Tenant is resolved from `/t/{slug}/...` and `/admin/{slug}/...`
- Backend stack: Java 21, Spring Boot 3.x, PostgreSQL, Flyway
- Telegram auth and notifications remain part of the backend
- Admin panel remains server-rendered MVP infrastructure
- Security: custom `OncePerRequestFilter` classes, not Spring `AuthenticationProvider`
- Service status model: `ServiceStatus` enum (ACTIVE, INACTIVE, DELETED) replaces boolean `active`

## Generalize now vs later

Generalize now:

- `tenant`
- `tenant_config`
- `service`
- `booking`
- tenant resolution infrastructure
- tenant-scoped security rules
- audit logging shape

Build only for `FOOD_ORDER` in phase 1:

- booking creation validation for food orders
- `booking_item`
- delivery date handling
- total price calculation from order items
- admin bookings list and status update
- Telegram booking confirmation and status notifications

Defer real delivery of other flows (tables NOT created yet — schema designed, migration deferred until APPOINTMENT phase):

- `slot` and `staff` — CRUD, availability, booking integration
- appointment availability rules
- request-only booking forms

## Phase completion status

### Phase 1: foundation — DONE

- Initialized Spring Boot project
- Flyway V1 baseline: `tenant`, `tenant_config`, `service`, `booking`, `booking_item`, `audit_log`
- V2: `audit_log.actor_id` changed to VARCHAR
- V3: `service.status` enum replaces boolean `active`
- Tenant context via `TenantContextFilter` (OncePerRequestFilter)
- Tenant-scoped repository conventions

Note: `slot` and `staff` tables were NOT included in V1. They remain planned for APPOINTMENT phase.

### Phase 2: food order customer API — DONE

- `GET /t/{slug}/config`
- `GET /t/{slug}/services`
- `POST /t/{slug}/bookings`
- `GET /t/{slug}/bookings/my`
- `GET /t/{slug}/bookings/{id}`
- `POST /t/{slug}/bookings/{id}/cancel`

Rules:

- Booking type is derived from tenant type, not client input
- Non-FOOD_ORDER tenants rejected with 400 before field validation
- Mandatory/ignored fields per booking type are defined in the Booking Validation Rules table in yoobu-rnd.md — backend rejects invalid requests with 400
- Food order must have at least one item
- `booking.total_price` is computed server-side from `sum(booking_item.unit_price * quantity)`
- Service prices are copied into booking items at order time
- `deliveryDate` validated against tenant-local earliest allowed date via `TenantTimeService`

### Phase 3: Telegram auth — PARTIALLY DONE

Done:

- Telegram Mini App init data validation (HMAC-SHA256)
- Dev profile fallback via `X-Telegram-User-Id` header

Not done:

- Telegram notifications on booking create (to client and owner)
- Telegram notifications on status change
- `TelegramNotifier` class does not exist yet

### Phase 4: tenant admin MVP — DONE

- HTTP Basic auth via custom `TenantBasicAuthenticationFilter`
- Superadmin credentials also accepted on tenant admin endpoints
- Thymeleaf panel: booking list (filter by date/status), booking detail, status change
- Thymeleaf panel: service create/edit/soft delete
- REST API: admin endpoints for bookings and services

### Phase 5: superadmin — DONE

- HTTP Basic auth via custom `SuperAdminBasicAuthenticationFilter`
- Tenant CRUD (REST + Thymeleaf panel)
- Slug availability check
- Tenant detail view with config map
- Does not yet have read-only view into tenant data (bookings, services)

### Phase 6: test coverage — DONE

- Integration tests on Testcontainers (PostgreSQL)
- Cross-tenant isolation tests
- Full booking lifecycle tests
- Admin auth and access tests
- Service management and validation tests
- Admin and superadmin panel tests
- Tenant-local time/cutoff unit tests

## What's next

1. Angular Mini App — `FoodOrderModule` for Telegram (step 10 in rnd)
2. Telegram notifications — on booking create and status change (step 8 in rnd)
3. First live tenant

## Design constraints for future flows

To avoid rewriting later:

- `booking.type` stays universal
- customer fields remain on `booking`
- flow-specific fields may be nullable when irrelevant
- controllers and services branch by `tenant.type`
- frontend modules stay flow-specific even if backend stays shared
