# Placement Flow (Bitrix24 iframe apps on BlackHole)

How a Bitrix24 placement app (LEFT_MENU, CRM_*_DETAIL_TAB, etc.) running on a BlackHole server authenticates each operator as themselves. Read this before building or debugging any placement/iframe app.

## The Runtime (authoritative, 2026-05+)

1. Operator clicks the placement in Bitrix24. Bitrix loads the app's **handler URL** inside an iframe and POSTs the placement context + Bitrix auth.
2. The handler URL **must be** `https://vibecode.bitrix24.tech/v1/bitrix-handler`. It exchanges the OAuth code, mints a `vibe_session_*`, and mints a short-lived `__init` JWT (5-min TTL, HMAC-SHA256, one-time `jti`).
3. Handler `302`-redirects the iframe to `<appUrl>/?placement=…&member_id=…[&placement_options=<json>]&__init=<jwt>`.
4. **Gateway redeems `__init`** on the first GET to the BlackHole subdomain: verifies JWT, checks subdomain binding, marks `jti` used, mints the `_vibe_gw` cookie (HttpOnly, ~10 min sliding), then redirects to a clean URL without `__init`.
5. On every subsequent request the Gateway strips any incoming `X-Vibe-Authorization` (anti-spoof) and **injects** the real headers:

| Header | Meaning |
|---|---|
| `X-Vibe-Request-Id` | correlation id `req_…` |
| `X-Vibe-User-Id` | numeric Bitrix24 user id |
| `X-Vibe-Portal-Id` | portal UUID |
| `X-Vibe-User-Name` | display name |
| `X-Vibe-User-Role` | `ADMIN` or `MEMBER` |
| `X-Vibe-Authorization` | `Bearer vibe_session_*` |

6. App reads identity: `GET /v1/me` with `X-Api-Key: <app key>` + `Authorization: Bearer <token stripped from X-Vibe-Authorization>`. Read **`data.currentUser.bitrixUserId`** (NOT `userId`/`user.id`). Cache per token ~24h.

The browser never sees `vibe_session_*`. Never put it in JS/sessionStorage/localStorage, never read `_vibe_gw` in JS, never return the token in JSON.

**Session across in-app navigation:** the injected `X-Vibe-Authorization` arrives on gateway-routed requests, but auth can appear "lost" on the app's own subsequent page loads/redirects. Proven pattern: on first authenticated hit, set your OWN signed session cookie (HMAC over the user/portal identity, base64url) with `Max-Age=3600; HttpOnly; Secure; SameSite=None` — `SameSite=None` is mandatory because the app lives in an iframe (`Lax`/`Strict` cookies are silently dropped there). On later requests: prefer the `X-Vibe-Authorization` header, fall back to the signed cookie. Sign the cookie, don't store the raw token in it.

**RETIRED — do not use:** the pre-2026-05 contract that delivered the session as `access_token` in a POST-body HTML auto-submit form. `X-Vibe-Authorization` injected by the Gateway is the ONLY source now.

**Reading placement context:** use `URLSearchParams` on `window.location.search` (`placement`, `placement_options` → `JSON.parse`). Do NOT call `BX24.placement.info()` — it returns nulls across the handler redirect.

## Setup Invariants (must all hold or operators can't log in)

Inspect with `GET /v1/apps/:id` and `GET /v1/infra/servers/:id`.

1. **`app.handlerUrl` = `https://vibecode.bitrix24.tech/v1/bitrix-handler`.** Set at app creation; verify it after any change.
2. **`app.appUrl` must point to the deployed BlackHole server URL** (e.g. `https://app-<id>.vibecode.bitrix24.tech`). If `appUrl` is `null`, the handler cannot resolve the app to its server → **`APP_RESOLVE_FAILED`**. Fix: `PATCH /v1/apps/:id { appUrl: "https://app-<id>.vibecode.bitrix24.tech" }`. This is the common break when the server was created under a `vibe_api_` key (owner identity) while the app lives under the `vibe_app_` key — the two are not auto-linked.
3. **Placement bound** via `POST /v1/placements/bind { placement, handler, title }` where `handler` = the bitrix-handler URL. `placement.bind` requires a **commercial** Bitrix tariff (free → `502 BITRIX_UNAVAILABLE`). To force-refresh: `unbind` then `bind`.
4. **`accessPolicy`** on the server admits the operators (see below).
5. Server healthy: `status=running` **and** `blackholeStatus=CONNECTED`.

