# Section Routing

Use this file to choose the first VibeCode docs section.

## Routing Rules

- For any **deploy / redeploy / "my change won't ship"** task, open [deploy-playbook.md](deploy-playbook.md) first — the linear recipe + error map.
- Start with `quick-start` for a first app, deploy flow, or platform overview.
- Open `entity-api` for CRUD, search, aggregate, fields, pagination, and entity conventions.
- Open `keys-auth` for key choice, scopes, OAuth, or unauthorized errors.
- Open `infra` for servers, deploys, Black Hole, access policies, and lifecycle. For **deploy target choice** (Standalone Black Hole VM vs Galaxy App container, 512MB/OOM graduate, trial `bc-medium` gating, deploy/exec error codes) read `universal-knowledge` → "Server Kinds" + "Deploy / exec runtime error codes".
- Open [placement-flow.md](placement-flow.md) for any Bitrix24 iframe/placement app: handler/`__init`/gateway flow, `appUrl` linking, `accessPolicy`, and the gate/`APP_RESOLVE_FAILED`/`ERR_BLOCKED_BY_RESPONSE` troubleshooting tree.
- Open `ai` for chat completions, models, BYOK, usage, or OpenAI-compatible clients.
- Open `bots` for bot registration, polling, commands, and messages.
- Open `mcp` for agent integration, MCP transport, or IDE setup.
- Open `cli` for `curl`, OpenAPI, or `ocli`.
- Open `errors` for failing requests or retry guidance.
- Open `api-reference` for exact endpoint confirmation after higher-level docs.
- For **portal event push** (subscribe a server to Bitrix events, no polling) read `universal-knowledge` → "Portal Event Subscriptions" (UPPERCASE event names, OAuth-app key, commercial tier).
- For **Open Lines connectors** (`imconnector.*`) read `anti-footguns` #20–21 first: needs a native local-app OAuth token, and connector outgoing events are a known platform gap.

## URL Map

- Quick start: `https://vibecode.bitrix24.tech/docs/quickstart`
- Entity API: `https://vibecode.bitrix24.tech/docs/entity-api`
- Keys and auth: `https://vibecode.bitrix24.tech/docs/keys-auth`
- Infrastructure: `https://vibecode.bitrix24.tech/docs/infra`
- AI Router: `https://vibecode.bitrix24.tech/docs/ai`
- Bots: `https://vibecode.bitrix24.tech/docs/bots`
- MCP: `https://vibecode.bitrix24.tech/docs/mcp`
- CLI and cURL: `https://vibecode.bitrix24.tech/docs/cli`
- Errors: `https://vibecode.bitrix24.tech/docs/errors`
- API reference: `https://vibecode.bitrix24.tech/docs/api-reference`
