# Universal Knowledge

## Start Here

- Start with `GET /v1/me` when a real key is available. It reveals entities, scopes, and deployment capabilities for that key.
- Base API URL is `https://vibecode.bitrix24.tech/v1`.
- Primary auth header is `X-Api-Key: <key>`; AI Router also accepts `Authorization: Bearer <key>`.
- Most VibeCode APIs return `success` plus `data`; AI Router returns raw OpenAI-compatible responses.

## Keys And Access

- `vibe_api_` is the default choice for personal scripts, dashboards, and automations tied to one Bitrix24 user.
- `vibe_app_` is for team apps with Bitrix24 OAuth. These requests often need both `X-Api-Key` and `Authorization: Bearer <session token>`.
- For Bitrix24 iframe/placement flow, `vibe_session_*` is passed in POST body as `access_token`, not in the query string.
- `vibe_live_` is a management key for administration tasks, not the default for ordinary entity work.
- The key prefix tells you which auth model and runtime behavior to expect before reading the rest of the task.

## Entity API Invariants

- Most entities follow one REST pattern: `GET list`, `GET by id`, `POST create`, `PATCH update`, `DELETE delete`, `POST search`, `POST aggregate`, `GET fields`.
- In requests, use camelCase field names by default.
- In responses, field format depends on the underlying Bitrix24 API: many CRM entities come back in camelCase, while legacy entities like tasks and users may come back in `UPPER_CASE`.
- User fields are an exception: keep their native Bitrix24 names such as `UF_CRM_*` or `ufCrm_*`; do not normalize them.
- For large reads, VibeCode can auto-paginate and aggregate multiple Bitrix24 calls into one VibeCode response.
- Prefer auto-pagination via `limit`; do not hand-roll page walking unless the endpoint clearly requires it.
- Search and aggregate can do more work server-side than plain list endpoints; prefer them for filtered reports and dashboards.

## Infrastructure Invariants

- App port is always `3000` unless a task explicitly proves otherwise. Black Hole tunnels exactly that internal server port.
- Do not reason about `:3000` as a local machine port. Inside the VM it is only the app's internal port and is isolated from the user's laptop ports.
- New app servers are typically created in Black Hole mode and exposed through an HTTPS subdomain such as `https://app-{id}.vibecode.bitrix24.tech`.
- Treat server readiness as a two-part condition: `status=running` and `blackholeStatus=CONNECTED`.
- For deploy automation that expects JSON, use `POST /v1/infra/servers/:id/deploy?stream=false`; otherwise the endpoint may stream progress as SSE.
- Infrastructure API is VibeCode's own platform API, not a Bitrix24 wrapper. Do not assume Bitrix24 method naming or limitations there.

## Performance And Scale

- Prefer Batch API or per-entity batch operations for mass updates instead of long client loops.
- Large filtered reads may use windowed search automatically; disable only when you have a concrete reason.
- Default rate limit shown in auth docs is `10` requests per second, so bulk jobs should batch and retry instead of firing naive parallel bursts.

## Error Handling

- Distinguish auth failures (`AUTH_REQUIRED`, `INVALID_KEY`, `KEY_REVOKED`, `SCOPE_DENIED`) from transient platform failures (`RATE_LIMIT`, `BITRIX_UNAVAILABLE`, some `BITRIX_ERROR` cases).
- `502 BITRIX_UNAVAILABLE` usually means retry with backoff, not immediate schema changes.
- Preserve request context for debugging, especially response code, payload, and request time. If available, keep `X-Request-Id`.
- When an entity call fails, check scopes, endpoint path, and field names before assuming the platform is broken.
