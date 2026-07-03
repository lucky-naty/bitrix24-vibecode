# bitrix24-vibecode

GitHub-ready export of the `bitrix24-vibecode` skill for AI coding agents.

## Repository layout

This repository is intentionally minimal. The repository root is the skill root, so users can either:

- copy the whole folder into an agent skill directory
- add it as a git submodule or vendored skill in another repository

## What is included

- `SKILL.md`: the skill itself
- `references/`: supporting reference files
  - `deploy-playbook.md`: linear deploy → edit → redeploy recipe with a symptom→cause→fix error map (start here for any deploy task)
  - `section-routing.md`: map a task to the right VibeCode docs section
  - `universal-knowledge.md`: cross-section platform invariants (auth, Entity API, infra, **Standalone vs Galaxy deploy**, rate limits, error handling)
  - `anti-footguns.md`: preflight checklist of predictable mistakes
  - `placement-flow.md`: Bitrix24 iframe/placement apps on Black Hole — per-operator identity, gateway runtime, setup invariants, symptom→fix tree
  - `bizproc-robot-flow.md`: Bitrix24 bizproc automation robots via VibeCode — registration, OAuth bootstrap, runtime callback, `bizproc.event.send` completion, native REST v3 work
- `agents/openai.yaml`: optional OpenAI/Codex UI metadata

## Compatibility

- Claude-style skill setups: use `SKILL.md` and `references/`
- Codex/OpenAI-compatible setups: use `SKILL.md`, `references/`, and optionally `agents/openai.yaml`

The core skill is platform-neutral. The only runtime-specific file in this repository is `agents/openai.yaml`.

## Install

Copy this repository folder into your agent skills directory:

- Claude-style example: `~/.claude/skills/bitrix24-vibecode/`
- Codex-style example: `~/.codex/skills/bitrix24-vibecode/`

The final layout should look like this:

```text
<skills-root>/
  bitrix24-vibecode/
    SKILL.md
    references/
    agents/openai.yaml   # optional
```

## Purpose

This skill is not for turning the user into a VibeCode expert. It is for making the agent reliable when the user gives a normal task and expects the agent to figure out the platform details on its own.

## What is implemented

The skill already includes:

- a main workflow in `SKILL.md` for routing, minimal context loading, and task execution
- a docs routing reference so the agent opens the right VibeCode section first
- a universal knowledge reference with platform invariants that apply across many tasks
- a short anti-footguns checklist for implementation and debugging
- a placement/iframe flow reference for per-operator Bitrix24 apps on Black Hole
- a bizproc-robot flow reference for task/CRM automation robots
- optional OpenAI/Codex metadata in `agents/openai.yaml`

The repository is public and can be used as a cross-compatible skill source for Claude-style and Codex/OpenAI-style setups.

## Why this skill

Without this skill, an agent working on VibeCode tasks can easily waste time or make the wrong assumptions about:

- Bitrix24 entity access
- VibeCode platform APIs
- infrastructure and Black Hole deployment
- multiple auth models

This skill reduces that risk by helping the agent:

- find the right docs section without reading the whole documentation set
- remember the platform invariants that matter across many tasks
- avoid predictable mistakes in deploy, auth, Entity API, and bulk operations
- use the docs as internal guidance while still executing ordinary user requests

## What this improves

- App deploys become more reliable because the agent remembers the Black Hole model, port `3000`, readiness checks, and deploy response behavior.
- Integration tasks become safer because the agent is less likely to choose the wrong auth path or confuse VibeCode APIs with native Bitrix24 APIs.
- Entity API work becomes cleaner because the agent keeps request field rules, response format differences, and user field exceptions straight.
- Bulk sync and reporting logic become more efficient because the agent reaches for batch, search, and aggregate earlier instead of naive loops.
- Debugging becomes faster because the agent separates retryable platform failures from auth, scope, and schema mistakes.

## What the skill knows

The skill helps the agent keep the following platform rules in mind:

- choose the right auth model between `vibe_api_`, `vibe_app_`, and `vibe_live_`
- treat VibeCode platform APIs and native Bitrix24 APIs as different layers
- use Entity API conventions correctly, including camelCase requests and user field exceptions
- remember that Black Hole deploys are built around the internal app port `3000`
- wait for both `running` and `CONNECTED` before treating a deployed app as reachable
- pick the right deploy target: Standalone Black Hole VM vs Galaxy App container (512 MB cap, OOM graduate path, trial plan gating)
- use `stream=false` when deploy automation expects JSON instead of SSE
- prefer batch, search, and aggregate when bulk work would otherwise become a slow client loop
- use exact error-code enums (`RATE_LIMITED`, `BITRIX_UNAVAILABLE`, `OAUTH_REQUIRED`, ...) and the two-tier rate model (platform 300/min + portal ~10/sec)
- route per-operator placement apps and bizproc automation robots correctly instead of guessing the auth and callback contracts

