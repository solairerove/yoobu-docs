# RND WEB State (Source-Derived)

Generated from source/config/template/spec files in `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web` only.

## 1. Module and Routing Structure

### 1.1 Angular module structure

| Item | State | Evidence |
|---|---|---|
| NgModules (`@NgModule`) | None in app source | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app` |
| Boot mode | Standalone bootstrap with `bootstrapApplication(AppComponent, appConfig)` | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/main.ts` |
| Router setup | `provideRouter(appRoutes)` in `ApplicationConfig` | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/app.config.ts` |
| HTTP setup | `provideHttpClient(withInterceptors([telegramInitDataInterceptor]))` | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/app.config.ts` |

### 1.2 Full routing tree

| Route | Type | Target | Guards | Resolvers | Lazy boundary | Params consumed | Query params consumed |
|---|---|---|---|---|---|---|---|
| `` | redirect | `t/demo` | None | None | N/A | None | None |
| `t/:slug` | loadComponent | `TenantShellComponent` | None | None | Yes (`loadComponent`) | `slug` via `ActivatedRoute.paramMap` | None via Angular router |
| `**` | wildcard redirect | `t/demo` | None | None | N/A | None | None |

Evidence:
- `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/app.routes.ts`
- `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/tenant/tenant-shell.component.ts`

## 2. Components

### 2.1 Component inventory

| Component class | Selector | Standalone status | Declared in module | File |
|---|---|---|---|---|
| `AppComponent` | `app-root` | Standalone (has `imports`) | None | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/app.component.ts` |
| `TenantShellComponent` | `app-tenant-shell` | Standalone (has `imports`) | None | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/tenant/tenant-shell.component.ts` |
| `UnsupportedFlowComponent` | `app-unsupported-flow` | Standalone | None | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/unsupported/unsupported-flow.component.ts` |
| `FoodOrderHomeComponent` | `app-food-order-home` | Standalone | None | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-home.component.ts` |
| `FoodOrderMenuComponent` | `app-food-order-menu` | Standalone | None | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-menu.component.ts` |
| `FoodOrderCartBarComponent` | `app-food-order-cart-bar` | Standalone | None | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-cart-bar.component.ts` |
| `FoodOrderCheckoutComponent` | `app-food-order-checkout` | Standalone | None | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-checkout.component.ts` |
| `FoodOrderBookingsComponent` | `app-food-order-bookings` | Standalone | None | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-bookings.component.ts` |
| `FoodOrderSuccessCardComponent` | `app-food-order-success-card` | Standalone | None | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-success-card.component.ts` |

### 2.2 Inputs/Outputs, injections, lifecycle, template behavior

| Component | Inputs | Outputs | Injected services/tokens | Lifecycle hooks | Template behavior summary | Telegram SDK calls from component |
|---|---|---|---|---|---|---|
| `AppComponent` | None | None | None | None | Renders only `<router-outlet />` | None |
| `TenantShellComponent` | None | None | `DOCUMENT`, `ActivatedRoute`, `TenantApiService`, `TelegramService` | No Angular lifecycle interface; uses RxJS pipeline over route params | Shows hero/status cards; applies tenant theme CSS var; dynamic feature render via `ngComponentOutlet`; handles loading/error | Calls `telegram.init()`, `telegram.setMainButton(null)` |
| `UnsupportedFlowComponent` | `config: TenantConfig` (required signal input) | None | None | None | Renders unavailable card with tenant type | None |
| `FoodOrderHomeComponent` | `config: TenantConfig` | None | `FoodOrderFlowFacade` (provider local to component) | No lifecycle interface; constructor `effect()` calls `facade.setConfig` | Switches between Menu/My orders tabs, loading/error/empty states, success card, menu, cart bar, checkout, bookings detail | No direct SDK calls; delegates via facade |
| `FoodOrderMenuComponent` | `services: ServiceItem[]`, `selectedCount: number`, `currencyCode: string`, `quantities: Record<number, number>`, `defaultUnit: string='item'` | `increaseRequested: number`, `decreaseRequested: number` | None | None | Renders service cards with quantity controls, selected states, currency formatting | None |
| `FoodOrderCartBarComponent` | `checkoutOpen: boolean`, `selectedCount: number`, `selectedTotal: number`, `currencyCode: string` | `openRequested: void` | None | None | Fixed bottom cart CTA with count/total and contextual label | None |
| `FoodOrderCheckoutComponent` | `open`, `localMode`, `submitting`, `submitError`, `repeatOrderBanner`, `form: FormGroup`, `isFirstOrder`, `customerNameHint`, `customerPhoneHint`, `customerNoteHint`, `currencyCode`, `selectedItems`, `selectedCount`, `selectedTotal` | `closeRequested`, `repeatOrderBannerDismissed`, `submitRequested` | None | None | Local-mode card or Telegram-mode bottom sheet; form fields + validation hints + order summary | None |
| `FoodOrderBookingsComponent` | `bookings`, `loading`, `error`, `paymentQrUrl`, `selectedBookingId`, `selectedBooking`, `confirmingPaymentBookingId`, `paymentError`, `cancellingBookingId`, `cancelError`, `currencyCode` | `refreshRequested`, `bookingSelected`, `repeatRequested`, `paymentConfirmRequested`, `cancelRequested` | None | None | Open orders/history sections, selectable list, status badges/timeline, QR/tracking links, receipt | None |
| `FoodOrderSuccessCardComponent` | `booking`, `fallbackCurrency`, `paymentQrUrl`, `confirmingPaymentBookingId`, `paymentError` | `paymentConfirmRequested`, `newOrderRequested`, `ordersRequested` | None | None | Displays submitted booking snapshot + optional QR + payment confirm + tracking + actions | None |

## 3. Services

### 3.1 `TenantApiService`

File: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/services/tenant-api.service.ts`

