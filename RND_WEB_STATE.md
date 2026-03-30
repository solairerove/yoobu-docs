# RND_WEB_STATE.md — Yoobu Web Frontend State Document

Generated: 2026-03-30
Source: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web`
Framework: Angular 19 standalone components

---

## Table of Contents

1. [Module and Routing Structure](#1-module-and-routing-structure)
2. [Components](#2-components)
3. [Services](#3-services)
4. [TypeScript Interfaces and Types](#4-typescript-interfaces-and-types)
5. [HTTP Layer](#5-http-layer)
6. [Telegram Mini App Integration](#6-telegram-mini-app-integration)
7. [Theming and Styling](#7-theming-and-styling)
8. [State Flow Diagrams](#8-state-flow-diagrams)
9. [Environment and Build Config](#9-environment-and-build-config)
10. [Guards and Resolvers](#10-guards-and-resolvers)
11. [Error and Loading States](#11-error-and-loading-states)
12. [Test Coverage](#12-test-coverage)
13. [Not Yet Implemented](#13-not-yet-implemented)

---

## 1. Module and Routing Structure

### Architecture

No NgModules. The entire app uses Angular 19 **standalone components**. Providers are registered in `src/app/app.config.ts`. The router is initialized with `provideRouter()`.

### Bootstrap

`src/main.ts`:
```typescript
bootstrapApplication(AppComponent, appConfig)
```

`src/app/app.config.ts`:
```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(appRoutes),
    provideHttpClient(withInterceptors([telegramInitDataInterceptor]))
  ]
};
```

### Route Table

File: `src/app/app.routes.ts`

| Path | Component / Target | Load Strategy | Notes |
|---|---|---|---|
| `` (empty) | — | — | `redirectTo: 't/demo'`, `pathMatch: 'full'` |
| `t/:slug` | `TenantShellComponent` | Lazy (`loadComponent`) | Entry point for all tenant traffic |
| `**` | — | — | `redirectTo: 't/demo'` |

**Route parameters consumed by `t/:slug`:**

| Param | Type | Consumer |
|---|---|---|
| `slug` | `string` | `TenantShellComponent` reads via `ActivatedRoute.paramMap` |

No query parameters are declared in routing; none are consumed at the route level.

**Child routes:** None defined in the route config. `TenantShellComponent` performs its own dynamic component outlet via `NgComponentOutlet` — it does not use child routes.

**Lazy-load boundaries (inside TenantShellComponent):**

| Condition | Loaded Component | File |
|---|---|---|
| `config.type === 'FOOD_ORDER'` | `FoodOrderHomeComponent` | `src/app/features/food-order/food-order-home.component.ts` |
| Any other `type` | `UnsupportedFlowComponent` | `src/app/features/unsupported-flow/unsupported-flow.component.ts` |

---

## 2. Components

### Component Inventory

| Class | Selector | File | Standalone | Change Detection |
|---|---|---|---|---|
| `AppComponent` | `app-root` | `src/app/app.component.ts` | Yes | Default |
| `TenantShellComponent` | `app-tenant-shell` | `src/app/tenant/tenant-shell.component.ts` | Yes | Default |
| `FoodOrderHomeComponent` | `app-food-order-home` | `src/app/features/food-order/food-order-home.component.ts` | Yes | Default |
| `FoodOrderMenuComponent` | `app-food-order-menu` | `src/app/features/food-order/food-order-menu.component.ts` | Yes | OnPush |
| `FoodOrderCartBarComponent` | `app-food-order-cart-bar` | `src/app/features/food-order/food-order-cart-bar.component.ts` | Yes | OnPush |
| `FoodOrderCheckoutComponent` | `app-food-order-checkout` | `src/app/features/food-order/food-order-checkout.component.ts` | Yes | OnPush |
| `FoodOrderBookingsComponent` | `app-food-order-bookings` | `src/app/features/food-order/food-order-bookings.component.ts` | Yes | OnPush |
| `FoodOrderSuccessCardComponent` | `app-food-order-success-card` | `src/app/features/food-order/food-order-success-card.component.ts` | Yes | OnPush |
| `UnsupportedFlowComponent` | `app-unsupported-flow` | `src/app/features/unsupported-flow/unsupported-flow.component.ts` | Yes | Default |

---

### `AppComponent`

File: `src/app/app.component.ts`

- **Imports:** `[RouterOutlet]`
- **Template:** `<router-outlet />`
- **Injected services:** None
- **Inputs/Outputs:** None
- **Lifecycle hooks:** None
- **Telegram SDK calls:** None

---

### `TenantShellComponent`

File: `src/app/tenant/tenant-shell.component.ts`

**Injected services:**

| Token | Purpose |
|---|---|
| `DOCUMENT` | DOM access for CSS custom property application |
| `ActivatedRoute` | Reads `:slug` param |
| `TenantApiService` | Fetches tenant config |
| `TelegramService` | Initializes Telegram Web App, hides main button |

**Inputs/Outputs:** None (receives route data implicitly).

**Reactive view model (`vm$`):**

```
route.paramMap
  → map slug
  → distinctUntilChanged
  → tap(init telegram, hide main button)
  → switchMap(slug → getConfig(slug))
    → tap(applyTheme)
    → switchMap(resolveFeatureComponent(type))
    → map({ config, component, error: null })
    → catchError → of({ config: null, component: null, error: '...' })
  → shareReplay({ bufferSize: 1, refCount: true })
```

**Private methods:**

| Method | Signature | Behavior |
|---|---|---|
| `resolveFeatureComponent` | `(type: TenantType): Promise<Type<unknown>>` | Returns lazy-loaded component class |
| `applyTheme` | `(config: TenantConfig): void` | Sets `--yoobu-primary` on `document.documentElement`; defaults to `#ff6b35` if `primaryColor` is null |

**Template logic:**

- If `vm$.error`: renders error message string in a status card.
- If `vm$` is null/loading: renders "Loading…" skeleton.
- If resolved: renders `NgComponentOutlet` with `component` class and passes `config` as input.

**Telegram SDK calls:**
- `telegram.init()` — called on each slug change
- `telegram.setMainButton(null)` — hides main button on slug change

**Lifecycle hooks:** None (uses `async` pipe and RxJS).

---

### `FoodOrderHomeComponent`

File: `src/app/features/food-order/food-order-home.component.ts`

**Providers:** `[FoodOrderFlowFacade]` — scoped to this component tree.

**Inputs:**

| Name | Type | Required |
|---|---|---|
| `config` | `TenantConfig` | Yes (signal input) |

**Outputs:** None.

**Injected services:** `FoodOrderFlowFacade`

**Signals exposed to template** (all proxied from facade/store):