## Typical user requests

The user does not need to ask special documentation questions. Typical requests are ordinary work requests such as:

- `Deploy this application to VibeCode.`
- `Build a Bitrix24 integration for this app.`
- `Fix why the deployed application is not reachable.`
- `Implement lead synchronization and make it stable for large volumes.`
- `Set up the backend so the app can read and update CRM entities.`

## Changelog

### v1.4.4

- Coverage-audit refinement (`universal-knowledge`): the "~940 unwrapped methods" figure was a raw method-name count. Corrected to **~930 by method name — an upper bound**: some capabilities are exposed as fields/filters rather than wrapped methods (e.g. deal↔contact bindings via `contactIds` in `/v1/deals`, not `crm.deal.contact.items.*`), so check the OpenAPI spec for the *capability* before assuming a gap. Split the gap list into **zero-trace** (absent from spec AND guide: imopenlines sessions/operators, `entity.*`, `biconnector.*`, voximplant admin) vs **effectively absent** (`lists.*`, `landing.*` beyond sites/pages, userfield definition management, task time-tracking `elapseditem`).

### v1.4.3

- Empirical platform audit (2026-07-02, 201 GET endpoints probed per key + behavioral diffs on live entities):
  - **No raw Bitrix REST passthrough** (`/v1/rest` → 404). Coverage: ~940 of 1171 native methods have no VibeCode wrapper; the uncovered areas (imopenlines sessions, `lists.*`, `entity.*`, `biconnector.*`, userfield definitions, task time-tracking, voximplant, `landing.*`) are listed in section-routing — route them to native REST immediately.
  - **Key-capability matrix**: bare `vibe_app_` reaches only platform resources (portal data → `401 TOKEN_MISSING`); `vibe_app_`+session = everything `vibe_api_` can PLUS the bizproc surface. App key in `Authorization: Bearer` → misleading `INVALID_SESSION`.
  - New anti-footguns #23–29: contact multifields live in `fm[]` (top-level `PHONE`/`EMAIL` are null), date filters use Mongo-style operators (`[$gte]`, not `[from]`), `meta.hasMore` lies on the exact-final page, unknown body fields silently dropped (chat `text` vs `message`), `?select=` ignored on GET-by-id, `POST /v1/timelines` ignores `authorId` (Bitrix core), catalog keeps native `propertyN` naming.
  - Machine-readable ground truth: `GET /v1/openapi.json` (~411 paths) and `GET /v1/guide`.

### v1.4.2

- Bizproc OAuth bootstrap: new **zero-config platform-callback + polling** path (`GET /v1/oauth/authorize` without redirect_uri → poll `GET /v1/oauth/poll` until `complete`; single-use token retrieval). Localhost-listener flow kept for when you must control the redirect. Footgun: an app key in `Authorization: Bearer` → `INVALID_SESSION` (app keys go in `X-Api-Key`).
- Placement iframe session pattern: signed HMAC cookie (`HttpOnly; Secure; SameSite=None` — mandatory in an iframe) as fallback when `X-Vibe-Authorization` isn't present on in-app navigation.
- Clarified that API `placement.bind` alone is sufficient — the catalog-publication wizard is optional and not required on your own portal.
- API key settings surface: expiry date, per-second rate limit, IP whitelist, section access — check these before rotating a suddenly failing key.
- Activities `ownerTypeId` constants (`4` = company, `2` = deal); env-var conventions (`VIBECODE_API_KEY` local, `VIBE_APP_KEY` on the server).
- Noted that direct portal REST may be unreachable from an agent sandbox (DNS timeout) while the VibeCode wrapper works — egress restriction, not a code bug.

### v1.4.1

- Documented the demo-tariff scope: all platform operations work on the demo/trial tier; the only restriction is server plan choice (`micro` only, bigger plans → `PLAN_NOT_ALLOWED_ON_TRIAL`).

### v1.4.0

- Added `references/deploy-playbook.md` — a linear **deploy → edit → redeploy** recipe for people who aren't server experts: know-your-3-facts preflight (owner key, server `kind`, id), `.tar.gz` packaging, `?stream=false` + wait for `running`+`CONNECTED`, the iterate loop (what NOT to recreate), an `exec` cheat-sheet, Galaxy differences, and a symptom→cause→fix error table. Wired into `SKILL.md` and `section-routing.md` as the first stop for any deploy task.
- Fixed a stale `RATE_LIMIT` reference in `SKILL.md` to the real `RATE_LIMITED` (+ other retryable codes).

