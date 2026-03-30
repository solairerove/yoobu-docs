# Task: Extract actual API state from yoobu-api project

You are analyzing a Java 21 + Spring Boot 3.x project. Your goal is to produce a single structured report of the **actual implemented** API surface — endpoints, DTOs, validation rules, database schema, security config. No opinions, no suggestions — only what exists in code right now.

## What to scan

1. **Controllers** — every class annotated with `@RestController` or `@Controller`. For each:
   - Full path mapping (class-level + method-level)
   - HTTP method
   - Request body type (if any)
   - Response body type
   - Path variables and query parameters
   - Security annotations or filter chain assignment

2. **DTOs** — every class used as request or response body. For each:
   - All fields with types
   - Validation annotations (`@NotNull`, `@NotBlank`, `@Size`, etc.)
   - Jackson annotations (`@JsonProperty`, `@JsonIgnore`, `@JsonFormat`, etc.)
   - Nested types (inline or reference)

3. **Entities** — every class annotated with `@Entity`. For each:
   - Table name
   - All columns with types, nullable, defaults
   - Relationships (`@ManyToOne`, `@OneToMany`, etc.)
   - Enums used (list values)

4. **Flyway migrations** — list all migration files in order. For the latest schema state:
   - All tables with columns, types, constraints, indexes
   - Note any differences between entity definitions and actual migration SQL

5. **Security configuration** — `SecurityFilterChain` beans:
   - Which paths are protected by what mechanism
   - Authentication providers
   - CORS/CSRF settings

6. **Tenant resolution** — how tenant context is set:
   - Interceptor or filter class
   - What path pattern it matches
   - Where tenant is stored (ThreadLocal, request attribute, etc.)

7. **Telegram integration** — current state:
   - InitData validation: implemented or stubbed?
   - Notification sending: which events trigger messages?
   - Dev fallback: how does local development bypass Telegram auth?

8. **Booking validation** — how booking creation handles different `BookingType` values:
   - Which fields are required per type?
   - Which fields are ignored per type?
   - Where is this logic implemented (controller, service, validator)?

9. **Application configuration** — `application.yml` / `application.properties`:
   - Active profiles
   - Custom `app.*` properties
   - Database config shape (not credentials)

## Output format

Produce ONE markdown document with this exact structure:

```
# yoobu-api — Actual API State
Generated from source code on: [date]

## Endpoints

### Client API (/t/{slug}/...)

| Method | Path | Request Body | Response Body | Auth | Notes |
|--------|------|-------------|---------------|------|-------|
| GET | /t/{slug}/config | — | TenantConfigResponse | none | |
| ... | | | | | |

### Admin API (/admin/{slug}/...)

[same table format]

### Superadmin API (/superadmin/...)

[same table format]

## DTOs

### TenantConfigResponse
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| slug | String | — | |
| ... | | | |

[repeat for every DTO]

## Entities

### tenant
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | BIGSERIAL | no | — | PK |
| ... | | | | |

[repeat for every entity]

## Enums

### BookingStatus
Values: NEW, CONFIRMED, DONE, CANCELLED

[repeat for every enum]

## Booking Validation Matrix

| Field | FOOD_ORDER | APPOINTMENT | CATALOG_REQUEST |
|-------|-----------|-------------|-----------------|
| customer_name | required | required | required |
| ... | | | |

Source: [class and method where this is enforced]

## Security

### Filter chains
- /admin/{slug}/** → [mechanism, provider]
- /superadmin/** → [mechanism, provider]
- /t/{slug}/** → [mechanism or open]

### Tenant resolution
- Interceptor: [class name]
- Pattern: [what it matches]
- Storage: [ThreadLocal / request attribute / etc.]

## Flyway Migrations

| Version | Filename | Summary |
|---------|----------|---------|
| V1 | V1__baseline.sql | Creates tenant, service, booking, ... |
| ... | | |

## Schema vs Entity Drift

[List any differences between Flyway SQL and JPA entity definitions. If none — state "No drift detected."]

## Telegram Integration State

- InitData validation: [implemented / stubbed / missing]
- Notifications on booking create: [yes/no, to whom]
- Notifications on status change: [yes/no, to whom]
- Daily summary: [implemented / missing]
- Dev fallback: [describe mechanism]

## Configuration Shape

```yaml
app:
  [actual structure from application.yml, no secrets]
```
```

## Rules

- Report ONLY what is implemented. Do not infer intent from comments or TODOs.
- If a section has nothing implemented, write "Not implemented."
- If entity and migration disagree, report both versions.
- If validation logic is split across multiple classes, list all locations.
- Do not suggest improvements or next steps.