| Signal | Type | Source |
|---|---|---|
| `store` | `FoodOrderStore` | `facade.store` |
| `showLocalCheckoutButtons` | `boolean` | `facade.showLocalCheckoutButtons` |
| `checkoutForm` | `FormGroup` | `facade.checkoutForm` |
| `selectedBookingId` | `Signal<number \| null>` | `facade.selectedBookingId` |
| `selectedBooking` | `Signal<BookingResponse \| null>` | `facade.selectedBooking` |
| `checkoutOpen` | `Signal<boolean>` | `facade.checkoutOpen` |
| `submitting` | `Signal<boolean>` | `facade.submitting` |
| `submitError` | `Signal<string \| null>` | `facade.submitError` |
| `repeatOrderBanner` | `Signal<string \| null>` | `facade.repeatOrderBanner` |
| `submittedBooking` | `Signal<BookingResponse \| null>` | `facade.submittedBooking` |
| `confirmingPaymentBookingId` | `Signal<number \| null>` | `facade.confirmingPaymentBookingId` |
| `paymentError` | `Signal<string \| null>` | `facade.paymentError` |
| `cancellingBookingId` | `Signal<number \| null>` | `facade.cancellingBookingId` |
| `cancelError` | `Signal<string \| null>` | `facade.cancelError` |
| `vm` | `Signal<FoodOrderVm>` | `facade.vm` |
| `bookingsVm` | `Signal<MyBookingsVm>` | `facade.bookingsVm` |
| `activeView` | `Signal<'menu' \| 'orders'>` | `facade.activeView` |
| `earliestDeliveryDate` | `Signal<string>` | `facade.earliestDeliveryDate` |
| `currencyCode` | `Signal<string>` | computed from `config().currency` via `normalizeCurrencyCode` |
| `selectedQuantities` | `Signal<Record<number, number>>` | computed from `store.quantities()` |

**Constructor:** `effect(() => facade.setConfig(this.config()))` — re-runs when `config` input changes.

**Lifecycle hooks:** None (uses signals and effects).

**Telegram SDK calls:** Delegated to facade.

**Template:** Renders all child components conditionally based on signals. View switch (`menu` / `orders`) via button click → `facade.setActiveView()`.

---

### `FoodOrderMenuComponent`

File: `src/app/features/food-order/food-order-menu.component.ts`

**Inputs:**

| Name | Type | Required |
|---|---|---|
| `services` | `ServiceItem[]` | Yes |
| `selectedCount` | `number` | Yes |
| `currencyCode` | `string` | Yes |
| `quantities` | `Record<number, number>` | Yes |
| `defaultUnit` | `string` | No (default: `'item'`) |

**Outputs:**

| Name | Payload |
|---|---|
| `increaseRequested` | `number` (serviceId) |
| `decreaseRequested` | `number` (serviceId) |

**Protected methods:**

| Method | Signature | Purpose |
|---|---|---|
| `quantityFor` | `(serviceId: number): number` | Looks up quantity from `quantities` input |
| `maxed` | `(serviceId: number): boolean` | Returns true if quantity === 9 |
| `serviceInitial` | `(name: string): string` | Returns first character of name for placeholder |
| `trackByServiceId` | `(_index: number, service: ServiceItem): number` | NgFor track function |

**Template:** 2-column responsive grid of service cards. Each card: optional `<img>` (or initial placeholder), name, description, unit label, formatted price, quantity stepper (`–` / count / `+`). Selected cards get accent border. `+` button is disabled when `maxed()`.

**Injected services:** None.

---

### `FoodOrderCartBarComponent`

File: `src/app/features/food-order/food-order-cart-bar.component.ts`

**Inputs:**

| Name | Type | Required |
|---|---|---|
| `checkoutOpen` | `boolean` | Yes |
| `selectedCount` | `number` | Yes |
| `selectedTotal` | `number` | Yes |
| `currencyCode` | `string` | Yes |

**Outputs:**

| Name | Payload |
|---|---|
| `openRequested` | `void` |

**Template:** Fixed-position bottom bar. Label shows `"X items · {total}"` or `"Your order"` if checkout is open. Action label: `"Checkout ›"` or `"Review ›"`. Gradient orange background, white text. Visible only in local/dev mode (renders inside `FoodOrderHomeComponent` conditional).

**Injected services:** None.

---

### `FoodOrderCheckoutComponent`

File: `src/app/features/food-order/food-order-checkout.component.ts`

**Inputs:**

| Name | Type | Required | Default |
|---|---|---|---|
| `open` | `boolean` | Yes | — |
| `localMode` | `boolean` | Yes | — |
| `submitting` | `boolean` | Yes | — |
| `submitError` | `string \| null` | Yes | — |
| `repeatOrderBanner` | `string \| null` | No | `null` |
| `form` | `FormGroup` | Yes | — |
| `customerNameHint` | `string \| null` | No | `null` |
| `customerPhoneHint` | `string \| null` | No | `null` |
| `deliveryAddressHint` | `string \| null` | No | `null` |
| `customerNoteHint` | `string \| null` | No | `null` |
| `currencyCode` | `string` | No | `'VND'` |
| `selectedItems` | `CheckoutSelection[]` | Yes | — |
| `selectedCount` | `number` | Yes | — |
| `selectedTotal` | `number` | Yes | — |
| `earliestDeliveryDate` | `string \| null` | No | `null` |

`CheckoutSelection` (local interface):
```typescript
interface CheckoutSelection {
  quantity: number;
  service: ServiceItem;
}
```

**Outputs:**

| Name | Payload |
|---|---|
| `closeRequested` | `void` |
| `repeatOrderBannerDismissed` | `void` |
| `submitRequested` | `void` |

**Computed:**

| Signal | Type | Logic |
|---|---|---|
| `isCutoffActive` | `boolean` | Returns `true` if `earliestDeliveryDate > today` |

**Template:**
- In `localMode`: card layout (no overlay).
- In Telegram mode: bottom-sheet overlay with drag handle, scrim backdrop.
- Form fields: `customerName`, `customerPhone`, `deliveryAddress`, `deliveryDate` (required, min = `earliestDeliveryDate`), `note`.
- Cutoff hint line shown if `isCutoffActive`.
- Repeat order banner with dismiss button.
- Submit error or placeholder hint text below form.
- Submit button visible only in `localMode`.
- Right column: order review card listing items with qty × unit price and total.

**Injected services:** None.

---

### `FoodOrderBookingsComponent`

File: `src/app/features/food-order/food-order-bookings.component.ts`

**Inputs:**

| Name | Type | Required | Default |
|---|---|---|---|
| `bookings` | `BookingResponse[]` | Yes | — |
| `loading` | `boolean` | Yes | — |
| `error` | `string \| null` | Yes | — |
| `paymentQrUrl` | `string \| null` | No | `null` |
| `selectedBookingId` | `number \| null` | Yes | — |
| `selectedBooking` | `BookingResponse \| null` | Yes | — |
| `confirmingPaymentBookingId` | `number \| null` | Yes | — |
| `paymentError` | `string \| null` | Yes | — |
| `cancellingBookingId` | `number \| null` | Yes | — |
| `cancelError` | `string \| null` | Yes | — |
| `currencyCode` | `string` | No | `'VND'` |

**Outputs:**

| Name | Payload |
|---|---|
| `refreshRequested` | `void` |
| `bookingSelected` | `number` (bookingId) |
| `repeatRequested` | `number` (bookingId) |
| `paymentConfirmRequested` | `number` (bookingId) |
| `cancelRequested` | `number` (bookingId) |

**Computed signals:**

| Signal | Logic |
|---|---|
| `currentBookings` | `bookings` filtered to status ∈ `[NEW, PAYMENT_PENDING, CONFIRMED, DELIVERING]` |
| `previousBookings` | `bookings` filtered to status ∈ `[DONE, CANCELLED]` |
| `allBookings` | `[...currentBookings, ...previousBookings]` |
| `selectedDisplayBooking` | `selectedBooking` if set, else first of `currentBookings`, else first of `previousBookings` |

**Protected methods:**

