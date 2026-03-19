# Admin Panel R&D for Angular Migration

## Purpose

This document describes the current admin surface implemented with Spring MVC + Thymeleaf and defines the behavior that a standalone Angular replacement must preserve.

Primary migration target:

- tenant admin UI: `/admin/{slug}/panel/**`
- superadmin UI: `/superadmin/panel/**`

Current server-rendered pages:

- tenant admin bookings list
- tenant admin booking detail
- tenant admin services list
- tenant admin create service
- tenant admin edit service
- superadmin tenants list
- superadmin tenant detail
- superadmin create tenant
- superadmin edit tenant

Out of scope for this document:

- public tenant app at `/t/{slug}/**`
- back-office process design beyond what directly affects the admin app

This is intentionally implementation-facing so another AI agent can use it as a build brief.

## Current Architecture

The current admin product is server-rendered, but most CRUD data already exists behind JSON endpoints.

HTML pages:

- `/admin/{slug}/panel/bookings`
- `/admin/{slug}/panel/bookings/{bookingId}`
- `/admin/{slug}/panel/services`
- `/admin/{slug}/panel/services/new`
- `/admin/{slug}/panel/services/{serviceId}/edit`
- `/superadmin/panel/tenants`
- `/superadmin/panel/tenants/{tenantId}`
- `/superadmin/panel/tenants/new`
- `/superadmin/panel/tenants/{tenantId}/edit`

JSON endpoints already available:

- `/admin/{slug}/bookings`
- `/admin/{slug}/bookings/{bookingId}`
- `/admin/{slug}/bookings/{bookingId}/status`
- `/admin/{slug}/services`
- `/admin/{slug}/services/{serviceId}`
- `/superadmin/tenants`
- `/superadmin/tenants/{tenantId}`
- `/superadmin/tenants/slug-availability`

Implication:

- Angular should be built as a standalone frontend consuming the existing `/admin/...` and `/superadmin/...` JSON APIs first.
- A page-for-page Angular rewrite does not require backend feature invention for the core admin CRUD surface.

## Authentication and Access Model

Admin auth is HTTP Basic.

Tenant admin:

- endpoints under `/admin/*/**` require `Authorization: Basic ...`
- challenge realm: `Yoobu Tenant Admin: {slug}`
- credentials are tenant-specific
- superadmin credentials are also accepted on tenant admin endpoints

Superadmin:

- endpoints under `/superadmin/**` require `Authorization: Basic ...`
- challenge realm: `Yoobu Super Admin`

Shared backend behavior:

- CSRF is disabled
- CORS is enabled for `/admin/**` and `/superadmin/**`

Implications for Angular:

- browser-native Basic auth prompts are not sufficient for a standalone SPA
- the app needs an explicit auth/session handling strategy
- reusing Basic auth is possible for the first migration, but means storing credentials client-side and setting headers manually
- if Angular is hosted on another origin, CORS is already conceptually supported, but auth UX remains weak

Role model to preserve:

- superadmin can use superadmin routes
- tenant admin can use tenant admin routes only for its own slug
- superadmin can also open tenant admin routes for any tenant slug

Recommended note for next phase:

- keep current backend auth for the first migration spike if speed matters
- plan a follow-up auth redesign if the standalone app is intended for serious production use

## Scope Model

The admin product has two scopes.

Tenant admin scope:

- scoped by `{slug}`
- strict tenant isolation
- wrong tenant credentials against another slug return `401`
- bookings and services management currently only work for `TenantType.FOOD_ORDER`

Superadmin scope:

- global tenant management by `tenantId`
- create tenants
- edit tenants
- inspect tenant settings and credentials metadata
- jump directly into a tenant admin panel by slug

Angular must treat both of these as first-class contexts:

- `{slug}` for tenant admin
- `tenantId` for superadmin tenant management

## Existing Navigation

Tenant admin navigation today:

- `Bookings`
- `Services`

Tenant admin default route:

- `/admin/{slug}/panel`
- redirects to `/admin/{slug}/panel/bookings`

Superadmin navigation today:

- `Tenants`
- `Create tenant`

Superadmin default route:

- `/superadmin/panel`
- redirects to `/superadmin/panel/tenants`

