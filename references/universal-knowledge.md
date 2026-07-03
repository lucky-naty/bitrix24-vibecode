# Universal Knowledge

## Start Here

- Start with `GET /v1/me` when a real key is available. It reveals entities, scopes, and deployment capabilities for that key.
- Machine-readable ground truth: `GET /v1/openapi.json` (full endpoint spec, ~411 paths) and `GET /v1/guide` (structured API guide). Prefer these over memory for exact request shapes.
- **There is NO raw Bitrix REST passthrough** (`/v1/rest` → 404; verified 2026-07). A native method with no entity wrapper is unreachable through ANY vibe key — plan a native webhook/OAuth path for it immediately instead of hunting the docs. Coverage audit (2026-07, demo portal): ~930 of 1171 native methods have no VibeCode wrapper **by method name** — an upper bound: some functionality is exposed as fields/filters instead of wrapped methods (e.g. deal-contact bindings via `contactIds` in `/v1/deals` rather than `crm.deal.contact.items.*`), so check the OpenAPI spec for the *capability* before assuming a gap. Zero-trace areas (confirmed absent from spec AND guide): imopenlines sessions/operators, entity.* (key-value), biconnector.*, voximplant admin. Effectively absent: lists.*, landing.* (only sites/pages wrapped), userfield definition management, task time-tracking (elapseditem).
- Base API URL is `https://vibecode.bitrix24.tech/v1`.
- Primary auth header is `X-Api-Key: <key>`. `Authorization: Bearer <key>` is an accepted alternative for `vibe_api_` and `vibe_live_` keys (OpenAI-compatible clients / AI Router). For `vibe_app_`, both are required: `X-Api-Key` + `Authorization: Bearer <session token>`.
- Most VibeCode APIs return `success` plus `data`; AI Router returns raw OpenAI-compatible responses.

## Keys And Access

- `vibe_api_` — **portal data via Vibe API.** Tied to one portal, every request acts as the key owner, no session token needed. Default choice for personal scripts, dashboards, server integrations, and cron/unattended jobs (works precisely because it needs no interactive session).
- `vibe_app_` — **embed an app in the portal + Bitrix24 OAuth.** Tied to an OAuth app; each request runs as the *authorizing user*, so it needs BOTH `X-Api-Key` and `Authorization: Bearer <session token>`. Required for per-operator identity (placement apps, bizproc robots).
- Empirical key-capability matrix (2026-07, 201 GET endpoints per key): bare `vibe_app_` (no session) reaches only platform resources (`/v1/me`, `/v1/bots`, `/v1/infra/*`, `/v1/feedback`, oauth flow) — every portal-data endpoint returns `401 TOKEN_MISSING`. `vibe_app_`+session reaches everything `vibe_api_` does PLUS the bizproc surface (`/v1/bizproc-templates|activities|robots`, `/v1/triggers`), which answers `403 OAUTH_REQUIRED` to personal keys. Session TTL is 24h with no refresh token — daemons must re-auth daily (zero-config poll flow: see bizproc-robot-flow.md).
- For Bitrix24 iframe/placement flow, the Gateway injects the session as the `X-Vibe-Authorization: Bearer vibe_session_*` header — the app reads it from there. The pre-2026-05 contract that delivered the session as `access_token` in the POST body is RETIRED; do not read it from the body. See [placement-flow.md](placement-flow.md).
- `vibe_live_` — **platform administration only.** NOT tied to one portal and has **no access to Bitrix24 entity data** — never reach for it to do ordinary CRM/task/entity work.
- The key prefix tells you which auth model and runtime behavior to expect before reading the rest of the task.
- Any key can be **read-only** (a write attempt → `403 WRITE_BLOCKED_READONLY_KEY`); a portal policy can also force read-only (`403 KEY_POLICY_READONLY_REQUIRED`). Check the key's scope/policy before designing a write path. `vibe:infra` scope is required for infrastructure calls (`403 INFRA_SCOPE_REQUIRED` otherwise).
- A key carries more settings than mode/scopes: **expiry date, per-second rate limit, IP whitelist**, and section access — all configurable in the dashboard, rights editable after creation. When a previously working key suddenly 401/403s, check expiry and IP whitelist before rotating it.

## Entity API Invariants

