# yoobu-api — Actual API State
Generated from source code on: 2026-03-14

## Endpoints

### Client API (/t/{slug}/...)

| Method | Path | Request Body | Response Body | Auth | Notes |
|--------|------|-------------|---------------|------|-------|
| GET | `/t/{slug}/config` | — | `TenantConfigResponse` | none | `TenantContextFilter` resolves active tenant by `slug`; 404 if tenant inactive or missing. |
| GET | `/t/{slug}/services` | — | `List<ServiceResponse>` | none | Returns services with `status = ACTIVE` for current tenant. |
| POST | `/t/{slug}/bookings` | `CreateBookingRequest` | `BookingResponse` | Telegram user required | `@TelegramPrincipal` resolved from `X-Telegram-Init-Data`; in `dev` profile also accepts `X-Telegram-User-Id`. Flow only supports tenants with `TenantType.FOOD_ORDER`; created `Booking.type` is always `ORDER`. |
| GET | `/t/{slug}/bookings/my` | — | `List<BookingResponse>` | Telegram user required | Returns only bookings for current tenant and current Telegram user. Food-order tenants only. |
| GET | `/t/{slug}/bookings/{bookingId}` | — | `BookingResponse` | Telegram user required | Returns only caller's own booking inside current tenant. Food-order tenants only. |
| POST | `/t/{slug}/bookings/{bookingId}/cancel` | — | `BookingResponse` | Telegram user required | Changes status to `CANCELLED`; returns 409 if current status is `DONE`. Food-order tenants only. |

### Admin API (/admin/{slug}/...)

| Method | Path | Request Body | Response Body | Auth | Notes |
|--------|------|-------------|---------------|------|-------|
| GET | `/admin/{slug}/services` | — | `List<ServiceResponse>` | HTTP Basic | Tenant admin credentials from `tenant_config` (`admin_username`, `admin_password`) or superadmin credentials. Food-order tenants only. |
| POST | `/admin/{slug}/services` | `AdminUpsertServiceRequest` | `ServiceResponse` | HTTP Basic | Returns `201 Created`. If request `status` is null, service status becomes `ACTIVE`. Food-order tenants only. |
| PUT | `/admin/{slug}/services/{serviceId}` | `AdminUpsertServiceRequest` | `ServiceResponse` | HTTP Basic | Rejects `status = DELETED` with 400; delete must use delete endpoint. Food-order tenants only. |
| DELETE | `/admin/{slug}/services/{serviceId}` | — | — | HTTP Basic | Returns `204 No Content`. Soft-delete: sets `status = DELETED`, `deleted_at`, `updated_at`. Food-order tenants only. |
| GET | `/admin/{slug}/bookings` | — | `List<BookingResponse>` | HTTP Basic | Optional query params: `status` (`BookingStatus`), `deliveryDate` (`ISO date`). Food-order tenants only. |
| GET | `/admin/{slug}/bookings/{bookingId}` | — | `BookingResponse` | HTTP Basic | Food-order tenants only. |
| PUT | `/admin/{slug}/bookings/{bookingId}/status` | `UpdateStatusRequest` | `BookingResponse` | HTTP Basic | Updates booking status without transition validation. Food-order tenants only. |
| GET | `/admin/{slug}/panel` | — | redirect string | HTTP Basic | MVC controller; redirects to `/admin/{slug}/panel/bookings`. |
| GET | `/admin/{slug}/panel/bookings` | — | HTML view `admin/panel/bookings` | HTTP Basic | Query params: `status`, `deliveryDate`. Model includes `bookings`, `selectedStatus`, `deliveryDate`, `statuses`. |
| GET | `/admin/{slug}/panel/bookings/{bookingId}` | — | HTML view `admin/panel/booking-detail` | HTTP Basic | Model includes `booking`, `statuses`, `statusForm`. |
| POST | `/admin/{slug}/panel/bookings/{bookingId}/status` | form params -> `BookingStatusForm` | redirect string or HTML view | HTTP Basic | `@ModelAttribute("statusForm")`; on validation error re-renders detail view. |
| GET | `/admin/{slug}/panel/services` | — | HTML view `admin/panel/services` | HTTP Basic | Model includes `services`. |
| GET | `/admin/{slug}/panel/services/new` | — | HTML view `admin/panel/service-form` | HTTP Basic | Model includes default `ServiceForm` with `sortOrder = 0`, `status = ACTIVE`. |
| POST | `/admin/{slug}/panel/services` | form params -> `ServiceForm` | redirect string or HTML view | HTTP Basic | On validation error re-renders form. Internally converted to `AdminUpsertServiceRequest`. |
| GET | `/admin/{slug}/panel/services/{serviceId}/edit` | — | HTML view `admin/panel/service-form` | HTTP Basic | Loads existing service through `AdminCatalogService#getAdminService`. |
| POST | `/admin/{slug}/panel/services/{serviceId}` | form params -> `ServiceForm` | redirect string or HTML view | HTTP Basic | On validation error re-renders form. |
| POST | `/admin/{slug}/panel/services/{serviceId}/delete` | — | redirect string | HTTP Basic | Internally calls same soft-delete flow as REST delete. |

