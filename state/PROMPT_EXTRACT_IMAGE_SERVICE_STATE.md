# Task: Generate complete Rust service state document from source code

Scan the entire image upload service codebase and produce a single Markdown document (`RND_UPLOAD_STATE.md`) that captures the full current state. Do not reference any existing documentation — read only from source code, configs, migrations (if any), and Cargo.toml.

## What to extract

### 1. Endpoints
For every handler/route:
- HTTP method, full path (including path variables, query params)
- Request: content type (multipart, JSON, etc.), body/part structure, size limits
- Response: status codes, body type, headers (Content-Type, Content-Disposition, etc.)
- Auth mechanism: how the caller is authenticated (API key, shared secret, forwarded Telegram header, none)
- Error responses: all error cases with status codes and body format

### 2. Request/response types
Every struct used for request parsing or response serialization:
- All fields with Rust type, serde attributes (rename, default, skip, flatten)
- Validation: custom validators, size limits, format checks

### 3. Storage layer
- Where files are stored (local filesystem path, S3, object storage)
- File naming strategy (UUID, hash, original name preserved?)
- Directory structure
- Retention/cleanup policy (if any)
- Max file size, allowed MIME types/extensions
- Image processing: resizing, compression, format conversion, thumbnail generation

### 4. Database (if any)
- Schema (from migrations or ORM definitions)
- What metadata is persisted per upload (filename, size, content type, uploader, tenant, timestamps)
- Indexes, constraints

### 5. Configuration
- All config sources: env vars, config files, CLI args
- Every config field with type, default value, what it controls
- Profiles/environments if applicable

### 6. Integration with Yoobu backend
- How the Java backend calls this service (HTTP? shared storage? message queue?)
- How URLs are returned/stored (does Java backend store the URL? does frontend call upload service directly?)
- Auth between services: shared secret, internal network trust, forwarded headers
- Tenant isolation: how uploads are scoped per tenant

### 7. Dependencies
From Cargo.toml:
- All non-trivial dependencies with versions and what they're used for
- Feature flags enabled

### 8. Error handling
- Error type hierarchy (custom error enum, anyhow, thiserror)
- How errors map to HTTP responses
- Logging: what's logged at what level

### 9. Build and deployment
- Dockerfile (if exists)
- Railway config (if exists)
- How it runs: binary name, default port, startup sequence

### 10. Test coverage
For each test:
- What it covers
- Integration vs unit
- How file storage is mocked/sandboxed

### 11. Not yet implemented
- Endpoints or features referenced in code but stubbed/TODO
- Config keys read but never set
- Dead code paths

## Format rules
- Tables for structured data
- Code blocks for Rust struct definitions and config examples
- No opinions, no recommendations
- Actual file paths, struct names, function names as references