# Tech Spec: Image Service (Rust) + Java Integration

## Architecture

```
                    ┌──────────────────────────────────────────┐
                    │          Railway Project                  │
                    │                                          │
  Browser/Admin ──► │  Java API (public)                       │
                    │    │                                     │
                    │    │ POST http://image-service            │
                    │    │      .railway.internal:3000/upload   │
                    │    │ Headers:                             │
                    │    │   Authorization: Bearer {SECRET}     │
                    │    │   X-Tenant-Id: 42                   │
                    │    │   X-Upload-Path: services/17         │
                    │    │ Body: multipart file                 │
                    │    ▼                                     │
                    │  Rust Image Service (private network)    │
                    │    │                                     │
                    │    │ validate → resize/crop → webp       │
                    │    │ PUT to R2                            │
                    │    │                                     │
                    │    ▼                                     │
                    │  Returns:                                │
                    │  { "url": "https://media.yoobu.app/..." }│
                    │                                          │
                    └──────────────────────────────────────────┘
                                      │
                                      ▼
                              Cloudflare R2 + CDN
                          https://media.yoobu.app
```

Java API: public, handles auth, domain logic, DB writes.
Rust Image Service: private network only, handles files, R2, image processing.

---

## Part 1: Rust Image Service

### Overview

Standalone HTTP service. Accepts multipart upload, validates, processes, uploads to R2, returns CDN URL. No database access. No domain knowledge.

### Stack

| Crate | Purpose |
|---|---|
| `axum` | HTTP framework |
| `tokio` | Async runtime |
| `aws-sdk-s3` | R2 upload (S3-compatible) |
| `image` | Resize, crop, format conversion |
| `tower-http` | Middleware (request size limits, tracing) |
| `tracing` + `tracing-subscriber` | Structured logging |

### Endpoint

```
POST /upload
Content-Type: multipart/form-data
```

#### Request

Headers:

| Header | Required | Description |
|---|---|---|
| `Authorization` | yes | `Bearer {INTERNAL_API_KEY}` — shared secret with Java |
| `X-Tenant-Id` | yes | Tenant ID (integer as string). Used for bucket path prefix. |
| `X-Upload-Path` | yes | One of: `services/{serviceId}` or `payment/qr`. Determines subfolder in bucket. |

Body: single file part named `file`.

#### Response

Success (200):

```json
{
  "url": "https://media.yoobu.app/42/services/17-1719312000000.webp"
}
```

Validation error (400):

```json
{
  "error": "File exceeds 2 MB limit"
}
```

```json
{
  "error": "Unsupported format. Allowed: jpeg, png, webp"
}
```

Auth error (401):

```json
{
  "error": "Unauthorized"
}
```

Missing header (400):

```json
{
  "error": "Missing header: X-Tenant-Id"
}
```

R2 upload failure (502):

```json
{
  "error": "Storage upload failed"
}
```

#### Health check

```
GET /health
Response: 200 { "status": "ok" }
```

### Processing Pipeline

```
1. Validate auth header (constant-time compare)
2. Parse headers: X-Tenant-Id, X-Upload-Path
3. Read multipart file into memory
4. Validate:
   a. File size ≤ 2 MB (raw bytes)
   b. Magic bytes → actual content type (don't trust Content-Type header)
   c. Allowed: JPEG, PNG, WebP
5. Process image:
   a. Decode
   b. Resize: max 1200px on longest side, preserve aspect ratio
   c. Encode to WebP (quality 80)
6. Build object key: {tenantId}/{uploadPath}-{timestamp}.webp
   Examples:
     42/services/17-1719312000000.webp
     42/payment/qr-1719312000000.webp
7. Upload to R2:
   - Content-Type: image/webp
   - Cache-Control: public, max-age=31536000, immutable
8. If previous image exists for same logical path:
   NOT handled here. Java decides whether to delete old image.
   See "Deletion endpoint" below.
9. Return { "url": "{CDN_BASE_URL}/{key}" }
```

### Deletion Endpoint

```
DELETE /object
Headers:
  Authorization: Bearer {INTERNAL_API_KEY}
  X-Object-Key: 42/services/17-1719312000000.webp

Response 204: deleted
Response 404: not found (or already deleted — idempotent)
Response 401: unauthorized
```

Java calls this after successful upload when replacing an existing image (old URL → extract key → DELETE).

### Configuration (env vars)

| Variable | Description | Example |
|---|---|---|
| `PORT` | Listen port | `3000` |
| `INTERNAL_API_KEY` | Shared secret with Java | `sk-...` (generate random 32+ chars) |
| `R2_ENDPOINT` | R2 S3 endpoint | `https://{account_id}.r2.cloudflarestorage.com` |
| `R2_ACCESS_KEY` | R2 access key ID | |
| `R2_SECRET_KEY` | R2 secret access key | |
| `R2_BUCKET` | Bucket name | `yoobu-media` |
| `R2_REGION` | Region (R2 uses `auto`) | `auto` |
| `CDN_BASE_URL` | Public CDN URL prefix | `https://media.yoobu.app` |
| `MAX_FILE_SIZE` | Max upload bytes (optional) | `2097152` (2 MB, default) |
| `MAX_IMAGE_DIMENSION` | Max px longest side (optional) | `1200` (default) |
| `WEBP_QUALITY` | WebP encode quality (optional) | `80` (default) |

