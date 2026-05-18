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

- explain which key type to use for a new integration
- build a script against Entity API without mixing request and response field conventions
- debug why a deployed app is not reachable through Black Hole
- decide when to use batch, search, or aggregate instead of client-side loops
- separate retryable platform errors from auth or schema mistakes

## Example prompts

Use this skill with prompts like:

- `Explain how to choose between vibe_api_, vibe_app_, and vibe_live_.`
- `Build a VibeCode CRM contacts sync and check the likely footguns first.`
- `Debug why my app deploy succeeded but the service is still unreachable.`
- `Which VibeCode docs section should I open first for OAuth app auth?`

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