| Method | Signature | Purpose |
|---|---|---|
| `canRepeat` | `(b: BookingResponse): boolean` | Based on status |
| `canCancel` | `(b: BookingResponse): boolean` | Based on status |
| `canConfirmPayment` | `(b: BookingResponse): boolean` | Based on status |
| `isStatus` | `(status: string, expected: string): boolean` | Case-insensitive comparison |
| `bookingStatusLabel` | `(status: BookingStatus): string` | Human-readable label |
| `bookingTimeline` | `(status: BookingStatus): Array<{label, description, state}>` | Timeline steps with state |
| `deliveryTrackingUrl` | `(b: BookingResponse): string \| null` | Returns `trackingUrl` if `DELIVERING` |
| `bookingCurrency` | `(b: BookingResponse): string` | Uses booking's `currency` or falls back to input |
| `displayAddress` | `(address: string \| null): string` | Formats or returns placeholder |
| `bookingItemsPreview` | `(b: BookingResponse): string` | Short text summary of items |
| `receiptItems` | `(b: BookingResponse): BookingResponse['items']` | Returns `b.items` |

**Template:**
- Header: "My orders" + refresh button.
- Loading / error / empty states.
- Hint: "Tap a card to see order details".
- Booking list: sorted (current first, previous after). Each card: `#id`, status badge, delivery date, address, item preview, total.
- Booking detail panel (for `selectedDisplayBooking`):
   - `#id` + status badge.
   - Timeline: Order placed → Payment → Confirmed → Delivering → Delivered. States: `pending` / `current` / `complete` / `cancelled`.
   - Payment QR card (if status is `NEW` or `PAYMENT_PENDING` and QR URL available).
   - Action buttons: Track delivery (link), Repeat, I paid (`confirmPayment`), Cancel.
   - Receipt card: delivery date, contact, address, note.
   - Item list with unit prices and total.

**Injected services:** None.

---

### `FoodOrderSuccessCardComponent`

File: `src/app/features/food-order/food-order-success-card.component.ts`

**Inputs:**

| Name | Type | Required | Default |
|---|---|---|---|
| `booking` | `BookingResponse` | Yes | — |
| `fallbackCurrency` | `string` | Yes | — |
| `paymentQrUrl` | `string \| null` | No | `null` |
| `confirmingPaymentBookingId` | `number \| null` | No | `null` |
| `paymentError` | `string \| null` | No | `null` |

**Outputs:**

| Name | Payload |
|---|---|
| `paymentConfirmRequested` | `number` (bookingId) |
| `newOrderRequested` | `void` |
| `ordersRequested` | `void` |

**Protected methods:**

| Method | Signature | Purpose |
|---|---|---|
| `bookingCurrency` | `(): string` | Booking currency or fallback |
| `statusLabel` | `(status: BookingStatus): string` | Human-readable status |
| `canConfirmPayment` | `(): boolean` | True if status is `NEW` |
| `effectivePaymentQrUrl` | `(): string \| null` | `booking.paymentQrUrl ?? paymentQrUrl` |
| `shouldShowPaymentQr` | `(): boolean` | Status is `NEW` or `PAYMENT_PENDING` and QR URL present |
| `shouldShowTrackingLink` | `(): boolean` | Status is `DELIVERING` and `trackingUrl` present |
| `deliveryTrackingUrl` | `(): string \| null` | Returns `booking.trackingUrl` |

**Template:**
- Eyebrow: "Order #id sent".
- Message: customer name + delivery date + status.
- Meta: total price, item count, created timestamp.
- Payment QR image + "I paid" button (if `shouldShowPaymentQr`).
- "Track delivery" link (if `shouldShowTrackingLink`).
- Payment error text (if `paymentError`).
- Action buttons: "New order", "My orders".

**Injected services:** None.

---

### `UnsupportedFlowComponent`

File: `src/app/features/unsupported-flow/unsupported-flow.component.ts`

**Inputs:**

| Name | Type | Required |
|---|---|---|
| `config` | `TenantConfig` | Yes |

**Outputs:** None.

**Template:** Static. Displays "Unavailable" label, `config.type` value, and message "This section is not available yet."

**Injected services:** None.

---

## 3. Services

### `TenantApiService`

File: `src/app/core/services/tenant-api.service.ts`
`@Injectable({ providedIn: 'root' })`

**State held:** None (stateless service).

**Constants:**

| Constant | Value |
|---|---|
| `REQUEST_TIMEOUT_MS` | `10_000` |
| `RETRY_COUNT` | `2` |

**HTTP methods:**

| Method | HTTP | Path | Request Body | Response Type |
|---|---|---|---|---|
| `getConfig(slug)` | GET | `/api/t/:slug/config` | — | `Observable<TenantConfig>` |
| `getServices(slug)` | GET | `/api/t/:slug/services` | — | `Observable<ServiceItem[]>` |
| `createBooking(slug, request)` | POST | `/api/t/:slug/bookings` | `CreateBookingRequest` | `Observable<BookingResponse>` |
| `getMyBookings(slug)` | GET | `/api/t/:slug/bookings/my` | — | `Observable<BookingResponse[]>` |
| `getBooking(slug, bookingId)` | GET | `/api/t/:slug/bookings/:bookingId` | — | `Observable<BookingResponse>` |
| `confirmBookingPayment(slug, bookingId)` | POST | `/api/t/:slug/bookings/:bookingId/confirm-payment` | — | `Observable<BookingResponse>` |
| `cancelBooking(slug, bookingId)` | POST | `/api/t/:slug/bookings/:bookingId/cancel` | — | `Observable<BookingResponse>` |

**`withRetry<T>` private method:**
- Applies `timeout(10_000)`.
- On error, retries up to 2 times.
- Backoff delays: attempt 0 → 0ms, attempt 1 → 2000ms, attempt 2 → 4000ms.

**Error handling:** Errors from `withRetry` bubble up as `Observable` errors to callers. No local catching.

---

### `TelegramService`

File: `src/app/core/services/telegram.service.ts`
`@Injectable({ providedIn: 'root' })`

**Internal interface (not exported):**
```typescript
interface TelegramWebApp {
  initData?: string;
  version?: string;
  ready(): void;
  expand(): void;
  showAlert?(message: string, callback?: () => void): void;
  showConfirm?(message: string, callback?: (confirmed: boolean) => void): void;
  MainButton?: {
    setText(text: string): void;
    show(): void;
    hide(): void;
    enable?(): void;
    disable?(): void;
    onClick?(callback: () => void): void;
    offClick?(callback: () => void): void;
  };
  HapticFeedback?: {
    impactOccurred(style: 'light' | 'medium' | 'heavy' | 'rigid' | 'soft'): void;
    notificationOccurred(type: 'error' | 'success' | 'warning'): void;
    selectionChanged(): void;
  };
}
```

**State held:**

| Field | Type | Initial | Description |
|---|---|---|---|
| `webAppSignal` | `Signal<TelegramWebApp \| null>` | `null` | Resolved WebApp object |
| `mainButtonHandler` | `Signal<(() => void) \| null>` | `null` | Current click handler |

**Constants:**

| Constant | Value |
|---|---|
| `POPUP_API_MIN_VERSION` | `'6.2'` |
| `INIT_DATA_SESSION_KEY` | `'tg_init_data_cache'` |

**Public methods:**