### Superadmin API (/superadmin/...)

| Method | Path | Request Body | Response Body | Auth | Notes |
|--------|------|-------------|---------------|------|-------|
| GET | `/superadmin/tenants` | — | `List<TenantSummaryResponse>` | HTTP Basic | Uses custom superadmin basic auth filter with `app.superadmin.*`. |
| GET | `/superadmin/tenants/{tenantId}` | — | `TenantDetailResponse` | HTTP Basic | 404 if tenant not found. |
| GET | `/superadmin/tenants/slug-availability?slug=...` | — | `TenantSlugAvailabilityResponse` | HTTP Basic | Returns `available = false` for existing slug; blank slug becomes `false`. |
| POST | `/superadmin/tenants` | `CreateTenantRequest` | `TenantSummaryResponse` | HTTP Basic | Returns 200. Creates tenant row and related `tenant_config` keys. |
| PUT | `/superadmin/tenants/{tenantId}` | `UpdateTenantRequest` | `TenantSummaryResponse` | HTTP Basic | Slug is immutable through API; optional config values may be removed when blank. |
| GET | `/superadmin/panel` | — | redirect string | HTTP Basic | MVC controller; redirects to `/superadmin/panel/tenants`. |
| GET | `/superadmin/panel/tenants` | — | HTML view `superadmin/panel/tenants` | HTTP Basic | Model includes `tenants`. |
| GET | `/superadmin/panel/tenants/{tenantId}` | — | HTML view `superadmin/panel/tenant-detail` | HTTP Basic | Model includes `tenant`. |
| GET | `/superadmin/panel/tenants/new` | — | HTML view `superadmin/panel/tenant-form` | HTTP Basic | Model includes `tenantForm`, `tenantTypes`, `formMode=create`. |
| POST | `/superadmin/panel/tenants` | form params -> `SuperAdminTenantForm` | redirect string or HTML view | HTTP Basic | Validates duplicate slug and non-blank `adminPassword` in controller in addition to bean validation. |
| GET | `/superadmin/panel/tenants/{tenantId}/edit` | — | HTML view `superadmin/panel/tenant-form` | HTTP Basic | Populates form from `TenantDetailResponse`. |
| POST | `/superadmin/panel/tenants/{tenantId}` | form params -> `SuperAdminTenantForm` | redirect string or HTML view | HTTP Basic | Internally converted to `UpdateTenantRequest`. |

## DTOs

### BookingItemRequest
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `serviceId` | `Long` | `@NotNull` | — |
| `quantity` | `int` | `@Min(1)` | — |

### BookingItemResponse
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `serviceName` | `String` | — | — |
| `quantity` | `int` | — | — |
| `unitPrice` | `BigDecimal` | — | — |

### BookingResponse
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `id` | `Long` | — | — |
| `type` | `BookingType` | — | — |
| `status` | `BookingStatus` | — | — |
| `customerName` | `String` | — | — |
| `totalPrice` | `BigDecimal` | — | — |
| `deliveryDate` | `LocalDate` | — | — |
| `note` | `String` | — | — |
| `items` | `List<BookingItemResponse>` | — | — |
| `createdAt` | `OffsetDateTime` | — | — |

### CreateBookingRequest
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `customerName` | `String` | `@NotBlank` | — |
| `customerPhone` | `String` | `@NotBlank` | — |
| `deliveryDate` | `LocalDate` | `@NotNull` | Also validated in service against tenant-local earliest allowed date. |
| `note` | `String` | — | Optional. |
| `items` | `List<BookingItemRequest>` | `@NotEmpty`, element `@Valid` | Each item validated through nested `BookingItemRequest`. |

### UpdateStatusRequest
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `status` | `BookingStatus` | `@NotNull` | — |