- Most entities follow one REST pattern: `GET list`, `GET by id`, `POST create`, `PATCH update`, `DELETE delete`, `POST search`, `POST aggregate`, `GET fields`.
- In requests, use camelCase field names by default.
- In responses, field format depends on the underlying Bitrix24 API: many CRM entities come back in camelCase, while legacy entities like tasks and users may come back in `UPPER_CASE`.
- User fields are an exception: keep their native Bitrix24 names such as `UF_CRM_*` or `ufCrm_*`; do not normalize them.
- For large reads, VibeCode can auto-paginate and aggregate multiple Bitrix24 calls into one VibeCode response.
- Prefer auto-pagination via `limit`; do not hand-roll page walking unless the endpoint clearly requires it. `limit` defaults to `50`, max `5000`; when `limit > 50` VibeCode fetches multiple Bitrix24 pages and merges them into one response.
- Search and aggregate can do more work server-side than plain list endpoints; prefer them for filtered reports and dashboards. `POST /{entity}/search` uses windowed search (time-sliced); disable with `autoWindow: false` only for a concrete reason.
- Activities filter by owner via `ownerTypeId`: `4` = CRM company, `2` = CRM deal (native Bitrix constants — keep them as magic numbers with a named constant in code).
- Bulk CRUD on one entity: `POST /{entity}/batch` (`{ action, items }`, ≤50 items → `BATCH_LIMIT_EXCEEDED`). Cross-entity: separate `POST /v1/batch`.

## Infrastructure Invariants

- App port is always `3000` unless a task explicitly proves otherwise. Black Hole tunnels exactly that internal server port.
- Do not reason about `:3000` as a local machine port. Inside the VM it is only the app's internal port and is isolated from the user's laptop ports.
- New app servers are typically created in Black Hole mode and exposed through an HTTPS subdomain such as `https://app-{id}.vibecode.bitrix24.tech`.
- Treat server readiness as a two-part condition: `status=running` and `blackholeStatus=CONNECTED`.
- For deploy automation that expects JSON, use `POST /v1/infra/servers/:id/deploy?stream=false`; otherwise the endpoint may stream progress as SSE.
- Infrastructure API is VibeCode's own platform API, not a Bitrix24 wrapper. Do not assume Bitrix24 method naming or limitations there.

## Deploying To A Black Hole VM

- Two deploy paths: the runtime deploy `POST /v1/infra/servers/:id/deploy`, and direct shell via `POST /v1/infra/servers/:id/exec` (run arbitrary commands on the VM). Many real deploys end up using `exec` (push files, write `.env`, start a service).
- **Use the API key that CREATED the server.** Both `exec` and server-scoped calls reject other keys with `WRONG_KEY: "Server exists but belongs to a different API key."` The creating key may be a `vibe_api_` or a `vibe_app_` key — check which one lists the server via `GET /v1/infra/servers`.
- Runtime deploy `source.content` must be a **`.tar.gz`** archive (base64), NOT `.zip`. A zip needs `unzip`, whose install can fail on a fresh VM (`"Failed to install unzip ... Use a .tar.gz archive"`). Large `source.content` POSTs intermittently fail with `TypeError: terminated` (upload cut) — just retry.
- A freshly provisioned VM may have `apt`/`dpkg` locked during cloud-init; the deploy `runtime` step then fails with `"Another command is running"`. Wait a minute and retry; do not hammer it.
- Black Hole VMs ship with Node preinstalled (e.g. `v18`) and often **no Docker**. Run the app as a `systemd` service (`WorkingDirectory=/opt/app`, `ExecStart=/usr/bin/node src/index.js`, `Restart=always`) listening on `:3000`; Black Hole tunnels that port to the subdomain. (`dotenv` loads `/opt/app/.env` from the working dir.)
- Env-var conventions: keep the personal key as `VIBECODE_API_KEY=vibe_api_...` locally, and put the app key on the server as `VIBE_APP_KEY=vibe_app_...` so deployed requests run under the app's identity, not your personal key.
- `exec` command gotchas: the body is JSON, so keep the command **ASCII-only** — non-ASCII (e.g. Cyrillic) breaks Content-Length (`"Request body size did not match Content-Length"`). The remote shell is **dash**, not bash: no `()` groups, no process substitution, no `[[ ]]`; quote glob-y args (e.g. `grep -E '^[A-Z_]+='`). `exec` is rate-limited to ~10 calls/min, so push large files in few big chunks, not many small ones.
- **`exec` command string is capped at 10000 chars** (`VALIDATION_ERROR: too_big, maximum 10000`). To push a file bigger than that: base64 it, split into ≤~7000-char chunks, `printf '%s' '<chunk>' >> /tmp/f.b64` (first chunk `>` to truncate, rest `>>`), then `base64 -d /tmp/f.b64 | tar xz -C <dir>`. One small final `exec` does the decode + `systemctl restart`.
- **App source snapshots (2026+):** every *successful* deploy auto-saves the app's sources (content-hashed, so an unchanged deploy adds nothing); history keeps recent + daily + weekly slices, "published" versions kept indefinitely. Visible/downloadable in the cabinet's "Исходники приложений" registry — a built-in rollback/handoff safety net, no manual backup step. Don't build your own source-archiving on top of deploy.

