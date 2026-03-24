# Tech Spec: Image Upload & CDN

## Scope

1. Загрузка изображений для карточки товара (одно фото на `CatalogService`)
2. Загрузка QR-кода оплаты через админку (замена ручного ввода URL в `payment_qr_url`)
3. CDN-раздача всех изображений

---

## Storage: Cloudflare R2

Почему R2, а не S3: нулевой egress, S3-совместимый API, встроенный CDN через Custom Domain, бесплатный tier (10 GB / 10M запросов) покрывает стадию валидации с 2-3 тенантами на годы вперёд.

### Bucket structure

**Один бакет**: `yoobu-media`

```
yoobu-media/
  {tenantId}/
    services/
      {serviceId}-{timestamp}.{ext}     ← фото товара
    payment/
      qr-{timestamp}.{ext}              ← QR оплаты
```

`{timestamp}` — Unix millis на момент загрузки. Обеспечивает уникальность + cache bust при замене.

### Пример ключей

```
42/services/17-1719312000000.webp
42/payment/qr-1719312000000.png
```

---

## Изоляция тенантов: почему это не проблема

Картинки товаров и QR — это публичный контент. Клиент видит их в меню без авторизации (`GET /t/{slug}/services` — `permitAll`). QR показывается на экране оплаты — тоже публично.

Если условный тенант-2 узнает URL фото товара тенанта-1 — он увидит фото бургера. Это не чувствительные данные. Это каталожные фото, которые и так открыты через API.

**Вывод**: public-read на бакет через CDN. Без signed URLs, без per-tenant bucket, без ACL. Усложнение здесь — чистое over-engineering без threat model.

Единственное ограничение: **загрузка** (write) должна проходить через бэкенд с проверкой `tenantId`, чтобы тенант-2 не мог записать файл в путь тенанта-1.

---

## CDN

R2 Custom Domain: `media.yoobu.app` (или аналог), привязан к бакету.

Итоговый URL изображения:

```
https://media.yoobu.app/42/services/17-1719312000000.webp
```

Этот URL хранится в БД и отдаётся фронтенду as-is. Кэширование — на уровне Cloudflare (Cache-Control: public, max-age=31536000, immutable). При замене изображения timestamp в ключе меняется → новый URL → автоматический cache bust.

---

## Database Changes

### Migration V9: `service.image_url`

```sql
-- V9__service_image_url.sql
ALTER TABLE service ADD COLUMN image_url TEXT;
```

Nullable. Старые товары отображаются без картинки (фронтенд уже должен это обрабатывать).

`payment_qr_url` в `tenant_config` — без изменений. Оно уже текстовое поле с URL. Загрузка просто пишет CDN URL вместо внешнего.

---

## Backend

### Зависимости

```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
</dependency>
```

R2 — S3-совместимый. Используем AWS SDK v2 с endpoint override.

### Конфигурация

```yaml
app:
  media:
    r2-endpoint: ${R2_ENDPOINT}           # https://<account_id>.r2.cloudflarestorage.com
    r2-access-key: ${R2_ACCESS_KEY}
    r2-secret-key: ${R2_SECRET_KEY}
    r2-bucket: ${R2_BUCKET:yoobu-media}
    cdn-base-url: ${CDN_BASE_URL}         # https://media.yoobu.app
```

### `MediaStorageService`

```java
public class MediaStorageService {

    String uploadServiceImage(Long tenantId, Long serviceId, MultipartFile file);
    String uploadPaymentQr(Long tenantId, MultipartFile file);
    void deleteByKey(String objectKey);
}
```

`uploadServiceImage`:
1. Валидация: content type in (`image/jpeg`, `image/png`, `image/webp`), размер ≤ 2 MB
2. Генерация ключа: `{tenantId}/services/{serviceId}-{System.currentTimeMillis()}.{ext}`
3. Загрузка в R2 через S3 PutObject с `Content-Type` и `Cache-Control: public, max-age=31536000, immutable`
4. Возврат полного CDN URL: `{cdn-base-url}/{key}`
5. Удаление старого объекта, если `service.imageUrl` был не null (извлечь ключ из URL)

`uploadPaymentQr` — аналогично, ключ `{tenantId}/payment/qr-{timestamp}.{ext}`.

### Endpoint: загрузка фото товара

```
POST /admin/{slug}/services/{serviceId}/image
Content-Type: multipart/form-data
Part: file (binary)
Auth: HTTP Basic tenant
Response: 200 + ServiceResponse (с обновлённым imageUrl)
```