Public methods:

```ts
getConfig(slug: string): Observable<TenantConfig>
getServices(slug: string): Observable<ServiceItem[]>
createBooking(slug: string, request: CreateBookingRequest): Observable<BookingResponse>
getMyBookings(slug: string): Observable<BookingResponse[]>
getBooking(slug: string, bookingId: number): Observable<BookingResponse>
confirmBookingPayment(slug: string, bookingId: number): Observable<BookingResponse>
cancelBooking(slug: string, bookingId: number): Observable<BookingResponse>
```

HTTP calls:

| Method | URL path | Request body type | Response type | Headers attached |
|---|---|---|---|---|
| GET | `/api/t/{slug}/config` | none | `TenantConfig` | via interceptor (Telegram headers conditional) |
| GET | `/api/t/{slug}/services` | none | `ServiceItem[]` | via interceptor |
| POST | `/api/t/{slug}/bookings` | `CreateBookingRequest` | `BookingResponse` | via interceptor |
| GET | `/api/t/{slug}/bookings/my` | none | `BookingResponse[]` | via interceptor |
| GET | `/api/t/{slug}/bookings/{bookingId}` | none | `BookingResponse` | via interceptor |
| POST | `/api/t/{slug}/bookings/{bookingId}/confirm-payment` | `{}` | `BookingResponse` | via interceptor |
| POST | `/api/t/{slug}/bookings/{bookingId}/cancel` | `{}` | `BookingResponse` | via interceptor |

State held: private fields only (`http`, `baseUrl='/api/t'`), no subjects/signals for mutable UI state.

Error handling in service: none local; HTTP errors bubble to caller.

### 3.2 `TelegramService`

