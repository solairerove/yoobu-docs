# RND API State (Source-Derived Snapshot)

Scope: `src/main/java`, `src/main/resources/db/migration`, `src/main/resources/application.yml`, `src/test/java`.  
Excluded: existing docs/README.

## 1) Endpoints

### Client API (`/t/{slug}/...`)

| Controller.method | HTTP | Full path | Query params | Request body/form | Response | Auth | Behavioral notes |
|---|---|---|---|---|---|---|---|
| `CatalogController.getServices` | `GET` | `/t/{slug}/services` | none | none | `200` + `List<ServiceResponse>` | none | Tenant resolved by `TenantContextFilter`; if slug missing: `400 Tenant slug is missing`; unknown/inactive tenant: `404 Tenant not found`. Returns only `ServiceStatus.ACTIVE`. |
| `TenantPublicController.getConfig` | `GET` | `/t/{slug}/config` | none | none | `200` + `TenantConfigResponse` | none | Same tenant resolution behavior as above. |
| `BookingController.createBooking` | `POST` | `/t/{slug}/bookings` | none | `@Valid CreateBookingRequest` | `200` + `BookingResponse` | Telegram | Telegram principal from `TelegramUserArgumentResolver` (`X-Telegram-Init-Data`, or dev-only `X-Telegram-User-Id`). Validation errors -> `400`. Business errors include: `400 Tenant does not support food ordering`, `400 Delivery date must be on or after ...`, `400 Service not found`, `400 Service price is missing`. |
| `BookingController.getMyBookings` | `GET` | `/t/{slug}/bookings/my` | none | none | `200` + `List<BookingResponse>` | Telegram | Returns bookings for `(tenantId, telegramUserId)` only. Unauthorized Telegram headers -> `401 Invalid initData`. |
| `BookingController.getBooking` | `GET` | `/t/{slug}/bookings/{bookingId}` | none | none | `200` + `BookingResponse` | Telegram | Not owned/missing booking -> `404 Booking not found`. |
| `BookingController.cancelBooking` | `POST` | `/t/{slug}/bookings/{bookingId}/cancel` | none | none | `200` + `BookingResponse` | Telegram | `DONE` booking cancel -> `409 Completed booking cannot be cancelled`; not owned/missing -> `404 Booking not found`. |
| `BookingController.confirmPayment` | `POST` | `/t/{slug}/bookings/{bookingId}/confirm-payment` | none | none | `200` + `BookingResponse` | Telegram | Allowed only from `NEW` -> status moves to `PAYMENT_PENDING`; otherwise `409 Payment can only be confirmed for booking in NEW status`; not owned/missing -> `404 Booking not found`. |

### Admin API (`/admin/{slug}/...`)

| Controller.method | HTTP | Full path | Query params | Request body/form | Response | Auth | Behavioral notes |
|---|---|---|---|---|---|---|---|
| `AdminCatalogController.getServices` | `GET` | `/admin/{slug}/services` | none | none | `200` + `List<ServiceResponse>` | HTTP Basic tenant | Tenant realm: `Yoobu Tenant Admin: {slug}`. Superadmin credentials are also accepted in this filter. Returns non-`DELETED` services. |
| `AdminCatalogController.createService` | `POST` | `/admin/{slug}/services` | none | `@Valid AdminUpsertServiceRequest` | `201` + `ServiceResponse` | HTTP Basic tenant | Validation errors -> `400`. For non-food tenant -> `400 Tenant does not support food ordering`. |
| `AdminCatalogController.updateService` | `PUT` | `/admin/{slug}/services/{serviceId}` | none | `@Valid AdminUpsertServiceRequest` | `200` + `ServiceResponse` | HTTP Basic tenant | Missing service -> `404 Service not found`; `status=DELETED` in upsert payload -> `400 Use delete endpoint to remove a service`. |
| `AdminCatalogController.deleteService` | `DELETE` | `/admin/{slug}/services/{serviceId}` | none | none | `204` empty | HTTP Basic tenant | Soft-delete: sets status `DELETED`, `deletedAt`. Missing service -> `404 Service not found`. |
| `AdminBookingController.getBookings` | `GET` | `/admin/{slug}/bookings` | `status` optional (`BookingStatus`), `deliveryDate` optional (`ISO date`) | none | `200` + `List<BookingResponse>` | HTTP Basic tenant | Non-food tenant -> `400 Tenant does not support food ordering`. |
| `AdminBookingController.getBooking` | `GET` | `/admin/{slug}/bookings/{bookingId}` | none | none | `200` + `BookingResponse` | HTTP Basic tenant | Missing booking -> `404 Booking not found`. |
| `AdminBookingController.updateStatus` | `PUT` | `/admin/{slug}/bookings/{bookingId}/status` | none | `@Valid UpdateStatusRequest` | `200` + `BookingResponse` | HTTP Basic tenant | Validation (`trackingUrl` max 2048) -> `400`. Invalid transition -> `409`. Invalid tracking URL/scheme/host -> `400`. Optimistic lock conflict -> `409 Booking was modified by another request. Refresh and retry.` |
| `AdminPanelBookingController.panelHome` | `GET` | `/admin/{slug}/panel` and `/admin/{slug}/panel/` | none | none | redirect view (`redirect:/admin/{slug}/panel/bookings`) | HTTP Basic tenant | MVC redirect. |
| `AdminPanelBookingController.bookings` | `GET` | `/admin/{slug}/panel/bookings` | `status?`, `deliveryDate?`, `page=0` default, `size=10` default | none | view `admin/panel/bookings` | HTTP Basic tenant | Populates paging + status options by booking. |
| `AdminPanelBookingController.bookingDetail` | `GET` | `/admin/{slug}/panel/bookings/{bookingId}` | none | none | view `admin/panel/booking-detail` | HTTP Basic tenant | Missing booking propagates `404 Booking not found`. |
| `AdminPanelBookingController.updateStatus` | `POST` | `/admin/{slug}/panel/bookings/{bookingId}/status` | `returnTo` default `list` (`detail` supported) | `@Valid @ModelAttribute BookingStatusForm` | redirect to list/detail | HTTP Basic tenant | Binding errors -> flash error and redirect; service `ResponseStatusException` reason surfaced as flash error. Success flash: `Booking #{id} status updated to ...`. |
| `AdminPanelServiceController.services` | `GET` | `/admin/{slug}/panel/services` | `query?`, `page=0`, `size=20` | none | view `admin/panel/services` | HTTP Basic tenant | Uses paged admin service query and tenant currency. |
| `AdminPanelServiceController.newService` | `GET` | `/admin/{slug}/panel/services/new` | none | none | view `admin/panel/service-form` | HTTP Basic tenant | Form defaults: `sortOrder=0`, `status=ACTIVE`. |
| `AdminPanelServiceController.createService` | `POST` | `/admin/{slug}/panel/services` | none | `@Valid @ModelAttribute ServiceForm` | redirect `.../panel/services` or same form view on bind errors | HTTP Basic tenant | Service exception -> flash error + redirect. |
| `AdminPanelServiceController.editService` | `GET` | `/admin/{slug}/panel/services/{serviceId}/edit` | none | none | view `admin/panel/service-form` | HTTP Basic tenant | Missing service -> `404 Service not found`. |
| `AdminPanelServiceController.updateService` | `POST` | `/admin/{slug}/panel/services/{serviceId}` | none | `@Valid @ModelAttribute ServiceForm` | redirect `.../panel/services` or same form view on bind errors | HTTP Basic tenant | Service exception -> flash error + redirect. |
| `AdminPanelServiceController.deleteService` | `POST` | `/admin/{slug}/panel/services/{serviceId}/delete` | `confirmName?` | form param only | redirect to list or edit page | HTTP Basic tenant | Requires exact typed name match; mismatch -> flash error. |
| `AdminPanelServiceController.updateStatus` | `POST` | `/admin/{slug}/panel/services/{serviceId}/status` | none | `@Valid @ModelAttribute ServiceStatusForm` | redirect `.../panel/services` | HTTP Basic tenant | Binding errors -> flash error. Internally loads current service then performs full update with new status. |