Контроллер: `AdminCatalogController.uploadServiceImage`

Логика:
1. `TenantContext` → tenantId
2. `AdminCatalogService.getAdminService(serviceId)` — проверка существования + tenant scope
3. `MediaStorageService.uploadServiceImage(tenantId, serviceId, file)`
4. Обновить `CatalogService.imageUrl`, сохранить
5. Audit log: `logUpdate(entity="service", ...)`
6. Вернуть `ServiceResponse`

### Endpoint: загрузка QR через REST

```
POST /admin/{slug}/payment-qr
Content-Type: multipart/form-data
Part: file (binary)
Auth: HTTP Basic tenant
Response: 200 + { "paymentQrUrl": "https://media.yoobu.app/..." }
```

Логика:
1. `MediaStorageService.uploadPaymentQr(tenantId, file)`
2. Обновить `tenant_config` key `payment_qr_url` с новым CDN URL
3. Вернуть URL

### Endpoint: загрузка QR через superadmin

Нет нового endpoint. Superadmin tenant create/update уже принимают `paymentQrUrl` как строку. Superadmin может вставить CDN URL после загрузки через admin endpoint, или можно добавить аналогичный upload в superadmin panel form позже. Не блокер.

### Panel (Thymeleaf): загрузка в форме товара

На странице `admin/panel/service-form`:
- Добавить `<input type="file" name="imageFile">` под полем name/description
- Показать превью текущего `imageUrl`, если есть
- `AdminPanelServiceController.createService` / `updateService` — если `imageFile` присутствует и не пуст, вызвать `MediaStorageService.uploadServiceImage` после сохранения сервиса

На странице `admin/panel/tenant-detail` или отдельной форме:
- Аналогичный file input для QR

---

## Frontend

### `ServiceItem` — новое поле

```typescript
export interface ServiceItem {
  // ... existing fields
  imageUrl: string | null;    // ← NEW
}
```

### `ServiceResponse` (backend) — новое поле

```java
// record field:
String imageUrl
```

### `FoodOrderMenuComponent`

В карточке товара: если `service.imageUrl` не null — отрендерить `<img>` с CDN URL. Иначе — показать плейсхолдер (цветной прямоугольник с первой буквой, или иконку).

`loading="lazy"` на все img. Aspect ratio фиксирован (например 4:3 или 1:1). Object-fit: cover.

### `FoodOrderSuccessCardComponent` / `FoodOrderBookingsComponent`

QR: уже рендерится из `paymentQrUrl`. Никаких изменений. CDN URL — это просто другой URL.

---

## Ограничения и валидация

| Параметр | Значение |
|---|---|
| Допустимые типы | `image/jpeg`, `image/png`, `image/webp` |
| Макс. размер файла | 2 MB |
| Макс. одна картинка на товар | Замена: загрузка новой удаляет старую |
| Макс. один QR на тенанта | Замена: загрузка нового удаляет старый |

Серверная валидация — на уровне `MediaStorageService` до отправки в R2. Content-Type проверять по magic bytes, не доверять заголовку.

---

## Миграция существующих `paymentQrUrl`

Существующие значения `payment_qr_url` в `tenant_config` — это внешние URL (введённые вручную). Они продолжат работать. Фронтенд рендерит `<img src="...">` — ему без разницы, CDN это или внешний URL. Миграции данных не нужно.

---

## Порядок реализации

1. **V9 миграция** — добавить `service.image_url`
2. **`MediaStorageService`** — upload/delete с R2
3. **REST endpoint** — `POST /admin/{slug}/services/{serviceId}/image`
4. **Backend `ServiceResponse`** — добавить `imageUrl`
5. **Frontend `ServiceItem`** — добавить `imageUrl`, рендерить в меню
6. **Panel form** — file input в форме товара
7. **QR upload endpoint** — `POST /admin/{slug}/payment-qr`
8. **Panel QR upload** — file input в tenant detail

Шаги 1-5 — MVP. Шаги 6-8 — convenience.

---

## Не в scope

- Ресайз/кроп на бэкенде (клиент загружает готовое фото, потом)
- Множественные фото на товар (потом, если нужно)
- Drag-and-drop в panel (потом)
- Watermark / tenant branding на фото
- Image optimization pipeline (Sharp/libvips) — потом, когда будет нагрузка
