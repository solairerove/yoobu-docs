# Tech Spec: Delivery Cutoff Time — Write Path

## Context

Read path already works:
- `TenantTimeService.earliestDeliveryDate(Tenant, TenantSettings)` reads `cutoff_hour` + `cutoff_minute` from `tenant_config`
- `BookingService.createFoodOrder` validates delivery date against cutoff
- Logic: if current time ≥ cutoff → earliest delivery = tomorrow; else → today
- Missing cutoff config → no cutoff enforcement (any date today or later accepted)

What's missing: no endpoint or form writes `cutoff_hour` / `cutoff_minute`. Admin cannot configure cutoff.

## Scope

1. Superadmin REST API: accept `cutoffHour` + `cutoffMinute` in create/update tenant
2. Superadmin Panel (Thymeleaf): cutoff fields in tenant form
3. Admin Panel (Thymeleaf): cutoff display + edit in tenant settings section
4. Frontend: show cutoff info in checkout (informational)

---

## Database Changes

Нет. `tenant_config` — key-value store. Ключи `cutoff_hour` и `cutoff_minute` уже читаются. Нужно только записывать.

---

## Backend

### 1. `CreateTenantRequest` / `UpdateTenantRequest`

Добавить поля:

```java
// In both DTOs:
private Integer cutoffHour;    // nullable, 0-23
private Integer cutoffMinute;  // nullable, 0-59
```

Валидация: если одно из двух задано, оба должны быть заданы. `cutoffHour` в [0, 23], `cutoffMinute` в [0, 59]. Оба null — допустимо (cutoff отключён).

Реализация валидации — в `TenantManagementService`, не через Bean Validation (cross-field constraint на два nullable поля в custom annotation — overkill). Service-level:

```java
if ((cutoffHour == null) != (cutoffMinute == null)) {
    throw new ResponseStatusException(BAD_REQUEST, 
        "Both cutoffHour and cutoffMinute must be set, or both must be empty");
}
if (cutoffHour != null && (cutoffHour < 0 || cutoffHour > 23)) {
    throw new ResponseStatusException(BAD_REQUEST, "cutoffHour must be 0-23");
}
if (cutoffMinute != null && (cutoffMinute < 0 || cutoffMinute > 59)) {
    throw new ResponseStatusException(BAD_REQUEST, "cutoffMinute must be 0-59");
}
```

### 2. `TenantManagementService.createTenant` / `updateTenant`

Добавить запись в tenant_config:

```java
// After existing config writes:
if (request.getCutoffHour() != null) {
    upsertConfig(tenantId, "cutoff_hour", String.valueOf(request.getCutoffHour()));
    upsertConfig(tenantId, "cutoff_minute", String.valueOf(request.getCutoffMinute()));
} else {
    removeConfig(tenantId, "cutoff_hour");
    removeConfig(tenantId, "cutoff_minute");
}
```

`removeConfig`: удалить строку из `tenant_config` (или upsert с null value — зависит от текущей реализации `removeWhenBlank`). Текущий код уже использует `removeWhenBlank=true` для optional config fields — тот же паттерн.

### 3. `TenantDetailResponse`

`config` map уже содержит все tenant_config записи. `cutoff_hour` и `cutoff_minute` будут в map автоматически после записи. Ничего менять не нужно.

### 4. `SuperAdminTenantForm` (Thymeleaf MVC)

Добавить поля:

```java
private Integer cutoffHour;
private Integer cutoffMinute;
```

В `SuperAdminPanelController.createTenant` / `updateTenant` — маппинг из формы в `CreateTenantRequest` / `UpdateTenantRequest` (где уже стоит валидация).

В `SuperAdminPanelController.editTenant` — читать из `TenantDetailResponse.config`:

```java
form.setCutoffHour(parseIntOrNull(config.get("cutoff_hour")));
form.setCutoffMinute(parseIntOrNull(config.get("cutoff_minute")));
```

### 5. Thymeleaf template: `superadmin/panel/tenant-form`

Два input поля в секции delivery settings:

```html
<label>Order cutoff time (tenant timezone)</label>
<div>
  <input type="number" name="cutoffHour" min="0" max="23" 
         th:value="${form.cutoffHour}" placeholder="HH"/>
  :
  <input type="number" name="cutoffMinute" min="0" max="59" 
         th:value="${form.cutoffMinute}" placeholder="MM"/>
</div>
<small>Orders placed after this time cannot select today's date. 
       Leave empty to disable cutoff.</small>
```

### 6. Admin Panel: tenant settings view

Сейчас admin panel не имеет страницы tenant settings. Два варианта:

**Вариант A (минимальный)**: Показать cutoff read-only на странице bookings или services. Админ настраивает через superadmin. Для MVP с 2-3 тенантами — достаточно: superadmin (ты) настраиваешь cutoff при onboarding.

**Вариант B**: Новая страница `GET /admin/{slug}/panel/settings` с формой для cutoff. Требует новый endpoint, контроллер, шаблон.

