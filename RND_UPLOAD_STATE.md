# yoobu-media — Service State Document

Generated from source code. No existing documentation consulted.

---

## 1. Endpoints

### `GET /health`

| Field | Value |
|---|---|
| Method | GET |
| Path | `/health` |
| Auth | None |
| Request body | None |
| Response status | 200 |
| Response content-type | `application/json` |
| Response body | `{"status":"ok"}` |

No error cases.

---

### `POST /upload`

| Field | Value |
|---|---|
| Method | POST |
| Path | `/upload` |
| Auth | `Authorization: Bearer <token>` (constant-time compared against `INTERNAL_API_KEY`) |
| Request content-type | `multipart/form-data` |
| Multipart field | `file` (binary image data) |
| Required headers | `X-Tenant-Id`, `X-Upload-Path`, `Authorization` |
| Body size limit | `MAX_FILE_SIZE + 65536` bytes (axum `DefaultBodyLimit`) |

**Success response**

| Status | Content-Type | Body |
|---|---|---|
| 200 | `application/json` | `{"url":"<cdn_base_url>/<key>"}` |

**Error responses**

| Status | Condition | Body |
|---|---|---|
| 401 | Missing or wrong Bearer token | `{"error":"Unauthorized"}` |
| 400 | Missing `Authorization` header | `{"error":"Unauthorized"}` |
| 400 | Missing `X-Tenant-Id` header | `{"error":"Missing header: X-Tenant-Id"}` |
| 400 | Missing `X-Upload-Path` header | `{"error":"Missing header: X-Upload-Path"}` |
| 400 | File exceeds `MAX_FILE_SIZE` | `{"error":"File exceeds 2 MB limit"}` |
| 400 | Multipart field `file` absent | `{"error":"Multipart field 'file' not found"}` |
| 400 | Multipart parse error | `{"error":"<axum multipart error message>"}` |
| 400 | Unrecognized image format (magic bytes) | `{"error":"Unsupported format. Allowed: jpeg, png, webp"}` |
| 400 | File too small to read magic bytes (<12 bytes) | `{"error":"File is too small to be a valid image"}` |
| 400 | Image crate cannot decode bytes | `{"error":"Cannot decode image: <image crate message>"}` |
| 500 | WebP encoder init failure | `{"error":"<webp encoder error>"}` |
| 502 | S3/R2 PUT failed | `{"error":"Storage upload failed"}` |

**Object key format**

```
{X-Tenant-Id}/{X-Upload-Path}-{unix_epoch_ms}.webp

Examples:
  42/services/17-1719312000000.webp
  42/payment/qr-1719312000000.webp
```

**Returned URL format**

```
{CDN_BASE_URL}/{X-Tenant-Id}/{X-Upload-Path}-{unix_epoch_ms}.webp

Example:
  https://media.yoobu.app/42/services/17-1719312000000.webp
```

---

### `DELETE /object`

| Field | Value |
|---|---|
| Method | DELETE |
| Path | `/object` |
| Auth | `Authorization: Bearer <token>` |
| Required headers | `X-Object-Key`, `Authorization` |
| Request body | None |

**Success response**

| Status | Body |
|---|---|
| 204 | Empty |

**Error responses**

| Status | Condition | Body |
|---|---|---|
| 401 | Missing or wrong Bearer token | `{"error":"Unauthorized"}` |
| 400 | Missing `X-Object-Key` header | `{"error":"Missing header: X-Object-Key"}` |
| 502 | S3/R2 DELETE network/auth failure | `{"error":"Storage upload failed"}` |

Note: S3/R2 DELETE is idempotent — a key that does not exist returns 204, not an error.

---

## 2. Request/Response Types

No serde structs for request parsing. Inputs come from HTTP headers and multipart fields, extracted manually.

### Response struct (inline, not a named struct)

Upload success — constructed with `serde_json::json!` macro:

```rust
Json(json!({ "url": url }))        // POST /upload 200
Json(json!({ "status": "ok" }))    // GET /health 200
Json(json!({ "error": message }))  // all error responses
```