### Superadmin API (`/superadmin/...`)

| Controller.method | HTTP | Full path | Query params | Request body/form | Response | Auth | Behavioral notes |
|---|---|---|---|---|---|---|---|
| `SuperAdminTenantController.getTenants` | `GET` | `/superadmin/tenants` | none | none | `200` + `List<TenantSummaryResponse>` | HTTP Basic superadmin | Missing/invalid basic header -> `401` with realm `Yoobu Super Admin`. |
| `SuperAdminTenantController.getTenant` | `GET` | `/superadmin/tenants/{tenantId}` | none | none | `200` + `TenantDetailResponse` | HTTP Basic superadmin | Missing tenant -> `404 Tenant not found`. |
| `SuperAdminTenantController.getSlugAvailability` | `GET` | `/superadmin/tenants/slug-availability` | `slug` required | none | `200` + `TenantSlugAvailabilityResponse` | HTTP Basic superadmin | `available` false if slug exists or slug blank. |
| `SuperAdminTenantController.createTenant` | `POST` | `/superadmin/tenants` | none | `@Valid CreateTenantRequest` | `200` + `TenantSummaryResponse` | HTTP Basic superadmin | Validation errors -> `400`; duplicate slug -> `409 Tenant slug already exists`; invalid payment QR -> `400 paymentQrUrl must be a valid absolute http(s) URL`. |
| `SuperAdminTenantController.updateTenant` | `PUT` | `/superadmin/tenants/{tenantId}` | none | `@Valid UpdateTenantRequest` | `200` + `TenantSummaryResponse` | HTTP Basic superadmin | Missing tenant -> `404`; invalid payment QR -> `400`. |
| `SuperAdminAuditController.getAuditLogs` | `GET` | `/superadmin/audit` | `tenantId?`, `entity?`, `action?`, `actorId?`, `createdFrom?` ISO datetime, `createdTo?` ISO datetime, `page=0`, `size=20` | none | `200` + `AuditLogPageResponse` | HTTP Basic superadmin | Size capped to 50 in service. |
| `SuperAdminAuditController.exportAuditLogs` | `GET` | `/superadmin/audit/export` | same filters; `size=5000` | none | `200 text/csv` bytes | HTTP Basic superadmin | Size capped to 5000 in service; attachment filename `audit-log-YYYYMMDD-HHmmss.csv`. |
| `SuperAdminPanelController.panelHome` | `GET` | `/superadmin/panel` and `/superadmin/panel/` | none | none | redirect `redirect:/superadmin/panel/tenants` | HTTP Basic superadmin | MVC redirect. |
| `SuperAdminPanelController.tenants` | `GET` | `/superadmin/panel/tenants` | `query?`, `page=0`, `size=20` | none | view `superadmin/panel/tenants` | HTTP Basic superadmin | Paged tenant listing. |
| `SuperAdminPanelController.audit` | `GET` | `/superadmin/panel/audit` | `tenantId?`, `entity?`, `action?`, `actorId?`, `createdFrom?`, `createdTo?`, `page=0`, `size=20` | none | view `superadmin/panel/audit` | HTTP Basic superadmin | Datetime parse supports both `OffsetDateTime.parse` and local datetime (interpreted as UTC). |
| `SuperAdminPanelController.exportAudit` | `GET` | `/superadmin/panel/audit/export` | same filters; `size=5000` | none | `200 text/csv` bytes | HTTP Basic superadmin | Attachment response. |
| `SuperAdminPanelController.tenantDetail` | `GET` | `/superadmin/panel/tenants/{tenantId}` | none | none | view `superadmin/panel/tenant-detail` | HTTP Basic superadmin | Missing tenant -> `404`. |
| `SuperAdminPanelController.newTenant` | `GET` | `/superadmin/panel/tenants/new` | none | none | view `superadmin/panel/tenant-form` | HTTP Basic superadmin | Initializes empty `SuperAdminTenantForm`. |
| `SuperAdminPanelController.createTenant` | `POST` | `/superadmin/panel/tenants` | none | `@Valid @ModelAttribute SuperAdminTenantForm` | redirect to tenants list or same form view | HTTP Basic superadmin | Extra create validations: adminPassword required, slug uniqueness check via service. |
| `SuperAdminPanelController.editTenant` | `GET` | `/superadmin/panel/tenants/{tenantId}/edit` | none | none | view `superadmin/panel/tenant-form` | HTTP Basic superadmin | Loads values from `TenantDetailResponse.config`. |
| `SuperAdminPanelController.updateTenant` | `POST` | `/superadmin/panel/tenants/{tenantId}` | none | `@Valid @ModelAttribute SuperAdminTenantForm` | redirect `.../tenants/{tenantId}` or same form view | HTTP Basic superadmin | Binding errors keep edit form; service errors set `formError`. |

### Unscoped endpoint

| Controller.method | HTTP | Full path | Auth | Notes |
|---|---|---|---|---|
| `HealthController.health` | `GET` | `/health` | none | Returns `{ "status": "ok" }`; handled by default security chain (`permitAll`). |

