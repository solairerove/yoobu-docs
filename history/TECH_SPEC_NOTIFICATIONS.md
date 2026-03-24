# Tech Spec: Telegram Notifications

## Scope

1. Уведомления клиенту при смене статуса заказа
2. Уведомления админу (owner) при создании нового заказа
3. Уведомления админу при подтверждении оплаты клиентом

---

## Telegram Bot API: почему отдельный бот не нужен

Бот, в котором открывается Mini App, — это и есть бот для отправки сообщений. `tenant.bot_token` уже в БД. Telegram Bot API endpoint: `POST https://api.telegram.org/bot{token}/sendMessage`.

Единственное ограничение Telegram: бот может писать пользователю, только если тот инициировал диалог (нажал Start / отправил сообщение). При открытии Mini App через бот-меню это условие выполняется автоматически — Telegram требует Start перед запуском Mini App.

---

## Какие события → какие уведомления

### Клиенту (по `booking.telegram_user_id`)

| Событие | Триггер в коде | Сообщение |
|---|---|---|
| Заказ подтверждён | `updateBookingStatus` → `CONFIRMED` | ✅ Order #{id} confirmed. Delivery on {date}. |
| Заказ в доставке | `updateBookingStatus` → `DELIVERING` | 🚗 Order #{id} is on its way! {trackingUrl, если есть} |
| Заказ выполнен | `updateBookingStatus` → `DONE` | ✅ Order #{id} delivered. Thank you! |
| Заказ отменён (админом) | `updateBookingStatus` → `CANCELLED` | ❌ Order #{id} has been cancelled. |

Не уведомляем при: `NEW` (клиент сам создал), `PAYMENT_PENDING` (клиент сам подтвердил оплату).

### Админу (по `tenant.owner_telegram_id`)

| Событие | Триггер в коде | Сообщение |
|---|---|---|
| Новый заказ | `createFoodOrder` | 🆕 New order #{id} from {customerName}. Total: {total} {currency}. Delivery: {date}. |
| Клиент подтвердил оплату | `confirmMyBookingPayment` | 💰 Payment confirmed for order #{id} by {customerName}. |

Если `owner_telegram_id` is null — не отправляем, без ошибки.

---

## Архитектура

### Синхронно vs асинхронно

Асинхронно. Отправка уведомления не должна блокировать API response и не должна ронять транзакцию при ошибке Telegram API.

Простейший вариант без message broker: `@Async` + `@EventListener` в Spring.

```
BookingService.updateBookingStatus()
  → save booking
  → audit log
  → publish BookingStatusChangedEvent (Spring ApplicationEventPublisher)
  → return response

NotificationListener.onStatusChanged(event)  ← @Async @EventListener
  → resolve message text
  → TelegramBotApiClient.sendMessage(botToken, chatId, text)
  → log result (success/failure)
  → swallow exceptions (never propagate to caller)
```

Для 2-3 тенантов этот подход достаточен. Message broker (RabbitMQ, Redis streams) — потом, если появится нагрузка или потребуется retry with persistence.

### Retry

Telegram Bot API может вернуть 429 (rate limit) или 5xx. Простой retry: 3 попытки с exponential backoff (1s, 3s, 9s). После третьей неудачи — логировать и отпустить. Недоставленное уведомление о статусе — неприятно, но не критично. Клиент видит статус в Mini App при следующем открытии.

---

## Компоненты

### `TelegramBotApiClient`

HTTP-клиент для Telegram Bot API. Один метод, минимальный контракт.

```java
@Component
public class TelegramBotApiClient {

    /**
     * Sends a text message via Telegram Bot API.
     * Returns true on success, false on failure (logged internally).
     * Never throws.
     */
    boolean sendMessage(String botToken, Long chatId, String text, ParseMode parseMode);
}
```

Реализация: `RestClient` (Spring 6.1+) или `WebClient`. Endpoint: `https://api.telegram.org/bot{botToken}/sendMessage`. Body: `{ "chat_id": chatId, "text": text, "parse_mode": "HTML" }`.

`ParseMode`: `HTML` — проще для форматирования (bold, links) чем Markdown v2 с его escaping.

Логирование: на success — debug; на failure — warn с response body (содержит `description` от Telegram с причиной). Не логировать `botToken` — sensitive.

### `NotificationService`

Формирует тексты сообщений и определяет получателя.

```java
@Component
public class NotificationService {

    void notifyCustomerStatusChanged(Booking booking, BookingStatus newStatus, Tenant tenant);
    void notifyAdminNewOrder(Booking booking, Tenant tenant);
    void notifyAdminPaymentConfirmed(Booking booking, Tenant tenant);
}
```

Внутри каждого метода:
1. Проверить наличие `botToken` (у тенанта). Если null — return, без ошибки.
2. Для admin-уведомлений: проверить `owner_telegram_id`. Если null — return.
3. Сформировать текст (шаблон + данные из booking).
4. Вызвать `TelegramBotApiClient.sendMessage`.

### `BookingNotificationListener`

```java
@Component
public class BookingNotificationListener {

    @Async
    @EventListener
    void onBookingCreated(BookingCreatedEvent event) {
        notificationService.notifyAdminNewOrder(event.booking(), event.tenant());
    }

    @Async
    @EventListener
    void onBookingStatusChanged(BookingStatusChangedEvent event) {
        notificationService.notifyCustomerStatusChanged(
            event.booking(), event.newStatus(), event.tenant());
    }

    @Async
    @EventListener
    void onPaymentConfirmed(PaymentConfirmedEvent event) {
        notificationService.notifyAdminPaymentConfirmed(event.booking(), event.tenant());
    }
}
```

