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
14. Retry `RATE_LIMIT` and `BITRIX_UNAVAILABLE` with backoff.
15. Check scopes, endpoint semantics, and portal state before blaming `BITRIX_ERROR` on your code.
16. Do not assume every endpoint returns `success/data`; AI Router uses raw OpenAI-compatible responses.