No named response structs. No serde attributes.

---

## 3. Storage Layer

| Property | Value |
|---|---|
| Backend | S3-compatible object storage (Cloudflare R2 in prod, MinIO in local dev) |
| SDK | `aws-sdk-s3` v1, `force_path_style(true)` |
| Local storage | None — no filesystem writes |
| File naming | `{tenantId}/{uploadPath}-{unix_epoch_ms}.webp` |
| Directory structure | Flat prefix hierarchy: tenant ID is the top-level prefix |
| Output format | Always WebP, regardless of input format |
| Retention/cleanup | None — no TTL, no cleanup policy in code |
| Max file size | `MAX_FILE_SIZE` env var (default 2 097 152 bytes = 2 MB); checked after multipart read |
| Allowed input formats | JPEG, PNG, WebP (validated by magic bytes) |
| Content-Type set on PUT | `image/webp` |
| Cache-Control set on PUT | `public, max-age=31536000, immutable` |

**Image processing pipeline** (`src/processing.rs`)

1. Magic-byte detection (`detect_format`) — validates format before any decoding
2. `image::load_from_memory` — decodes JPEG/PNG/WebP to `DynamicImage`
3. `resize_if_needed` — if `width > MAX_IMAGE_DIMENSION` or `height > MAX_IMAGE_DIMENSION`, scales down with `FilterType::Lanczos3` preserving aspect ratio; no upscaling
4. `webp::Encoder::from_image` → `encode(WEBP_QUALITY)` — lossy WebP, quality range 0.0–100.0

No thumbnail generation. No lossless encoding path in use.

---

## 4. Database

None. No database dependency, no migrations, no ORM.

---

## 5. Configuration

All configuration via environment variables. No config files, no CLI args.

| Env var | Type | Default | Required | Controls |
|---|---|---|---|---|
| `INTERNAL_API_KEY` | `String` | — | Yes | Bearer token for all authenticated endpoints |
| `R2_ENDPOINT` | `String` | — | Yes | S3-compatible endpoint URL |
| `R2_ACCESS_KEY` | `String` | — | Yes | S3 access key ID |
| `R2_SECRET_KEY` | `String` | — | Yes | S3 secret access key |
| `R2_BUCKET` | `String` | — | Yes | Bucket name |
| `R2_REGION` | `String` | `"auto"` | No | S3 region string |
| `CDN_BASE_URL` | `String` | — | Yes | Base URL prepended to object keys in upload response |
| `PORT` | `u16` | `3000` | No | TCP port to bind |
| `MAX_FILE_SIZE` | `usize` | `2097152` | No | Max accepted file bytes (post-read check, not stream limit) |
| `MAX_IMAGE_DIMENSION` | `u32` | `1200` | No | Max width or height after resize |
| `WEBP_QUALITY` | `f32` | `80` | No | Lossy WebP quality, 0.0–100.0 |
| `RUST_LOG` | `String` | `image_service=debug,tower_http=debug` | No | Log filter directive for `tracing-subscriber` |

`Config::from_env()` is called once at startup. Missing required vars cause `eprintln` + `process::exit(1)`.

Axum `DefaultBodyLimit` is set to `MAX_FILE_SIZE + 65536` (extra headroom for multipart framing headers).

---

## 6. Integration with Yoobu Backend

| Property | Value |
|---|---|
| Protocol | HTTP (internal Railway private network) |
| Caller | Java API |
| Service URL (Railway) | `http://yoobu-media.railway.internal:3000` |
| Public domain | None — private network only |
| Upload call | `POST /upload` with `Authorization: Bearer`, `X-Tenant-Id`, `X-Upload-Path`, multipart `file` field |
| Delete call | `DELETE /object` with `Authorization: Bearer`, `X-Object-Key` |
| URL storage | Java backend receives `{"url":"..."}` from upload response and is responsible for persisting it |
| Shared secret | `INTERNAL_API_KEY` — Railway project-level variable shared between this service and the Java API |
| Tenant isolation | `X-Tenant-Id` header value becomes the top-level prefix in the object key; no cross-tenant enforcement in this service (trust the caller) |
| Frontend interaction | Frontend does not call this service directly |