### v1.3.2

- Deeper mining of real build sessions plus a full re-verification against the live docs.
- Added the **root cause of the `imconnector` wall**: the VibeCode `/v1/apps` scope whitelist rejects `imconnector` and `event` (`VALIDATION_ERROR: Invalid enum value`; only `im, imbot, imopenlines, …` pass) and placement `CONTACT_CENTER`, so a managed app structurally cannot register connectors — hence the native local app. Also `WRONG_AUTH_TYPE` on webhook register, and the `tokens.domain = oauth.bitrix.info` trap (use the real portal domain).
- Added diagnostics: `401 BH_LOGIN_REQUIRED` on a direct subdomain hit means the server is alive (normal gate, not a fault); a sleeping Black Hole server drops the direct `ONAPPINSTALL` callback (wake, then reinstall).
- Corrected Galaxy against docs: it does not self-connect (`blackholeStatus` stays `NONE`), `CONNECTED` is not a deploy precondition; the ~512 MB cap and `WAKE_IN_PROGRESS` / `GATEWAY_CONNECTION_TERMINATED` are marked as observed (not in the docs error table). Doc-confirmed deploy codes, `SNAPSHOT_REQUIRED` + `X-Skip-Source-Snapshot`, and the 10 ops/min limit retained.

### v1.3.1

- Sharpened the key-type guidance (`vibe_api_` portal data / no session / good for cron, `vibe_app_` per-operator OAuth identity, `vibe_live_` platform admin with **no** Bitrix24 entity data) and added read-only-key policy codes (`WRITE_BLOCKED_READONLY_KEY`, `KEY_POLICY_READONLY_REQUIRED`, `INFRA_SCOPE_REQUIRED`).
- Added more deploy/exec runtime codes from experience: `GATEWAY_CONNECTION_TERMINATED` (long `exec` drops the tunnel but keeps running on the VM — wait/retry) and `409 SNAPSHOT_REQUIRED` (external-URL source needs the snapshot step or `X-Skip-Source-Snapshot`).
- Noted that re-binding an already-bound event handler returns `ERROR_HANDLER_ALREADY` and should be treated as success (idempotent).

### v1.3.0

- Added `references/bizproc-robot-flow.md`: full bizproc automation-robot flow (registration with `vibe_app_` + `vibe_session`, FREE-tariff support, `documentType` for tasks, one-time OAuth bootstrap, native form-encoded runtime callback, `bizproc.event.send` completion, native REST v3 work).
- Added a **Standalone vs Galaxy App** deploy model to `universal-knowledge.md`: server `kind`, Galaxy container build/512 MB cap/OOM graduate path, deploy/exec runtime error codes, trial `bc-medium` gating.
- Corrected error handling against live docs: real enum names (`RATE_LIMITED`, `INVALID_API_KEY`, `OAUTH_REQUIRED`, `TOKEN_EXPIRED`, ...) instead of earlier shorthands, expanded retryable set, and the `BITRIX_UNAVAILABLE` write-duplicate caveat.
- Corrected the rate-limit model to two tiers: platform `300/min` per source (`429 RATE_LIMITED`, `X-RateLimit-*`) and portal `~10/sec` shared (`502 BITRIX_UNAVAILABLE`).
- Clarified OAuth: VibeCode session token is 24 h with no refresh (re-run authorize); native Bitrix app OAuth `refresh_token` is the separate self-renewing one.
- Added `imconnector.send.messages` runtime error codes and Entity API limits (`limit` default 50 / max 5000, `autoWindow`, single-entity `batch`).
- Verified all of the above against the current public VibeCode docs.

### v1.2.0

- Black Hole exec-deploy guide; `accessPolicy` and placement footguns; `placement-flow.md` reference.

## Source

The skill content was derived from the public VibeCode documentation:

- <https://vibecode.bitrix24.tech/docs>
- <https://vibecode.bitrix24.tech/docs/entity-api>
- <https://vibecode.bitrix24.tech/docs/keys-auth>
- <https://vibecode.bitrix24.tech/docs/infra>
- <https://vibecode.bitrix24.tech/docs/cli>
- <https://vibecode.bitrix24.tech/docs/errors>
- <https://vibecode.bitrix24.tech/docs/mcp>
- <https://vibecode.bitrix24.tech/docs/ai>
- <https://vibecode.bitrix24.tech/docs/bots>

Review and update the references if the public docs change.
