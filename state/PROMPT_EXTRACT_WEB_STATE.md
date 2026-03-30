# Task: Generate complete frontend state document from source code

Scan the entire `yoobu-web` codebase and produce a single Markdown document (`RND_WEB_STATE.md`) that captures the full current state of the Angular frontend. Do not reference any existing documentation — read only from source code, configs, and templates.

## What to extract

### 1. Module and routing structure
- All modules (eager and lazy-loaded), their declarations/imports/exports
- Full routing tree: every route with path, component, guards, resolvers, lazy-load boundaries
- Route parameters and query parameters consumed by each route

### 2. Components
For every component:
- Selector, standalone or module-declared
- Inputs and Outputs with types
- Template: what it renders, conditional blocks, loops, event bindings
- Injected services
- Lifecycle hooks used and what they do
- Telegram JS SDK calls made (if any): `MainButton`, `BackButton`, `showAlert`, `showConfirm`, `close`, `expand`, `ready`, `HapticFeedback`

### 3. Services
For every service:
- All public methods with signatures and return types
- HTTP calls: method, URL path, request body type, response type, headers attached
- State held (BehaviorSubject, signals, plain fields) — what shape, initial value, who reads/writes
- Error handling: what happens on HTTP error, what bubbles up to component

### 4. TypeScript interfaces and types
Every interface/type used for:
- API request bodies
- API response bodies
- Internal state shapes (cart, config, etc.)

List field by field with types and nullability. Flag any `any` usage.

### 5. HTTP layer
- HTTP interceptors: what each one does, in what order they run
- Base URL configuration: how API base path is resolved
- How `X-Telegram-Init-Data` (or dev fallback `X-Telegram-User-Id`) is attached
- Error interceptor behavior: retries, user feedback, logging

### 6. Telegram Mini App integration
- Where and how `window.Telegram.WebApp` is accessed
- `tg.ready()` / `tg.expand()` — where called
- `tg.initData` — where read, how passed to API
- `tg.MainButton` — where configured, label, onClick handler, show/hide logic
- `tg.BackButton` — where configured
- `tg.close()` — when called
- Theme params: are `tg.themeParams` or CSS variables from Telegram used?
- Viewport: `tg.viewportHeight`, `tg.viewportStableHeight` usage
- Any other SDK methods called

### 7. Theming and styling
- How `primaryColor` from `TenantConfig` is applied (CSS custom properties, inline styles, class bindings)
- Global styles vs component-scoped styles
- CSS framework or utility library used (if any)
- Responsive/mobile considerations (this runs in Telegram WebView)

### 8. State flow diagrams
For each user flow that exists in code:
- Step-by-step: which component → what user action → which service call → what state change → which navigation → what Telegram SDK call
- Flows to document: app init/tenant resolution, catalog browse, add to cart, checkout, view my bookings, view booking detail, cancel booking

### 9. Environment and build config
- `environment.ts` / `environment.prod.ts` contents
- `angular.json` build targets, output path, assets, custom webpack config (if any)
- `package.json` dependencies with versions (only non-trivial ones — skip Angular core)
- Proxy config for local dev (if exists)
- How the app is deployed to Railway

### 10. Guards and resolvers
For every route guard and resolver:
- What it checks / loads
- Redirect behavior on failure
- Dependencies injected

### 11. Error and loading states
- How loading states are shown (spinner, skeleton, Telegram native?)
- How API errors are displayed to the user
- How network-offline is handled (if at all)
- 404 / invalid slug behavior

### 12. Test coverage
For each spec file:
- What it covers
- How Telegram SDK is mocked
- HTTP mocking approach
- Any shared test utilities or fixtures

### 13. Not yet implemented
- Components, services, modules referenced in routing but not yet built
- Interfaces defined but not used
- Feature flags or commented-out code indicating planned work

## Format rules
- Use tables for structured data (routes, interfaces, component inventory)
- Use code blocks for TypeScript interface definitions and config files
- No opinions, no recommendations
- If something is ambiguous, state both interpretations
- Include actual file paths, class names, method names as references