## 2) DTOs / Form models

`Required` semantics below are from validation annotations and Java nullability/primitive types.

| DTO / Form | Field | Java type | Validation / default | Required? | Used by endpoint(s) |
|---|---|---|---|---|---|
| `CreateBookingRequest` | `customerName` | `String` | `@NotBlank` | required | `BookingController.createBooking` |
| `CreateBookingRequest` | `customerPhone` | `String` | `@NotBlank` | required | same |
| `CreateBookingRequest` | `deliveryAddress` | `String` | `@NotBlank` | required | same |
| `CreateBookingRequest` | `deliveryDate` | `LocalDate` | `@NotNull` | required | same |
| `CreateBookingRequest` | `note` | `String` | none | nullable | same |
| `CreateBookingRequest` | `items` | `List<BookingItemRequest>` | `@NotEmpty`, elements `@Valid` | required, non-empty | same |
| `BookingItemRequest` | `serviceId` | `Long` | `@NotNull` | required | nested in `CreateBookingRequest` |
| `BookingItemRequest` | `quantity` | `int` | `@Min(1)` | required (primitive) | nested in `CreateBookingRequest` |
| `UpdateStatusRequest` | `status` | `BookingStatus` | `@NotNull` | required | `AdminBookingController.updateStatus` |
| `UpdateStatusRequest` | `trackingUrl` | `String` | `@Size(max=2048)` | nullable | same |
| `AdminUpsertServiceRequest` | `name` | `String` | `@NotBlank` | required | `AdminCatalogController.createService`, `updateService` |
| `AdminUpsertServiceRequest` | `description` | `String` | none | nullable | same |
| `AdminUpsertServiceRequest` | `price` | `BigDecimal` | `@NotNull` | required | same |
| `AdminUpsertServiceRequest` | `unit` | `String` | none | nullable (`"шт"` fallback in service) | same |
| `AdminUpsertServiceRequest` | `durationMinutes` | `Integer` | none | nullable | same |
| `AdminUpsertServiceRequest` | `sortOrder` | `Integer` | none | nullable (`0` fallback in service) | same |
| `AdminUpsertServiceRequest` | `status` | `ServiceStatus` | none | nullable (`ACTIVE` fallback; `DELETED` rejected) | same |
| `CreateTenantRequest` | `slug` | `String` | `@NotBlank` | required | `SuperAdminTenantController.createTenant` |
| `CreateTenantRequest` | `name` | `String` | `@NotBlank` | required | same |
| `CreateTenantRequest` | `type` | `TenantType` | `@NotNull` | required | same |
| `CreateTenantRequest` | `botToken` | `String` | none | nullable | same |
| `CreateTenantRequest` | `ownerTelegramId` | `Long` | none | nullable | same |
| `CreateTenantRequest` | `timezone` | `String` | none | nullable (`Asia/Ho_Chi_Minh` fallback) | same |
| `CreateTenantRequest` | `currency` | `String` | none | nullable (`USD` fallback) | same |
| `CreateTenantRequest` | `primaryColor`, `logoUrl`, `welcomeMessage`, `checkoutNameHint`, `checkoutPhoneHint`, `checkoutNoteHint`, `paymentQrUrl` | `String` | none (`paymentQrUrl` validated if present) | nullable | same |
| `CreateTenantRequest` | `adminUsername` | `String` | `@NotBlank` | required | same |
| `CreateTenantRequest` | `adminPassword` | `String` | `@NotBlank` | required | same |
| `UpdateTenantRequest` | `name` | `String` | `@NotBlank` | required | `SuperAdminTenantController.updateTenant` |
| `UpdateTenantRequest` | `type` | `TenantType` | `@NotNull` | required | same |
| `UpdateTenantRequest` | `adminUsername` | `String` | `@NotBlank` | required | same |
| `UpdateTenantRequest` | `active` | `Boolean` | `@NotNull` | required | same |
| `UpdateTenantRequest` | other fields | mixed | none (`paymentQrUrl` validated if present) | nullable | same |
| `ServiceForm` (MVC) | `name` | `String` | `@NotBlank` | required | `AdminPanelServiceController.createService`, `updateService` |
| `ServiceForm` (MVC) | `price` | `BigDecimal` | `@NotNull` | required | same |
| `ServiceForm` (MVC) | `status` | `ServiceStatus` | `@NotNull`, default `ACTIVE` | required with default | same |
| `ServiceForm` (MVC) | `description`, `unit`, `durationMinutes`, `sortOrder` | mixed | none | nullable | same |
| `ServiceStatusForm` (MVC) | `status` | `ServiceStatus` | `@NotNull` | required | `AdminPanelServiceController.updateStatus` |
| `BookingStatusForm` (MVC) | `status` | `BookingStatus` | `@NotNull` | required | `AdminPanelBookingController.updateStatus` |
| `BookingStatusForm` (MVC) | `trackingUrl` | `String` | `@Size(max=2048)` | nullable | same |
| `SuperAdminTenantForm` (MVC) | `slug` | `String` | `@NotBlank` | required | `SuperAdminPanelController.createTenant` |
| `SuperAdminTenantForm` (MVC) | `name` | `String` | `@NotBlank` | required | create/update forms |
| `SuperAdminTenantForm` (MVC) | `type` | `TenantType` | `@NotNull`, default `FOOD_ORDER` | required with default | create/update forms |
| `SuperAdminTenantForm` (MVC) | `timezone` | `String` | default `Asia/Ho_Chi_Minh` | optional input | create/update forms |
| `SuperAdminTenantForm` (MVC) | `currency` | `String` | default `USD` | optional input | create/update forms |
| `SuperAdminTenantForm` (MVC) | `adminUsername` | `String` | `@NotBlank` | required | create/update forms |
| `SuperAdminTenantForm` (MVC) | `adminPassword` | `String` | none (create adds manual non-blank check) | optional on update, required on create | create/update forms |
| `SuperAdminTenantForm` (MVC) | `active` | `boolean` | default `true` | always present as primitive/default | update form |
| `TenantSummaryResponse` | all fields | record | no validation (output DTO) | n/a | superadmin tenant list/create/update responses |
| `TenantDetailResponse` | all fields incl `config` map | record | output DTO | n/a | superadmin tenant detail response |
| `TenantConfigResponse` | branding/checkout/payment fields | record | output DTO | n/a | `/t/{slug}/config` |
| `TenantSlugAvailabilityResponse` | `slug`, `available` | record | output DTO | n/a | `/superadmin/tenants/slug-availability` |
| `ServiceResponse` | service fields | record | output DTO | n/a | catalog/admin/service panel flows |
| `BookingResponse` | booking aggregate fields | record | output DTO | n/a | client/admin booking endpoints |
| `BookingItemResponse` | serviceName/quantity/unitPrice/currency | record | output DTO | n/a | nested inside `BookingResponse` |
| `AuditLogItemResponse` | audit row projection | record | output DTO | n/a | `/superadmin/audit*`, panel audit |
| `AuditLogPageResponse` | pagination wrapper | record | output DTO | n/a | `/superadmin/audit` |