### Project Structure

```
image-service/
├── Cargo.toml
├── Dockerfile
├── src/
│   ├── main.rs              # tokio::main, axum router, server start
│   ├── config.rs             # Env var parsing into Config struct
│   ├── auth.rs               # Auth middleware (Bearer token check)
│   ├── handler/
│   │   ├── mod.rs
│   │   ├── upload.rs         # POST /upload handler
│   │   ├── delete.rs         # DELETE /object handler
│   │   └── health.rs         # GET /health handler
│   ├── processing.rs         # Image validation, resize, webp encode
│   ├── storage.rs            # R2/S3 client wrapper (upload, delete)
│   └── error.rs              # AppError enum → axum IntoResponse
```

### Dockerfile

```dockerfile
FROM rust:1.83-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
# Cache dependencies
RUN mkdir src && echo "fn main() {}" > src/main.rs && cargo build --release && rm -rf src
COPY src/ src/
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/image-service /usr/local/bin/
CMD ["image-service"]
```

### Key Implementation Notes for AI Agent

**Auth middleware (auth.rs):**
- Extract `Authorization` header, strip `Bearer ` prefix
- Constant-time compare against `INTERNAL_API_KEY` from config
- Return 401 with JSON body on mismatch
- Apply to all routes except `/health`

**Upload handler (upload.rs):**
- Use `axum::extract::Multipart` to read file
- Read entire file into `Vec<u8>` (max 2 MB, not streaming — fine for this size)
- Check magic bytes: JPEG (`FF D8 FF`), PNG (`89 50 4E 47`), WebP (`52 49 46 46...57 45 42 50`)
- Call `processing::process_image(bytes)` → returns WebP `Vec<u8>`
- Build key from headers + timestamp (`std::time::SystemTime` → millis)
- Call `storage::upload(key, webp_bytes)` → returns ()
- Return JSON with full CDN URL

**Processing (processing.rs):**
- Use `image::load_from_memory(&bytes)` to decode
- `image::DynamicImage::resize` with max dimension, `FilterType::Lanczos3`
- Encode to WebP: use `image::codecs::webp::WebPEncoder` or `webp` crate
- Note: `image` crate's WebP encoder is lossy. If quality control needed, use `webp` crate directly.

**Storage (storage.rs):**
- Build `aws_sdk_s3::Client` from config at startup (shared via `axum::extract::State`)
- `PutObject`: bucket, key, body (`ByteStream::from(bytes)`), content_type, cache_control
- `DeleteObject`: bucket, key
- R2 endpoint override: `aws_sdk_s3::config::Builder::endpoint_url(r2_endpoint).force_path_style(true)`
- Region: `aws_sdk_s3::config::Region::new("auto")`

**Error handling (error.rs):**
- Enum: `Unauthorized`, `BadRequest(String)`, `StorageError(String)`, `ProcessingError(String)`
- Implement `IntoResponse` for each variant → appropriate status code + JSON body

---

## Part 2: Java Backend Changes

### Remove

If `MediaStorageService` already exists with direct R2 logic — remove R2-specific code. If not yet implemented — skip.

### New: `ImageServiceClient`

```java
@Component
public class ImageServiceClient {

    // Injected from config
    private final String imageServiceUrl;   // IMAGE_SERVICE_URL env var
    private final String internalApiKey;    // INTERNAL_API_KEY env var
    private final RestClient restClient;

    /**
     * Uploads image to image service.
     * Returns CDN URL on success.
     * Throws ResponseStatusException on failure.
     */
    public String upload(Long tenantId, String uploadPath, MultipartFile file) {
        // POST {imageServiceUrl}/upload
        // Headers:
        //   Authorization: Bearer {internalApiKey}
        //   X-Tenant-Id: {tenantId}
        //   X-Upload-Path: {uploadPath}
        // Body: multipart with file
        // Parse response JSON → extract "url" field
        // On 400 from image service → throw 400 to client with error message
        // On other error → throw 502
    }

    /**
     * Deletes object from storage.
     * Extracts key from CDN URL, calls DELETE on image service.
     * Failures logged, not propagated.
     */
    public void deleteByUrl(String cdnUrl) {
        // Extract key: remove CDN_BASE_URL prefix from cdnUrl
        // DELETE {imageServiceUrl}/object
        // Header: Authorization: Bearer {internalApiKey}
        // Header: X-Object-Key: {extracted key}
        // Log on failure, don't throw
    }
}
```

### Configuration

```yaml
app:
  image-service:
    url: ${IMAGE_SERVICE_URL:http://image-service.railway.internal:3000}
    api-key: ${INTERNAL_API_KEY}
```

New properties class:

```java
@ConfigurationProperties(prefix = "app.image-service")
public record ImageServiceProperties(String url, String apiKey) {}
```

### Changes to `AdminCatalogService`

