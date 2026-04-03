# TECH_SPEC_FIXES.md — Yoobu: Technical Debt & Immediate Fixes

Generated: 2026-04-01
Scope: Bugs, operational gaps, schema cleanup, caching layer
Priority order: P0 (live bugs) → P1 (operational blindspots) → P2 (structural debt) → P3 (performance)

---

## P0 — Live Bugs

### FIX-01: `BookingItem.currency` TypeScript/Java contract mismatch

**Status:** Bug in production  
**Layer:** Frontend (TypeScript model)

**Problem:**  
`BookingItemResponse` in Java contains three fields: `serviceName`, `quantity`, `unitPrice`.  
`BookingItem` in TypeScript defines a fourth field: `currency: string`.  
At runtime, `booking.items[0].currency` is `undefined`. Any code path that reads this field without a null guard produces silent incorrect behavior — not a thrown error.

**Affected file:**  
`src/app/core/models/booking.model.ts`

**Fix:**  
Remove `currency` from `BookingItem` interface. Currency is available at the `BookingResponse` level via `booking.currency`. Any display logic that needs item-level currency should read from the parent booking.

```typescript
// Before
export interface BookingItem {
  serviceName: string;
  quantity: number;
  unitPrice: number;
  currency: string; // REMOVE — not present in Java DTO
}

// After
export interface BookingItem {
  serviceName: string;
  quantity: number;
  unitPrice: number;
}
```

Audit all template and service usages of `item.currency` and replace with `booking.currency`.

**Effort:** 1–2 hours including audit  
**Risk of fix:** Low

---

### FIX-02: `RUST_LOG` default does not match crate name

**Status:** Operational blind spot in production  
**Layer:** Rust upload service (yoobu-media)

**Problem:**  
Default `RUST_LOG` is set to `image_service=debug,tower_http=debug`.  
The crate is named `yoobu-media`, not `image_service`.  
Result: in production, if `RUST_LOG` is not explicitly overridden in env, all application-level debug logs (upload handler, storage calls, processing pipeline) are silently dropped. Only `tower_http` request/response logs survive.

**Affected location:**  
`src/main.rs` — `tracing_subscriber` init  
`railway.toml` or Railway service env vars

**Fix — Option A (preferred):** Set correct default in `main.rs`:

```rust
// Before
std::env::var("RUST_LOG").unwrap_or_else(|_| "image_service=debug,tower_http=debug".to_string())

// After
std::env::var("RUST_LOG").unwrap_or_else(|_| "yoobu_media=debug,tower_http=debug".to_string())
```

Note: Rust module paths use underscores, not hyphens. Crate `yoobu-media` maps to module path `yoobu_media`.

**Fix — Option B:** Add explicit `RUST_LOG=yoobu_media=debug,tower_http=debug` to Railway service environment variables. Does not require a redeploy of code.

Option B is faster to ship. Option A is the correct permanent fix.

**Effort:** 15 minutes  
**Risk of fix:** None

---

## P1 — Operational Gaps

### FIX-03: Image deletion lifecycle not implemented

**Status:** Accumulating technical debt + future cost  
**Layer:** Java backend + Rust upload service

**Problem:**  
When a service image is replaced (re-upload) or a service is soft-deleted, the old object in Cloudflare R2 is never deleted. The Java backend already has `ImageServiceClient` with a `DELETE /object` call to the Rust service. The Rust service's `DELETE /object` endpoint is implemented and works. The call is simply never triggered.

**Fix:**

In `CatalogService.java` (or equivalent service layer), add deletion calls at two points:

1. **On service image replace:** Before persisting the new `image_url`, check if `existing.getImageUrl() != null`, extract the object key from the URL, and call `imageServiceClient.deleteObject(objectKey)`.

2. **On service soft-delete:** If `service.getImageUrl() != null` at the time of soft-delete, call `imageServiceClient.deleteObject(objectKey)` before or after setting `deleted_at`.

**Object key extraction:**  
The URL format is `{CDN_BASE_URL}/{objectKey}`. Strip the CDN base URL prefix to recover the key. Inject `CDN_BASE_URL` from config into the service layer.

**Error handling:**  
Delete failures should be logged but must not fail the primary operation (service update/delete). Wrap in try-catch, log at `warn` level.

```java
// Pseudocode
private void deleteImageIfPresent(CatalogService service) {
    if (service.getImageUrl() == null) return;
    try {
        String key = service.getImageUrl().replace(cdnBaseUrl + "/", "");
        imageServiceClient.deleteObject(key);
    } catch (Exception e) {
        log.warn("Failed to delete image object for service {}: {}", service.getId(), e.getMessage());
    }
}
```

**Effort:** 3–4 hours including logging and testing  
**Risk of fix:** Low. Delete is idempotent on R2 — nonexistent keys return 204.

---

### FIX-04: No caching on high-frequency read endpoints

**Status:** Not a bug today, will be at scale  
**Layer:** Java backend

**Problem:**  
`GET /t/{slug}/config` and `GET /t/{slug}/services` are called on every Mini App open. Both are DB reads. Both change rarely (config: on tenant update; services: on admin catalog edit). Under concurrent users on the same tenant, these generate redundant DB queries.

**Fix:**

Add Spring Cache (`@Cacheable`) on the service-layer methods that back these endpoints.

```java
// application.yml — add cache config
spring:
  cache:
    type: caffeine
  caffeine:
    spec: maximumSize=500,expireAfterWrite=60s
```

```java
@Cacheable(value = "tenant-config", key = "#slug")
public TenantConfigResponse getConfig(String slug) { ... }

@Cacheable(value = "tenant-services", key = "#slug")
public List<ServiceResponse> getServices(String slug) { ... }
```