## 3) Entities

### `Tenant` (`tenant`)

| Field | Java type | DB column/type | Nullability/default/constraints | Relationships | Used vs reserved |
|---|---|---|---|---|---|
| `id` | `Long` | `id BIGSERIAL` | PK | none | used |
| `slug` | `String` | `slug VARCHAR(100)` | `NOT NULL`, `UNIQUE` | none | used in routing/auth |
| `name` | `String` | `name VARCHAR(255)` | `NOT NULL` | none | used |
| `type` | `TenantType` | `type VARCHAR(30)` | `NOT NULL` | none | used for flow gating |
| `botToken` | `String` | `bot_token VARCHAR(255)` | nullable | none | used by Telegram validator |
| `ownerTelegramId` | `Long` | `owner_telegram_id BIGINT` | nullable | none | stored/exposed only |
| `timezone` | `String` | `timezone VARCHAR(50)` | `NOT NULL`, DB default `'Asia/Ho_Chi_Minh'` | none | used in tenant time formatting/logic |
| `active` | `boolean` | `active BOOLEAN` | `NOT NULL`, DB default `TRUE` | none | used by `findBySlugAndActiveTrue` |
| `createdAt` | `OffsetDateTime` | `created_at TIMESTAMPTZ` | `NOT NULL`, DB default `NOW()` | none | used for sorting/exposure |

### `TenantConfig` (`tenant_config`)

| Field | Java type | DB column/type | Nullability/default/constraints | Relationships | Used vs reserved |
|---|---|---|---|---|---|
| `id` | `Long` | `id BIGSERIAL` | PK | none | used |
| `tenant` | `Tenant` | `tenant_id BIGINT` | `NOT NULL`, FK `tenant(id)` | `@ManyToOne(fetch = LAZY, optional = false)` | used |
| `key` | `String` | `key VARCHAR(100)` | `NOT NULL`, unique with tenant at DB (`UNIQUE (tenant_id,key)`) | none | used |
| `value` | `String` | `value TEXT` | nullable | none | used |

### `CatalogService` (`service`)

| Field | Java type | DB column/type | Nullability/default/constraints | Relationships | Used vs reserved |
|---|---|---|---|---|---|
| `id` | `Long` | `id BIGSERIAL` | PK | none | used |
| `tenant` | `Tenant` | `tenant_id BIGINT` | `NOT NULL`, FK `tenant(id)` | `@ManyToOne(fetch = LAZY, optional = false)` | used |
| `name` | `String` | `name VARCHAR(255)` | `NOT NULL` | none | used |
| `description` | `String` | `description TEXT` | nullable | none | used |
| `price` | `BigDecimal` | `price NUMERIC(10,2)` | nullable in DB, required by API validation | none | used |
| `unit` | `String` | `unit VARCHAR(50)` | nullable, DB default `'шт'` | none | used |
| `durationMinutes` | `Integer` | `duration_minutes INT` | nullable | none | used |
| `status` | `ServiceStatus` | `status VARCHAR(20)` | `NOT NULL`, DB default `ACTIVE` | none | used (`ACTIVE/INACTIVE/DELETED`) |
| `sortOrder` | `int` | `sort_order INT` | `NOT NULL`, DB default `0` | none | used |
| `deletedAt` | `OffsetDateTime` | `deleted_at TIMESTAMPTZ` | nullable | none | used for soft-delete audit |
| `createdAt` | `OffsetDateTime` | `created_at TIMESTAMPTZ` | `NOT NULL`, DB default `NOW()` | none | used |
| `updatedAt` | `OffsetDateTime` | `updated_at TIMESTAMPTZ` | `NOT NULL`, DB default `NOW()` | none | used |

### `Booking` (`booking`)

| Field | Java type | DB column/type | Nullability/default/constraints | Relationships | Used vs reserved |
|---|---|---|---|---|---|
| `id` | `Long` | `id BIGSERIAL` | PK | none | used |
| `tenant` | `Tenant` | `tenant_id BIGINT` | `NOT NULL`, FK `tenant(id)` | `@ManyToOne(fetch = LAZY, optional = false)` | used |
| `type` | `BookingType` | `type VARCHAR(20)` | `NOT NULL` | none | currently always set to `ORDER`; other enum values reserved |
| `telegramUserId` | `Long` | `telegram_user_id BIGINT` | `NOT NULL` | none | used |
| `customerName` | `String` | `customer_name VARCHAR(255)` | `NOT NULL` | none | used |
| `customerPhone` | `String` | `customer_phone VARCHAR(50)` | nullable | none | used |
| `deliveryAddress` | `String` | `delivery_address TEXT` | nullable in DB, required by API validation | none | used |
| `status` | `BookingStatus` | `status VARCHAR(20)` | `NOT NULL`, DB default `NEW` | none | used |
| `trackingUrl` | `String` | `tracking_url TEXT` | nullable | none | used |
| `note` | `String` | `note TEXT` | nullable | none | used |
| `totalPrice` | `BigDecimal` | `total_price NUMERIC(10,2)` | nullable | none | used |
| `deliveryDate` | `LocalDate` | `delivery_date DATE` | nullable in DB, required by API validation | none | used |
| `slotId` | `Long` | `slot_id BIGINT` | nullable | none | reserved (no service/controller usage) |
| `service` | `CatalogService` | `service_id BIGINT` | nullable FK `service(id)` | `@ManyToOne(fetch = LAZY)` | reserved (no service/controller usage) |
| `deletedAt` | `OffsetDateTime` | `deleted_at TIMESTAMPTZ` | nullable | none | used in repository filters |
| `createdAt` | `OffsetDateTime` | `created_at TIMESTAMPTZ` | `NOT NULL`, DB default `NOW()` | none | used |
| `updatedAt` | `OffsetDateTime` | `updated_at TIMESTAMPTZ` | `NOT NULL`, DB default `NOW()` | none | used |
| `version` | `Long` | `version BIGINT` | `NOT NULL`, DB default `0`, optimistic lock | `@Version` | used |

### `BookingItem` (`booking_item`)