| Method | Signature | Behavior |
|---|---|---|
| `init` | `(): void` | Sets `webAppSignal` from `window.Telegram.WebApp`. Side-effect via `effect()`: calls `ready()` and `expand()` when WebApp resolves. |
| `getInitData` | `(): string \| null` | 1) Tries `webApp.initData` (trimmed); 2) falls back to `getInitDataFromLaunchParams()` (parses `tgWebAppData` from URL query or hash); 3) falls back to `readCachedInitData()` (localStorage). Caches result. |
| `getDevTelegramUserId` | `(): string \| null` | Returns `'101'` only if `hostname` is `'localhost'` or `'127.0.0.1'`. Otherwise `null`. |
| `isLocalhost` | `(): boolean` | True if hostname is localhost/127.0.0.1. |
| `setMainButton` | `(text: string \| null, enabled = true): void` | `null` → `hide()`. String → `setText()`, then `disable()/enable()` per flag, then `show()`. |
| `onMainButtonClick` | `(handler: (() => void) \| null): void` | Updates `mainButtonHandler` signal. Effect auto-wires `onClick` / `offClick`. |
| `hapticLight` | `(): void` | Calls `HapticFeedback.impactOccurred('light')`. |
| `hapticSelection` | `(): void` | Calls `HapticFeedback.selectionChanged()`. |
| `alert` | `(message: string): Promise<void>` | Uses `showAlert` if WebApp supports popup API (v ≥ 6.2); else `window.alert()`. |
| `confirm` | `(message: string): Promise<boolean>` | Uses `showConfirm` if supported; else `window.confirm()`. |

**Private methods:** `resolveWebApp`, `supportsPopupApi`, `isVersionAtLeast`, `parseVersionParts`, `cacheInitData`, `readCachedInitData`, `getInitDataFromLaunchParams`, `readInitDataFromParams`.

---

### `FoodOrderStore`

File: `src/app/features/food-order/food-order.store.ts`
`@Injectable({ providedIn: 'root' })`

**State held (private signals):**

| Signal | Type | Initial |
|---|---|---|
| `slugSignal` | `Signal<string \| null>` | `null` |
| `servicesSignal` | `Signal<ServiceItem[]>` | `[]` |
| `quantitiesSignal` | `Signal<Record<number, number>>` | `{}` |

**Public computed signals:**

| Signal | Type | Description |
|---|---|---|
| `services` | `ServiceItem[]` | Current service catalog |
| `quantities` | `Record<number, number>` | Current cart quantities |
| `selectedItems` | `Array<{service: ServiceItem, quantity: number}>` | Services with quantity > 0 |
| `selectedCount` | `number` | Sum of all quantities |
| `selectedTotal` | `number` | Sum of `service.price * quantity` |

**Constants:**

| Constant | Value |
|---|---|
| `MAX_ITEM_QUANTITY` | `9` |

**Public methods:**

| Method | Signature | Behavior |
|---|---|---|
| `setTenant` | `(slug: string): void` | If slug changed: clears services and cart |
| `setServices` | `(services: ServiceItem[]): void` | Updates catalog; removes quantities for services no longer in catalog |
| `quantityFor` | `(serviceId: number): number` | Returns current quantity or `0` |
| `increase` | `(serviceId: number): void` | Increments quantity; capped at `MAX_ITEM_QUANTITY` |
| `decrease` | `(serviceId: number): void` | Decrements; removes entry when quantity reaches `0` |
| `setQuantities` | `(quantities: Record<number, number>): void` | Bulk-sets; filters to valid service IDs; caps at `9` |
| `clearCart` | `(): void` | Resets `quantitiesSignal` to `{}` |

---

### `FoodOrderFlowFacade`

File: `src/app/features/food-order/food-order-flow.facade.ts`
`@Injectable()` — not `providedIn: 'root'`; provided in `FoodOrderHomeComponent.providers`.

**Injected dependencies:**

| Token | Purpose |
|---|---|
| `TenantApiService` | All HTTP calls |
| `FormBuilder` | Builds `checkoutForm` |
| `TelegramService` | Main button, haptics, alerts, confirms |
| `DestroyRef` | Cleans up subscriptions |
| `FoodOrderStore` | Cart and service state |

**Form (`checkoutForm`):**

| Control | Type | Validators |
|---|---|---|
| `customerName` | `string` | `required` |
| `customerPhone` | `string` | `required`, pattern `/^[+]?[\d\s\-\(\)\.]{6,20}$/` |
| `deliveryAddress` | `string` | None |
| `deliveryDate` | `string` | `required` |
| `note` | `string` | None |

Default value for `deliveryDate`: computed by `defaultDeliveryDate()` (today or earliest allowed).

**Signals (public):**

| Signal | Type | Initial |
|---|---|---|
| `bookingsReloadKey` | `number` | `0` |
| `selectedBookingId` | `number \| null` | `null` |
| `selectedBooking` | `BookingResponse \| null` | `null` |
| `activeView` | `'menu' \| 'orders'` | `'menu'` |
| `checkoutOpen` | `boolean` | `false` |
| `submitting` | `boolean` | `false` |
| `submitError` | `string \| null` | `null` |
| `repeatOrderBanner` | `string \| null` | `null` |
| `submittedBooking` | `BookingResponse \| null` | `null` |
| `confirmingPaymentBookingId` | `number \| null` | `null` |
| `paymentError` | `string \| null` | `null` |
| `cancellingBookingId` | `number \| null` | `null` |
| `cancelError` | `string \| null` | `null` |

**Computed signals (public):**

| Signal | Type | Logic |
|---|---|---|
| `earliestDeliveryDate` | `string` | From `config.earliestDeliveryDate` or cutoff logic |
| `vm` | `FoodOrderVm` | Wraps `services`, `loading`, `error` |
| `bookingsVm` | `MyBookingsVm` | Wraps `bookings`, `loading`, `error` |
| `isFirstOrder` | `boolean` | True if no bookings yet |

**Public methods:**

| Method | Signature | Behavior |
|---|---|---|
| `setConfig` | `(config: TenantConfig): void` | Updates config; resets state if slug changed; triggers service reload |
| `increase` | `(serviceId: number): void` | `store.increase()`; haptic; clears error banners |
| `decrease` | `(serviceId: number): void` | `store.decrease()`; closes checkout if cart empty |
| `openCheckout` | `(): void` | Sets `checkoutOpen(true)` |
| `closeCheckout` | `(): void` | Sets `checkoutOpen(false)` |
| `startNewOrder` | `(): void` | Clears `submittedBooking`, clears cart, returns to menu |
| `refreshBookings` | `(): void` | Increments `bookingsReloadKey` |
| `selectBooking` | `async (bookingId: number): Promise<void>` | Sets `selectedBookingId`; loads booking detail; handles race conditions with request versioning |
| `cancelBooking` | `async (bookingId: number): Promise<void>` | `telegram.confirm()` → `api.cancelBooking()` → updates local state + refreshes bookings |
| `confirmPayment` | `async (bookingId: number): Promise<void>` | `api.confirmBookingPayment()` → handles 409 (already paid) → `telegram.alert()` on error |
| `repeatBooking` | `async (bookingId: number): Promise<void>` | Loads booking → matches items to current services by normalized name → prefills form and cart → shows alert if some items unavailable |
| `dismissRepeatOrderBanner` | `(): void` | Sets `repeatOrderBanner(null)` |
| `submitOrder` | `async (): Promise<void>` | Validates form (required fields, date ≥ earliest) → `telegram.confirm()` → `api.createBooking()` → sets `submittedBooking` → refreshes bookings → `telegram.alert()` on error |
| `setActiveView` | `(view: 'menu' \| 'orders'): void` | Updates `activeView`; refreshes bookings |

