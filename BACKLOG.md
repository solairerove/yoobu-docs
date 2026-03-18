# Backlog

This document tracks product and engineering ideas to consider outside of Jira.

## Ideas

1. [x] API to check whether a suggested or requested tenant is unique before creation.
2. [x] As a super admin, I want to be able to check the services and catalogs of a tenant admin.
3. [x] ~~rename service as catalog~~
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
15. [ ] pagination on superadmin page?
16. [ ] aop for auditing
17. [x] superadmin should be able to activate/deactivate services?
18. [x] on admin panel there should be a confirmation dialog for deleting a service
19. [x] on admin panel for booking and service statuses i think we should able to update status right here, rather then on detailed form.
20. [ ] phone place holder `+84...` as part of the config, Note `no onion, gate code, delivery code` as well
21. [x] Preserve customer form values between orders instead of resetting name/phone every time.
22. [x] Add Telegram-native confirm/alert UX for cancel and submit failures.
23. [x] Add booking status emphasis and maybe a compact timeline/receipt view.
24. [ ] Then, if needed, split checkout and bookings into smaller components so this screen stops growing into a monolith.
25. [ ] reorder booking
26. [ ] recent order status is not updated
27. [ ] if phone or name are not filled, show a warning
28. [ ] new angular admin panel [doc](https://github.com/solairerove/yoobu-docs/blob/master/admin-panel-rnd.md)
29. [ ] Motion polish Subtle transitions for card selection, cart-bar updates, tab switching, and checkout opening. Small enough to stay Telegram-friendly, but enough to make the UI feel deliberate. 
30. [x] Quantity control refinement The + / - controls work, but they still look a bit generic. Better pressed/active states and slightly tighter visual balance would improve the whole menu. 
31. [ ] Cleaner menu copy There are still a few lines that read like product-demo text rather than a real app. Tightening those would make the UI feel more production-ready. 
32. [ ] Theme consistency pass Some surfaces still mix similar whites and borders. A pass to unify surface contrast, shadows, and accent usage would make the whole app feel more cohesive.