File: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/telegram/telegram.service.ts`

Public methods:

```ts
init(): void
getInitData(): string | null
getDevTelegramUserId(): string | null
isLocalhost(): boolean
setMainButton(text: string | null, enabled = true): void
onMainButtonClick(handler: (() => void) | null): void
alert(message: string): Promise<void>
confirm(message: string): Promise<boolean>
```

State held:

| Field | Kind | Shape | Initial |
|---|---|---|---|
| `webAppSignal` | signal | `TelegramWebApp \| null` | `null` |
| `mainButtonHandler` | signal | `(() => void) \| null` | `null` |

Reads/writes:
- `init()` sets `webAppSignal`.
- `resolveWebApp()` lazy-resolves `window.Telegram.WebApp` and caches in `webAppSignal`.
- `onMainButtonClick()` writes handler signal.
- Effects bind/unbind `MainButton.onClick/offClick` and call `ready/expand` when webApp exists.

Error handling:
- `alert()` and `confirm()` try Telegram popup APIs when available and version-compatible; fallback to browser dialogs on absence/throw.
- No thrown error from fallback path (resolves promise).

### 3.3 `FoodOrderFlowFacade` (feature-scoped injectable)

File: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-flow.facade.ts`

Public methods:

```ts
setConfig(config: TenantConfig): void
increase(serviceId: number): void
decrease(serviceId: number): void
openCheckout(): void
closeCheckout(): void
startNewOrder(): void
refreshBookings(): void
selectBooking(bookingId: number): Promise<void>
cancelBooking(bookingId: number): Promise<void>
confirmPayment(bookingId: number): Promise<void>
repeatBooking(bookingId: number): Promise<void>
dismissRepeatOrderBanner(): void
submitOrder(): Promise<void>
setActiveView(view: 'menu' | 'orders'): void
```

Injected dependencies:
- `TenantApiService`
- `FormBuilder`
- `TelegramService`
- `FoodOrderStore`

State held:

| Field | Kind | Shape | Initial |
|---|---|---|---|
| `checkoutForm` | reactive form | `{customerName, customerPhone, deliveryAddress, deliveryDate, note}` | empty + `deliveryDate=today` |
| `bookingsReloadKey` | signal | `number` | `0` |
| `selectedBookingId` | signal | `number \| null` | `null` |
| `selectedBooking` | signal | `BookingResponse \| null` | `null` |
| `activeView` | signal | `'menu' \| 'orders'` | `'menu'` |
| `checkoutOpen` | signal | `boolean` | `false` |
| `submitting` | signal | `boolean` | `false` |
| `submitError` | signal | `string \| null` | `null` |
| `repeatOrderBanner` | signal | `string \| null` | `null` |
| `submittedBooking` | signal | `BookingResponse \| null` | `null` |
| `confirmingPaymentBookingId` | signal | `number \| null` | `null` |
| `paymentError` | signal | `string \| null` | `null` |
| `cancellingBookingId` | signal | `number \| null` | `null` |
| `cancelError` | signal | `string \| null` | `null` |
| `customerDetailsDraft` | signal | `CustomerDetailsDraft` | empty strings |
| `customerDetailsHydrated` | signal | `boolean` | `false` |
| `configSignal` | signal | `TenantConfig \| null` | `null` |
| `vmSignal` | signal | `FoodOrderVm` | `{services:[],loading:true,error:null}` |
| `bookingsVmSignal` | signal from Rx stream | `MyBookingsVm` | `{bookings:[],loading:true,error:null}` |
| `showLocalCheckoutButtons` | plain boolean | from `telegram.isLocalhost()` | computed once at construct |

HTTP usage and error behavior:
- `getServices` in `loadServices`: success updates store/vm; error sets vm error message.
- `getMyBookings` in stream: error mapped to `{bookings:[], loading:false, error:'Could not load your orders.'}`.
- `getBooking` in `selectBooking`: error sets `cancelError='Could not load the order details.'`.
- `cancelBooking`: on error sets `cancelError` and triggers `telegram.alert(...)`.
- `confirmBookingPayment`: if `409`, alerts and refreshes/reloads booking; other errors set `paymentError` + alert.
- `createBooking` in `submitOrder`: error reopens checkout, sets `submitError`, alerts user.

