# Deploy Playbook (linear recipe for deploy → edit → redeploy)

A step-by-step path for shipping and iterating a VibeCode Bitrix24 app without the "dance with a tambourine." Follow it top to bottom. When a step fails, jump to the error map at the bottom, fix the named cause, then resume — do **not** restart from scratch. Deeper detail lives in [universal-knowledge.md](universal-knowledge.md); this file is the ordered checklist.

## 0. Know your three facts first (skipping this causes most of the pain)

1. **Which API key OWNS the server.** `exec` and every server-scoped call reject any other key with `WRONG_KEY`. Find it: `GET /v1/infra/servers` with each key you have — the one that lists the server is the owner. Use ONLY that key for all deploy steps.
2. **Server `kind`** — `STANDALONE` (a Black Hole VM) or `GALAXY_APP` (a shared-host container). The deploy contract differs (see §5). Get it from `GET /v1/infra/servers/:id`.
3. **Server id and subdomain** (`https://app-<id>.vibecode.bitrix24.tech`).

## 1. Preflight (10 seconds, saves 10 deploys)

- `GET /v1/me` with the owner key → confirms scopes + that the key can deploy.
- `GET /v1/infra/servers/:id` → confirm `status` and `blackholeStatus`.
- The app must listen on **port 3000** inside the VM. Not 8080, not `$PORT` — `3000`. Black Hole tunnels exactly that port.

## 2. Package the code — `.tar.gz`, never `.zip`

- Build a **gzip tarball** and base64 it into `source.content`.
- A `.zip` needs `unzip`, which often isn't installed on a fresh VM and fails with *"Use a .tar.gz archive"*. Don't use zip.
- Do NOT commit a `Dockerfile` for Galaxy — it's auto-generated (see §5).

## 3. Deploy — one call, then WAIT

```
POST /v1/infra/servers/:id/deploy?stream=false
  { "source": { "content": "<base64 .tar.gz>" }, "runtime": "node20", "start": "node src/index.js" }
```

- `?stream=false` → you get clean JSON. Without it the endpoint may stream SSE and your tool "hangs" waiting.
- **Then poll `GET /v1/infra/servers/:id` until `status=running` AND `blackholeStatus=CONNECTED`.** Treating the app as live before BOTH are true is the #1 reason "it didn't work, redeploy" — the deploy was fine, you just checked too early.
- A fresh VM may briefly report `"Another command is running"` (cloud-init holds `apt`/`dpkg`). Wait ~60s, retry once. Don't hammer.

## 4. Verify it's actually reachable

- Open `https://app-<id>.vibecode.bitrix24.tech` **the way the app is meant to be opened** (through the Bitrix placement, or with auth).
- A direct unauthenticated curl returning **`401 BH_LOGIN_REQUIRED`** does NOT mean it's broken — that's the normal gate and proves the server is up. See [placement-flow.md](placement-flow.md).

## 5. The edit → redeploy loop (this is the part that felt like a "dance")

To ship a change, you repeat **only §2 → §3 → §4**. You do NOT:

- ❌ re-create the app or server (`POST /v1/apps`, `POST /v1/infra/servers`) — the server already exists; recreating it changes the owner key and breaks `appUrl`.
- ❌ switch keys mid-flow — same owner key every time.
- ❌ hand-craft `exec` file chunks unless a single file is genuinely huge (see below).
- ❌ change the port, `accessPolicy`, or `handlerUrl` to "try to fix" a deploy that's just still starting.

Fast redeploy = new `.tar.gz` → `POST /deploy?stream=false` → wait for `running`+`CONNECTED` → check. That's the whole loop.

### If you must push files via `exec` instead of a full deploy

`exec` is for small surgical changes (write `.env`, restart a service). Its traps:
- Remote shell is **dash**, not bash — no `[[ ]]`, no `()` groups, no process substitution.
- **ASCII only** — Cyrillic in the command body breaks Content-Length. Write non-ASCII content to a file and `--data-binary @file`.
- Command string capped at **10000 chars**, and `exec` is limited to **~10 calls/min**. For a big file: base64 it, split into ≤7000-char chunks appended to `/tmp/f.b64`, then `base64 -d /tmp/f.b64 | tar xz -C <dir>` + one `systemctl restart`.
- A long `exec` (e.g. `apt install`) may return `GATEWAY_CONNECTION_TERMINATED` even though the command **keeps running on the VM** — wait and poll, don't blindly re-run.

### Galaxy App (`kind=GALAXY_APP`) differences

- Deploy **builds the container** — no `CONNECTED` wait (its `blackholeStatus` stays `NONE`, that's normal).
- `source.content` ONLY; `runtime` + single-line `install` + `start` are required. No custom Dockerfile.
- Container is memory-tight (~512 MB observed). If it crashes after start with an OOM hint → graduate to a dedicated VM: `POST /v1/infra/servers { placement: "dedicated", graduateFrom: <appId>, plan, provider, region }` then deploy.

## Error map (symptom → cause → fix)

| You see | It means | Do |
|---|---|---|
| `WRONG_KEY` | wrong API key for this server | use the key that CREATED the server (§0.1) |
| `SERVER_NOT_READY` / not `CONNECTED` | you deployed/checked too early | wait for `running`+`CONNECTED`, then proceed |
| `"Another command is running"` | cloud-init holds apt/dpkg | wait ~60s, retry once |
| `"Use a .tar.gz archive"` / unzip fail | you sent a `.zip` | repackage as `.tar.gz` (§2) |
| `TypeError: terminated` on deploy | upload got cut | just retry the same deploy |
| `EXEC_BUSY` / `WAKE_IN_PROGRESS` / `GATEWAY_CONNECTION_TERMINATED` | one op at a time / server waking / long cmd dropped tunnel | wait and retry/poll, don't parallelize |
| `409 SNAPSHOT_REQUIRED` | deploying from an external URL source | use `source.content`, or header `X-Skip-Source-Snapshot: <reason>` |
| app opens but acts as wrong user | using a `vibe_api_` key for per-operator work | see [placement-flow.md](placement-flow.md) |
| `401 BH_LOGIN_REQUIRED` on direct curl | normal gate | not an error — server is alive (§4) |

For anything auth/scope/entity-shaped, or codes not listed here, see [anti-footguns.md](anti-footguns.md) and the error section of [universal-knowledge.md](universal-knowledge.md). Fix permanent mistakes (auth, scope, path, field shape, zip-vs-tar) before retrying; only retry the transient codes above.