### AdminUpsertServiceRequest
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `name` | `String` | `@NotBlank` | — |
| `description` | `String` | — | Optional. |
| `price` | `BigDecimal` | `@NotNull` | — |
| `unit` | `String` | — | Null/blank becomes `"шт"` in service layer. |
| `durationMinutes` | `Integer` | — | Optional. |
| `sortOrder` | `Integer` | — | Null becomes `0` in service layer. |
| `status` | `ServiceStatus` | — | Null becomes `ACTIVE`; `DELETED` is rejected on create/update. |

### ServiceResponse
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `id` | `Long` | — | — |
| `name` | `String` | — | — |
| `description` | `String` | — | — |
| `price` | `BigDecimal` | — | — |
| `unit` | `String` | — | — |
| `durationMinutes` | `Integer` | — | — |
| `sortOrder` | `int` | — | — |
| `status` | `ServiceStatus` | — | — |

### CreateTenantRequest
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `slug` | `String` | `@NotBlank` | Stored as-is; uniqueness checked in service. |
| `name` | `String` | `@NotBlank` | — |
| `type` | `TenantType` | `@NotNull` | — |
| `botToken` | `String` | — | Optional; blank trimmed to null. |
| `ownerTelegramId` | `Long` | — | Optional. |
| `timezone` | `String` | — | Blank/null normalized to `Asia/Ho_Chi_Minh`. |
| `primaryColor` | `String` | — | Optional config key `primary_color`. |
| `logoUrl` | `String` | — | Optional config key `logo_url`. |
| `welcomeMessage` | `String` | — | Optional config key `welcome_message`. |
| `adminUsername` | `String` | `@NotBlank` | Stored in `tenant_config.admin_username`. |
| `adminPassword` | `String` | `@NotBlank` | BCrypt-hashed and stored in `tenant_config.admin_password`. |

### UpdateTenantRequest
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `name` | `String` | `@NotBlank` | — |
| `type` | `TenantType` | `@NotNull` | — |
| `botToken` | `String` | — | Blank trimmed to null. |
| `ownerTelegramId` | `Long` | — | Optional. |
| `timezone` | `String` | — | Blank/null normalized to `Asia/Ho_Chi_Minh`. |
| `primaryColor` | `String` | — | Blank removes config key. |
| `logoUrl` | `String` | — | Blank removes config key. |
| `welcomeMessage` | `String` | — | Blank removes config key. |
| `adminUsername` | `String` | `@NotBlank` | Overwrites stored username. |
| `adminPassword` | `String` | — | Blank keeps previous password hash. |
| `active` | `Boolean` | `@NotNull` | `false` makes tenant inaccessible through `TenantContextFilter`. |

### TenantConfigResponse
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `slug` | `String` | — | — |
| `name` | `String` | — | — |
| `type` | `TenantType` | — | — |
| `primaryColor` | `String` | — | From config key `primary_color`. |
| `logoUrl` | `String` | — | From config key `logo_url`. |
| `welcomeMessage` | `String` | — | From config key `welcome_message`. |

### TenantDetailResponse
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `id` | `Long` | — | — |
| `slug` | `String` | — | — |
| `name` | `String` | — | — |
| `type` | `TenantType` | — | — |
| `active` | `boolean` | — | — |
| `timezone` | `String` | — | — |
| `botToken` | `String` | — | Optional. |
| `ownerTelegramId` | `Long` | — | Optional. |
| `createdAt` | `OffsetDateTime` | — | — |
| `config` | `Map<String, String>` | — | Includes stored config entries such as `admin_username`, `primary_color`, `logo_url`, `welcome_message`. |

### TenantSlugAvailabilityResponse
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `slug` | `String` | — | — |
| `available` | `boolean` | — | — |

### TenantSummaryResponse
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `id` | `Long` | — | — |
| `slug` | `String` | — | — |
| `name` | `String` | — | — |
| `type` | `TenantType` | — | — |
| `active` | `boolean` | — | — |
| `timezone` | `String` | — | — |
| `createdAt` | `OffsetDateTime` | — | — |

### BookingStatusForm
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `status` | `BookingStatus` | `@NotNull` | MVC form model for `/admin/{slug}/panel/bookings/{bookingId}/status`; not used as JSON body. |

### ServiceForm
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `name` | `String` | `@NotBlank` | MVC form model; converted to `AdminUpsertServiceRequest`. |
| `description` | `String` | — | — |
| `price` | `BigDecimal` | `@NotNull` | — |
| `unit` | `String` | — | — |
| `durationMinutes` | `Integer` | — | — |
| `sortOrder` | `Integer` | — | — |
| `status` | `ServiceStatus` | `@NotNull` | Default `ACTIVE`. |