## Server Kinds: Standalone (Black Hole VM) vs Galaxy App

Two hosting models. Check `kind` on the server; the deploy contract and lifecycle differ.

- **Standalone (`kind=STANDALONE`)** — a real cloud VM in Black Hole mode. Deploy requires `status=running` + `blackholeStatus=CONNECTED` (else `NOT_BLACKHOLE` in OPEN mode, or `SERVER_NOT_READY` while provisioning). Create: `POST /v1/infra/servers` needs `provider, name, plan, region`; two-step create → poll for CONNECTED → deploy. Full VM lifecycle applies (start/stop/wake/reboot/sleep/refresh + access-policy + port PATCH). Counted in `/v1/me` `infra.limits.used`. Returns one-time `ssh.password` / `ssh.privateKey` at create — save immediately.
- **Galaxy App (`kind=GALAXY_APP`)** — a container on a shared host. It does NOT self-connect: `blackholeStatus` stays `NONE`, code is loaded right after create, so a `CONNECTED` tunnel is NOT a deploy precondition (docs). NOT counted in `infra.limits.used` (bounded by host capacity). **No v1 lifecycle actions** (start/stop/wake/access-policy/port) — a portal admin manages it via the galaxy admin surface; only owner `DELETE /v1/infra/servers/:id` applies. *Observed (not doc-stated):* the container is memory-limited (~512 MB — an oversized app crashes after start with an OOM hint; use the graduate path below).

Galaxy deploy contract:
- **Deploy immediately after create**, or pass `source` at create (Galaxy only; on a non-Galaxy portal `source` at create → `400 SOURCE_AT_CREATE_GALAXY_ONLY`).
- `source.content` ONLY (`source.url` / `source.versionId` → `400 GALAXY_DEPLOY_CONTENT_ONLY`).
- `runtime` (e.g. `"node20"`) AND `start` are REQUIRED (missing → `400 GALAXY_DEPLOY_RUNTIME_REQUIRED`). `install` must be single-line (newline → `400 GALAXY_DEPLOY_INVALID_INSTALL`).
- Dockerfile is AUTO-GENERATED from `runtime`/`install`/`start`/`port` (`start` baked as CMD); a user Dockerfile in the archive is ignored. App listens on `port` (default 3000); tunnel proxies to it.
- Galaxy runtime error codes: `409 GALAXY_BUILD_BUSY`, `409 GALAXY_SLEEPING`, `502 GALAXY_HOST_UNREACHABLE`, `502 GALAXY_APP_BUILD_FAILED` (returns buildLog tail), `502 GALAXY_APP_START_FAILED` (app crashed after start, e.g. OOM at 512 MB — returns buildLog tail).
- **OOM graduate path:** on `GALAXY_APP_START_FAILED` with an OOM hint, re-create dedicated: `POST /v1/infra/servers { placement: "dedicated", graduateFrom: <appId>, plan, provider, region }` then deploy.
- **Trial/plan gating:** Galaxy forces plan `bc-medium`, which a trial blocks → `PLAN_NOT_ALLOWED_ON_TRIAL`. On a Galaxy-First portal, pass `placement: "dedicated"` to get a standalone `bc-micro` VM instead (e.g. `provider=bitrix-cloud`, `region=ru-central`). Other billing gates: `BILLING_EXHAUSTED`, `TRIAL_EXPIRED`, `COMMERCIAL_PLAN_REQUIRED`.
- **Demo tariff (verified live):** ALL platform operations work on the demo/trial tier — the only restriction is server **plan choice: `micro` only** (bigger plans → `PLAN_NOT_ALLOWED_ON_TRIAL`). Do not assume a failure is tariff-gated; check the error code first.

## Deploy / exec runtime error codes