Expected Angular route structure:

- `/admin/:slug`
- `/admin/:slug/bookings`
- `/admin/:slug/bookings/:bookingId`
- `/admin/:slug/services`
- `/admin/:slug/services/new`
- `/admin/:slug/services/:serviceId/edit`
- `/superadmin`
- `/superadmin/tenants`
- `/superadmin/tenants/new`
- `/superadmin/tenants/:tenantId`
- `/superadmin/tenants/:tenantId/edit`

If the app is deployed under another base path, route mapping can differ, but the page model should remain the same.

## Feature Inventory

### A. Tenant Admin

### A1. Bookings List

Current page:

- `/admin/{slug}/panel/bookings`

Purpose:

- primary landing page for tenant admin
- shows all bookings for the current tenant
- supports lightweight filtering
- allows inline status updates directly from the table

Data source:

- `GET /admin/{slug}/bookings`
- optional query params:
  - `status`
  - `deliveryDate` in `YYYY-MM-DD`

Displayed columns:

- booking ID
- customer name
- booking status
- delivery date
- total price
- view link

Existing filters:

- status dropdown
  - options are all `BookingStatus` values:
    - `NEW`
    - `CONFIRMED`
    - `DONE`
    - `CANCELLED`
- delivery date input

Existing inline actions:

- per-row status select
- per-row `Update` action
- per-row `View` link to booking detail

Empty state:

- `No bookings found.`

Behavior details to preserve:

- list is sorted by backend, newest first
- filter state is reflected in request query params
- inline status change is allowed from list view without opening detail
- current Thymeleaf flow redirects back to list after update; Angular should update local state or refetch list

### A2. Booking Detail

Current page:

- `/admin/{slug}/panel/bookings/{bookingId}`

Purpose:

- inspect one booking
- update its status from a dedicated screen

Data source:

- `GET /admin/{slug}/bookings/{bookingId}`

Displayed summary fields:

- status
- customer name
- delivery date
- total price
- created timestamp
- note
  - if absent, show `N/A`

Displayed items table:

- service name
- quantity
- unit price

Actions:

- status select
- `Save status`

Status update endpoint:

- `PUT /admin/{slug}/bookings/{bookingId}/status`

Request:

```json
{
  "status": "CONFIRMED"
}
```

Behavior details to preserve:

- detail page initializes status selector from current booking status
- all booking statuses are selectable
- current Thymeleaf flow redirects back to list after update
- Angular can preserve that or stay on detail and refresh, but the choice should be explicit

### A3. Services List

Current page:

- `/admin/{slug}/panel/services`

Purpose:

- manage tenant catalog services
- create new service
- edit existing service
- toggle service status inline

Data source:

- `GET /admin/{slug}/services`

Displayed columns:

- name
- price
- unit
- duration
- sort order
- status
- edit link

Existing toolbar action:

- `New service`

Existing inline actions:

- per-row status select
- per-row `Update` action
- per-row `Edit` link

Service statuses currently used in UI:

- `ACTIVE`
- `INACTIVE`

Important backend rule:

- `DELETED` exists in backend enum but must not be set through normal create/update flow
- deletion is a separate endpoint

Display details:

- duration shows `N/A` when null
- status select gets different styling for active vs inactive

Empty state:

- `No services yet.`

Behavior details to preserve:

- list excludes soft-deleted services
- list ordering is backend-driven by `sortOrder ASC, id ASC`
- inline status updates reuse the existing service data and only replace status

### A4. Create Service

Current page:

- `/admin/{slug}/panel/services/new`

Purpose:

- create a new catalog item/service

Create endpoint:

- `POST /admin/{slug}/services`

Request body shape:

```json
{
  "name": "Pasta",
  "description": "Fresh pasta",
  "price": 14.50,
  "unit": "bowl",
  "durationMinutes": 25,
  "sortOrder": 2,
  "status": "ACTIVE"
}
```

Fields in current form:

- `name` required
- `description` optional
- `price` required
- `unit` optional
- `durationMinutes` optional
- `sortOrder` optional
- `status` required in UI, defaults to `ACTIVE`

Current defaults:

- `sortOrder = 0`
- `status = ACTIVE`