**Constructor-level effects:**
1. Main button state management: tracks `checkoutOpen`, `store.selectedCount`, `submitting` → calls `telegram.setMainButton()` and `telegram.onMainButtonClick()` accordingly.
2. `bookingsVm` kept hot (accessed in effect to ensure signal graph stays live).

---

## 4. TypeScript Interfaces and Types

### `src/app/core/models/booking.model.ts`

```typescript
export interface CreateBookingItem {
  serviceId: number;       // Service ID from catalog
  quantity: number;        // Units ordered
}

export interface CreateBookingRequest {
  customerName: string;
  customerPhone: string;
  deliveryAddress: string;
  deliveryDate: string;        // ISO date string
  note: string | null;
  items: CreateBookingItem[];
}

export interface BookingItem {
  serviceName: string;
  quantity: number;
  unitPrice: number;
  currency: string;
}

export interface BookingResponse {
  id: number;
  type: 'ORDER' | 'APPOINTMENT' | 'REQUEST' | string;  // string allows unknown future types
  status: BookingStatus;
  customerName: string;
  customerPhone: string;
  deliveryAddress: string | null;
  totalPrice: number;
  currency: string;
  deliveryDate: string;        // ISO date string
  note: string | null;
  trackingUrl: string | null;
  paymentQrUrl: string | null;
  items: BookingItem[];
  createdAt: string;           // ISO datetime string
}

export type BookingStatus =
  | 'NEW'
  | 'PAYMENT_PENDING'
  | 'CONFIRMED'
  | 'DELIVERING'
  | 'DONE'
  | 'CANCELLED';
```

No `any` usage.

---

### `src/app/core/models/service.model.ts`

```typescript
export interface ServiceItem {
  id: number;
  name: string;
  description: string | null;
  imageUrl: string | null;
  price: number;
  unit: string | null;
  durationMinutes: number | null;
  sortOrder: number;
  status: 'ACTIVE' | 'INACTIVE' | 'DELETED';
}
```

No `any` usage.

---

### `src/app/core/models/tenant-config.model.ts`

```typescript
export type TenantType = 'FOOD_ORDER' | 'APPOINTMENT' | 'CATALOG_REQUEST';

export interface TenantConfig {
  slug: string;
  name: string;
  type: TenantType;
  currency?: string | null;
  paymentQrUrl?: string | null;
  primaryColor: string | null;
  logoUrl: string | null;
  welcomeMessage: string | null;
  checkoutNameHint?: string | null;
  checkoutPhoneHint?: string | null;
  checkoutDeliveryHint?: string | null;
  checkoutNoteHint?: string | null;
  cutoffHour?: number | null;
  cutoffMinute?: number | null;
  earliestDeliveryDate?: string | null;
}
```

Fields marked `?` are optional (may be absent from response). `primaryColor`, `logoUrl`, `welcomeMessage` are non-optional but nullable.

No `any` usage.

---

### Internal interfaces in facade (`food-order-flow.facade.ts`)

```typescript
interface FoodOrderVm {
  services: ServiceItem[];
  loading: boolean;
  error: string | null;
}

interface MyBookingsVm {
  bookings: BookingResponse[];
  loading: boolean;
  error: string | null;
}
```

### Internal interface in checkout component

```typescript
interface CheckoutSelection {
  quantity: number;
  service: ServiceItem;
}
```

### Internal interface in telegram service

```typescript
interface TelegramWebApp {
  initData?: string;
  version?: string;
  ready(): void;
  expand(): void;
  showAlert?(message: string, callback?: () => void): void;
  showConfirm?(message: string, callback?: (confirmed: boolean) => void): void;
  MainButton?: { /* see §3 TelegramService */ };
  HapticFeedback?: { /* see §3 TelegramService */ };
}
```

---

## 5. HTTP Layer

### Interceptor

File: `src/app/core/interceptors/telegram-init-data.interceptor.ts`

**Function name:** `telegramInitDataInterceptor` (`HttpInterceptorFn`)

**Registered in:** `app.config.ts` via `provideHttpClient(withInterceptors([telegramInitDataInterceptor]))`

**Constants:**

| Constant | Value |
|---|---|
| `TELEGRAM_INIT_DATA_WAIT_TIMEOUT_MS` | `1500` |
| `TELEGRAM_INIT_DATA_POLL_INTERVAL_MS` | `50` |

**Decision logic:**

```
1. getDevTelegramUserId() → non-null (localhost)?
   → If yes: clone request with X-Telegram-User-Id: '101'
   → Continue to next handler

2. Request URL contains '/bookings'?
   → Poll for init data every 50ms, up to 1500ms total
   → If init data found: clone request with X-Telegram-Init-Data header
   → If timeout with no data: throw Error('Telegram session unavailable')

3. Otherwise (non-bookings, no dev user):
   → Try getInitData() immediately
   → If found: add X-Telegram-Init-Data header
   → If not found: pass request through unmodified
```

**Headers set:**

| Header | Value | When |
|---|---|---|
| `X-Telegram-Init-Data` | `telegram.getInitData()` | Real Telegram session |
| `X-Telegram-User-Id` | `'101'` | Localhost development |

### Base URL

All requests use the path `/api/t/:slug/...`. In production, Caddy proxies `/api/*` to `$BACKEND_URL` (strips `/api` prefix). In development, Angular's dev-server proxy rewrites `/api/t` → `/t` against `http://localhost:8080`.

### Error handling in HTTP layer

- The interceptor throws `Error('Telegram session unavailable')` for bookings requests with no init data.
- `TenantApiService.withRetry()` handles timeouts and transient errors via retry with backoff.
- No global error interceptor beyond `telegramInitDataInterceptor`.

---

## 6. Telegram Mini App Integration

### SDK Loading

`src/index.html`:
```html
<script src="https://telegram.org/js/telegram-web-app.js" defer></script>
```

The `defer` attribute means the SDK is loaded after HTML parsing but before `DOMContentLoaded`. Angular bootstraps after this point.

### `window.Telegram.WebApp` Access

Accessed only in `TelegramService` (`src/app/core/services/telegram.service.ts`) via `resolveWebApp()` private method, which reads `(window as any).Telegram?.WebApp`. Result is stored in `webAppSignal`.

### `tg.ready()` / `tg.expand()`

Called inside an `effect()` in `TelegramService.init()`. Triggered when `TenantShellComponent` calls `telegram.init()` on each slug navigation.

### `tg.initData`

- Read in `TelegramService.getInitData()`.
- Fallback chain: `webApp.initData` → URL params (`tgWebAppData`) → localStorage cache.
- Passed to API via `X-Telegram-Init-Data` header by `telegramInitDataInterceptor`.

### `tg.MainButton`

| Operation | Where called | Logic |
|---|---|---|
| `setText(text)` | `TelegramService.setMainButton()` | Sets label |
| `show()` / `hide()` | `TelegramService.setMainButton()` | Based on `text` being null or not |
| `enable()` / `disable()` | `TelegramService.setMainButton()` | Based on `enabled` param |
| `onClick(handler)` | `TelegramService` effect (wires handler when `mainButtonHandler` signal changes) | Used for order submit |
| `offClick(handler)` | `TelegramService` effect (unwires previous handler) | Cleanup |