## accessPolicy enum

`PATCH /v1/infra/servers/:id/access-policy { accessPolicy: <VALUE> }`. Accepted values (from API validation; docs omit them):

`OWNER_ONLY` | `NAMED_USERS` | `DEPARTMENT` | `PORTAL` | `AUTHENTICATED` | `PUBLIC`

- `OWNER_ONLY` — only the server owner.
- `NAMED_USERS` / `DEPARTMENT` — explicit allow-list.
- **`PORTAL`** — all users of the bound Bitrix24 portal. **This is the right choice for a placement app** that all operators of one portal should use.
- `AUTHENTICATED` — requires a VibeCode/Network account; ordinary Bitrix operators without one are rejected at the gate.
- `PUBLIC` — any visitor.

Security: loosening `accessPolicy` directly affects access. Never change it silently — confirm with the user first.

## Troubleshooting (symptom → cause → fix)

- **Browser shows BlackHole "Войти через Bitrix24 / Bitrix24 Network" gate, redirect to `auth2.bitrix24.net`** → the iframe reached `appUrl` WITHOUT `__init` (the placement is not routing through `bitrix-handler`). The visitor gate is the fallback for direct visitors. In an iframe its login page fails with **`ERR_BLOCKED_BY_RESPONSE`** because `auth2.bitrix24.net` sets frame-busting headers. Fix: ensure the placement handler = bitrix-handler (rebind), and `app.appUrl` is set, so `__init` is delivered and the gate never shows. The gate is NOT a network/proxy problem — proxy changes won't fix it.
- **`APP_RESOLVE_FAILED` "Не удалось определить приложение по application token"** at the handler → app has no resolvable server. Almost always `app.appUrl` is `null`. Fix per invariant 2. (Secondary cause: the Bitrix install token was never captured — platform-side; file feedback.)
- **`USER_AUTH_REQUIRED`** from `POST /v1/bitrix-handler` → no Bitrix auth in the POST. Expected when probing the handler manually; on a real placement click Bitrix supplies it.
- **`401 BH_LOGIN_REQUIRED`** ("This is a Black Hole app subdomain…") on a direct curl to `https://app-<id>.vibecode.bitrix24.tech` → this is the NORMAL gate for an unauthenticated direct hit, and it PROVES the server exists and is running. Do not read it as "server down" or "handler broken" — real placement/robot traffic arrives through the Gateway and bypasses it.
- **Operator opens app but acts as the wrong (shared/webhook) user** → you're using a `vibe_api_` key (single owner identity) for the action, or not forwarding the per-user bearer. Send user-context calls with `X-Api-Key` + the injected bearer.
- **Send/transfer/close fails with member/access error** → the operator isn't a member of that chat. Add them server-side (admin webhook `im.chat.user.add`) then retry.
- **App never flips to `INSTALLED`, or the `ONAPPINSTALL` callback never arrives** → the BlackHole server was **sleeping/hibernating** when Bitrix fired the direct install POST, so it was lost (unlike tunnelled event-subscriptions, a direct install callback does NOT auto-wake the server). Wake the server (`status=running` + `CONNECTED`), then click "Переустановить"/reinstall so the callback lands.

## Key-type reminder

- `vibe_api_` — single owner identity (everything acts as one user). Wrong for per-operator placement actions.
- `vibe_app_` — OAuth app; per-user identity via the gateway bearer. Required for placement apps where each operator acts as themselves. Needs `X-Api-Key` + `Authorization: Bearer <session>`.