Backend normalization rules:

- blank `unit` becomes `"шт"`
- null `sortOrder` becomes `0`
- null `status` becomes `ACTIVE`

Current validation visible in Thymeleaf:

- name must not be blank
- price must not be null

### A5. Edit Service

Current page:

- `/admin/{slug}/panel/services/{serviceId}/edit`

Purpose:

- edit all service fields
- delete service

Load endpoint:

- `GET /admin/{slug}/services/{serviceId}`

Update endpoint:

- `PUT /admin/{slug}/services/{serviceId}`

Delete endpoint:

- `DELETE /admin/{slug}/services/{serviceId}`

Behavior details to preserve:

- form loads all existing values
- inactive status must remain selected correctly when editing an inactive service
- delete action requires confirmation
- delete is soft delete on backend
- repeated delete returns `404 Service not found`

Current delete UX:

- separate delete button
- browser confirm dialog with `Delete this service?`

### B. Superadmin

### B1. Tenants List

Current page:

- `/superadmin/panel/tenants`

Purpose:

- entry point for superadmin operations
- list all tenants
- provide quick navigation into tenant detail and tenant admin

Data source:

- `GET /superadmin/tenants`

Displayed columns:

- tenant name
- slug
- type
- status
- timezone
- created timestamp
- open admin panel link

Existing actions:

- open tenant detail
- open tenant admin panel
- create new tenant

Empty state:

- `No tenants yet.`

Behavior details to preserve:

- list is backend sorted by `createdAt DESC`
- each row links to both superadmin detail and tenant admin
- active status is displayed clearly

### B2. Tenant Detail

Current page:

- `/superadmin/panel/tenants/{tenantId}`

Purpose:

- inspect operational tenant metadata in one place
- expose quick links to related surfaces

Data source:

- `GET /superadmin/tenants/{tenantId}`

Displayed tenant fields:

- ID
- slug
- type
- timezone
- owner Telegram ID
- created timestamp
- active flag

Displayed integration/config fields:

- bot token
- primary color
- logo URL
- welcome message
- checkout phone hint
- checkout note hint
- admin username

Quick links shown today:

- tenant admin panel URL
- public catalog API URL

Existing actions:

- open tenant admin panel
- edit tenant

Behavior details to preserve:

- missing optional fields render as `N/A`
- detail page is operational, not just informational
- direct tenant admin link is part of the workflow

### B3. Create Tenant

Current page:

- `/superadmin/panel/tenants/new`

Purpose:

- provision a new tenant, including admin credentials and branding/config defaults

Create endpoint:

- `POST /superadmin/tenants`

Create request shape:

```json
{
  "slug": "panel-tenant",
  "name": "Panel Tenant",
  "type": "FOOD_ORDER",
  "botToken": "panel-bot",
  "ownerTelegramId": 123456,
  "timezone": "Asia/Ho_Chi_Minh",
  "primaryColor": "#112233",
  "logoUrl": "https://cdn.example.com/logo.png",
  "welcomeMessage": "Hello from panel",
  "checkoutPhoneHint": "+84...",
  "checkoutNoteHint": "No onion, gate code, delivery code",
  "adminUsername": "panel-admin",
  "adminPassword": "panel-secret"
}
```

Fields in current form:

- `slug` required, immutable after create
- `name` required
- `type` required, defaults to `FOOD_ORDER`
- `botToken` optional
- `ownerTelegramId` optional
- `timezone` optional, defaults to `Asia/Ho_Chi_Minh`
- `primaryColor` optional
- `logoUrl` optional
- `welcomeMessage` optional
- `checkoutPhoneHint` optional
- `checkoutNoteHint` optional
- `adminUsername` required
- `adminPassword` required on create

Existing client-side behavior in Thymeleaf:

- slug availability is checked asynchronously
- submit is disabled until slug is verified as available
- checking is debounced on input and triggered on blur
- if availability check fails, create is blocked

Slug availability endpoint:

- `GET /superadmin/tenants/slug-availability?slug=...`

Response:

```json
{
  "slug": "panel-tenant",
  "available": true
}
```

Validation/behavior details to preserve:

