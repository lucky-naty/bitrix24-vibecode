# bitrix24-vibecode

GitHub-ready export of the `bitrix24-vibecode` skill for AI coding agents.

## Repository layout

This repository is intentionally minimal. The repository root is the skill root, so users can either:

- copy the whole folder into an agent skill directory
- add it as a git submodule or vendored skill in another repository

## What is included

- `SKILL.md`: the skill itself
- `references/`: supporting reference files
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
- use `stream=false` when deploy automation expects JSON instead of SSE
- prefer batch, search, and aggregate when bulk work would otherwise become a slow client loop

## Typical user requests

The user does not need to ask special documentation questions. Typical requests are ordinary work requests such as:

- `Deploy this application to VibeCode.`
- `Build a Bitrix24 integration for this app.`
- `Fix why the deployed application is not reachable.`
- `Implement lead synchronization and make it stable for large volumes.`
- `Set up the backend so the app can read and update CRM entities.`

## Source

The skill content was derived from the public VibeCode documentation:

- <https://vibecode.bitrix24.tech/docs>
- <https://vibecode.bitrix24.tech/docs/entity-api>
- <https://vibecode.bitrix24.tech/docs/keys-auth>
- <https://vibecode.bitrix24.tech/docs/infra>
- <https://vibecode.bitrix24.tech/docs/cli>
- <https://vibecode.bitrix24.tech/docs/errors>
- <https://vibecode.bitrix24.tech/docs/mcp>

Review and update the references if the public docs change.
