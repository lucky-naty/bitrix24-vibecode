# Anti-Footguns

Use this file as a short preflight checklist before implementation or debugging.

## Frequent Mistakes

1. Run `GET /v1/me` first when a real key is available.
2. Match the key type before designing the flow: `vibe_api_`, `vibe_app_`, `vibe_live_`.
3. For `vibe_app_` flows, check whether both `X-Api-Key` and bearer token are required.
4. Do not mix VibeCode platform APIs with native Bitrix24 APIs.
5. Default request fields to camelCase.
6. Keep `UF_CRM_*` and `ufCrm_*` in native form.
7. Do not assume every response comes back in camelCase.
8. Treat port `3000` as the app port inside the remote VM, not a local port.
9. Do not change the app port unless the task explicitly requires it.
10. Wait for both `status=running` and `blackholeStatus=CONNECTED`.
11. Use `stream=false` when deploy automation expects JSON instead of SSE.
12. Prefer batch, search, and aggregate over naive client loops.
13. Retry `RATE_LIMIT` and `BITRIX_UNAVAILABLE` with backoff.
14. Check scopes, endpoint semantics, and portal state before blaming `BITRIX_ERROR` on your code.
15. Read higher-level docs before jumping to API Reference.
