# Backlog

This document tracks product and engineering ideas to consider outside of Jira.

## Ideas

1. [x] API to check whether a suggested or requested tenant is unique before creation.
2. [x] As a super admin, I want to be able to check the services and catalogs of a tenant admin.
3. [ ] ~~rename service as catalog~~
4. [x] {{host}}/admin/dark-kitchen-dn/bookings?status=NEW&deliveryDate=2026-03-14 doesn't work
5. [x] SecurityConfig — сделать до админки, не после. Thymeleaf-контроллеры для /admin/{slug}/** сразу должны жить за правильным filter chain. Если SecurityConfig ещё не написан — это первое, что делается. Два chain: один для /admin/{slug}/** (Basic Auth против tenant_config), второй для /superadmin/** (Basic Auth против env vars). Клиентские /t/{slug}/** остаются открытыми (auth через Telegram header позже). Писать админку без security config — потом переделывать всю маршрутизацию.
6. [x] Superadmin read-only view на данные тенантов — это не "потом решу", это операционная необходимость с первого живого тенанта. Не нужна полная админка — достаточно GET /superadmin/tenants/{id}/bookings и GET /superadmin/tenants/{id}/services. Пять строк в контроллере. Без этого первый баг-репорт от тенанта = лезть в psql.
7. [x] Thymeleaf админку делай уродливой. Без CSS-фреймворков, без попыток "потом переиспользовать шаблоны". Это throwaway-код. Чем меньше в него вложено, тем легче выкинуть. Один layout, таблицы, формы, всё.
8. [x] Даты и таймзоны — закрыть огрехи до Telegram, не после. delivery_date это DATE без таймзоны. Cutoff-логика (cutoff_hour/cutoff_minute) привязана к tenant.timezone. Если сейчас где-то используется системное время сервера вместо tenant timezone — это баг, который проявится в проде мгновенно (сервер в Railway может быть в UTC, тенант в Asia/Ho_Chi_Minh, +7 часов разницы). Убедись что ZonedDateTime.now(tenant.getTimezone()) используется везде, где есть cutoff или "сегодня/завтра".
9. [x] Порядок Telegram-интеграции: сначала бекенд (initData validation + notifications), потом Angular Mini App. Это позволит тестировать уведомления через curl + admin panel ещё до того, как фронт существует. Notifications можно проверить руками: создать заказ через API, увидеть сообщение в Telegram.
10. [x] Superadmin логинится в любую /admin/{slug}/ — это не фича, это операционная необходимость. Тенант пишет "у меня заказ странный", ты открываешь его админку и видишь то же, что видит он. В SecurityConfig это просто: filter chain для /admin/{slug}/** принимает либо tenant credentials из tenant_config, либо superadmin credentials из env vars. Один AuthenticationProvider проверяет оба источника. Не два filter chain на один path — один chain, два provider в цепочке.
11. [x] Superadmin panel на /superadmin/ — отдельный filter chain, отдельные Thymeleaf views. Тут живёт то, что тенанту не показывают: список тенантов, создание/деактивация, конфиг-редактор, read-only просмотр данных тенанта (bookings, services). Это не пересекается с tenant admin UI ни по маршрутам, ни по шаблонам.
12. [x] tenant crud by superadmin
13. [x] audits
14. [x] show tenants with active not active statuses on superadmin page
15. [x] Add search/sort/pagination for bookings, services, and tenants lists. Right now they render full tables only, so operations degrade as data grows: bookings.html, services.html, tenants.html.
16. [ ] aop for auditing
17. [x] superadmin should be able to activate/deactivate services?
18. [x] on admin panel there should be a confirmation dialog for deleting a service
19. [x] on admin panel for booking and service statuses i think we should able to update status right here, rather then on detailed form.
20. [x] phone place holder `+84...` as part of the config, Note `no onion, gate code, delivery code` as well (ui left)
21. [x] Preserve customer form values between orders instead of resetting name/phone every time.
22. [x] Add Telegram-native confirm/alert UX for cancel and submit failures.
23. [x] Add booking status emphasis and maybe a compact timeline/receipt view.
24. [x] Then, if needed, split checkout and bookings into smaller components so this screen stops growing into a monolith.
25. [ ] UI reorder booking
26. [ ] UI recent order status is not updated
27. [ ] UI if phone or name are not filled, show a warning
28. [ ] ~~new angular admin panel [doc](https://github.com/solairerove/yoobu-docs/blob/master/admin-panel-rnd.md)~~
29. [x] UI Motion polish Subtle transitions for card selection, cart-bar updates, tab switching, and checkout opening. Small enough to stay Telegram-friendly, but enough to make the UI feel deliberate. 
30. [x] Quantity control refinement The + / - controls work, but they still look a bit generic. Better pressed/active states and slightly tighter visual balance would improve the whole menu. 
31. [x] Cleaner menu copy There are still a few lines that read like product-demo text rather than a real app. Tightening those would make the UI feel more production-ready. 
32. [x] UI Theme consistency pass Some surfaces still mix similar whites and borders. A pass to unify surface contrast, shadows, and accent usage would make the whole app feel more cohesive.
33. [x] booking status can be changed from done to new
34. [x] List pages need workflow scaling improvements. bookings.html, services.html, and tenants.html have no search/pagination and show raw timestamps, so usability will degrade as data grows.
35. [ ] ADMIN ~~Mobile table UX is hard to scan. admin-panel.css hides table headers and converts cells to blocks, but rows don’t add per-cell labels, so column meaning is lost on phone.~~
36. [x] sensitive data is exposed, bot token
37. [x] ADMIN Format timestamps and monetary values for readability (local timezone, stable currency format) instead of raw values in tables/detail pages.
38. [x] ADMIN Add flash feedback for service/tenant create/edit/delete flows too (we only added this for booking status), so operators always get explicit success/error outcomes.
39. [x] ADMIN Add stronger destructive-action guards for service delete (for example typed confirmation), not only browser confirm() in service-form.html.
40. [x] Add business validation for booking status consistency in public APIs too (for example clear behavior if admin tries to modify cancelled/completed bookings via API race conditions).
41. [x] ADMIN Add an admin-facing audit log screen (you already log everything), so superadmin can inspect who changed status/config and when.
42. [ ] pagination/sorting but for API REST admin endpoints too.
43. [ ] pagination/sorting but for API REST public endpoints too.
44. [x] ADMIN Нормальные поля времени в фильтре (не raw ISO text), чтобы не ошибаться руками. 
45. [x] ADMIN Локализация/человекочитаемые лейблы для entity/action (booking -> Заказы, UPDATE_STATUS -> Изменение статуса). 
46. [x] ADMIN Аккуратный diff-view: показывать только измененные ключи вместо обрезанного JSON. 
47. [x] ADMIN Базовый “safety cap”: ограничить size например до 50 в UI. 
48. [x] ADMIN Экспорт текущей выборки в CSV (очень полезно для разборов инцидентов). 
49. [x] ADMIN Индекс под новые фильтры, если аудит быстро растет (created_at, action, возможно составной).
50. [x] UI вынести еще пару общих UI-стилей (.eyebrow, .ghost-button) в общие стили, чтобы уменьшить дублирование между компонентами.
51. [x] UI Route + tenant bootstrap tests for tenant-shell.component.ts: slug changes, config load error fallback, and applyTheme side effects.
52. [x] UI HTTP auth-header tests for telegram-init-data.interceptor.ts: init-data header vs localhost dev-user header vs no header.
53. [x] UI Telegram service behavior tests for telegram.service.ts: fallback paths (alert/confirm), main-button enable/disable, handler attach/detach cleanup.
54. [x] UI UI integration tests for food-order-home.component.ts: menu/orders switch, cart bar visibility, checkout open/close wiring.
55. [x] Presentational component interaction tests for food-order-bookings.component.ts and food-order-checkout.component.ts: emitted outputs and disabled/loading states.
56. [x] Currency as part of the tenant config and it should be visible in the admin panel and in the public API for ui.
57. [ ] geolocation for delivery address.
58. [ ] изменение валюты работает, но куренси должна стать частью ордер айтема. чтобы именение валюты не влияло на существующие и созданные товары
59. [x] убрать ид сервиса из каталога, потому что не дает никакой инфы