Telegram interactions in facade:
- Main button text/show/hide/enable-disable and click handler wiring via `telegram.setMainButton(...)` and `telegram.onMainButtonClick(...)`.
- `telegram.confirm(...)` on submit/cancel/repeat-replace.
- `telegram.alert(...)` for validation and API failure feedback.

### 3.4 `FoodOrderStore`

File: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order.store.ts`

Public methods:

```ts
setTenant(slug: string): void
setServices(services: ServiceItem[]): void
quantityFor(serviceId: number): number
increase(serviceId: number): void
decrease(serviceId: number): void
setQuantities(quantities: Record<number, number>): void
clearCart(): void
```

State held:

| Field | Kind | Shape | Initial |
|---|---|---|---|
| `slugSignal` | signal | `string \| null` | `null` |
| `servicesSignal` | signal | `ServiceItem[]` | `[]` |
| `quantitiesSignal` | signal | `Record<number, number>` | `{}` |

Derived state:
- `selectedItems`, `selectedCount`, `selectedTotal`, `services`, `quantities`.

Error handling: none; pure in-memory state operations.

## 4. TypeScript Interfaces and Types

### 4.1 API request/response models

File: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/models/booking.model.ts`

```ts
export interface CreateBookingItem {
  serviceId: number;
  quantity: number;
}

export interface CreateBookingRequest {
  customerName: string;
  customerPhone: string;
  deliveryAddress: string;
  deliveryDate: string;
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
  type: 'ORDER' | 'APPOINTMENT' | 'REQUEST' | string;
  status: BookingStatus;
  customerName: string;
  customerPhone: string;
  deliveryAddress: string | null;
  totalPrice: number;
  currency: string;
  deliveryDate: string;
  note: string | null;
  trackingUrl: string | null;
  items: BookingItem[];
  createdAt: string;
}

export type BookingStatus =
  | 'NEW'
  | 'PAYMENT_PENDING'
  | 'CONFIRMED'
  | 'DELIVERING'
  | 'DONE'
  | 'CANCELLED'
  | string;
```

File: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/models/service.model.ts`

```ts
export interface ServiceItem {
  id: number;
  name: string;
  description: string | null;
  price: number;
  unit: string | null;
  durationMinutes: number | null;
  sortOrder: number;
  status: 'ACTIVE' | 'INACTIVE' | 'DELETED';
}
```

File: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/models/tenant-config.model.ts`

```ts
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
  checkoutNoteHint?: string | null;
}
```

### 4.2 Internal state shapes

From `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-flow.facade.ts`:

```ts
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

interface CustomerDetailsDraft {
  customerName: string;
  customerPhone: string;
  deliveryAddress: string;
}
```

From `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-checkout.component.ts`:

```ts
interface CheckoutSelection {
  quantity: number;
  service: ServiceItem;
}
```

From `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/telegram/telegram.service.ts`:

```ts
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
}

interface TelegramWindow extends Window {
  Telegram?: {
    WebApp?: TelegramWebApp;
  };
}
```

`any` usage flag:
- No `any` type usage found in `src/**/*.ts` (`rg -n "\bany\b" src` returned none except comment text).

## 5. HTTP Layer

### 5.1 Interceptors and execution order

| Interceptor | Order | Behavior | File |
|---|---|---|---|
| `telegramInitDataInterceptor` | 1 (only interceptor) | Initializes Telegram service, resolves init data, conditionally attaches `X-Telegram-Init-Data` or local fallback `X-Telegram-User-Id` | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/http/telegram-init-data.interceptor.ts` |

Order source: `withInterceptors([telegramInitDataInterceptor])` in `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/app.config.ts`.

### 5.2 Base URL and path resolution

- API client base: `TenantApiService.baseUrl = '/api/t'`.
- Dev (`ng serve`): `/api/t` proxied to `http://localhost:8080/t` via `proxy.conf.json` rewrite.
- Railway Docker runtime: nginx rewrites `/api/{x}` to `/{x}` and proxies to `${BACKEND_URL}`.