Рекомендация: **вариант A**. Cutoff меняется раз при настройке тенанта. Superadmin form — достаточно.

### 7. `TenantConfigResponse` (client API)

Добавить поля для фронтенда:

```java
// New fields in TenantConfigResponse record:
Integer cutoffHour,    // nullable
Integer cutoffMinute   // nullable
```

`TenantConfigService.getCurrentTenantConfig` — маппинг из `TenantSettings.delivery()`:

```java
TenantSettings.DeliverySettings delivery = settings.delivery();
// delivery.cutoffHour() and delivery.cutoffMinute() already parsed
```

---

## Frontend

### 1. `TenantConfig` interface

```typescript
export interface TenantConfig {
  // ... existing fields
  cutoffHour?: number | null;     // NEW
  cutoffMinute?: number | null;   // NEW
}
```

### 2. `FoodOrderFlowFacade` — checkout form default date

Текущее поведение: `deliveryDate` default = today.

Новое поведение: если `cutoffHour` и `cutoffMinute` заданы, и текущее время (в timezone клиента) ≥ cutoff → default = tomorrow.

Проблема: фронтенд не знает tenant timezone. Бэкенд знает и уже вычисляет `earliestDeliveryDate`. Два варианта:

**Вариант A**: Добавить `earliestDeliveryDate` в `TenantConfigResponse`. Бэкенд вычисляет, фронтенд использует как default и min для date picker.

**Вариант B**: Фронтенд сам вычисляет из cutoffHour/cutoffMinute. Некорректно — фронтенд не знает tenant timezone, только системное время клиента.

**Рекомендация: вариант A.** Одно поле, zero logic на фронтенде:

```java
// In TenantConfigResponse:
LocalDate earliestDeliveryDate  // nullable — null means "today"
```

```typescript
// In TenantConfig:
earliestDeliveryDate?: string | null;  // ISO date
```

Фронтенд:
- Default для `deliveryDate` в форме = `earliestDeliveryDate ?? today`
- `min` attribute на date input = same value
- Hint под полем: если cutoff задан и `earliestDeliveryDate` > today → показать "Orders for today are closed. Earliest delivery: {date}"

### 3. `FoodOrderCheckoutComponent`

Date input:

```html
<input type="date" 
       [min]="earliestDeliveryDate" 
       formControlName="deliveryDate">
<small *ngIf="isCutoffActive">
  Orders for today are closed. Earliest delivery: {{earliestDeliveryDate}}
</small>
```

`isCutoffActive`: computed from `earliestDeliveryDate > today` (local date comparison).

### 4. Валидация на submit

Frontend-side: проверить `deliveryDate >= earliestDeliveryDate`. Показать ошибку если нет.

Backend-side: уже работает — `BookingService.validateDeliveryDate` бросает `400 Delivery date must be on or after ...`.

---

## Полный список изменений

### Backend

| Файл / Класс | Изменение |
|---|---|
| `CreateTenantRequest` | + `cutoffHour`, `cutoffMinute` |
| `UpdateTenantRequest` | + `cutoffHour`, `cutoffMinute` |
| `TenantManagementService` | Валидация + write `cutoff_hour`/`cutoff_minute` в tenant_config |
| `SuperAdminTenantForm` | + `cutoffHour`, `cutoffMinute` |
| `SuperAdminPanelController` | Маппинг form ↔ request, read from config on edit |
| `tenant-form.html` (superadmin) | Два input поля |
| `TenantConfigResponse` | + `cutoffHour`, `cutoffMinute`, `earliestDeliveryDate` |
| `TenantConfigService` | Маппинг delivery settings + вычисление earliestDeliveryDate |

### Frontend

| Файл / Класс | Изменение |
|---|---|
| `TenantConfig` interface | + `cutoffHour`, `cutoffMinute`, `earliestDeliveryDate` |
| `FoodOrderFlowFacade` | Default deliveryDate = earliestDeliveryDate ?? today |
| `FoodOrderCheckoutComponent` | min на date input, hint когда cutoff активен |

---

## Тестирование

### Backend

| Тест | Что проверять |
|---|---|
| `SuperAdminTenantControllerIT` | Create/update с cutoff → читается обратно в detail. Partial (hour без minute) → 400. Очистка (null/null) → cutoff убран. |
| `ServiceManagementAndValidationIT` | Уже тестирует delivery date validation с timezone. Добавить: create tenant с cutoff → booking с today после cutoff → 400. |
| `TenantManagementService` unit | Валидация cross-field: one null other not → error. Range: hour=24 → error. |

### Frontend

| Тест | Что проверять |
|---|---|
| `food-order-flow.facade.spec.ts` | Default deliveryDate uses earliestDeliveryDate from config when present |
| `food-order-checkout.component.spec.ts` | min attribute on date input, cutoff hint visibility |

---

## Не в scope

- Admin panel self-service cutoff editing (superadmin-only for now)
- Per-day cutoff override (e.g., "no orders on Sunday")
- Multiple delivery slots / time windows
- Cutoff per-service (different cutoff for different items)