### Events

```java
public record BookingCreatedEvent(Booking booking, Tenant tenant) {}
public record BookingStatusChangedEvent(Booking booking, BookingStatus oldStatus, BookingStatus newStatus, Tenant tenant) {}
public record PaymentConfirmedEvent(Booking booking, Tenant tenant) {}
```

Публикуются через `ApplicationEventPublisher` в `BookingService`:
- `createFoodOrder` → `BookingCreatedEvent`
- `updateBookingStatus` → `BookingStatusChangedEvent`
- `confirmMyBookingPayment` → `PaymentConfirmedEvent`

---

## Шаблоны сообщений

HTML parse mode. Минимальное форматирование — Telegram не email.

### Customer: status changed

```
CONFIRMED:
✅ <b>Order #{id} confirmed</b>
Delivery: {deliveryDate}

DELIVERING:
🚗 <b>Order #{id} is on the way</b>
{trackingUrl ? "Track: " + trackingUrl : ""}

DONE:
✅ <b>Order #{id} delivered</b>
Thank you!

CANCELLED:
❌ <b>Order #{id} cancelled</b>
```

### Admin: new order

```
🆕 <b>New order #{id}</b>
Customer: {customerName}
Phone: {customerPhone}
Total: {totalPrice} {currency}
Delivery: {deliveryDate}
{note ? "Note: " + note : ""}
Items:
{items.map(i -> "• " + i.serviceName + " × " + i.quantity).join("\n")}
```

### Admin: payment confirmed

```
💰 <b>Payment confirmed</b>
Order #{id}
Customer: {customerName}
Total: {totalPrice} {currency}
```

---

## Конфигурация

### `@Async` setup

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("notificationExecutor")
    public Executor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(4);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("tg-notify-");
        executor.initialize();
        return executor;
    }
}
```

`@Async("notificationExecutor")` на listener-методах.

### Telegram Bot API rate limits

Per-bot: ~30 сообщений/сек общий, ~1 сообщение/сек per chat. Для 2-3 тенантов с десятками заказов в день — не проблема. Retry на 429 с `retry_after` из response body — достаточно.

---

## Изменения в существующем коде

### `BookingService`

```java
// Inject:
private final ApplicationEventPublisher eventPublisher;

// In createFoodOrder, after save + audit:
eventPublisher.publishEvent(new BookingCreatedEvent(booking, tenant));

// In updateBookingStatus, after save + audit:
eventPublisher.publishEvent(new BookingStatusChangedEvent(booking, oldStatus, newStatus, tenant));

// In confirmMyBookingPayment, after save + audit:
eventPublisher.publishEvent(new PaymentConfirmedEvent(booking, tenant));
```

Events публикуются ПОСЛЕ commit-а? Нет — Spring `@TransactionalEventListener(phase = AFTER_COMMIT)` гарантирует доставку события только после успешного коммита. Использовать `@TransactionalEventListener` вместо `@EventListener`:

```java
@Async("notificationExecutor")
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
void onBookingCreated(BookingCreatedEvent event) { ... }
```

Это предотвращает отправку уведомления, если транзакция откатилась.

### `Tenant` entity

`owner_telegram_id` — уже есть в entity и schema. Уже nullable. Записывается через superadmin create/update. Никаких миграций.

### `BookingService.cancelMyBooking`

Клиент сам отменяет свой заказ — не уведомлять клиента (он и так знает). Но уведомить админа? На этом этапе — нет. Админ видит в панели. Если реальные пользователи попросят — добавим.

---

## Database changes

Нет. Все нужные поля уже в schema:
- `tenant.bot_token` — токен бота для отправки
- `tenant.owner_telegram_id` — chat_id админа
- `booking.telegram_user_id` — chat_id клиента

---

## Тестирование

### Unit: `NotificationService`

- Формирование текстов для каждого статуса
- Null botToken → no-op
- Null ownerTelegramId → no-op для admin-уведомлений

### Unit: `TelegramBotApiClient`

- Success response → true
- 429 → retry → success
- 3x failure → false, logged

### Integration

- `BookingLifecycleIT` — проверить что events публикуются (mock `ApplicationEventPublisher` или verify через `@SpyBean`)
- Не тестировать реальный Telegram API в CI — mock `TelegramBotApiClient`

---

## Локализация

Сейчас — English only. Шаблоны захардкожены в `NotificationService`. Когда (если) появятся вьетнамские, русские тенанты — вынести в tenant_config (`notification_language`) и i18n resource bundles. Не сейчас.

---

## Порядок реализации

1. `TelegramBotApiClient` — HTTP клиент + retry
2. Events (3 record-а)
3. `BookingService` — publish events
4. `AsyncConfig`
5. `NotificationService` — шаблоны + отправка
6. `BookingNotificationListener` — wiring
7. Тесты

---

## Не в scope

- Inline keyboards в сообщениях (кнопки "Open order" с deep link в Mini App) — потом
- Уведомление при отмене клиентом → админу — потом, если попросят
- Webhook от Telegram (incoming messages / commands) — не нужно сейчас
- Уведомление при создании заказа → клиенту (он только что создал, он знает)
- Кастомизация шаблонов через tenant_config — потом
- Batching (дайджест заказов) — потом
