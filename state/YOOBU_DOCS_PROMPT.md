# Task: Produce unified product documentation from all service state extracts

You have three source-of-truth documents generated directly from code:
- `RND_API_STATE.md` — Java backend (REST API, admin panels, security, DB)
- `RND_WEB_STATE.md` — Angular frontend (Telegram Mini App client)
- `RND_UPLOAD_STATE.md` — Rust upload service (image storage/processing)

## Goal

Produce a single `YOOBU_DOCS.md` — the canonical documentation for the Yoobu platform. This replaces all prior documents. A new developer reads this one file and understands the entire system.

## Document structure

### 1. System overview
- What Yoobu is (one paragraph, from what the code does)
- Stack: all three services, database, hosting, auth
- Repository/service map: which repo does what, how they connect

### 2. Architecture
- Service topology: how the three services relate (who calls whom, shared DB or not, network boundaries)
- Multi-tenancy model: how tenant isolation works across all services
- Request lifecycle: Telegram Mini App → frontend → Java backend → Rust upload service (where applicable)
- Security model: all auth mechanisms across all services
- Data flow: which service/layer owns what

### 3. API contract reference
For every endpoint across all services, grouped by service and audience:
- Client API (Java backend `/t/{slug}/...`)
- Admin API (Java backend `/admin/{slug}/...`)
- Superadmin API (Java backend `/superadmin/...`)
- Upload API (Rust service)
- Mark which endpoints the frontend calls, which are backend-to-backend, which are admin-only

### 4. Data models
#### 4.1 API contracts (shared boundaries)
- Frontend ↔ Java backend: field-by-field with both types, flag mismatches
- Java backend ↔ Rust upload service: field-by-field, flag mismatches
- Frontend → Rust upload service (if direct calls exist): same treatment

#### 4.2 Backend-only models (Java)
- Entities, MVC forms, internal DTOs

#### 4.3 Upload service models (Rust)
- Request/response structs, stored metadata

#### 4.4 Schema vs Entity drift (Java)

### 5. Enums
Single table across all services. Flag value mismatches.

### 6. Database
- Full schema from Flyway (Java backend)
- Upload service schema (if separate DB or tables)
- Migration history
- Unused schema elements

### 7. Frontend architecture
- Same as before: module tree, components, state, theming
- How image upload integrates into the frontend (if it does)

### 8. Upload service architecture
- Storage strategy, image processing pipeline
- File lifecycle: upload → storage → URL generation → consumption
- Tenant scoping of uploads
- Cleanup/retention

### 9. Telegram Mini App integration
- Same as before, plus any upload-related SDK interactions (camera, file picker)

### 10. User flows (end-to-end)
All previous flows, plus:
- **Image upload**: who initiates → which service handles → where stored → how URL propagates to frontend
- Update any existing flows where image upload is now involved (service catalog images, tenant logos, etc.)

### 11. Configuration
- Java backend: application.yml, env vars, tenant_config keys
- Rust upload service: config, env vars
- Frontend: environment, proxy, build
- Deployment: all three services

### 12. Inter-service communication
Dedicated section:
- Every HTTP call between services: caller, callee, method, path, auth, timeout, retry
- Shared resources: database, file storage, config
- Failure modes: what happens when upload service is down? How does Java backend handle it?

### 13. Test coverage
Unified across all three codebases. Note gaps.

### 14. Contract mismatches
Full cross-check across all three service boundaries:
| Boundary | Endpoint/DTO | Field/Aspect | Service A state | Service B state | Verdict |

### 15. Not yet implemented
Single list across all repos.

## Rules
- Source code extracts are ground truth
- No opinions, no recommendations
- Every claim traces to a specific class/struct/method/file/migration
- Flag ambiguities explicitly
- Tables for structured data, code blocks for schemas, prose only for architecture
- Document must be copy-pasteable into a repo as-is