---
name: bitrix24-vibecode
description: Use when working with VibeCode Bitrix24 apps, integrations, bots, or AI Router workflows.
---

# VibeCode Docs

## Overview

Use this skill to route through VibeCode docs quickly, keep platform invariants straight, and avoid re-deriving basics.

## Workflow

1. Identify the task type before reading docs in depth.
2. Load [references/section-routing.md](references/section-routing.md) and open only the most relevant section first.
3. Load [references/universal-knowledge.md](references/universal-knowledge.md) only when the task touches build, deploy, auth, API conventions, scaling, or error handling.
4. Load [references/anti-footguns.md](references/anti-footguns.md) only before implementation or debugging.
5. Confirm endpoint-level details from the live docs only when the answer depends on exact parameter names, current limits, or current behavior.
6. Distinguish VibeCode platform behavior from native Bitrix24 behavior.

## Load Minimally

- For explanation-only questions, start with `section-routing` and stop unless the answer needs platform invariants.
- For implementation or debugging, read `universal-knowledge` before writing code.
- For exact request shapes or current behavior, verify against the live docs instead of relying on memory.

## Task Patterns

### App, Bot, Or Integration Build

Use quick start plus the target domain section. Default flow:

1. Inspect capabilities with `/v1/me`
2. Build against Entity API or another target API
3. Create infrastructure if deployment is needed
4. Deploy and wait for `running` plus `CONNECTED`

### Auth And Access Questions

Open keys/auth first. Decide from the key prefix whether the task is personal (`vibe_api_`), OAuth app (`vibe_app_`), or management (`vibe_live_`). For OAuth app requests, remember that a bearer session token may also be required for user-context calls.

### Placement / Iframe App (per-operator identity)

For any Bitrix24 placement app (LEFT_MENU, CRM_*_DETAIL_TAB, etc.) on a BlackHole server where each operator must act as themselves, read [references/placement-flow.md](references/placement-flow.md) first. It covers the handler/`__init`/gateway runtime, the setup invariants (`handlerUrl` = bitrix-handler, `appUrl` linked to the server, `accessPolicy=PORTAL`), and the symptom→fix tree for the gate / `auth2.bitrix24.net` / `ERR_BLOCKED_BY_RESPONSE` / `APP_RESOLVE_FAILED` failures.

### Bizproc Robot (task/CRM automation)

For any Bitrix24 bizproc automation robot registered through VibeCode, read [references/bizproc-robot-flow.md](references/bizproc-robot-flow.md) first. It covers registration (`POST /v1/bizproc-robots`, `vibe_app_`+`vibe_session`, works on FREE tariff, `documentType` for tasks), the one-time browser OAuth bootstrap for the `vibe_session`, the native form-encoded runtime callback (`event_token`, `auth[application_token]`), completion via `bizproc.event.send` (HTTP 200 ≠ completion), and doing the work via native REST v3.

### Entity Work

Open Entity API first, then API Reference only if exact endpoint confirmation is needed. Prefer search, aggregate, and batch capabilities over naive client-side loops when the task is report-heavy or bulk-oriented.

### Infrastructure And Deployment

Open infrastructure docs first, then load `universal-knowledge` for platform invariants such as Black Hole, port `3000`, readiness rules, and deploy response mode.

### Debugging

Open error docs early for real failures. Separate permanent mistakes from retryable failures:

- Fix auth, scope, path, and field-shape issues before retrying.
- Retry `RATE_LIMIT` and `BITRIX_UNAVAILABLE` with backoff.
- Treat many `BITRIX_ERROR` responses as upstream or portal-specific issues, not proof that VibeCode docs are wrong.

## Answering Style

- Surface invariant rules explicitly when they prevent common mistakes.
- If a detail is likely to drift, verify it from the live docs rather than relying on memory.
- Make it clear when a statement comes from universal platform behavior versus one specific endpoint or section.
- Prefer concise implementation guidance over long prose.

## Resources

- [references/section-routing.md](references/section-routing.md): map task types to doc sections and URLs.
- [references/universal-knowledge.md](references/universal-knowledge.md): cross-section platform postulates that should be applied broadly.
- [references/anti-footguns.md](references/anti-footguns.md): short preflight checklist for implementation and debugging.
- [references/placement-flow.md](references/placement-flow.md): Bitrix24 iframe/placement apps on BlackHole — handler/`__init`/gateway runtime, per-operator identity, setup invariants (`handlerUrl`, `appUrl`, `accessPolicy`), and a symptom→fix troubleshooting tree.
- [references/bizproc-robot-flow.md](references/bizproc-robot-flow.md): Bitrix24 bizproc automation robots via VibeCode — registration (`vibe_app_`+`vibe_session`, FREE tariff, `documentType`), one-time OAuth bootstrap, runtime callback contract, `bizproc.event.send` completion, native REST v3 work.