Cache eviction must be triggered when:
- Tenant config is updated (superadmin PUT)
- Services are created/updated/deleted/toggled (admin API)

```java
@CacheEvict(value = "tenant-config", key = "#slug")
public TenantSummaryResponse updateTenant(String slug, ...) { ... }

@CacheEvict(value = "tenant-services", key = "#slug")
public ServiceResponse upsertService(String slug, ...) { ... }
```

**Dependencies:**  
Add `com.github.ben-manes.caffeine:caffeine` to `pom.xml`. No Redis required at current scale. Caffeine is in-process, zero ops overhead.

**Effort:** 4–6 hours including eviction coverage and verification  
**Risk of fix:** Medium. Cache eviction gaps (missed `@CacheEvict`) cause stale data. Requires thorough mapping of all write paths per cached entity.

---

## P2 — Structural Debt

### FIX-05: Remove deprecated `booking.service_id` column and JPA mapping

**Status:** Dead code in schema and entity  
**Layer:** Java backend + PostgreSQL

**Problem:**  
`booking.service_id` was the original item reference before `booking_item` table was introduced. It remains:
- As a column in the `booking` table (nullable FK → `service.id`)
- As a `@ManyToOne CatalogService service` field in the `Booking` JPA entity

It is never written by current booking creation logic. It is never read by any response mapping. It is a trap for future developers and creates a misleading FK relationship.

**Fix:**

Step 1 — Flyway migration:
```sql
-- V11__drop_booking_service_id.sql
ALTER TABLE booking DROP COLUMN service_id;
```

Step 2 — Remove from JPA entity:
```java
// Remove from Booking.java
@ManyToOne
@JoinColumn(name = "service_id")
private CatalogService service; // DELETE THIS
```

Step 3 — Verify no JPQL/HQL or repository methods reference `booking.service`.

**Effort:** 2 hours  
**Risk of fix:** Low. Column is nullable and not written. Verify no raw SQL queries reference it before dropping.

---

### FIX-06: Remove deprecated `booking_item.currency` column

**Status:** Superseded field, adds confusion  
**Layer:** Java backend + PostgreSQL

**Problem:**  
`booking_item.currency` was added in V6 as the currency source of truth, then superseded by `booking.currency` in V10 and made nullable. It is no longer the source of truth and should not be read for currency display.

**Fix:**

Step 1 — Verify the JPA `BookingItem` entity does not expose this field in any response mapping. If it does, remove it from the DTO mapping first.

Step 2 — Flyway migration:
```sql
-- V12__drop_booking_item_currency.sql
ALTER TABLE booking_item DROP COLUMN currency;
```

**Effort:** 1–2 hours  
**Risk of fix:** Low after confirming no active DTO mapping reads this column.

---

### FIX-07: `booking.slot_id` — document or drop

**Status:** Reserved column, never used  
**Layer:** PostgreSQL

**Problem:**  
`booking.slot_id` is a nullable BIGINT column added in V1. No FK. Never written. Never read. Documentation says "Reserved, unused."

**Decision required (not a pure tech decision):**  
If APPOINTMENT flow is on the roadmap within the next quarter, document the intended semantic and leave it. If APPOINTMENT is more than a quarter out, drop it to keep the schema clean.

**If dropping:**
```sql
-- V13__drop_booking_slot_id.sql
ALTER TABLE booking DROP COLUMN slot_id;
```

**Effort:** 30 minutes  
**Risk of fix:** None

---

## P3 — Reliability

### FIX-08: Object key collision in upload service

**Status:** Edge case, not yet observed  
**Layer:** Rust upload service

**Problem:**  
Object keys are `{tenantId}/{uploadPath}-{unix_epoch_ms}.webp`. Two uploads within the same millisecond for the same tenant and upload path produce the same key. The second PUT silently overwrites the first. In practice, admin uploads are not concurrent so this has not triggered, but it is a latent issue.

**Fix:**  
Append a random suffix to the key. A 6–8 character alphanumeric token from `rand::thread_rng()` is sufficient.

```rust
use rand::distributions::Alphanumeric;
use rand::{thread_rng, Rng};

let suffix: String = thread_rng()
    .sample_iter(&Alphanumeric)
    .take(8)
    .map(char::from)
    .collect();

let key = format!("{}/{}-{}-{}.webp", tenant_id, upload_path, epoch_ms, suffix);
```

Add `rand` to `Cargo.toml`.

**Effort:** 1 hour  
**Risk of fix:** Low. Key format change does not affect existing stored URLs.

---

## Summary Table

| ID | Layer | Priority | Issue | Effort |
|----|-------|----------|-------|--------|
| FIX-01 | Frontend | P0 | `BookingItem.currency` undefined at runtime | 1–2h |
| FIX-02 | Rust service | P0 | `RUST_LOG` wrong module name, no app logs in prod | 15min |
| FIX-03 | Java + Rust | P1 | Image deletion not triggered on replace/soft-delete | 3–4h |
| FIX-04 | Java | P1 | No caching on config/services endpoints | 4–6h |
| FIX-05 | Java + DB | P2 | Dead `booking.service_id` column and JPA mapping | 2h |
| FIX-06 | Java + DB | P2 | Superseded `booking_item.currency` column | 1–2h |
| FIX-07 | DB | P2 | Unused `booking.slot_id` — drop or document | 30min |
| FIX-08 | Rust service | P3 | Object key collision on same-millisecond uploads | 1h |

---

*End of TECH_SPEC_FIXES.md*