### 5.3 Telegram headers

Header attachment logic in interceptor:
1. `telegram.getDevTelegramUserId()` read first.
2. For `/bookings` URLs with no dev user id, it polls for init data up to 1500ms every 50ms.
3. If init data present => add `X-Telegram-Init-Data`.
4. Else if dev user id present => add `X-Telegram-User-Id`.
5. Else => no Telegram auth headers.

### 5.4 Error interceptor behavior

- No dedicated error interceptor exists.
- No retry interceptor exists.
- No centralized HTTP logging interceptor exists.
- User feedback for failures is handled in `FoodOrderFlowFacade` via local state + `TelegramService.alert`.

## 6. Telegram Mini App Integration

### 6.1 Access points and methods

| Concern | Implementation | File |
|---|---|---|
| SDK script load | `<script src="https://telegram.org/js/telegram-web-app.js"></script>` | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/index.html` |
| `window.Telegram.WebApp` access | `resolveWebApp()` reads `document.defaultView.Telegram.WebApp` | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/telegram/telegram.service.ts` |
| `tg.ready()` / `tg.expand()` | effect executes when web app signal becomes available | same file |
| `tg.initData` read | `getInitData()` reads `webApp.initData` or fallback from launch params (`tgWebAppData`) | same file |
| API header pass-through | interceptor sets `X-Telegram-Init-Data` | `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/http/telegram-init-data.interceptor.ts` |
| `tg.MainButton` | configured in `TelegramService.setMainButton`, click binding in `onMainButtonClick`; business logic in `FoodOrderFlowFacade` | service + facade files |
| `tg.BackButton` | Not used | N/A |
| `tg.close()` | Not used | N/A |
| `tg.showAlert` | Wrapped by `TelegramService.alert` with fallback to `window.alert` | service file |
| `tg.showConfirm` | Wrapped by `TelegramService.confirm` with fallback to `window.confirm` | service file |
| `HapticFeedback` | Not used | N/A |
| `themeParams` | Not used | N/A |
| `viewportHeight`/`viewportStableHeight` | Not used | N/A |

### 6.2 MainButton labels and show/hide logic

In `FoodOrderFlowFacade` constructor effect:
- If `submittedBooking` exists or selected item count is `0` -> hide main button (`setMainButton(null)`), clear click handler.
- If cart has items and checkout closed -> text: `Checkout • {formatted total}`.
- If checkout open -> text: `Submitting...` when submitting else `Place order • {formatted total}`; disabled during submit.
- click action:
  - if checkout closed -> `openCheckout()`
  - if checkout open -> `submitOrder()`

### 6.3 Ambiguity note

- `getInitData()` supports both Telegram runtime `initData` and URL launch param `tgWebAppData`. If both exist, runtime value is used.

## 7. Theming and Styling

### 7.1 `primaryColor` application

| Mechanism | Where |
|---|---|
| Root CSS variable `--yoobu-primary` set on `document.documentElement` using `TenantConfig.primaryColor` (fallback `#ff6b35`) | `TenantShellComponent.applyTheme` in `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/tenant/tenant-shell.component.ts` |
| Local shell style variable binding `[style.--yoobu-primary]="vm.config?.primaryColor || '#ff6b35'"` | same file |
| Downstream components consume CSS vars (`--yoobu-primary`, borders, shadows, surfaces) | component style blocks + `/src/styles.css` |

### 7.2 Global vs component styles