### SuperAdminTenantForm
| Field | Type | Validation | Notes |
|-------|------|-----------|-------|
| `slug` | `String` | `@NotBlank` | MVC form model; used on create and update forms, but update flow does not write slug back to entity. |
| `name` | `String` | `@NotBlank` | — |
| `type` | `TenantType` | `@NotNull` | Default `FOOD_ORDER`. |
| `botToken` | `String` | — | — |
| `ownerTelegramId` | `Long` | — | — |
| `timezone` | `String` | — | Default `Asia/Ho_Chi_Minh`. |
| `primaryColor` | `String` | — | — |
| `logoUrl` | `String` | — | — |
| `welcomeMessage` | `String` | — | — |
| `adminUsername` | `String` | `@NotBlank` | — |
| `adminPassword` | `String` | — | Controller additionally requires non-blank password on create form. |
| `active` | `boolean` | — | Default `true`. |

## Entities

### tenant
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | `BIGSERIAL` | no | — | PK |
| `slug` | `VARCHAR(100)` | no | — | Unique |
| `name` | `VARCHAR(255)` | no | — | JPA does not declare length; SQL does. |
| `type` | `VARCHAR(30)` | no | — | Enum `TenantType` stored as string |
| `bot_token` | `VARCHAR(255)` | yes | — | — |
| `owner_telegram_id` | `BIGINT` | yes | — | — |
| `timezone` | `VARCHAR(50)` | no | `'Asia/Ho_Chi_Minh'` | — |
| `active` | `BOOLEAN` | no | `TRUE` | `TenantContextFilter` only loads active tenants |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | no | `NOW()` | — |
Indexes: unique constraint on `slug`.

### tenant_config
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | `BIGSERIAL` | no | — | PK |
| `tenant_id` | `BIGINT` | no | — | FK -> `tenant(id)` |
| `key` | `VARCHAR(100)` | no | — | — |
| `value` | `TEXT` | yes | — | — |
Constraints: unique (`tenant_id`, `key`).

### service
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | `BIGSERIAL` | no | — | PK |
| `tenant_id` | `BIGINT` | no | — | FK -> `tenant(id)` |
| `name` | `VARCHAR(255)` | no | — | — |
| `description` | `TEXT` | yes | — | — |
| `price` | `NUMERIC(10,2)` | yes | — | Booking creation rejects active service with null price |
| `unit` | `VARCHAR(50)` | yes | `'шт'` | Service layer also defaults blank/null to `"шт"` |
| `duration_minutes` | `INT` | yes | — | — |
| `sort_order` | `INT` | no | `0` | Service layer also defaults null to `0` |
| `status` | `VARCHAR(20)` | no | `'ACTIVE'` | Enum `ServiceStatus` stored as string |
| `deleted_at` | `TIMESTAMP WITH TIME ZONE` | yes | — | Set on soft delete |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | no | `NOW()` | — |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | no | `NOW()` | — |
Indexes: `idx_service_tenant` on (`tenant_id`, `status`, `sort_order`, `id`) where `status <> 'DELETED'`.

### booking
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | `BIGSERIAL` | no | — | PK |
| `tenant_id` | `BIGINT` | no | — | FK -> `tenant(id)` |
| `type` | `VARCHAR(20)` | no | — | Entity requires explicit value; service currently writes `ORDER` |
| `telegram_user_id` | `BIGINT` | no | — | — |
| `customer_name` | `VARCHAR(255)` | no | — | — |
| `customer_phone` | `VARCHAR(50)` | yes | — | Request DTO still requires non-blank value |
| `status` | `VARCHAR(20)` | no | `'NEW'` | Enum `BookingStatus` stored as string |
| `note` | `TEXT` | yes | — | — |
| `total_price` | `NUMERIC(10,2)` | yes | — | Calculated after `booking_item` insert |
| `delivery_date` | `DATE` | yes | — | Request DTO requires value in implemented create flow |
| `slot_id` | `BIGINT` | yes | — | Present in schema and entity; not used by controllers/services |
| `service_id` | `BIGINT` | yes | — | FK -> `service(id)`; present in schema and entity; not used by implemented create flow |
| `deleted_at` | `TIMESTAMP WITH TIME ZONE` | yes | — | No controller sets it |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | no | `NOW()` | — |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | no | `NOW()` | — |
Indexes: `idx_booking_tenant` (`tenant_id`), `idx_booking_tenant_date` (`tenant_id`, `delivery_date`), `idx_booking_tenant_status` (`tenant_id`, `status`), `idx_booking_telegram_user` (`tenant_id`, `telegram_user_id`).

