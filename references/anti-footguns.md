# Anti-Footguns

Use this file as a short preflight checklist before implementation or debugging.

## Frequent Mistakes

1. Run `GET /v1/me` first when a real key is available.
2. Match the key type before designing the flow: `vibe_api_`, `vibe_app_`, `vibe_live_`.
3. For `vibe_app_` flows, check whether both `X-Api-Key` and bearer token are required.
4. For iframe or placement flow, read the session from the Gateway-injected `X-Vibe-Authorization: Bearer vibe_session_*` header. Do NOT read `access_token` from the POST body — that contract is retired. See [placement-flow.md](placement-flow.md).
4b. Placement setup invariants: `app.handlerUrl` = `https://vibecode.bitrix24.tech/v1/bitrix-handler` AND `app.appUrl` points to the deployed BlackHole server URL. `appUrl: null` causes `APP_RESOLVE_FAILED`. A direct iframe hit without `__init` shows the visitor gate (auth2.bitrix24.net), which fails in-iframe with `ERR_BLOCKED_BY_RESPONSE` — not a network/proxy issue.
4c. For a placement app meant for all operators of one portal, set the server `accessPolicy` to `PORTAL`, not `AUTHENTICATED` (the latter needs a VibeCode account). Enum: `OWNER_ONLY | NAMED_USERS | DEPARTMENT | PORTAL | AUTHENTICATED | PUBLIC`.
5. Do not mix VibeCode platform APIs with native Bitrix24 APIs.
6. Default request fields to camelCase.
7. Keep `UF_CRM_*` and `ufCrm_*` in native form.
8. Do not assume every response comes back in camelCase.
9. Treat port `3000` as the app port inside the remote VM, not a local port.
10. Do not change the app port unless the task explicitly requires it.
11. Wait for both `status=running` and `blackholeStatus=CONNECTED`.
12. Use `stream=false` when deploy automation expects JSON instead of SSE.
13. Prefer auto-pagination, batch, search, and aggregate over hand-rolled client loops.
14. Retry `429 RATE_LIMITED` (honor `retryAfter`), `502 BITRIX_UNAVAILABLE`, `504 QUEUE_TIMEOUT`, `ERROR_LOOP_DETECTED`, `500 INTERNAL_ERROR` with backoff. (`RATE_LIMIT` is not a real code — it is `RATE_LIMITED`.) For a WRITE, `GET`-check before retrying `BITRIX_UNAVAILABLE` — blind retry can duplicate.
15. Check scopes, endpoint semantics, and portal state before blaming `BITRIX_ERROR` on your code.
16. Do not assume every endpoint returns `success/data`; AI Router uses raw OpenAI-compatible responses.
17. `accessPolicy` is per-SERVER, not per-app. A Black Hole VM can co-host several apps, so `PUBLIC` exposes ALL of them to the internet. Confirm with the user before loosening it, and revert to `PORTAL` immediately after any temporary change.
18. A placement (e.g. `LEFT_MENU`) renders in the portal only AFTER the app is installed/published. If `placement.bind` returns `alreadyBound` but the menu item is missing, the app is not installed — not a binding bug.
19. "Приложение не найдено" / `APP_RESOLVE_FAILED` when opening a placement = the app↔server link is broken. Re-`PATCH /v1/apps/:id { appUrl }` to the server subdomain root (`https://app-<id>.vibecode.bitrix24.tech`) to force the relink. This breaks when the server and app were created under different keys.
20. `imconnector.*` (register/activate/send for Open Lines connectors) needs a **real Bitrix app OAuth token**. The VibeCode managed app (`handlerUrl = bitrix-handler`) cannot supply one → `401`, and there is no VibeCode wrapper for `imconnector`/`imopenlines` methods. Workaround: a **separate NATIVE local Bitrix app** (server type, scope `imopenlines,imconnector,im`) — do its OAuth `authorization_code` flow server-side (the `oauth/authorize` page is on the *portal* domain, so it dodges the BlackHole gate; only the final callback lands on the gated app domain and rides the user's gateway-session cookie), store the token, call `imconnector.*` against `https://<portal>/rest/`.
20b. **`imconnector.send.messages` runtime errors** (native, incoming push works today): `IMCONNECTOR_NOT_SPECIFIED_CORRECT_CONNECTOR` (bad `CONNECTOR` value), `NOT_ACTIVE_LINE` (line deleted/disabled — re-activate via `imconnector.activate { ACTIVE: 1 }`), `PROVIDER_UNSUPPORTED_TYPE_INCOMING_MESSAGE` (bad `type_message`), `IMCONNECTOR_NOT_ALL_THE_REQUIRED_DATA` (empty/invalid `user`), `IMCONNECTOR_NOT_SPECIFIED_CORRECT_COMMAND` (bad command). Bitrix caps the connector `name` at 25 chars.
21b. **Bizproc robot registration needs a `vibe_app_` OAuth key + `vibe_session`** — a `vibe_api_` key → `OAUTH_REQUIRED`. Robot registration WORKS on the FREE tariff (unlike `placement.bind`/`event.bind`). Default `documentType` is CRM; to fire on tasks pass `documentType: ["tasks","Bitrix\\Tasks\\Integration\\Bizproc\\Document\\Task","TASK"]`. See [bizproc-robot-flow.md](bizproc-robot-flow.md).
21c. **A subscription robot (`useSubscription:"Y"`) is NOT completed by an HTTP 200** — it hangs until you `POST /rest/bizproc.event.send { event_token, auth: <callback auth.access_token>, return_values }`. `event_token` is one-time, taken from each callback. HTTP 200 only acknowledges receipt.
21d. **`accessPolicy=PORTAL` is enough for robot callbacks** — real callbacks ride managed infra and pass the gate; `PUBLIC` is not needed. A direct external curl to the subdomain under `PORTAL` IS gated — do not conclude the handler is unreachable from that curl test.
22. **Connector outgoing events are a known gap (as of 2026-06 — verify before relying).** When an operator replies, Bitrix should fire `OnImConnectorMessageAdd`; today it does NOT reach the app. The VibeCode event-subscription tunnel only delivers events for connectors owned by the **managed** app, but `imconnector.register` (per #20) is done by a **local** app — so the connector's events go to neither (`recentDeliveries: []`). Binding `event.bind('OnImConnectorMessageAdd', <local-app handler>)` returns `true` and the handler is reachable, yet Bitrix still emits nothing. The vendor has confirmed connector-method + connector-event support is in development. Net: connector **incoming** (`imconnector.send.messages`, push from your app) works now; **outgoing** (operator→connector) is blocked upstream — don't burn time debugging your handler. It is NOT a tariff issue (demo exposes all commercial features) and NOT an install issue (`app.info INSTALLED:true`, connector `STATUS:true`).
