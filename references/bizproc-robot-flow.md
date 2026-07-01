# Bizproc Robot Flow (Bitrix24 automation robots via VibeCode)

How a Bitrix24 bizproc automation robot is registered, wired to tasks/CRM, and how its handler runs on a BlackHole server. Read this before building or debugging any robot. Verified live (2026-06) on two real portals.

## What a robot is (vs event subscription)

A robot is an **automation-rule step**: it fires inside a document's automation/bizproc, receives the selected property values, does work, and (in subscription mode) signals completion back to the bizproc engine. It is NOT the same as a portal event subscription — see `universal-knowledge.md` "Portal Event Subscriptions" for the push-event tunnel. Use a robot when the user drops it into task/deal automation; use an event subscription when you need "any change to this entity" pushed to you.

## Registration (VibeCode wrapper)

- **Endpoint:** `POST /v1/bizproc-robots`. Delete: `DELETE /v1/bizproc-robots/{code}`.
- **Body is camelCase:** `{ code, handler, name, useSubscription: "Y", properties: { <key>: { name, type: "bool" } } }`. `handler` = your BlackHole robot callback URL (e.g. `https://app-<id>.vibecode.bitrix24.tech/robot`).
- **Requires a `vibe_app_` OAuth app key + a `vibe_session` bearer.** A personal `vibe_api_` key → `OAUTH_REQUIRED: "bizproc-robots requires an OAuth app key. Personal webhook keys cannot use Bitrix24 bizproc methods that need application context."` No single key does both entity work and robot registration — plan for the app key here.
- **Works on the FREE tariff.** Unlike `placement.bind` / `event.bind` (commercial only), robot registration succeeds on `ru_demo`/free. Only placement + event-subscription binds need a commercial tier.
- **Default document binding is CRM.** To bind to TASK automation you MUST pass `documentType: ["tasks", "Bitrix\\Tasks\\Integration\\Bizproc\\Document\\Task", "TASK"]`. Without it the robot only appears in CRM and never fires on tasks.

## Getting the `vibe_session` for registration (one-time browser bootstrap)

The bizproc methods need a `vibe_session_*` bearer minted from the app's OAuth. One browser click bootstraps it. Note: per keys-auth the VibeCode OAuth token lives **24h and does NOT ship a refresh token** — to renew, re-run the `authorize` → `token` exchange (the `state`↔cookie binding means the GET must happen in a real browser). This is distinct from the NATIVE Bitrix app OAuth `refresh_token` below, which self-renews headless.

1. redirect_uri must be **pre-registered EXACTLY** on the app: `PATCH /v1/apps/:id { redirectUris: [...] }`. A `http://localhost:<port>/callback` listener is the simplest, proven bootstrap target (no deploy dependency, no gate). The redirect_uri sent to `authorize` must string-match a registered entry or `authorize` returns `400 INVALID_REDIRECT_URI` — the "any localhost port/path" shortcut is unreliable, register the exact `http://localhost:<port>/callback` you will use.
2. Open `GET /v1/oauth/authorize?app_key=<key>&redirect_uri=<registered>&state=<16+ chars>` **in a real browser**. The browser must make the GET — `state` binds to a cookie, so a server-side curl breaks the flow.
3. Capture `code` at the redirect (local listener, or read the tab URL). Exchange: `POST /v1/oauth/token { app_key, code, redirect_uri }` → `access_token: vibe_session_*` (~24h).
4. **`scope` is NOT passed on the authorize URL** — it is inherited from the scopes registered on the app. Verify/patch app scopes via `GET`/`PATCH /v1/apps/:id`. Baseline for a task↔CRM robot: `crm` (read deal → companyId), `task` (update task), `bizproc` (register robot).

**App-URL callback instead of localhost** is possible but has two failure modes: the redirect_uri must be in `redirectUris`, and `accessPolicy=PORTAL` may gate the browser's direct GET to the subdomain (`/oauth/callback` never reached). If you must use it, temporarily set `PUBLIC` for the bootstrap then revert to `PORTAL`. Prefer localhost.