| Field | Java type | DB column/type | Nullability/default/constraints | Relationships | Used vs reserved |
|---|---|---|---|---|---|
| `id` | `Long` | `id BIGSERIAL` | PK | none | used |
| `booking` | `Booking` | `booking_id BIGINT` | `NOT NULL`, FK `booking(id) ON DELETE CASCADE` | `@ManyToOne(fetch = LAZY, optional = false)` | used |
| `service` | `CatalogService` | `service_id BIGINT` | `NOT NULL`, FK `service(id)` | `@ManyToOne(fetch = LAZY, optional = false)` | used |
| `quantity` | `int` | `quantity INT` | `NOT NULL`, DB check `quantity > 0` | none | used |
| `unitPrice` | `BigDecimal` | `unit_price NUMERIC(10,2)` | `NOT NULL` | none | used |
| `currency` | `String` | `currency VARCHAR(10)` | `NOT NULL` | none | used |

### `AuditLog` (`audit_log`)

| Field | Java type | DB column/type | Nullability/default/constraints | Relationships | Used vs reserved |
|---|---|---|---|---|---|
| `id` | `Long` | `id BIGSERIAL` | PK | none | used |
| `tenantId` | `Long` | `tenant_id BIGINT` | `NOT NULL` | none | used |
| `entity` | `String` | `entity VARCHAR(50)` | `NOT NULL` | none | used |
| `entityId` | `Long` | `entity_id BIGINT` | `NOT NULL` | none | used |
| `action` | `String` | `action VARCHAR(50)` | `NOT NULL` | none | used |
| `actorId` | `String` | `actor_id VARCHAR(255)` | nullable | none | used |
| `oldValue` | `String` | `old_value TEXT` | nullable | none | used |
| `newValue` | `String` | `new_value TEXT` | nullable | none | used |
| `createdAt` | `OffsetDateTime` | `created_at TIMESTAMPTZ` | `NOT NULL`, DB default `NOW()` | none | used |

## 4) Enums

| Enum | Values | Referenced in |
|---|---|---|
| `BookingStatus` | `NEW`, `PAYMENT_PENDING`, `CONFIRMED`, `DELIVERING`, `DONE`, `CANCELLED` | `Booking`, `BookingService` transition matrix, `UpdateStatusRequest`, `BookingStatusForm`, booking/admin controllers |
| `BookingType` | `ORDER`, `APPOINTMENT`, `REQUEST` | `Booking.type`, `BookingService.createFoodOrder` sets `ORDER` |
| `ServiceStatus` | `ACTIVE`, `INACTIVE`, `DELETED` | `CatalogService.status`, catalog repositories/services/controllers/forms |
| `TenantType` | `FOOD_ORDER`, `APPOINTMENT`, `CATALOG_REQUEST` | `Tenant.type`, tenant create/update DTOs/forms/services; food-order gating in `BookingService` and `AdminCatalogService` |

## 5) Database schema (Flyway-derived)

### Final reconstructed schema (after V1..V8)

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

CREATE INDEX idx_service_tenant
    ON service(tenant_id, status, sort_order, id)
    WHERE status <> 'DELETED';

CREATE INDEX idx_booking_tenant ON booking(tenant_id);
CREATE INDEX idx_booking_tenant_date ON booking(tenant_id, delivery_date);
CREATE INDEX idx_booking_tenant_status ON booking(tenant_id, status);
CREATE INDEX idx_booking_telegram_user ON booking(tenant_id, telegram_user_id);

