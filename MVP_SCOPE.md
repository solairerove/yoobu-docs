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

Defer real delivery of other flows (tables created in V1 migration, no business logic until food flow is stable):

- `slot` and `staff` — CRUD, availability, booking integration
- appointment availability rules
- request-only booking forms

## Recommended phase order

### Phase 1: foundation

- Initialize Spring Boot project
- Add Flyway and baseline schema for `tenant`, `tenant_config`, `service`, `booking`, `booking_item`, `slot`, `staff`, `audit_log` (slot and staff tables created empty — no logic until APPOINTMENT phase)
- Add tenant context and tenant resolver
- Add tenant-scoped repository conventions

Exit condition:

- App starts
- Migrations run
- Tenant is resolved from slug

### Phase 2: food order customer API

- `GET /t/{slug}/config`
- `GET /t/{slug}/services`
- `POST /t/{slug}/bookings`
- `GET /t/{slug}/bookings/my`
- `GET /t/{slug}/bookings/{id}`
- `POST /t/{slug}/bookings/{id}/cancel`

Rules:

- Booking type is derived from tenant type, not client input
- Mandatory/ignored fields per booking type are defined in the Booking Validation Rules table in yoobu-rnd.md — backend rejects invalid requests with 400
- Food order must have at least one item
- `booking.total_price` is computed server-side from `sum(booking_item.unit_price * quantity)`
- Service prices are copied into booking items at order time

Exit condition:

- Customer can create and inspect a food order safely inside one tenant

### Phase 3: Telegram auth and notifications

- Validate Telegram Mini App init data
- Add local dev fallback header
- Send owner notification on new order
- Send client confirmation on order creation
- Send client notification on status change

Exit condition:

- Telegram user identity reaches booking flow
- Notifications are delivered from the tenant bot

### Phase 4: tenant admin MVP

- HTTP Basic auth for one tenant admin
- Booking list by date/status
- Booking detail view
- Status update endpoint/view
- Service create/update/deactivate

Exit condition:

- Owner can manage orders without direct chat handling

### Phase 5: extension preparation

- Harden tenant isolation tests
- Define typed config migration path for security-sensitive settings
- Design `APPOINTMENT` on top of the same booking aggregate
- Add `slot` and `staff` only when food flow is stable

## Design constraints for future flows

To avoid rewriting later:

- `booking.type` stays universal
- customer fields remain on `booking`
- flow-specific fields may be nullable when irrelevant
- controllers and services branch by `tenant.type`
- frontend modules stay flow-specific even if backend stays shared

## Immediate next build slice

If implementation starts now, the smallest useful slice is:

1. Spring Boot skeleton
2. Flyway V1
3. `tenant` + `service` + `booking` + `booking_item` + `slot` + `staff` (tables only)
4. tenant resolution from slug
5. food order create/list/detail endpoints

That slice is enough to validate the domain model before adding Telegram auth and admin UI.