## Runtime callback (native Bitrix → your handler)

At runtime Bitrix (not VibeCode) POSTs to `handler` as `application/x-www-form-urlencoded` in **bracket-form** (`a[b][c]=v` → unflatten to nested). Real robot callbacks ride VibeCode managed infra and **pass an `accessPolicy=PORTAL` gate** — `PUBLIC` is NOT needed. (A direct external curl to the subdomain under `PORTAL` IS gated; don't let that curl test mislead you — managed callbacks still arrive.)

Callback body fields:

| Field | Meaning |
|---|---|
| `auth[application_token]` | shared secret — verify against your stored `APP_TOKEN` |
| `auth[domain]` | portal domain for the reply calls |
| `auth[access_token]` | token to use for the work + the completion ack |
| `event_token` | one-time token to ack completion (subscription mode) |
| `document_id[...]` | parse the task/deal id from here |
| `properties[...]` | the robot property values the operator set |

**Capturing `APP_TOKEN` (bring-up mode):** on first install `APP_TOKEN` is unknown. Run a bring-up mode that logs the received `auth[application_token]` and lets the first callback through instead of rejecting it; then pin the value into `.env` and enforce the check.

## Completion signal (useSubscription:"Y")

**HTTP 200 acknowledges receipt only — it does NOT complete the robot.** A subscription robot that never acks **hangs forever** in the automation. Completion is a separate native call:

```
POST https://<domain>/rest/bizproc.event.send        (v2, form-encoded, bracket-form)
  event_token        = <event_token from callback>    (one-time, per callback)
  auth               = <auth.access_token from callback>
  return_values[key] = value                           ({} if nothing to return)
```

Flow that works live:

```
1. receive /robot callback → respond 200 immediately
2. verify auth[application_token] == APP_TOKEN
3. do the work (see below)
4. bizproc.event.send { event_token, auth, return_values: {} }        ← completes the robot
5. on error: bizproc.event.send { ..., return_values: { error: "1", message } }
```

Use `useSubscription: "N"` for a synchronous robot that completes on the HTTP response (no ack) — simpler, but you lose the ability to report an error back into the bizproc and to retry. Subscription mode is more robust.

## Doing the work (native REST, not VibeCode entity API)

The handler calls **native Bitrix REST directly** with the callback's `auth.access_token`, not the VibeCode entity API — some fields are only writable that way.

- **REST v3 update:** `POST https://<domain>/rest/api/tasks.task.update?auth=<token>` with JSON `{ id, fields }`. The task flags `requireResult` / `allowsChangeDeadline` / `requireDeadlineChangeReason` are writable here (`result: true`) but are NOT exposed via the VibeCode entity API or `/v1/batch`.
- **Token expiry retry:** if the callback token is `expired_token`/`invalid_token`, refresh the app token once and retry: `GET https://oauth.bitrix.info/oauth/token/?grant_type=refresh_token&client_id=&client_secret=&refresh_token=` → new `access_token`. This is the standard Bitrix app OAuth (`.env`: `APP_CLIENT_ID/SECRET/ACCESS_TOKEN/REFRESH_TOKEN`), distinct from the `vibe_session` used for registration. The `refresh_token` arrives in the same token-exchange response (or in the install callback as `REFRESH_ID`) and self-renews headless — browser is a one-time bootstrap, not per-request.

## Footguns

- Registering a robot with a `vibe_api_` key → `OAUTH_REQUIRED`. Use `vibe_app_` + `vibe_session`.
- Omitting `documentType` → robot lands in CRM only, never fires on tasks.
- Treating HTTP 200 as completion → subscription robot hangs. Always `bizproc.event.send`.
- Using your stored app token for the reply instead of the callback's `auth.access_token`.
- Cyrillic in a POST body breaks Content-Length on inline `curl -d` / `exec` → write the body to a UTF-8 file and `--data-binary @file`.
- Confusing the two token systems: `vibe_session` (VibeCode OAuth, registers the robot) vs Bitrix app OAuth `access_token`/`refresh_token` (native REST, does the work). They are not interchangeable.
