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

### Tenant filter enforcement strategy

Status:

- Not fully designed

Debt:

- Need a repeatable convention for tenant-aware repositories and service methods
- Need tests that fail if data leaks across tenants

## Configuration model

### Overuse of `tenant_config`

Status:

- Accepted for MVP

Risk:

- Security, scheduling, branding, and business settings become mixed in one generic key/value table

Likely follow-up:

- Move security-sensitive or operationally critical settings to typed columns or dedicated tables

### Tenant admin credentials stored in config

Status:

- Accepted for MVP with bcrypt hash only

Risk:

- Weak lifecycle for password rotation, auditing, and future multi-user admin support

Likely follow-up:

- Introduce `admin_user` table with tenant-scoped identities and roles

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

- MVP will likely use direct synchronous Bot API calls

Risk:

- Transient Telegram failures can affect request flow or leave message state inconsistent

Likely follow-up:

- Add retry policy, outbox pattern, or async delivery

## Appointment flow

### Double-booking prevention

Status:

- Not implemented in MVP phase 1

Planned approach:

- Use explicit `slot` rows
- Lock slot row in booking transaction
- Fail if slot is already booked

Open question:

- Whether additional DB constraints are needed beyond pessimistic locking

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