### booking_item
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | `BIGSERIAL` | no | — | PK |
| `booking_id` | `BIGINT` | no | — | FK -> `booking(id)` with `ON DELETE CASCADE` |
| `service_id` | `BIGINT` | no | — | FK -> `service(id)` |
| `quantity` | `INT` | no | — | Check constraint `quantity > 0` |
| `unit_price` | `NUMERIC(10,2)` | no | — | Copied from service price at booking time |

### audit_log
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | `BIGSERIAL` | no | — | PK |
| `tenant_id` | `BIGINT` | no | — | No FK constraint in SQL |
| `entity` | `VARCHAR(50)` | no | — | — |
| `entity_id` | `BIGINT` | no | — | — |
| `action` | `VARCHAR(50)` | no | — | `CREATE`, `UPDATE`, `UPDATE_STATUS`, `DELETE`, `CANCEL` are written in code |
| `actor_id` | `VARCHAR(255)` | yes | — | Changed from `BIGINT` to `VARCHAR(255)` in V2 |
| `old_value` | `TEXT` | yes | — | JSON string |
| `new_value` | `TEXT` | yes | — | JSON string |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | no | `NOW()` | — |
Indexes: `idx_audit_tenant_entity` (`tenant_id`, `entity`, `entity_id`).

## Enums

### BookingStatus
Values: `NEW`, `CONFIRMED`, `DONE`, `CANCELLED`

### BookingType
Values: `ORDER`, `APPOINTMENT`, `REQUEST`

### ServiceStatus
Values: `ACTIVE`, `INACTIVE`, `DELETED`

### TenantType
Values: `FOOD_ORDER`, `APPOINTMENT`, `CATALOG_REQUEST`

## Booking Validation Matrix

| Field | FOOD_ORDER | APPOINTMENT | CATALOG_REQUEST |
|-------|-----------|-------------|-----------------|
| `customerName` | required (`@NotBlank`) | not implemented; request rejected before type-specific field handling | not implemented; request rejected before type-specific field handling |
| `customerPhone` | required (`@NotBlank`) | not implemented | not implemented |
| `deliveryDate` | required (`@NotNull`) and must be `>= tenantTimeService.earliestDeliveryDate(...)` | not implemented | not implemented |
| `note` | optional | not implemented | not implemented |
| `items` | required, non-empty (`@NotEmpty`) | not implemented | not implemented |
| `items[].serviceId` | required (`@NotNull`), service must exist in current tenant with `status = ACTIVE` | not implemented | not implemented |
| `items[].quantity` | required, minimum `1` (`@Min(1)`) | not implemented | not implemented |
| `Booking.type` persisted value | always `ORDER` | not implemented | not implemented |
| `slotId` | ignored; not present in request DTO | ignored | ignored |
| `booking.service` root field | ignored; not present in request DTO | ignored | ignored |

Source: `BookingController#createBooking`, `BookingService#createFoodOrder`, `BookingService#requireFoodOrderTenant`, `BookingService#validateDeliveryDate`, `BookingService#toBookingItem`, `CreateBookingRequest`, `BookingItemRequest`

## Security

### Filter chains
- `/superadmin/**` -> custom `SuperAdminBasicAuthenticationFilter`; credentials from `app.superadmin.username` and `app.superadmin.password`; any request authenticated; CSRF disabled; form login disabled; logout disabled; HTTP Basic disabled in Spring and replaced by custom filter; anonymous disabled.
- `/admin/*/**` -> `TenantContextFilter` then custom `TenantBasicAuthenticationFilter`; credentials from current tenant config keys `admin_username` and `admin_password` (BCrypt), plus superadmin credentials are also accepted here; any request authenticated; CSRF disabled; form login disabled; logout disabled; HTTP Basic disabled in Spring and replaced by custom filter; anonymous disabled.
- `/t/*/**` -> `TenantContextFilter`; all requests permitted at Spring Security level; CSRF disabled; form login disabled; logout disabled; HTTP Basic disabled. Telegram auth is enforced only on controller parameters that require `@TelegramPrincipal`.
- default (`/**`) -> permit all; CSRF disabled; anonymous enabled.

Authentication providers: Not implemented. Authentication is performed directly inside custom `OncePerRequestFilter` classes.

CORS: Not implemented.