CREATE INDEX idx_audit_tenant_entity ON audit_log(tenant_id, entity, entity_id);
CREATE INDEX idx_audit_created_at_id ON audit_log(created_at DESC, id DESC);
CREATE INDEX idx_audit_tenant_created_at_id ON audit_log(tenant_id, created_at DESC, id DESC);
CREATE INDEX idx_audit_action_created_at_id ON audit_log(action, created_at DESC, id DESC);
```

### Migration history

| Version | Filename | One-line summary |
|---|---|---|
| 1 | `V1__baseline_schema.sql` | Initial tables (`tenant`, `tenant_config`, `service`, `booking`, `booking_item`, `audit_log`) + base indexes. |
| 2 | `V2__audit_actor_id_as_string.sql` | `audit_log.actor_id` changed from numeric to `VARCHAR(255)`. |
| 3 | `V3__service_status_model.sql` | Added `service.status`, backfilled from `active/deleted_at`, rebuilt service index, dropped `active`. |
| 4 | `V4__audit_log_filter_indexes.sql` | Added audit feed/filter indexes by `created_at`, `(tenant_id, created_at)`, `(action, created_at)`. |
| 5 | `V5__booking_optimistic_lock.sql` | Added `booking.version` (`NOT NULL DEFAULT 0`) for optimistic locking. |
| 6 | `V6__booking_item_currency.sql` | Added `booking_item.currency`, backfilled from tenant `currency` config with fallback `USD`, set `NOT NULL`. |
| 7 | `V7__booking_delivery_address.sql` | Added `booking.delivery_address` (`TEXT`). |
| 8 | `V8__booking_tracking_url.sql` | Added `booking.tracking_url` (`TEXT`). |

## 6) Schema vs Entity drift

| Area | Flyway schema | Entity mapping | Difference |
|---|---|---|---|
| `tenant_config` uniqueness | `UNIQUE (tenant_id, key)` | `TenantConfig` has no `@Table(uniqueConstraints=...)` | DB-enforced unique constraint not declared in JPA metadata. |
| FK delete behavior | `booking_item.booking_id` uses `ON DELETE CASCADE` | `BookingItem.booking` has no cascade remove/orphan config in JPA | Delete cascade exists only at DB layer. |
| DB defaults (`tenant`) | defaults on `timezone`, `active`, `created_at` | `Tenant` fields have no `columnDefinition/default` metadata | Defaults rely on DB; service usually sets values explicitly on create. |
| DB defaults (`service`) | defaults on `unit`, `status`, `sort_order`, `created_at`, `updated_at` | `CatalogService` has no default metadata | Defaults in DB, while service code also sets values at write time. |
| DB defaults (`booking`) | defaults on `status`, `created_at`, `updated_at`, `version` | `Booking` has no default metadata (except `@Version`) | Defaults in DB; service sets most values explicitly. |
| DB default (`audit_log.created_at`) | `DEFAULT NOW()` | no default metadata in `AuditLog` | Timestamp default DB-side; service sets `createdAt` explicitly. |
| `audit_log.actor_id` length | `VARCHAR(255)` | `@Column(name="actor_id")` with implicit provider length | Explicit SQL length not mirrored as explicit annotation parameter. |

## 7) Security

### Security filter chains (`SecurityConfig`)

| Order | Bean method | Matcher | Authz | Custom filters and order |
|---|---|---|---|---|
| 1 | `superAdminSecurityFilterChain` | `/superadmin/**` | `anyRequest().authenticated()` | `SuperAdminBasicAuthenticationFilter` added **before** `BasicAuthenticationFilter` |
| 2 | `adminSecurityFilterChain` | `/admin/*/**` | `anyRequest().authenticated()` | `TenantContextFilter` added **before** `BasicAuthenticationFilter`; `TenantBasicAuthenticationFilter` added **after** `TenantContextFilter` |
| 3 | `tenantPublicSecurityFilterChain` | `/t/*/**` | `anyRequest().permitAll()` | `TenantContextFilter` added **before** `AnonymousAuthenticationFilter` |
| 4 | `defaultSecurityFilterChain` | fallback | `permitAll` | no custom filters |

All chains disable CSRF, form login, logout, HTTP Basic framework handler; custom filters handle basic auth where needed.  
`FilterRegistrationBean` for all custom filters is disabled to avoid servlet container auto-registration outside Spring Security chain.

### Custom filters behavior

| Filter | Credential source | Validation logic | Failure response |
|---|---|---|---|
| `SuperAdminBasicAuthenticationFilter` | `Authorization: Basic ...` | Decodes credentials, compares exact username/password against `SecurityProperties.superadmin` | `401` + `WWW-Authenticate: Basic realm="Yoobu Super Admin"` with reason (`Missing...`, `Invalid...`) |
| `TenantBasicAuthenticationFilter` | `Authorization: Basic ...` | Requires `TenantContext` first; loads tenant admin settings from `TenantSettingsService` (`admin_username`, `admin_password` hash). Valid if BCrypt matches tenant creds. Also accepts superadmin plain credentials as bypass. | `401` + tenant realm (`Yoobu Tenant Admin: {slug}`) with reason (`Missing...`, `Invalid...`, `Admin credentials are not configured`) |
| `TenantContextFilter` | URL path segment | Extracts slug from URI segments after `t` or `admin`, loads active tenant via `TenantRepository.findBySlugAndActiveTrue` and stores in `TenantContext` thread local | Missing slug -> `400 Tenant slug is missing`; missing/inactive tenant -> `404 Tenant not found` |

### Tenant context

| Item | Behavior |
|---|---|
| Attachment points | `TenantContextFilter` is attached to `/admin/*/**` and `/t/*/**` chains, not to `/superadmin/**`. |
| Slug parsing | `requestUri.split("/")`, first segment `t` or `admin`, next non-blank segment = tenant slug. |
| Storage | `TenantContext` static `ThreadLocal<Tenant>` with `setCurrentTenant`, `getCurrentTenant`, `requireCurrentTenant`, `getRequiredTenantId`, `clear`. |

### Telegram auth

| Component | Behavior |
|---|---|
| `TelegramInitDataValidator.validate` | Parses query-string-like init data into sorted map, removes `hash`, computes Telegram WebApp hash using HMAC-SHA256 (`secretKey = HMAC("WebAppData", botToken)` then HMAC over data check string), compares with constant-time `MessageDigest.isEqual`. Extracts `user` JSON, requires non-null `id`. |
| Tenant coupling | Uses `TenantContext.getCurrentTenant().botToken`; missing tenant or missing token -> `401 Invalid initData`. |
| Resolver | `TelegramUserArgumentResolver` supports only `@TelegramPrincipal TelegramUser` params. Reads header `X-Telegram-Init-Data`; if absent and active profile matches `dev`, allows fallback from `X-Telegram-User-Id` (returns synthetic `TelegramUser(id, "Dev", "User", null)`). |
| Dev fallback | Enabled only when `environment.matchesProfiles("dev")`. Invalid numeric header in dev -> `401 Invalid initData`. Non-dev without init data -> `401 Invalid initData`. |

## 8) Service layer logic

### `BookingService`

| Public method | Called by | Behavior / rules | Audit log |
|---|---|---|---|
| `createFoodOrder(CreateBookingRequest, Long telegramUserId)` | `BookingController.createBooking` | Requires tenant type `FOOD_ORDER`; validates delivery date via `TenantTimeService` cutoff; builds booking + items from active services only; computes total; item currency from tenant pricing currency (trimmed, fallback `USD`). | `logCreate(entity="booking", actorId=telegramUserId)` |
| `getMyBookings(Long)` | `BookingController.getMyBookings` | Tenant + user scoped list, excludes soft-deleted. | none |
| `getMyBooking(Long, Long)` | `BookingController.getBooking` | Tenant + user scoped single fetch. | none |
| `cancelMyBooking(Long, Long)` | `BookingController.cancelBooking` | Disallows cancel from `DONE`; sets `CANCELLED`. | `logAction(..., action="CANCEL", actorId=telegramUserId, old/new snapshot)` |
| `confirmMyBookingPayment(Long, Long)` | `BookingController.confirmPayment` | Only allowed from `NEW`; sets `PAYMENT_PENDING`. | `logAction(..., action="CONFIRM_PAYMENT", actorId=telegramUserId, old/new snapshot)` |
| `getAdminBookings(BookingStatus, LocalDate)` | `AdminBookingController.getBookings` | Tenant-scoped admin list with optional status/date filters. | none |
| `getAdminBookingsPage(BookingStatus, LocalDate, int, int)` | `AdminPanelBookingController.bookings` | Same filters, pageable (`size` normalized: `<1 => 10`, max `100`). | none |
| `getAdminBooking(Long)` | `AdminBookingController.getBooking`, `AdminPanelBookingController.bookingDetail` | Tenant-scoped single fetch. | none |
| `updateBookingStatus(Long, BookingStatus)` / overload with tracking URL | REST and panel update handlers | Enforces transition matrix (`NEW -> {NEW,CANCELLED}`, `PAYMENT_PENDING -> {PAYMENT_PENDING,CONFIRMED,CANCELLED}`, `CONFIRMED -> {CONFIRMED,DELIVERING,CANCELLED}`, `DELIVERING -> {DELIVERING,DONE,CANCELLED}`, `DONE -> {DONE,CANCELLED}`, `CANCELLED -> {CANCELLED}`); validates tracking URL (http/https + host) when provided; preserves existing tracking URL when `trackingUrl == null`; blank tracking URL clears it; optimistic lock conflicts mapped to `409`. | `logAction(..., action="UPDATE_STATUS", actorId=currentActorId())` |
| `getAllowedAdminStatuses(BookingStatus)` | panel controllers | Returns allowed enum subset for UI controls. | none |

### `AdminCatalogService`

| Public method | Called by | Behavior / rules | Audit log |
|---|---|---|---|
| `getAdminServices()` | `AdminCatalogController.getServices` | Requires `FOOD_ORDER`; returns all non-`DELETED`. | none |
| `getAdminServicesPage(String,int,int)` | `AdminPanelServiceController.services` | Optional name contains filter; pageable (`<1 => 20`, max `100`). | none |
| `getAdminService(Long)` | panel edit/delete/status flows | Tenant + non-deleted lookup. | none |
| `createService(AdminUpsertServiceRequest)` | REST create, panel create | Requires `FOOD_ORDER`; applies defaults (`unit="шт"` when blank, `sortOrder=0` when null, status default `ACTIVE`); rejects status `DELETED` in upsert. | `logCreate(entity="service", actorId=currentActorId())` |
| `updateService(Long, AdminUpsertServiceRequest)` | REST update, panel update/status | Same rules; if status not `DELETED` then clears `deletedAt`; updates `updatedAt`. | `logUpdate(entity="service", actorId=currentActorId(), old/new)` |
| `deleteService(Long)` | REST delete, panel delete | Soft delete: status `DELETED`, sets `deletedAt`. | `logAction(action="DELETE", entity="service", actorId=currentActorId(), old/new)` |

### `CatalogQueryService`

| Public method | Called by | Behavior |
|---|---|---|
| `getActiveServices()` | `CatalogController.getServices` | Tenant-scoped `ACTIVE` services ordered by `sortOrder,id`. |

### `TenantManagementService`

| Public method | Called by | Behavior / rules | Audit log |
|---|---|---|---|
| `getAllTenants()` | `SuperAdminTenantController.getTenants` | Returns all tenants ordered by createdAt desc. | none |
| `getAllTenantsPage(String,int,int)` | `SuperAdminPanelController.tenants` | Optional query against name/slug; pageable (`<1 => 20`, max `100`). | none |
| `getTenant(Long)` | superadmin REST + panel detail/edit | Loads tenant + full config map (`TenantSettings.asMap`). | none |
| `isSlugAvailable(String)` | slug-availability endpoint, panel create validation | `true` only for non-blank trimmed slug absent in DB. | none |
| `createTenant(CreateTenantRequest)` | superadmin REST + panel create | Rejects duplicate slug (`409`); sets defaults (`timezone` fallback `Asia/Ho_Chi_Minh`, active true, currency fallback `USD`); stores tenant config entries including hashed admin password and normalized payment QR URL. | `logCreate(entity="tenant", tenantId=createdTenantId, actorId=currentActorId(), snapshot)` |
| `updateTenant(Long, UpdateTenantRequest)` | superadmin REST + panel update | Updates tenant core fields; upserts configs; blank optional fields removed when `removeWhenBlank=true`; blank `adminPassword` keeps existing hash; currency maintained/seeded to default if missing; payment QR normalized/validated. | `logUpdate(entity="tenant", tenantId=tenantId, actorId=currentActorId(), old/new snapshot)` |

### `TenantSettingsService`

| Public method | Called by | Behavior |
|---|---|---|
| `getSettings(Long tenantId)` | auth, booking, tenant config mapping, tenant management snapshots | Loads all `tenant_config` rows and builds `TenantSettings`. |
| `getCurrentTenantSettings()` | tenant/public/panel flows | Same using `TenantContext.getRequiredTenantId()`. |

### `TenantConfigService`

| Public method | Called by | Behavior |
|---|---|---|
| `getCurrentTenantConfig()` | `TenantPublicController.getConfig` | Reads current tenant + settings and maps to `TenantConfigResponse`. |

### `TenantTimeService`

| Public method | Called by | Behavior |
|---|---|---|
| `now(Tenant)` / `today(Tenant)` | tests, time helpers | Uses injected `Clock` and tenant `ZoneId`. |
| `earliestDeliveryDate(Tenant, TenantSettings)` | `BookingService.validateDeliveryDate` | Uses optional `cutoff_hour` + `cutoff_minute`; if now is at/after cutoff, earliest date is tomorrow, else today. Incomplete/invalid cutoff config throws `IllegalStateException`. |

### `AuditLogService`

| Public method | Called by | Behavior |
|---|---|---|
| `logCreate`, `logUpdate`, `logAction` | `BookingService`, `AdminCatalogService`, `TenantManagementService` | Persists serialized old/new snapshots with UTC timestamp. |
| `currentActorId()` | business services | Reads Spring Security principal; supports numeric/string principals. |
| `search(...)` | `SuperAdminAuditController.getAuditLogs`, `SuperAdminPanelController.audit` | Filter by tenant/entity/action/actor/time range; sort `createdAt DESC, id DESC`; page size normalized (`default 20`, max `50`). |
| `searchForExport(...)` | audit export endpoints | Same filters, first page only, size capped at `5000`. |
| `exportLimit()` | panel audit page | Returns `5000`. |

### `AuditLogCsvExporter`

| Public method | Called by | Behavior |
|---|---|---|
| `toCsv(List<AuditLogItemResponse>)` | superadmin REST export + panel export | Emits CSV headers + rows; wraps values in quotes; escapes quotes; prefixes `'=+-@` leading cells with `'` to avoid spreadsheet formula injection. |

### `AuditLogChangeFormatter`

| Public method | Called by | Behavior |
|---|---|---|
| `buildDiffLines(AuditLogItemResponse)` | panel audit view, csv exporter | For map payloads, compares top-level keys and renders `key: old -> new`; else `value: old -> new`; returns fallback messages for no changes. |
| `summarizeChanges(AuditLogItemResponse)` | csv exporter | Joins diff lines by ` | `. |

## 9) Configuration

### `application.yml` (full)

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

`application-*.yml` in `src/main/resources`: not present.

### `AppProperties` fields and consumption

`AppProperties` class is not present in the source tree.  
`app.*` keys are bound via:

| Class | Prefix | Fields | Consumed by |
|---|---|---|---|
| `SecurityProperties` | `app` | `superadmin.username`, `superadmin.password` | `SecurityConfig` -> `SuperAdminBasicAuthenticationFilter`; also superadmin fallback in `TenantBasicAuthenticationFilter` |
| `CorsProperties` | `app.cors` | `allowedOriginPatterns`, `allowedMethods`, `allowedHeaders`, `exposedHeaders`, `allowCredentials`, `maxAge` | `CorsConfig.corsConfigurationSource` |

### Tenant config keys (`tenant_config.key`) and readers

| Key | Written by | Read by |
|---|---|---|
| `admin_username` | `TenantManagementService.createTenant/updateTenant` | `TenantSettings.admin()`, `TenantBasicAuthenticationFilter`, superadmin tenant detail/panel |
| `admin_password` | create/update (BCrypt hash) | `TenantSettings.admin()`, `TenantBasicAuthenticationFilter` |
| `currency` | create/update (default `USD`) | `TenantSettings.pricing()`, `BookingService` currency assignment, panel/service money formatting context |
| `primary_color` | create/update | `TenantSettings.branding()`, `/t/{slug}/config`, panel |
| `logo_url` | create/update | same |
| `welcome_message` | create/update | same |
| `checkout_name_hint` | create/update | same |
| `checkout_phone_hint` | create/update | same |
| `checkout_note_hint` | create/update | same |
| `payment_qr_url` | create/update (normalized by `PaymentQrUrlValidator`) | `TenantSettings.payment()`, `/t/{slug}/config`, panel |
| `cutoff_hour` | not written by current controller/service flows | `TenantSettings.delivery()`, `TenantTimeService` |
| `cutoff_minute` | not written by current controller/service flows | `TenantSettings.delivery()`, `TenantTimeService` |

## 10) Test coverage

### Test infrastructure

| Class | Infra details |
|---|---|
| `IntegrationTestSupport` | `@SpringBootTest`, `@AutoConfigureMockMvc`, `@ActiveProfiles("dev")`; shared `PostgreSQLContainer("postgres:17-alpine")`; dynamic datasource/superadmin props; DB reset before each test via `TRUNCATE ... RESTART IDENTITY CASCADE`; helpers for superadmin/tenant basic auth and Telegram dev header. |
| Web MVC tests (`BookingControllerWebMvcTest`, `AdminBookingControllerWebMvcTest`) | `@WebMvcTest(..., addFilters=false)` + stubbed services + custom `TelegramInitDataValidator` bean. |
| Unit tests (`BookingServiceTest`, `AuditLogServiceTest`, etc.) | Proxy stubs/test doubles for repositories/services; no DB container. |

### Per test class coverage

| Test class | Coverage summary |
|---|---|
| `CorsConfigurationIT` | CORS headers on tenant public endpoint and admin preflight behavior without auth challenge. |
| `TelegramUserArgumentResolverTest` | Resolver parameter support, initData path, dev fallback header path, invalid/no header unauthorized behavior. |
| `AuditLogServiceTest` | Search page-size caps, export cap, principal extraction, enum/temporal serialization, nested JSON parsing from text fields. |
| `TenantContextTest` | `TenantContext` success/failure semantics. |
| `TenantSettingsServiceTest` | `getSettings` and `getCurrentTenantSettings` wiring to repository/context. |
| `TenantSettingsTest` | Duplicate-key override, read-only map, domain slices (`admin/branding/checkout/delivery/pricing`) and default currency fallback. |
| `TenantTimeServiceTest` | Timezone-aware dates, cutoff behavior, edge at exact cutoff, no cutoff behavior. |
| `PaymentQrUrlValidatorTest` | Valid/invalid URL normalization rules and error reason. |
| `BookingControllerWebMvcTest` | Client booking controller binding/validation and delegation for create/list/get/cancel/confirm endpoints. |
| `AdminBookingControllerWebMvcTest` | Admin status update binding (`trackingUrl` present/null), max-length validation. |
| `BookingServiceTest` | Total/currency computation, transition rejection/allowance cases, tracking URL normalization rules, page-size normalization, allowed-status matrix slices. |
| `TenantAdminAccessIT` | Tenant basic auth success/failure, superadmin access to tenant admin endpoints, missing credentials challenge. |
| `TenantIsolationIT` | Cross-tenant admin credential isolation and service visibility separation. |
| `TenantFoodOrderConstraintsIT` | Tenant type gating (`FOOD_ORDER` only), unknown/deleted service booking rejection, cross-tenant booking admin access denial. |
| `TenantAdminCatalogAndBookingIT` | End-to-end tenant admin service create + customer booking + admin listing/filtering. |
| `ServiceManagementAndValidationIT` | Public config output, payment QR update/clear reflection, service update/delete behavior, booking status filter, request payload validation, delivery-date validation by tenant timezone. |
| `BookingLifecycleIT` | Full booking lifecycle for customer and admin, ownership constraints, payment confirm rules, transition matrix outcomes, tracking URL propagation, audit entries, currency immutability for existing bookings after tenant currency change. |
| `BookingOptimisticLockingIT` | JPA optimistic locking conflict on stale booking update (`version` behavior). |
| `SuperAdminTenantControllerIT` | Superadmin auth/challenge paths, tenant create/read/update, slug availability, duplicate slug, payment QR validation, password keep/rotate behavior, deactivation effects. |
| `SuperAdminAuditControllerIT` | Audit API filtering/pagination/caps/auth/export CSV. |
| `SuperAdminPanelIT` | Superadmin Thymeleaf panel tenant CRUD forms, credential rotation, duplicate slug form error, panel audit rendering/filter/export links. |
| `AdminPanelIT` | Tenant admin Thymeleaf panel rendering, booking/service form mutations, inline status updates, delete confirmation flow, tracking URL UI behavior, flash messaging and redirect behavior. |
| `AuditLogIndexPlanIT` | `EXPLAIN` index usage assertions for audit queries (`idx_audit_*`). |

## 11) Not yet implemented / currently unused

### Schema/entity elements with no active controller/service flow

| Item | Location | Current status |
|---|---|---|
| Appointment/request booking types | `BookingType.APPOINTMENT`, `BookingType.REQUEST` | No controller/service path creates these; current booking creation always sets `BookingType.ORDER`. |
| Booking slot linkage | `booking.slot_id`, `Booking.slotId` | No controller/service logic reads/writes `slotId`. |
| Direct booking->service link | `booking.service_id`, `Booking.service` | No controller/service logic uses this relation; itemized model uses `booking_item.service_id`. |
| Delivery cutoff config write path | `TenantConfigKeys.CUTOFF_HOUR`, `TenantConfigKeys.CUTOFF_MINUTE` | Read by `TenantTimeService`, but no current create/update endpoint/form sets these keys. |
| `TenantConfigRepository.findByTenantIdAndKey` | repository method | Declared but not used by current production code paths. |

### Referenced-in-comments/docs missing classes

No missing classes were identified from in-code comments in scanned source files.