---

## 7. Dependencies

| Crate | Version | Used for |
|---|---|---|
| `axum` | 0.7.9 | HTTP framework, routing, extractors, multipart |
| `tokio` | 1.50.0 | Async runtime (`features = ["full"]`) |
| `tower-http` | 0.5 | `TraceLayer` middleware for request/response logging |
| `tracing` | 0.1.44 | Structured logging macros (`info!`, `error!`) |
| `tracing-subscriber` | 0.3.23 | Log subscriber with `EnvFilter` and `fmt` output |
| `serde` | 1.0 | Derive macros for serialization (`features = ["derive"]`) |
| `serde_json` | 1.0 | JSON serialization, `json!` macro |
| `aws-sdk-s3` | 1.127.0 | S3-compatible PUT/DELETE via R2 or MinIO |
| `aws-credential-types` | 1 | `Credentials::new` for hardcoded key/secret (`features = ["hardcoded-credentials"]`) |
| `image` | 0.25 | Decode JPEG/PNG/WebP, resize (`features = ["jpeg","png","webp"]`, `default-features = false`) |
| `webp` | 0.3.1 | Encode `DynamicImage` to lossy WebP; bundles libwebp C sources |
| `bytes` | 1 | `Bytes` buffer for S3 `ByteStream` |
| `subtle` | 2.6.1 | Constant-time byte comparison for Bearer token verification |
| `thiserror` | 2.0.18 | Derive `std::error::Error` + `Display` for `AppError` |

Dev dependencies: `tokio` (same version, `features = ["full"]`) — used for async test runtime in integration tests.

---

## 8. Error Handling

### Error type

`src/error.rs` — `AppError` enum with `#[derive(thiserror::Error)]`:

```rust
pub enum AppError {
    Unauthorized,
    BadRequest(String),
    StorageError,
    ProcessingError(String),
}
```

### HTTP mapping

| Variant | HTTP Status |
|---|---|
| `Unauthorized` | 401 |
| `BadRequest(msg)` | 400 |
| `StorageError` | 502 |
| `ProcessingError(msg)` | 500 |

`AppError` implements axum `IntoResponse`. All errors serialize to `{"error":"<message>"}` JSON.

### Logging

| Location | Level | Fields |
|---|---|---|
| `storage.rs` — R2 upload failure | `error` | `key`, `error` (debug fmt of SDK error) |
| `storage.rs` — R2 delete failure | `error` | `key`, `error` |
| `handler/upload.rs` — before PUT | `info` | `key`, `size_bytes` |
| `handler/delete.rs` — before DELETE | `info` | `key` |
| `tower-http TraceLayer` | `debug` (default) | Method, path, status, latency (all requests) |

Log level controlled by `RUST_LOG`. Default: `image_service=debug,tower_http=debug`.

---

## 9. Build and Deployment

### Dockerfile

Two-stage build:

| Stage | Base image | Purpose |
|---|---|---|
| `builder` | `rust:slim-bookworm` | Compile release binary; includes gcc/make for libwebp C sources |
| runtime | `debian:bookworm-slim` | Run binary; only `ca-certificates` added for TLS to R2 |

Layer-caching optimization: manifests (`Cargo.toml`, `Cargo.lock`) and a stub `src/main.rs` are compiled first. Real sources are copied and built second so dependency compilation is cached when only `src/` changes.

Binary path in image: `/usr/local/bin/yoobu-media`

Exposed port: `3000`

Entrypoint: `CMD ["yoobu-media"]`

### railway.toml

```toml
[build]
builder = "DOCKERFILE"
dockerfilePath = "Dockerfile"
watchPatterns = ["Dockerfile", "Cargo.toml", "Cargo.lock", "src/**", "railway.toml"]

[deploy]
healthcheckPath = "/health"
healthcheckTimeout = 300
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 10
```

### docker-compose.yml (local dev)

