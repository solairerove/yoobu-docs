# TASK: Fix Telegram initData loss on page reload in Mini App

## Context

Project: `yoobu-web` — Angular 19+ standalone SPA, Telegram Mini App.

File to modify (single file, no others):
`src/app/core/telegram/telegram.service.ts`

## Problem

When the user reloads the page inside an already-open Telegram Mini App, `window.Telegram.WebApp.initData` becomes an empty string. The Telegram client injects `initData` only once — at the moment of Mini App launch. On reload the SDK script re-executes but the native client does not re-inject the value.

The HTTP interceptor (`telegram-init-data.interceptor.ts`) polls `telegram.getInitData()` for up to 1500ms. The poll resolves immediately because the value is not null/undefined — it is an empty string. The result: no `X-Telegram-Init-Data` header is attached, backend returns `401`.

## Required Change

In `TelegramService`, modify `getInitData()` to cache a successfully obtained `initData` value into `sessionStorage` on first read, and fall back to that cached value on subsequent calls when the live value is absent or empty.

### Exact logic

```
getInitData():
  1. Try to get live value:
     - webApp?.initData (non-empty string)
     - OR getLaunchParam('tgWebAppData') (non-empty string)
  2. If live value found:
     - write it to sessionStorage under key 'tg_init_data_cache'
     - return it
  3. If live value is absent or empty:
     - read sessionStorage.getItem('tg_init_data_cache')
     - return it (may be null if never cached)
```

### Session key constant

Declare as a private readonly field on the class:
```typescript
private readonly INIT_DATA_SESSION_KEY = 'tg_init_data_cache';
```

## Constraints

- Modify **only** `getInitData()` and add the private constant. No other methods, no other files.
- Do not change the method signature — it must remain `getInitData(): string | null`.
- Do not add any new imports.
- `sessionStorage` is available as a global — use it directly without injection.
- Non-empty string check: treat empty string `''` as absent (same as `null`/`undefined`). Use `|| null` idiom or explicit check — your choice, but be consistent.

## What NOT to change

- `telegram-init-data.interceptor.ts` — no changes needed, it already calls `getInitData()`
- `app.config.ts`
- Any component or facade
- Any backend file

## Acceptance criteria

1. First load inside Telegram: `initData` is non-empty → cached in `sessionStorage` → header attached → `200` from backend.
2. User reloads page: `initData` is empty → `getInitData()` returns cached value from `sessionStorage` → header attached → `200` from backend.
3. Outside Telegram (localhost dev): `initData` is empty, `sessionStorage` has no cached value → `getInitData()` returns `null` → interceptor falls back to `X-Telegram-User-Id` dev header. Existing dev flow unaffected.
4. New tab (different session): `sessionStorage` is isolated per tab → no cross-tab leakage.
