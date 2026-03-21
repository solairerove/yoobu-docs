# Yoobu API Tech Debt Register

This file tracks deliberate deferrals for MVP.

## Multitenancy safety

### PostgreSQL Row Level Security

Status:

- Deferred until after MVP foundation is stable

Why deferred:

- Application-level tenant filtering is simpler to ship initially
- RLS adds operational and query-debugging complexity early

Risk:

- A missed tenant filter in application code could expose cross-tenant data

Mitigation before RLS:

- Every tenant-owned table must include `tenant_id`
- Repository methods must always filter by tenant
- Integration tests must cover cross-tenant isolation
- Logs must include tenant identifiers

Later implementation note:

- Evaluate session-variable or connection-setting based policies
- Add RLS only after repository/query patterns settle

### Tenant filter enforcement testing

Status:

- Convention defined, automated enforcement missing

Convention (defined in yoobu-rnd.md):

- TenantResolver interceptor resolves tenant from slug, stores in TenantContext (ThreadLocal)
- All repositories and services read tenantId from TenantContext, never from request parameters

Debt:

- No integration tests that fail if a query omits tenant_id filter
- No compile-time or startup-time check that all repository methods are tenant-scoped

## Configuration model

### Overuse of `tenant_config`

Status:

- Accepted for MVP

Risk:

- Security, scheduling, branding, and business settings become mixed in one generic key/value table

Likely follow-up:

- Move security-sensitive or operationally critical settings to typed columns or dedicated tables

### Tenant admin credentials in tenant_config

Status:

- Decided: Spring Security HTTP Basic Auth. Credentials stored as `admin_username` and `admin_password` (bcrypt hash) in `tenant_config`. Separate filter chain for `/admin/{slug}/**`.

Risk:

- No password rotation mechanism
- No audit trail for admin logins
- Single credential set per tenant — no multi-user admin support

Likely follow-up:

- Introduce `admin_user` table with tenant-scoped identities and roles
- Optional: Telegram-based admin login via `owner_telegram_id`

## Product scope debt

### Single flow shipped first

Status:

- Intentional phased delivery

Debt:

- `APPOINTMENT` and `CATALOG_REQUEST` remain design-only until food ordering is stable

Constraint:

- New code for `FOOD_ORDER` must not block later addition of other booking types

### One admin per tenant

Status:

- Accepted for MVP

Debt:

- No staff roles, no delegated access, no password ownership workflow

## Telegram integration

### `auth_date` expiry check

Status:

- Deferred per product document

Risk:

- Long-lived init data may be accepted longer than desired during MVP

Likely follow-up:

- Reject init data older than 24 hours

### Notification delivery reliability

Status:

- Decided: synchronous HTTP POST to Telegram Bot API via `TelegramNotifier.sendMessage(botToken, chatId, text)`. No retry, no outbox.

Risk:

- Transient Telegram failures can affect request flow or leave message state inconsistent

Likely follow-up:

- Add retry policy, outbox pattern, or async delivery

## Appointment flow

### Double-booking prevention

Status:

- Not implemented in MVP phase 1

Current state:

- `slot` table exists with `is_booked` flag
- Admin creates slots manually via `POST /admin/{slug}/slots` (start time + end time + optional staff)
- Client sees only unbooked slots via `GET /t/{slug}/slots?date=...`
- No transactional lock on slot during booking creation yet

Planned approach:

- Lock slot row in booking transaction (SELECT FOR UPDATE or equivalent)
- Set `is_booked = true` atomically with booking insert
- Fail with 409 if slot is already booked

Open question:

- Whether additional DB constraints (unique partial index on slot where is_booked = true + booking.slot_id) are needed beyond pessimistic locking

## Observability and operations

### Audit logging depth

Status:

- Basic audit table planned

Debt:

- Exact payload shape, retention, and sensitive field handling are not defined

### Structured logging

Status:

- Required but not designed

Debt:

- Need a standard for including `tenant_id`, booking id, and request identifiers in logs

## Process rule

Any meaningful shortcut taken during MVP should be added here before being treated as complete.