**Label and state logic (in `FoodOrderFlowFacade` effect):**
- Cart is empty + checkout closed: `setMainButton(null)` (hidden)
- Cart has items + checkout closed: `setMainButton("Checkout (N items)")`, enabled
- Checkout open + not submitting: `setMainButton("Place Order")`, enabled
- Checkout open + submitting: `setMainButton("Placing order…")`, disabled

### `tg.BackButton`

Not used anywhere in the codebase.

### `tg.close()`

Not called anywhere in the codebase.

### `tg.showAlert` / `tg.showConfirm`

Wrapped in `TelegramService.alert()` and `TelegramService.confirm()`. Used in:

| Call | Where | Trigger |
|---|---|---|
| `confirm("Confirm your order?")` | `FoodOrderFlowFacade.submitOrder()` | Before submitting |
| `confirm("Cancel this order?")` | `FoodOrderFlowFacade.cancelBooking()` | Before cancelling |
| `alert(errorMessage)` | `FoodOrderFlowFacade.confirmPayment()` | On payment confirm error |
| `alert(errorMessage)` | `FoodOrderFlowFacade.submitOrder()` | On submit error |
| `alert(message)` | `FoodOrderFlowFacade.repeatBooking()` | If some items unavailable |

Fallback: `window.alert()` / `window.confirm()` when SDK not available or version < 6.2.

### `tg.HapticFeedback`

| Call | Where | Trigger |
|---|---|---|
| `hapticLight()` → `impactOccurred('light')` | `FoodOrderFlowFacade.increase()` | Adding item to cart |
| `hapticSelection()` → `selectionChanged()` | `FoodOrderFlowFacade.decrease()` | Removing item from cart |

### Theme params

`tg.themeParams` is **not used**. Telegram CSS variables are **not used**. Theming is entirely tenant-driven via `TenantConfig.primaryColor`.

### Viewport

`tg.viewportHeight` and `tg.viewportStableHeight` are **not used** in the codebase.

---

## 7. Theming and Styling

### Primary Color Application

`TenantShellComponent.applyTheme()`:
```typescript
document.documentElement.style.setProperty(
  '--yoobu-primary',
  config.primaryColor ?? '#ff6b35'
);
```

This sets a CSS custom property on `:root`. All components consume `--yoobu-primary` via `var(--yoobu-primary)` in their scoped styles.

### CSS Custom Properties

File: `src/styles.css`

**Color tokens:**

| Variable | Default value | Description |
|---|---|---|
| `--yoobu-primary` | `#ff6b35` | Primary accent (overridden per tenant) |
| `--yoobu-primary-strong` | `#e6521f` | Darker primary |
| `--yoobu-primary-soft` | `rgba(255,107,53,0.12)` | Translucent primary |
| `--yoobu-surface` | — | Page background |
| `--yoobu-surface-strong` | — | Stronger background |
| `--yoobu-surface-card` | — | Card background |
| `--yoobu-surface-card-soft` | — | Soft card surface |
| `--yoobu-surface-card-strong` | — | Strong card surface |
| `--yoobu-surface-tint` | — | Tinted surface |
| `--yoobu-surface-muted` | — | Muted surface |
| `--yoobu-ink` | `#24160f` | Primary text (dark brown) |
| `--yoobu-muted` | `#6b5d56` | Secondary text |
| `--yoobu-border` | — | Default border |
| `--yoobu-border-soft` | — | Soft border |
| `--yoobu-border-accent-soft` | — | Soft accent border |
| `--yoobu-border-accent` | — | Accent border |

**Shadow tokens:** `--yoobu-shadow`, `--yoobu-shadow-soft`, `--yoobu-shadow-xs`, `--yoobu-shadow-sm`, `--yoobu-shadow-accent`, `--yoobu-shadow-modal`.

**Status badge colors (global CSS):**

| Status | Color |
|---|---|
| `PAYMENT_PENDING` | `#0d47a1` (blue) |
| `CONFIRMED` | `#9a6800` (gold) |
| `DELIVERING` | `#5e35b1` (purple) |
| `DONE` | `#2e7d32` (green) |
| `CANCELLED` | `#a52a2a` (brown) |

### CSS Framework / Utility Library

None. All styles are hand-written CSS. No Tailwind, Bootstrap, Material, or other framework.

### Global Styles

**Font:** `"SF Pro Display", "Avenir Next", "Segoe UI", sans-serif`

**Background:** Radial and linear gradient.

**Global resets:** `box-sizing: border-box`, button/input inherit font.

**Global utility classes:**

| Class | Purpose |
|---|---|
| `.eyebrow` | Uppercase label with letter-spacing, primary color |
| `.ghost-button` | White button with border |
| `.head-action` | Small secondary button |
| `.ui-copy` | Muted text, 1.45 line-height |
| `.ui-status-card` | Styled container card |
| `.ui-status-card.error` | Error styling variant |

### Responsive / Mobile

- Max-width container: `720px`.
- Grid breakpoints at `640px` (2-column → 1-column) and `430px` (fine-tuning).
- Fixed bottom cart bar uses `safe-area-inset-bottom` for notch/home-bar padding.
- Checkout in Telegram mode renders as a bottom sheet (absolute positioned overlay).
- `user-scalable=no` in viewport meta.

---

## 8. State Flow Diagrams

### Flow 1: App Init / Tenant Resolution

```
User opens Telegram Mini App (URL: /t/:slug)
  → AppComponent renders <router-outlet>
  → Router matches t/:slug → lazy-loads TenantShellComponent
  → TenantShellComponent.vm$ starts
  → telegram.init() called
    → TelegramService sets webAppSignal from window.Telegram.WebApp
    → effect fires: tg.ready(), tg.expand()
  → telegram.setMainButton(null) — hide main button
  → tenantApi.getConfig(slug) HTTP GET /api/t/:slug/config
    → interceptor: checks for init data (non-bookings path, passes through)
    → on success: applyTheme(config) → sets --yoobu-primary
    → resolveFeatureComponent(config.type)
      → FOOD_ORDER → lazy-loads FoodOrderHomeComponent
      → else → lazy-loads UnsupportedFlowComponent
    → NgComponentOutlet renders chosen component with config input
  → on error: shows error message in status card
```

### Flow 2: Catalog Browse

```
FoodOrderHomeComponent created with config input
  → effect fires: facade.setConfig(config)
    → store.setTenant(slug)
    → loads services: tenantApi.getServices(slug) GET /api/t/:slug/services
      → interceptor: adds X-Telegram-Init-Data or X-Telegram-User-Id
      → on success: store.setServices(services)
      → facade.vm signal updates → loading=false, services=[...]
  → template renders FoodOrderMenuComponent with services, quantities
  → User scrolls through service cards
```

### Flow 3: Add to Cart

```
User taps "+" on a service card
  → FoodOrderMenuComponent emits increaseRequested(serviceId)
  → FoodOrderHomeComponent calls facade.increase(serviceId)
  → facade: store.increase(serviceId)
    → quantitiesSignal updated
    → selectedCount, selectedTotal recomputed
  → facade: telegram.hapticLight()
  → Telegram SDK: HapticFeedback.impactOccurred('light')
  → facade effect: main button state syncs
    → store.selectedCount > 0: setMainButton("Checkout (N items)", enabled=true)
    → onMainButtonClick(handler: openCheckout)
  → template: service card shows updated quantity stepper
  → If selectedCount > 0 and localMode: FoodOrderCartBarComponent becomes visible
```