Beyond the deploy-archive gotchas above, the Deploy API surfaces (doc-confirmed): `409 SERVER_NOT_READY` (still provisioning), `409 AGENT_NOT_CONNECTED` (tunnel agent not `CONNECTED`), `409 EXEC_BUSY` (a `build`/`deploy`/`upload` already running — one at a time per server), `403 UPLOAD_PATH_DENIED` (forbidden upload path), plus `SERVER_WAKE_BLOCKED` / `REPAIR_BLOCKED`. Deploy API is rate-limited to **10 operations/minute per server** (separate from the ~10-calls/min `exec` limit noted above).
- *Observed (not in the docs error table but seen live):* `409 WAKE_IN_PROGRESS` ("server is being woken by another request" — wait, don't hammer) and `GATEWAY_CONNECTION_TERMINATED` (arrives even with HTTP 200): a long `exec` (e.g. `apt install`) tears the tunnel, but the command **keeps running on the VM** — treat like `EXEC_BUSY`, wait/poll for the result, do NOT re-run blindly.
- **`409 SNAPSHOT_REQUIRED`**: deploying with `source: { url }` from an external CDN requires the source-snapshot step; bypass only with header `X-Skip-Source-Snapshot: <non-empty reason ≤200 chars>`. `source.content` deploys don't hit this.

## Portal Event Subscriptions (push, no polling)

- Subscribe a BlackHole server to a Bitrix24 portal event and the platform tunnels each event to your app — no public URL needed: `POST /v1/infra/servers/:id/event-subscriptions { "event": "...", "appPath": "/your/handler" }`. List/journal via `GET` (`recentDeliveries`), remove via `DELETE …/:subId`.
- **Event name MUST be ALL-UPPERCASE**: `ONTASKADD`, `ONCRMDEALUPDATE`, `ONIMCONNECTORMESSAGEADD`. Mixed case (`OnImConnectorMessageAdd`) → `INVALID_EVENT`.
- Use the **OAuth-app key that owns the server** (`vibe_app_`); a plain `vibe_api_` server without an OAuth app → `400 NOT_OAUTH_APP`. The platform performs the Bitrix `event.bind` itself (under the server's managed app), so `event.bind` needs a **commercial** portal tier (free → `502 BIND_FAILED`).
- Delivery is a normal Bitrix event POST (`application/x-www-form-urlencoded`) to `appPath`; verify `auth[application_token]` and reply 2xx (platform retries + wakes a sleeping server). `recentDeliveries: []` with `status: ACTIVE` means the bind exists but Bitrix has fired nothing to it yet — see the imconnector caveat in anti-footguns.
- Re-binding an already-bound handler returns `ERROR_HANDLER_ALREADY` / `handler_already` — treat it as success (idempotent), not a failure.

## Performance And Scale

- Prefer Batch API or per-entity batch operations for mass updates instead of long client loops.
- Large filtered reads may use windowed search automatically; disable only when you have a concrete reason.
- Two independent rate limits apply per key:
  - **Platform (VibeCode):** `300` requests/minute per source, sliding 60s window. Exposed via `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset`. Exceeded → `429 RATE_LIMITED` (carries `retryAfter`).
  - **Portal (Bitrix24):** ~`10` req/sec, shared across all portal keys (actual in `rateLimit.requestsPerSecond`). Exceeded → `502 BITRIX_UNAVAILABLE`.
- Bulk jobs should batch and retry instead of firing naive parallel bursts.

## Error Handling

- Distinguish auth failures (`401 MISSING_API_KEY`, `401 INVALID_API_KEY`, `401 INVALID_APP_KEY`, `401 TOKEN_EXPIRED`, `403 OAUTH_REQUIRED`, `403 SCOPE_DENIED`) from transient platform failures (`429 RATE_LIMITED`, `502 BITRIX_UNAVAILABLE`, `504 QUEUE_TIMEOUT`, some `422 BITRIX_ERROR` cases). Error names are exact enum strings — the pre-2026 shorthands `AUTH_REQUIRED`/`INVALID_KEY`/`KEY_REVOKED`/`RATE_LIMIT` are NOT real codes; a deleted key returns `INVALID_API_KEY`.
- Retryable (docs ship auto-retry for these): `RATE_LIMITED` (honor `retryAfter`, ~1–2s), `QUEUE_TIMEOUT`, `BITRIX_UNAVAILABLE` (~5s), `ERROR_LOOP_DETECTED`, `INTERNAL_ERROR`.
- `502 BITRIX_UNAVAILABLE` usually means retry with backoff, not immediate schema changes. For a WRITE, do a `GET` check before blind retry — a slow portal may still apply the write after timeout, so a blind retry creates a duplicate.
- Preserve request context for debugging, especially response code, payload, and request time. If available, keep `X-Request-Id`.
- When an entity call fails, check scopes, endpoint path, and field names before assuming the platform is broken.
- Direct REST calls to `https://<portal>/rest/` may be unreachable from an agent/sandbox environment (DNS timeout) while the VibeCode wrapper works fine — that's an egress restriction, not a code bug. Route portal work through the VibeCode API when the environment can't reach the portal directly.
