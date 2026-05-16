
---

1. audit — стартуй отсюда

7 файлов, 1 таблица, 0 зависимостей от других доменов.

Таблица идеальная:
audit_log(id, tenant_id, entity, entity_id, action, actor_id, old_value, new_value, created_at)
Нет FK-constraints — tenant_id просто BIGINT. Append-only. Три эндпоинта:
- POST /logs — записать событие
- GET /logs — фильтрация
- GET /logs/export — CSV

На Rust: axum + sqlx. 2-3 дня работы. Java-монолит переключаешь на HTTP-клиент вместо локального бина.

---

2. catalog — следующий шаг

10 файлов, 1 таблица (service), FK только на tenant_id.

Два REST-контроллера (публичный read-only + admin CRUD). Зависит от tenant и audit (которые к тому моменту уже будут сервисами или просто tenant_id передаётся параметром).

---

3. notification — почти бесплатно

6 файлов, 0 таблиц. Просто шлёт сообщения в Telegram через бот. Можно сделать отдельный сервис который слушает вебхук или очередь от booking.

---
Что НЕ трогать первым

- booking — 6 зависимостей, optimistic locking, event publishing
- tenant — фундамент, всё на нём держится
- admin — Thymeleaf UI, не сервис