### Flow 4: Checkout

```
User taps "Checkout" (Telegram: main button / local: cart bar)
  → facade.openCheckout()
  → checkoutOpen signal = true
  → facade effect: main button updates
    → setMainButton("Place Order", enabled=true)
    → onMainButtonClick(handler: submitOrder)
  → FoodOrderCheckoutComponent renders (open=true)
    → Form pre-filled with persisted customer details
    → Order review card shows selected items

User fills form and taps "Place Order"
  → facade.submitOrder() called (Telegram: via main button handler)
  → Validation:
    → customerName required
    → customerPhone required + pattern match
    → deliveryDate >= earliestDeliveryDate
  → On invalid: submitError signal set, returns
  → telegram.confirm("Confirm your order?")
    → User taps OK
  → submitting = true
  → setMainButton("Placing order…", enabled=false)
  → tenantApi.createBooking(slug, request) POST /api/t/:slug/bookings
    → interceptor: polls for init data up to 1500ms
    → adds X-Telegram-Init-Data header
  → On success:
    → submittedBooking = response
    → checkoutOpen = false
    → store.clearCart()
    → refreshBookings()
    → setMainButton(null) — hide main button
  → On error:
    → submitting = false
    → submitError set
    → telegram.alert(errorMessage)
```

### Flow 5: View My Bookings

```
User taps "My orders" tab
  → facade.setActiveView('orders')
  → activeView signal = 'orders'
  → refreshBookings(): bookingsReloadKey++
  → bookingsVm signal triggers reload
    → tenantApi.getMyBookings(slug) GET /api/t/:slug/bookings/my
      → interceptor: polls for init data
    → on success: bookings list updated, loading=false
  → FoodOrderBookingsComponent renders
    → currentBookings computed (active statuses)
    → previousBookings computed (done/cancelled)
    → First booking auto-selected as selectedDisplayBooking
```

### Flow 6: View Booking Detail

```
User taps a booking card
  → FoodOrderBookingsComponent emits bookingSelected(bookingId)
  → FoodOrderHomeComponent calls facade.selectBooking(bookingId)
  → selectedBookingId = bookingId
  → tenantApi.getBooking(slug, bookingId) GET /api/t/:slug/bookings/:bookingId
    → Request versioning prevents stale updates from earlier slow responses
  → On success: selectedBooking = response
  → Detail panel renders:
    → Timeline with status state
    → QR code if PAYMENT_PENDING/NEW and paymentQrUrl present
    → Action buttons based on status
```

### Flow 7: Cancel Booking

```
User taps "Cancel" on booking detail
  → FoodOrderBookingsComponent emits cancelRequested(bookingId)
  → FoodOrderHomeComponent calls facade.cancelBooking(bookingId)
  → telegram.confirm("Cancel this order?")
    → User taps OK
  → cancellingBookingId = bookingId
  → tenantApi.cancelBooking(slug, bookingId) POST .../cancel
  → On success:
    → Update local selectedBooking status to CANCELLED
    → cancellingBookingId = null
    → refreshBookings()
  → On error:
    → cancelError set
    → cancellingBookingId = null
```

### Flow 8: Repeat Order

```
User taps "Repeat" on a previous booking
  → FoodOrderBookingsComponent emits repeatRequested(bookingId)
  → FoodOrderHomeComponent calls facade.repeatBooking(bookingId)
  → tenantApi.getBooking(slug, bookingId) — loads full booking details
  → For each booking item:
    → Normalize name (lowercase, trim)
    → Find matching service in current catalog by normalized name
  → store.setQuantities(matchedQuantities)
  → checkoutForm.patchValue({ customerName, customerPhone, deliveryAddress })
  → deliveryDate reset to defaultDeliveryDate()
  → If some items not found in catalog:
    → telegram.alert("Some items are no longer available...")
    → repeatOrderBanner set with warning message
  → facade.openCheckout()
  → activeView = 'menu' (to show checkout over menu)
```

---

## 9. Environment and Build Config

### `environment.ts` / `environment.prod.ts`

No `environment.ts` or `environment.prod.ts` files exist in the codebase. API base URL is hardcoded as `/api/t` in `TenantApiService`. No environment file switching is configured in `angular.json`.

### `angular.json`

| Setting | Value |
|---|---|
| `projectType` | `application` |
| `sourceRoot` | `src` |
| `outputPath` | `dist/yoobu-web` |
| `index` | `src/index.html` |
| `browser` | `src/main.ts` |
| `styles` | `["src/styles.css"]` |
| `polyfills` | `["zone.js"]` |
| `assets` | `["src/favicon.ico", "src/assets"]` |
| `budgets (prod)` | initial: 500kB warning / 1MB error; component style: 2kB warning / 4kB error |
| `extractLicenses` | `true` (prod) |
| `sourceMap` | `false` (prod) |

Dev server proxy config: `proxy.conf.json` applied via `proxyConfig` in serve target.

No custom Webpack config.

### `package.json` (v0.0.1) — non-trivial dependencies

| Package | Version |
|---|---|
| `@angular/common` | `^19.0.0` |
| `@angular/core` | `^19.0.0` |
| `@angular/forms` | `^19.0.0` |
| `@angular/platform-browser` | `^19.0.0` |
| `@angular/platform-browser-dynamic` | `^19.2.20` |
| `@angular/router` | `^19.0.0` |
| `rxjs` | `^7.8.0` |
| `zone.js` | `^0.15.0` |
| `tslib` | `^2.8.0` |

Dev dependencies: `@angular/cli`, `@angular/compiler-cli`, `@types/jasmine`, `jasmine-core`, `karma`, `karma-chrome-launcher`, `karma-coverage`, `karma-jasmine`, `karma-jasmine-html-reporter`, `typescript`.

### `proxy.conf.json`

```json
{
  "/api/t": {
    "target": "http://localhost:8080",
    "secure": false,
    "changeOrigin": true,
    "pathRewrite": { "^/api/t": "/t" }
  },
  "/api/admin": {
    "target": "http://localhost:8080",
    "secure": false,
    "changeOrigin": true,
    "pathRewrite": { "^/api/admin": "/admin" }
  },
  "/api/superadmin": {
    "target": "http://localhost:8080",
    "secure": false,
    "changeOrigin": true,
    "pathRewrite": { "^/api/superadmin": "/superadmin" }
  }
}
```

### Deployment (Railway)

**Dockerfile:**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM caddy:2-alpine
COPY deploy/caddy/Caddyfile /etc/caddy/Caddyfile
COPY --from=build /app/dist/yoobu-web/browser /srv
EXPOSE 8080
ENV PORT=8080
```

**`deploy/caddy/Caddyfile`:**
```
:{$PORT}