| Service | Image | Ports | Purpose |
|---|---|---|---|
| `minio` | `minio/minio:latest` | 9000 (S3 API), 9001 (console) | Local S3-compatible storage |
| `minio-init` | `minio/mc:latest` | — | Creates `yoobu-media` bucket, sets anonymous public policy |
| `yoobu-media` | Built from `Dockerfile` | 3000 | Service under development |

`CDN_BASE_URL` in local compose: `http://localhost:9000/yoobu-media`

`INTERNAL_API_KEY` in local compose: `dev-internal-key-change-in-prod`

### Binary

| Property | Value |
|---|---|
| Binary name | `yoobu-media` |
| Default port | `3000` (env `PORT`) |
| Bind address | `0.0.0.0:{PORT}` |
| Startup sequence | Parse env → build `StorageClient` → build router → bind TCP → serve |
| Startup failure modes | Missing required env vars → `exit(1)`; TCP bind failure → `exit(1)` |

---

## 10. Test Coverage

All tests are in the same file as the code under test (Rust inline `#[cfg(test)]` modules). No separate test files.

### `src/auth.rs` — `mod tests`

| Test name | Type | What it covers |
|---|---|---|
| `valid_token_passes` | Unit | `verify_token` returns `true` for identical strings |
| `wrong_token_fails` | Unit | `verify_token` returns `false` for different same-length strings |
| `empty_token_fails` | Unit | `verify_token` returns `false` for empty provided token |
| `different_length_fails` | Unit | `verify_token` returns `false` when lengths differ |
| `prefix_match_fails` | Unit | `verify_token` returns `false` when provided is a prefix of expected |

No mock or network required.

### `src/processing.rs` — `mod tests`

| Test name | Type | What it covers |
|---|---|---|
| `detects_jpeg` | Unit | `detect_format` identifies JPEG magic bytes |
| `detects_png` | Unit | `detect_format` identifies PNG magic bytes |
| `detects_webp` | Unit | `detect_format` identifies WEBP magic bytes |
| `rejects_unknown_format` | Unit | `detect_format` returns `Err` for unrecognized bytes |
| `rejects_too_small` | Unit | `detect_format` returns `Err` for <12 byte input |
| `resize_not_triggered_when_fits` | Unit | `resize_if_needed` does not resize 10×10 image within 1200px limit |
| `resize_scales_down_wide_image` | Unit | `resize_if_needed` scales 2400×600 down to ≤1200 on each axis |
| `process_image_produces_webp_output` | Integration | Full pipeline: real PNG bytes → `process_image` → output starts with `RIFF` + `WEBP` magic |

No HTTP requests. No storage mock. `process_image_produces_webp_output` encodes a real in-memory PNG and runs the full decode → resize → encode path.

No tests for: handlers, auth extraction from HTTP, storage client, config parsing, error-to-HTTP mapping.

---

## 11. Not Yet Implemented

| Item | Location | Notes |
|---|---|---|
| No rate limiting | — | No middleware or per-IP throttling |
| No request ID / correlation ID | — | `TraceLayer` adds spans but no `X-Request-Id` propagation |
| No content-type validation of multipart field | `handler/upload.rs` | `field.content_type()` is not checked; format validated by magic bytes only |
| No object key collision handling | `handler/upload.rs` | Two uploads in the same millisecond for the same tenant+path produce the same key; second write silently overwrites the first |
| No upload size streaming enforcement | `handler/upload.rs` | Full bytes read into memory before size check; `DefaultBodyLimit` is the only pre-read guard |
| No lossless WebP path | `processing.rs` | `encode_lossless()` exists in the `webp` crate but is not called |
| `RUST_LOG` default targets `image_service` module | `main.rs` | Binary and crate are named `yoobu-media`; filter `image_service=debug` targets a module name that does not match the package name — effective log level falls back to tower_http filter only unless overridden via env |
| No handler tests | — | Auth extraction, multipart parsing, header validation, and storage interaction are not tested |
| No config validation beyond type parsing | `config.rs` | `INTERNAL_API_KEY` minimum length not enforced in code (documented in `.env.example` comment only) |