- duplicate slug is rejected
- create form displays `Tenant slug already exists`
- slug cannot be changed later
- tenant is created active by default

### B4. Edit Tenant

Current page:

- `/superadmin/panel/tenants/{tenantId}/edit`

Purpose:

- edit tenant metadata and config
- rotate tenant admin credentials
- activate/deactivate tenant

Update endpoint:

- `PUT /superadmin/tenants/{tenantId}`

Update request shape:

```json
{
  "name": "Panel After",
  "type": "FOOD_ORDER",
  "botToken": null,
  "ownerTelegramId": null,
  "timezone": "Asia/Ho_Chi_Minh",
  "primaryColor": null,
  "logoUrl": "https://cdn.example.com/panel-updated.png",
  "welcomeMessage": "Updated from panel",
  "checkoutPhoneHint": "+1 555...",
  "checkoutNoteHint": "Ring bell twice",
  "adminUsername": "panel-admin-2",
  "adminPassword": "panel-secret-2",
  "active": true
}
```

Behavior details to preserve:

- slug is readonly in edit mode
- admin password is optional in edit mode
- blank edit password means keep current password
- changing admin credentials invalidates old tenant-admin login and enables new one
- optional config values may be cleared
- active flag is editable only in edit mode

Important backend normalization rules:

- blank optional config values may be removed from stored config
- blank timezone falls back to default timezone

## API Contracts to Reuse

### Tenant Admin APIs

#### Bookings

`GET /admin/{slug}/bookings`

Query params:

- `status` optional
- `deliveryDate` optional, `YYYY-MM-DD`

Response item:

```json
{
  "id": 1,
  "type": "ORDER",
  "status": "NEW",
  "customerName": "Alice",
  "customerPhone": "+48123456789",
  "totalPrice": 25.0,
  "deliveryDate": "2026-03-19",
  "note": "Leave at the door",
  "items": [
    {
      "serviceName": "Pizza",
      "quantity": 2,
      "unitPrice": 12.5
    }
  ],
  "createdAt": "2026-03-18T12:34:56Z"
}
```

`GET /admin/{slug}/bookings/{bookingId}`

- same response shape

`PUT /admin/{slug}/bookings/{bookingId}/status`

Request:

```json
{
  "status": "CONFIRMED"
}
```

#### Services

`GET /admin/{slug}/services`

Response item:

```json
{
  "id": 1,
  "name": "Pizza",
  "description": "Thin crust",
  "price": 12.5,
  "unit": "pcs",
  "durationMinutes": 20,
  "sortOrder": 1,
  "status": "ACTIVE"
}
```

`POST /admin/{slug}/services`

`PUT /admin/{slug}/services/{serviceId}`

Shared request:

```json
{
  "name": "Pizza Romana",
  "description": "Thin crust",
  "price": 13.5,
  "unit": "pcs",
  "durationMinutes": 20,
  "sortOrder": 1,
  "status": "ACTIVE"
}
```

`DELETE /admin/{slug}/services/{serviceId}`

- returns `204 No Content`

### Superadmin APIs

#### Tenants

`GET /superadmin/tenants`

Response item:

```json
{
  "id": 1,
  "slug": "dark-kitchen",
  "name": "Dark Kitchen",
  "type": "FOOD_ORDER",
  "active": true,
  "timezone": "Asia/Ho_Chi_Minh",
  "createdAt": "2026-03-18T12:34:56Z"
}
```

`GET /superadmin/tenants/{tenantId}`

Response:

```json
{
  "id": 1,
  "slug": "dark-kitchen",
  "name": "Dark Kitchen",
  "type": "FOOD_ORDER",
  "active": true,
  "timezone": "Asia/Ho_Chi_Minh",
  "botToken": "panel-bot",
  "ownerTelegramId": 123456,
  "createdAt": "2026-03-18T12:34:56Z",
  "config": {
    "primary_color": "#112233",
    "logo_url": "https://cdn.example.com/logo.png",
    "welcome_message": "Hello from panel",
    "checkout_phone_hint": "+84...",
    "checkout_note_hint": "No onion, gate code, delivery code",
    "admin_username": "panel-admin"
  }
}
```

`GET /superadmin/tenants/slug-availability?slug=dark-kitchen`

