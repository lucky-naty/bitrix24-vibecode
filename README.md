# bitrix24-vibecode

GitHub-ready export of the `bitrix24-vibecode` skill for AI coding agents.

## Repository layout

This repository is intentionally minimal. The repository root is the skill root, so users can either:

- copy the whole folder into an agent skill directory
- add it as a git submodule or vendored skill in another repository

## What is included

- `SKILL.md`: the skill itself
- `references/`: supporting reference files
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
