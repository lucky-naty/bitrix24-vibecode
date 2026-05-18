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

This skill helps an AI coding agent work with the VibeCode Bitrix24 documentation, especially for:

- choosing the right auth model
- routing to the correct docs section
- preserving cross-cutting platform invariants
- avoiding common implementation and deployment mistakes

## Why this skill

VibeCode combines several domains that are easy to mix up in real work:

- Bitrix24 entity access
- VibeCode platform APIs
- infrastructure and Black Hole deployment
- multiple auth models

This skill reduces wasted cycles by helping the agent:

- open the right docs section first
- apply platform invariants only when needed
- avoid common integration and deploy mistakes
- keep answers short for simple questions and deeper for implementation tasks

## Typical use cases

- deploy a VibeCode app without making the agent rediscover Black Hole and infrastructure rules
- build Bitrix24 integrations where the agent must choose the correct auth model on its own
- implement Entity API workflows without confusing request conventions, response formats, and user fields
- debug failed deploys, unreachable apps, and API errors with the right VibeCode-specific assumptions
- handle bulk sync and reporting tasks without wasting time on naive client-side loops

## Example prompts

Use this skill with prompts like:

- `Deploy this VibeCode app.`
- `Build an integration that syncs CRM contacts from Bitrix24.`
- `Fix why this deployed app is still unreachable.`
- `Implement bulk update logic for leads through VibeCode API.`

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