Response:

```json
{
  "slug": "dark-kitchen",
  "available": true
}
```

`POST /superadmin/tenants`

- request body is the create tenant payload described above

`PUT /superadmin/tenants/{tenantId}`

- request body is the update tenant payload described above

## Error Cases and Edge Cases

These are easy to miss and should be preserved or deliberately redesigned.

### Authentication failures

Current backend returns `401` for:

- missing Basic auth header
- invalid Basic auth header
- invalid tenant credentials
- invalid superadmin credentials

Angular must handle `401` globally.

### Tenant isolation

- wrong tenant credentials against another slug return `401`
- data must never bleed across slugs

### Unsupported tenant type

- tenant admin services and bookings assume `FOOD_ORDER`
- non-food-order tenants currently fail with `400 Tenant does not support food ordering`

Angular should not assume every future tenant type supports tenant admin bookings/services unchanged.

### Validation failures

Service create/update:

- blank `name` fails
- null `price` fails

Booking status update:

- null `status` fails

Tenant create/update:

- blank `slug` fails on create
- duplicate slug fails on create
- blank `name` fails
- blank `adminUsername` fails
- blank `adminPassword` fails on create

### Service deletion semantics

- delete is soft delete, not hard delete
- deleted services are hidden from admin service list and public catalog
- deleted services cannot be updated through normal flow because they become not found

### Inactive service semantics

- inactive services still appear in admin list
- inactive services disappear from public catalog
- bookings cannot be created against inactive services

This means the Angular app should visually distinguish:

- active
- inactive
- deleted is not a user-selectable state and should not appear in UI controls

### Tenant credential rotation semantics

- superadmin editing tenant admin username/password has immediate operational effect
- old tenant admin credentials stop working after rotation
- new credentials work immediately

## UI/UX Parity Requirements

Minimum parity for first Angular release:

- role-aware entry into superadmin and tenant admin
- superadmin tenants list
- superadmin tenant detail
- superadmin create tenant form
- superadmin edit tenant form
- live slug availability check in create tenant flow
- tenant-scoped routing by slug
- bookings list with status filter and delivery date filter
- inline booking status updates from list
- booking detail page with items and status editor
- services list
- inline service status updates from list
- create service form
- edit service form
- delete service with confirmation
- loading states
- empty states
- backend validation/error display

Non-critical presentational details that do not need exact parity:

- current CSS theme
- exact button shapes
- exact table-to-mobile stacking behavior

These interaction details should be preserved:

- fast switching between superadmin tenant management and tenant admin
- quick deep link from superadmin into tenant admin
- fast switching between bookings and services
- ability to change status without opening edit/detail page
- explicit navigation back to list after create/update/delete

## Suggested Angular Module Breakdown

Recommended feature structure:

- `admin-shell`
- `admin-auth`
- `superadmin`
- `tenant-admin`
- `tenants`
- `bookings`
- `services`
- shared `api`
- shared `ui`

Suggested screens/components:

- `AdminShellComponent`
- `AdminNavComponent`
- `SuperadminTenantsPageComponent`
- `SuperadminTenantDetailPageComponent`
- `SuperadminTenantFormPageComponent`
- `BookingsPageComponent`
- `BookingsFilterComponent`
- `BookingStatusSelectComponent`
- `BookingDetailPageComponent`
- `ServicesPageComponent`
- `ServiceFormPageComponent`
- `ServiceStatusSelectComponent`
- `ConfirmDialogComponent`

Suggested services:

- `AdminSessionService`
- `SuperadminTenantsApi`
- `AdminBookingsApi`
- `AdminServicesApi`

Suggested route resolver/guard responsibilities:

- determine active role context
- extract `slug`
- extract `tenantId`
- ensure credentials/session exists
- attach auth header
- optionally preload tenant, booking, and service detail data

## State and Data Flow Notes

For first implementation, simple request-response state is enough.

Recommended behavior:

- superadmin tenant list:
  - fetch on route enter
- superadmin tenant detail:
  - fetch on route enter
- superadmin tenant form:
  - reuse one form component for create and edit
  - create mode owns slug availability checks
  - edit mode keeps slug readonly