### Tenant resolution
- Interceptor/filter: `TenantContextFilter`
- Pattern: attached to `/admin/*/**` and `/t/*/**` security chains
- Storage: `TenantContext` static `ThreadLocal<Tenant>`
- Resolution rule: parses URI segments and uses the segment after `t` or `admin` as slug
- Lookup: `TenantRepository.findBySlugAndActiveTrue(slug)`
- Failure modes: 400 `"Tenant slug is missing"` if slug segment missing; 404 `"Tenant not found"` if tenant missing or inactive

## Flyway Migrations

| Version | Filename | Summary |
|---------|----------|---------|
| `V1` | `V1__baseline_schema.sql` | Creates `tenant`, `tenant_config`, `service`, `booking`, `booking_item`, `audit_log`; adds baseline indexes. |
| `V2` | `V2__audit_actor_id_as_string.sql` | Changes `audit_log.actor_id` from numeric to string. |
| `V3` | `V3__service_status_model.sql` | Adds `service.status`, backfills values from `active`/`deleted_at`, adds new partial index, drops `service.active`. |

## Schema vs Entity Drift

- `tenant.name`: SQL length is `VARCHAR(255)`; entity does not declare length.
- `tenant.timezone`, `tenant.active`, `tenant.created_at`: SQL defines defaults; entity does not model defaults.
- `tenant.slug`: SQL has unique constraint; entity models `unique = true`.
- `tenant_config`: SQL has unique (`tenant_id`, `key`) constraint; entity does not declare the unique constraint.
- `service.unit`, `service.sort_order`, `service.status`, `service.created_at`, `service.updated_at`: SQL defines defaults; entity does not model defaults.
- `service.name`: SQL length is `VARCHAR(255)`; entity does not declare length.
- `service.active`: removed in V3 SQL; entity also no longer has this field. No drift here.
- `booking.customer_name`: SQL length is `VARCHAR(255)`; entity does not declare length.
- `booking.status`, `booking.created_at`, `booking.updated_at`: SQL defines defaults; entity does not model defaults.
- `booking.customer_phone`: SQL allows null; request DTO requires non-blank in implemented create flow.
- `booking.slot_id` and `booking.service_id`: present in SQL and entity, but unused by implemented controllers/services.
- `booking_item.quantity`: SQL has check constraint `quantity > 0`; entity does not model the check constraint.
- `booking_item.booking_id`: SQL has `ON DELETE CASCADE`; entity does not model cascade behavior.
- `audit_log.actor_id`: latest SQL type `VARCHAR(255)` matches entity `String` after V2.
- `audit_log.created_at`: SQL defines default `NOW()`; entity does not model default.

## Telegram Integration State

- InitData validation: implemented in `TelegramInitDataValidator`; validates HMAC-SHA256 using tenant `bot_token`, requires `hash` and `user.id`, and returns `401 Invalid initData` on any failure.
- Notifications on booking create: no.
- Notifications on status change: no.
- Daily summary: missing.
- Dev fallback: in `dev` Spring profile, `TelegramUserArgumentResolver` accepts header `X-Telegram-User-Id` and creates `TelegramUser(id, "Dev", "User", null)` when `X-Telegram-Init-Data` is absent.

## Configuration Shape

Active profiles in `application.yml`: not configured.

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

## Test Coverage Map

| Test file | Current covered behavior |
|-----------|--------------------------|
| `SuperAdminTenantControllerIT` | Superadmin auth, tenant CRUD, slug availability, duplicate slug rejection, optional field clearing, password retention, inactive tenant behavior |
| `TenantAdminAccessIT` | Tenant admin auth success/failure, missing credentials, superadmin access to tenant admin endpoints |
| `TenantIsolationIT` | Cross-tenant isolation for services and admin auth realms |
| `TenantAdminCatalogAndBookingIT` | Happy path from service creation to public catalog and booking |
| `TenantFoodOrderConstraintsIT` | Restriction of booking/service flows to `FOOD_ORDER`, deleted service behavior, unknown service rejection |
| `BookingLifecycleIT` | Customer booking create/list/read/cancel, admin read/update, audit log writes, cancel conflict on `DONE` |
| `ServiceManagementAndValidationIT` | Public tenant config, service update/delete, booking filtering, bean validation failures, delivery-date cutoff validation |
| `AdminPanelIT` | Tenant admin MVC panel routes and form submissions |
| `SuperAdminPanelIT` | Superadmin MVC panel routes and form submissions |
| `TenantTimeServiceTest` | Tenant-local date and cutoff calculations |