- Global: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/styles.css`
  - color system CSS variables, base typography, button/shared utility classes (`.ghost-button`, `.ui-copy`, `.ui-status-card`).
- Component-scoped: inline `styles` arrays in each standalone component TS.

### 7.3 CSS framework/library

- No Bootstrap/Tailwind/Material utility framework usage detected.
- Styling is custom CSS only.

### 7.4 Responsive/mobile considerations

Implemented across components with:
- `@media (max-width: 640px)` layouts.
- Fixed bottom cart with `env(safe-area-inset-bottom)`.
- Checkout bottom sheet with `100dvh`/safe-area handling in Telegram mode.
- Nginx SPA fallback for deep links.

## 8. State Flow Diagrams

### 8.1 App init / tenant resolution

1. `AppComponent` router outlet loads route.
2. Route `t/:slug` lazy-loads `TenantShellComponent`.
3. `TenantShellComponent.vm$` reads `slug` from `ActivatedRoute.paramMap`.
4. On slug change: `telegram.init()` and `telegram.setMainButton(null)`.
5. `TenantApiService.getConfig(slug)` request.
6. On success: `applyTheme(primaryColor)`, render feature component by `config.type`.
7. On error: render tenant unavailable card.

### 8.2 Catalog browse

1. `FoodOrderHomeComponent` effect passes `config` into `FoodOrderFlowFacade.setConfig`.
2. `setConfig` resets per-tenant state and calls `loadServices`.
3. `TenantApiService.getServices(slug)` fetches menu.
4. On success: `FoodOrderStore.setServices`, vm updates, menu list renders.
5. User views cards in `FoodOrderMenuComponent`; no navigation occurs.

### 8.3 Add to cart

1. User taps `+` in `FoodOrderMenuComponent`.
2. Emits `increaseRequested(serviceId)` to `FoodOrderHomeComponent`.
3. Home calls `facade.increase(serviceId)` -> `store.increase(serviceId)`.
4. Computed store totals/count update.
5. UI updates menu selected states and cart bar.
6. Telegram main button effect updates label to checkout total.

### 8.4 Checkout (submit order)

1. User opens checkout via cart bar or Telegram MainButton click.
2. `checkoutOpen=true`; checkout form shown.
3. Submit action from local form or Telegram MainButton:
   - validates non-empty required fields and delivery address pattern.
   - on invalid -> `submitError` + `telegram.alert`.
4. On valid -> `telegram.confirm` with total.
5. If confirmed -> `TenantApiService.createBooking(slug, request)`.
6. On success:
   - set `submittedBooking`, `selectedBookingId`, `selectedBooking`
   - clear cart
   - close checkout
   - reset form (preserving drafted customer details)
   - refresh bookings stream key
7. On error:
   - keep checkout open
   - set `submitError`
   - `telegram.alert`.

### 8.5 View my bookings

1. User switches tab to `My orders` in `FoodOrderHomeComponent`.
2. `facade.setActiveView('orders')` and `refreshBookings()`.
3. `bookingsVmSignal` pipeline calls `getMyBookings(slug)`.
4. On success:
   - updates booking list vm
   - reconciles selected booking id/details
   - updates submitted active booking references
   - hydrates customer draft from latest booking once per tenant.
5. `FoodOrderBookingsComponent` renders grouped open/history sections.

### 8.6 View booking detail

1. User clicks booking row in `FoodOrderBookingsComponent`.
2. Emits `bookingSelected(id)` -> home -> `facade.selectBooking(id)`.
3. `selectBooking` sets `selectedBookingId`, active view, clears payment/cancel errors.
4. Calls `TenantApiService.getBooking(slug, id)`.
5. Uses `selectedBookingRequestVersion` to discard stale async responses.
6. On success sets `selectedBooking`; detail panel renders timeline/receipt/actions.
7. On error sets `cancelError='Could not load the order details.'`.

### 8.7 Cancel booking

1. User taps `Cancel order` action.
2. `facade.cancelBooking(id)` asks `telegram.confirm(...)`.
3. If confirmed -> set `cancellingBookingId` + clear `cancelError`.
4. Call `TenantApiService.cancelBooking(slug, id)`.
5. On success: update `selectedBooking`, possibly `submittedBooking`, and refresh bookings.
6. On error: set `cancelError`, show `telegram.alert`.
7. Finally clear `cancellingBookingId`.

## 9. Environment and Build Config

### 9.1 `environment.ts` / `environment.prod.ts`

- No `src/environments/environment.ts` or `src/environments/environment.prod.ts` found.

### 9.2 `angular.json` (build/serve/test key sections)

File: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/angular.json`