- bookings list:
  - route query params drive filters
  - refetch on filter change
- booking detail:
  - fetch on route enter
  - update status optimistically only if simple; otherwise refetch after save
- services list:
  - fetch on route enter
  - update inline status by patching row or refetching
- service form:
  - reuse one form component for create and edit
  - create mode initializes defaults
  - edit mode loads by `serviceId`

## Backend Gaps Not Blocking Migration

### No dedicated session API

- auth is still Basic auth for both tenant admin and superadmin
- Angular can work with it, but UX and credential storage need care

### No dedicated service status endpoint in JSON API

Current server-rendered list status toggle works by reading the full service and calling update with only a different status.

Angular options:

- reuse current `PUT /admin/{slug}/services/{serviceId}` with full payload
- or add a backend `PUT /admin/{slug}/services/{serviceId}/status` later

Recommendation:

- for migration speed, use current full update endpoint
- for cleaner frontend semantics, add a dedicated status endpoint later

### No dedicated superadmin dashboard API

- current superadmin UI is tenant management only
- that is not a blocker, but means the first Angular superadmin shell can stay intentionally narrow

## Recommended Phase 1 Scope

Phase 1 should aim for operational equivalence, not redesign.

Recommended first milestone:

- standalone Angular app renders both superadmin and tenant admin
- uses existing JSON endpoints
- uses current Basic auth model
- route structure mirrors current panel features
- no public app migration

That gets the team off Thymeleaf for the whole admin surface without coupling the first step to backend redesign.

## Acceptance Checklist for the Next AI Agent

The new Angular admin app should be considered functionally equivalent when all of the following are true:

- superadmin can authenticate
- superadmin tenants page loads all tenants
- superadmin can open tenant detail
- superadmin can create a tenant
- superadmin create flow performs slug availability checks
- superadmin can edit a tenant
- superadmin can rotate tenant admin credentials
- superadmin can jump directly into tenant admin for a tenant
- user can authenticate against a tenant with existing admin credentials
- app routes by tenant slug
- bookings page loads and shows existing bookings
- bookings page filters by status
- bookings page filters by delivery date
- bookings page can change booking status inline
- booking detail page loads summary and items
- booking detail page can change status
- services page loads existing services
- services page can change service status inline
- services page can open create form
- create form can create a service with all current fields
- edit form loads current values correctly, including `INACTIVE` status
- edit form can update a service
- edit form can delete a service after confirmation
- all API validation and auth failures are surfaced clearly

## Source Pointers

Primary current implementation:

- `src/main/java/com/yoobu/api/admin/panel/AdminPanelBookingController.java`
- `src/main/java/com/yoobu/api/admin/panel/AdminPanelServiceController.java`
- `src/main/java/com/yoobu/api/admin/panel/SuperAdminPanelController.java`
- `src/main/resources/templates/admin/panel/bookings.html`
- `src/main/resources/templates/admin/panel/booking-detail.html`
- `src/main/resources/templates/admin/panel/services.html`
- `src/main/resources/templates/admin/panel/service-form.html`
- `src/main/resources/templates/superadmin/panel/tenants.html`
- `src/main/resources/templates/superadmin/panel/tenant-detail.html`
- `src/main/resources/templates/superadmin/panel/tenant-form.html`

Primary JSON APIs:

- `src/main/java/com/yoobu/api/admin/AdminBookingController.java`
- `src/main/java/com/yoobu/api/catalog/AdminCatalogController.java`
- `src/main/java/com/yoobu/api/admin/SuperAdminTenantController.java`

Behavioral test coverage:

- `src/test/java/com/yoobu/api/admin/AdminPanelIT.java`
- `src/test/java/com/yoobu/api/admin/SuperAdminPanelIT.java`
- `src/test/java/com/yoobu/api/admin/TenantAdminAccessIT.java`
- `src/test/java/com/yoobu/api/admin/TenantIsolationIT.java`
- `src/test/java/com/yoobu/api/admin/TenantAdminCatalogAndBookingIT.java`
- `src/test/java/com/yoobu/api/admin/ServiceManagementAndValidationIT.java`
- `src/test/java/com/yoobu/api/admin/BookingLifecycleIT.java`