handle /api/* {
    uri strip_prefix /api
    reverse_proxy {$BACKEND_URL}
}

handle {
    root * /srv
    try_files {path} /index.html
    file_server
}
```

- Port: `$PORT` env var (default `8080`).
- API proxy: `/api/*` → `$BACKEND_URL` (strips `/api` prefix).
- SPA fallback: all unmatched paths → `/index.html`.
- Static files served from `/srv` (Angular build output).

---

## 10. Guards and Resolvers

**No route guards or resolvers are defined anywhere in the codebase.**

The `TenantShellComponent` performs its own guard-like behavior inline via the `vm$` observable:
- If `getConfig()` fails → renders error state (does not redirect).
- If type is unsupported → renders `UnsupportedFlowComponent` (does not redirect or throw).

---

## 11. Error and Loading States

### Loading States

| Location | Mechanism |
|---|---|
| Tenant config loading | `vm$` async pipe — renders "Loading" text while `vm$` has not emitted |
| Services loading | `facade.vm` signal — `loading: true` while `getServices()` in flight; template shows spinner/skeleton |
| Bookings loading | `facade.bookingsVm` signal — `loading: true`; template shows loading indicator |
| Order submit | `facade.submitting` signal — main button disabled + label changes to "Placing order…" |
| Payment confirm | `facade.confirmingPaymentBookingId` signal — button disabled while in flight |
| Booking cancel | `facade.cancellingBookingId` signal — button disabled while in flight |

No Telegram-native loading indicators (e.g. `MainButton.showProgress`) are used.

### API Error Display

| Error Source | Display Mechanism |
|---|---|
| Config load failure | Error string shown in `TenantShellComponent` status card |
| Service load failure | `facade.vm.error` → template shows error message |
| Bookings load failure | `facade.bookingsVm.error` → template shows error in bookings panel |
| Submit error | `facade.submitError` signal → shown inline in checkout form + `telegram.alert()` |
| Payment confirm error | `facade.paymentError` signal → shown in success card + `telegram.alert()` |
| Cancel error | `facade.cancelError` signal → shown in bookings detail |
| Telegram session missing | Interceptor throws; bubbles as observable error to calling service |

### Network-Offline Handling

Not handled. No offline detection, no service worker, no `navigator.onLine` checks.

### 404 / Invalid Slug

No 404 page. If a slug returns an HTTP error from `getConfig()`, `TenantShellComponent` catches it and renders a generic error message: `"This page is unavailable right now. Please try again later."`. The URL is not changed.

---

## 12. Test Coverage

### Test Files

| Spec file | Tests |
|---|---|
| `src/app/core/services/telegram.service.spec.ts` | TelegramService |
| `src/app/core/interceptors/telegram-init-data.interceptor.spec.ts` | `telegramInitDataInterceptor` |
| `src/app/features/food-order/food-order.store.spec.ts` | `FoodOrderStore` |
| `src/app/features/food-order/food-order-flow.facade.spec.ts` | `FoodOrderFlowFacade` (partial) |

---

### `TelegramService` spec

**Coverage:**
- Web app initialization: `ready()` and `expand()` called after `init()`.
- `getInitData()`: resolution chain — `webApp.initData` → launch params (`tgWebAppData` in query string and hash) → localStorage cache.
- localStorage caching: data written after retrieval.
- Main button: `setText`, `show`, `hide`, `enable`, `disable` called correctly by `setMainButton`.
- Main button click handler: `onClick` wired when handler set; `offClick` called with previous handler when replaced.
- `alert()`: uses `showAlert` if popup API supported (v ≥ 6.2); falls back to `window.alert`.
- `confirm()`: uses `showConfirm` if supported; falls back to `window.confirm`.
- Version checking: `6.2` supported; `6.1` not supported; `10.0` supported.

**Mocking approach:**
- `DOCUMENT` token injected with a mock object that has `defaultView.Telegram.WebApp`.
- `window.Telegram.WebApp` stubbed as a plain object with Jasmine spies.
- localStorage mocked via `jasmine.createSpyObj` or direct spy on `localStorage.getItem/setItem`.

**HTTP mocking:** Not applicable (no HTTP in TelegramService).

---

### `telegramInitDataInterceptor` spec

**Coverage:**
- Adds `X-Telegram-Init-Data` header when init data available.
- Adds `X-Telegram-User-Id: '101'` in dev/localhost mode.
- Passes request unmodified when no init data and non-bookings URL.
- Throws error for bookings URL when no init data after 1500ms.
- Waits (polls) up to 1500ms for init data on bookings requests.

**HTTP mocking:** `HttpTestingController` via `provideHttpClientTesting()`.

**Telegram mocking:** `TelegramService` provided as `jasmine.createSpyObj` with stubbed `getInitData` and `getDevTelegramUserId`.

---

### `FoodOrderStore` spec

**Coverage:**
- `selectedCount` computed: sums quantities.
- `selectedTotal` computed: sums `price × quantity`.
- `increase()`: increments; stops at `MAX_ITEM_QUANTITY` (9).
- `decrease()`: decrements; removes entry at 0.
- `setQuantities()`: bulk-sets; clamps at 9; filters to valid service IDs.
- `setServices()`: updates catalog; removes quantities for removed services.
- `setTenant()`: clears cart and services on slug change; no-op if slug unchanged.
- `selectedItems` computed: only services with quantity > 0.

**Setup:** `TestBed.configureTestingModule({})` — store instantiated via `inject(FoodOrderStore)`. Mock `ServiceItem` objects created inline.

**Mocking approach:** No mocks needed (pure signal state, no HTTP).

---

### `FoodOrderFlowFacade` spec (partial)

**Coverage (visible):**
- `earliestDeliveryDate` computed from `config.earliestDeliveryDate`.
- Dependencies mocked: `TenantApiService`, `TelegramService` as `jasmine.createSpyObj`.
- `FormBuilder` injected by facade.

**HTTP mocking:** `TenantApiService` fully stubbed (spy object).

**Telegram mocking:** `TelegramService` stubbed as spy object.

---

### Test utilities / conventions

- **No shared fixture files.** Each spec uses inline helper functions (e.g., `setRequiredInputs()`).
- **`fixture.componentRef.setInput()`** used for setting required signal inputs.
- **No HTTP mocking via `HttpTestingController` in component tests** — services are stubbed entirely.
- **`TestBed.configureTestingModule({ imports: [StandaloneComponent] })`** pattern for component tests.
- **No `fdescribe`/`fit` present in committed code** (reserved for local isolation).

---

## 13. Not Yet Implemented

### Tenant Types with No Feature Component

| `TenantType` value | Status |
|---|---|
| `'FOOD_ORDER'` | Implemented (`FoodOrderHomeComponent`) |
| `'APPOINTMENT'` | Not implemented — falls through to `UnsupportedFlowComponent` |
| `'CATALOG_REQUEST'` | Not implemented — falls through to `UnsupportedFlowComponent` |

`proxy.conf.json` includes `/api/admin` and `/api/superadmin` routes → no corresponding frontend routes or components exist.

### Interfaces Defined but Partially Used

- `TenantConfig.durationMinutes` field in `ServiceItem` — loaded from API but not displayed in any template.
- `TenantConfig.logoUrl` — loaded but not rendered in any visible template element (referenced in `TenantShellComponent` but display is ambiguous).
- `BookingResponse.type` — loaded; not displayed or used for branching in any component.

### Features with Structural Placeholder Only

No fully scaffolded-but-unfinished modules exist. The `UnsupportedFlowComponent` is the explicit placeholder for unimplemented tenant types.

### Commented-Out Code / Feature Flags

No feature flags in code. No commented-out feature blocks identified in source.

### `any` Usage

No `any` usage in model interfaces. Ambiguous: `TelegramWebApp` interface is accessed via `(window as any).Telegram?.WebApp` in `TelegramService.resolveWebApp()` — this is the sole `as any` cast in the codebase, used to access the untyped global SDK.

---

*End of RND_WEB_STATE.md*