```json
{
  "projects": {
    "yoobu-web": {
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:application",
          "options": {
            "outputPath": "dist/yoobu-web",
            "browser": "src/main.ts",
            "index": "src/index.html",
            "assets": [{ "glob": "**/*", "input": "public" }],
            "styles": ["src/styles.css"]
          },
          "configurations": {
            "production": {
              "budgets": [{ "type": "initial", "maximumWarning": "500kB", "maximumError": "1MB" }],
              "outputHashing": "all"
            },
            "development": {
              "optimization": false,
              "extractLicenses": false,
              "sourceMap": true
            }
          }
        },
        "serve": {
          "configurations": {
            "development": {
              "buildTarget": "yoobu-web:build:development",
              "proxyConfig": "proxy.conf.json"
            }
          }
        }
      }
    }
  }
}
```

Custom webpack config: none found.

### 9.3 `package.json` dependencies (non-trivial, skipping Angular core family)

| Package | Version |
|---|---|
| `rxjs` | `^7.8.0` |
| `zone.js` | `^0.15.0` |
| `@angular-eslint/*` tooling | `^19.8.1` |
| `eslint` | `^8.57.1` |
| `karma` + launchers/reporters | `^6.4.4` / related |
| `typescript` | `^5.6.0` |

Source: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/package.json`

### 9.4 Proxy config (local dev)

File: `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/proxy.conf.json`

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

### 9.5 Railway deployment path

Files:
- `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/railway.toml`
- `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/Dockerfile`
- `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/deploy/nginx/default.conf.template`
- `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/deploy/nginx/40-validate-backend-url.sh`

Flow:
1. Railway uses Dockerfile build.
2. Stage 1 builds Angular app via `npm ci` + `npm run build`.
3. Stage 2 serves static files via `nginx:1.27-alpine` on `PORT` (default 8080).
4. Entry script requires `BACKEND_URL` and forbids trailing slash.
5. Nginx proxies `/api/*` to backend and serves SPA fallback (`try_files ... /index.html`).

## 10. Guards and Resolvers

| Item | Present? | Details |
|---|---|---|
| Route guards (`canActivate`, `canMatch`, etc.) | No | No guard classes/functions referenced in `app.routes.ts` |
| Route resolvers | No | No resolver references in route config |

Redirect/failure behavior:
- Invalid route path (`**`) redirects to `t/demo`.
- Tenant config fetch failure handled inside `TenantShellComponent` view model (no router redirect).

## 11. Error and Loading States

### 11.1 Loading UX

| Area | Loading handling |
|---|---|
| Tenant shell | `"Loading"` status card while config unresolved |
| Menu catalog | `"Loading menu"` status card |
| Bookings | `"Loading orders"` status card |
| Submit/payment/cancel actions | Button labels switch to `Submitting...`, `Confirming...`, `Cancelling...` and disable actions |

No Telegram-native loading indicator usage found.

### 11.2 API error UX

| Area | User-facing behavior |
|---|---|
| Tenant config | Generic tenant unavailable card |
| Services fetch | vm error message `"Check the backend service or tenant data and try again."` |
| Bookings list | vm error `"Could not load your orders."` |
| Booking detail | `cancelError` used to display `"Could not load the order details."` |
| Submit/cancel/confirm payment/repeat | local error signals + `TelegramService.alert` popups in failure/validation branches |

### 11.3 Offline handling

- No explicit network offline detection (`navigator.onLine`, service worker offline strategies, retry backoff) found.

### 11.4 404 / invalid slug behavior

- URL-level unknown path -> route wildcard redirect to `t/demo`.
- Tenant slug not found/backend error -> shell stays on route and renders unavailable message.

## 12. Test Coverage

| Spec file | Coverage summary | Telegram mocking | HTTP mocking | Shared utilities/fixtures |
|---|---|---|---|---|
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/http/telegram-init-data.interceptor.spec.ts` | Header attachment logic for init data, dev fallback, no-header branch | `jasmine.SpyObj<TelegramService>` | `HttpTestingController` + `provideHttpClientTesting` | none |
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/core/telegram/telegram.service.spec.ts` | `ready/expand`, initData extraction, launch param parsing, MainButton wiring, popup fallback/version behavior | manual fake `defaultView.Telegram.WebApp` | none | `setupService(...)` helper |
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/tenant/tenant-shell.component.spec.ts` | slug change behavior, error fallback vm, theme application | `SpyObj<TelegramService>` | API mocked via `SpyObj<TenantApiService>` returning `of/throwError` | `createConfig`, `createBookingResponse` |
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-flow.facade.spec.ts` | service loading, stale detail response protection, repeat-order scenarios, submit validation/success, booking refresh, cancel/payment success/failure flows | `SpyObj<TelegramService>` (`confirm/alert/main button`) | API mocked via `SpyObj<TenantApiService>` + `Subject`/`throwError`/`HttpErrorResponse` | `createBookingResponse` |
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order.store.spec.ts` | selected totals/count, catalog pruning, set snapshot, tenant reset | N/A | N/A | local `serviceA/serviceB` fixtures |
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-home.component.spec.ts` | facade wiring, menu controls, tab switch, bookings visibility, repeat action routing, cart bar visibility | Facade methods mocked (Telegram not directly) | none | inline facade stub + service/config fixtures |
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-menu.component.spec.ts` | quantity button emissions, disabled decrease, selected card styling | N/A | N/A | `setRequiredInputs` helper |
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-cart-bar.component.spec.ts` | cart-bar click emission and label variants | N/A | N/A | `setRequiredInputs` helper |
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-checkout.component.spec.ts` | submit/close emissions, submitting disabled state, first-order hints, repeat banner dismiss | N/A | N/A | `createForm`, `setRequiredInputs` |
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-bookings.component.spec.ts` | refresh/select/repeat/cancel/payment events, grouping/ordering by status, address fallback, tracking link | N/A | N/A | booking fixture + helper |
| `/Users/solairerove/workspace/solairerove/yoobu/yoobu-web/src/app/features/food-order/food-order-success-card.component.spec.ts` | rendering, action events, payment confirm event, tracking link | N/A | N/A | booking fixture + helper |

## 13. Not Yet Implemented / Partially Implemented

| Item | Evidence | State |
|---|---|---|
| `TenantType` values `APPOINTMENT` and `CATALOG_REQUEST` | `TenantType` union in `tenant-config.model.ts`; `TenantShellComponent.resolveFeatureComponent` only maps `FOOD_ORDER` and falls back to `UnsupportedFlowComponent` | Typed but feature UIs not implemented |
| Routing for non-food flows | Only `t/:slug` shell route exists; no dedicated route trees for appointment/catalog | Not implemented |
| `tg.BackButton`, `tg.close`, `HapticFeedback`, theme params, viewport methods | No code references in `src/**/*.ts` | Not implemented in frontend integration |
| Environment files (`environment.ts`, `environment.prod.ts`) | No files found under `src/environments` | Not present |
| Guard/resolver layer | No guard/resolver classes or route config refs | Not present |
| Commented-out planned code / feature flags | No feature flags or commented-out blocks indicating roadmap found in scanned TS/CSS/HTML files | None found |

Ambiguity note:
- `BookingStatus` and `BookingResponse.type` include `| string`, so backend can send unenumerated values. UI methods usually normalize known values and otherwise display fallback raw status/title text.