```java
// Existing createService / updateService — no changes

// New method:
public ServiceResponse uploadServiceImage(Long serviceId, MultipartFile file) {
    CatalogService service = getAdminService(serviceId); // tenant-scoped
    Long tenantId = TenantContext.getRequiredTenantId();

    String oldImageUrl = service.getImageUrl();

    String newImageUrl = imageServiceClient.upload(
        tenantId,
        "services/" + serviceId,
        file
    );

    service.setImageUrl(newImageUrl);
    service.setUpdatedAt(OffsetDateTime.now(ZoneOffset.UTC));
    catalogServiceRepository.save(service);

    // Delete old image after successful save
    if (oldImageUrl != null) {
        imageServiceClient.deleteByUrl(oldImageUrl);
    }

    auditLogService.logUpdate("service", service.getId(), ...);

    return toServiceResponse(service);
}
```

### Changes to payment QR upload

Same pattern. `TenantManagementService` or new method in relevant service:

```java
public String uploadPaymentQr(Long tenantId, MultipartFile file) {
    TenantSettings settings = tenantSettingsService.getSettings(tenantId);
    String oldQrUrl = settings.payment().qrUrl(); // may be null

    String newQrUrl = imageServiceClient.upload(tenantId, "payment/qr", file);

    upsertConfig(tenantId, "payment_qr_url", newQrUrl);

    if (oldQrUrl != null && oldQrUrl.startsWith(cdnBaseUrl)) {
        // Only delete if it's our CDN URL (not external URL from before)
        imageServiceClient.deleteByUrl(oldQrUrl);
    }

    return newQrUrl;
}
```

### Controller Endpoints (unchanged from original image spec)

```
POST /admin/{slug}/services/{serviceId}/image
Content-Type: multipart/form-data
Auth: HTTP Basic tenant
Response: 200 + ServiceResponse

POST /admin/{slug}/payment-qr
Content-Type: multipart/form-data
Auth: HTTP Basic tenant
Response: 200 + { "paymentQrUrl": "https://media.yoobu.app/..." }
```

### Thymeleaf Panel

Same as original image spec — file inputs in service form and tenant detail. No changes from extraction.

---

## Shared Environment Variables

| Variable | Java | Rust | How to share in Railway |
|---|---|---|---|
| `INTERNAL_API_KEY` | yes | yes | Shared variable in Railway project, referenced by both services |

| Variable | Java only | Rust only |
|---|---|---|
| `IMAGE_SERVICE_URL` | yes (points to Rust) | — |
| `R2_ENDPOINT` | — | yes |
| `R2_ACCESS_KEY` | — | yes |
| `R2_SECRET_KEY` | — | yes |
| `R2_BUCKET` | — | yes |
| `CDN_BASE_URL` | — | yes |

Java never sees R2 credentials. Rust never sees database credentials.

---

## Railway Setup

1. In existing Railway project, add new service (Docker, from `image-service/` repo or subfolder)
2. Set service to **private networking only** — no public domain
3. Service name: `image-service` → internal URL becomes `http://image-service.railway.internal:3000`
4. Add shared variable `INTERNAL_API_KEY` at project level
5. Add R2-specific vars to image-service only
6. Add `IMAGE_SERVICE_URL=http://image-service.railway.internal:3000` to Java service

---

## Testing

### Rust Image Service

| Test | What |
|---|---|
| Unit: `processing.rs` | JPEG/PNG/WebP decode → resize → WebP encode. Bad file → error. Oversized → error. Magic bytes mismatch → error. |
| Unit: `auth.rs` | Valid token → pass. Invalid → 401. Missing → 401. |
| Integration: `upload handler` | Mock S3 client (or use `aws_smithy_mocks`). Full flow: multipart → process → mock upload → response URL. |
| Integration: `delete handler` | Mock S3 client. Delete existing key → 204. |

### Java Backend

| Test | What |
|---|---|
| Unit: `ImageServiceClient` | Mock HTTP (MockRestServiceServer or WireMock). Successful upload → returns URL. 400 from image service → throws 400. Timeout → throws 502. |
| Unit: `AdminCatalogService.uploadServiceImage` | Mock `ImageServiceClient`. Verify: old image URL delete called when replacing. |
| Integration: `AdminPanelIT` / new IT | Multipart upload through admin endpoint. Mock `ImageServiceClient` at bean level. Verify service entity updated with URL. |

No test hits real R2 or real Rust service in CI. Both sides mock the boundary.

---

## Bucket Structure (unchanged)

```
yoobu-media/
  {tenantId}/
    services/
      {serviceId}-{timestamp}.webp
    payment/
      qr-{timestamp}.webp
```

All files converted to WebP by Rust service regardless of input format. Timestamp is Unix millis at upload time.

---

## Order of Implementation

1. **Rust service** — full standalone, testable with curl
2. **Deploy to Railway** — private network, verify `/health`
3. **Java `ImageServiceClient`** — HTTP client + config
4. **Java endpoints** — `POST .../image`, `POST .../payment-qr`
5. **Java panel** — file inputs in Thymeleaf forms
6. **Frontend** — `imageUrl` field in `ServiceItem`, render in menu

Rust service can be developed and deployed independently. Java changes come after.
