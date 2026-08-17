---
name: Local Development
code: BRA106
version: 3
description: How to run Telos Brain locally with the CLI and Docker — start,
  stop, deploy, credentials, and pointing connectors at a host app via
  host.docker.internal. Use when developing a brain or a host application
  against a local Brain stack.
---

# Local Development

Run a full Telos Brain stack on your machine with the `brain` CLI. No Clerk,
no cloud credentials. The stack is SQL Server plus the Brain server in Docker;
ports bind to `127.0.0.1` only.

For schema authoring after the stack is up, use **BRA104** (guided interview)
or **BRA211** (auto-build from an existing app). For environment variables and
secrets, see **BRA202**. For connectors, see **BRA209**.

---

## 1. Requirements

- **Node.js 25+**
- **Docker** (Desktop on Mac/Windows, or Engine + Compose on Linux)
- **Git** and GitHub SSH access, if you scaffold with `brain init`
  (`git@github.com:telos-brain/starter-brain.git`)

---

## 2. Install the CLI

```bash
npm install -g @telos.ready/brain
brain --help
```

Upgrade later with `brain update`. Disable the daily npm update notice with
`TELOS_DISABLE_UPDATE_CHECK=1`.

---

## 3. Quick start

```bash
brain init                 # clones starter-brain into ./brain
cd brain
brain start                # Docker stack; creates .env.local if missing
# fill VOYAGE_API_KEY / LLM keys in .env.local
brain deploy --env local --instance local-brain
brain status               # API URL, org key, Compose project id, health
brain stop --project-id <id>   # tear down that stack (Compose id from status)
```

`brain start` does the following:

- Writes `brain.config.toml` (ports, image, instance name) if missing
- Creates `.env.local` from `.env.example` when missing — sets
  `TELOS_BRAIN_API_URL` and the well-known local `TELOS_BRAIN_ORG_API_KEY`;
  never overwrites an existing file
- Starts SQL + Brain server from `supporttelosready/telos-brain:latest`
- Boots a synthetic local SuperAdmin and a well-known organisation API key
- Opens the admin UI at `http://127.0.0.1:60061` (no sign-in)
- Prints the Compose project id — pass it to `brain stop --project-id …`

Useful options:

```bash
brain start --port 60061 --image supporttelosready/telos-brain:latest
brain start --brain local-brain
brain stop --project-id my-app-brain --reset   # also delete the local database volume
brain stop --all                              # stop every local Brain stack on this machine
```

---

## 4. Files the CLI creates

| Path | Role |
|---|---|
| `brain.config.toml` | Local ports, image, instance name, Compose project id. Safe to commit if it has no secrets. |
| `.env.local` | Deploy destination + secrets for `--env local`. **Do not commit.** |
| `.brain/` | Runtime state (compose file, encryption key). Gitignored. |
| `brain.lock` | Instance brain IDs (create vs migrate). Do not commit if it contains API keys. |

Destination always comes from `TELOS_BRAIN_API_URL` in the loaded env file
(`.env` or `.env.<env>` via `--env`). `brain.lock` only tracks instance brain
IDs.

---

## 5. Credentials and `.env.local`

`brain start` writes a well-known organisation API key that only works on a
local Docker stack (`BRAIN_LOCAL_DEV=true` in Development):

```
TELOS_BRAIN_ORG_API_KEY=tbk_local_dev_key_do_not_use_in_production_0001
TELOS_BRAIN_API_URL=http://127.0.0.1:60061
```

`TELOS_*` variables are CLI/deploy configuration. They are **never** uploaded
to the brain. Every other declared variable in `.env.local` **is** uploaded on
deploy (encrypted at rest). Blank values are skipped.

Before the first `brain deploy --env local`, fill at least:

| Variable | Why |
|---|---|
| `VOYAGE_API_KEY` | Required for the default embedding model (`voyage-3-lite`). Deploy fails without an embedding key. |
| `ANTHROPIC_API_KEY` | Needed if workflows use Anthropic (the default provider). |
| `OPENAI_API_KEY` | Needed for `openai/…` models, or if `embedding-model` is a `text-embedding-*` model. |
| `XAI_API_KEY` | Needed for `xai/…` models. |

See **BRA202** and **BRA210** for the full key list.

---

## 6. Commands (local)

### `brain start [path]`

Starts the Docker stack for `[path]` (default: current directory).

| Option | Description |
|---|---|
| `--port <port>` | Host port for API and admin UI (default: `60061`). |
| `--brain <instance>` | Local brain instance name (default: `local-brain`). |
| `--image <image>` | Docker image (default: `supporttelosready/telos-brain:latest`). |

### `brain update`

Upgrades the CLI (`npm install -g @telos.ready/brain@latest`). If the current
directory has a local Docker stack, also pulls the Brain image and recreates
the stack **without** `--reset` (database kept):

```bash
brain update
```

Equivalent manual steps:

```bash
brain status
docker pull supporttelosready/telos-brain:latest
brain stop --project-id <compose-id-from-status>
brain start
```

Compose interpolation is pinned to `.brain/` so the schema `.env` (deploy
secrets, comments, backticks) is never auto-loaded as a Compose env file.

### `brain status [path]`

Prints API URL, organisation API key, admin URL, SQL port, instance, Compose
project id, and health.

### `brain stop`

Stops a stack by Compose project id (the **Compose** row in `brain status`).
A container name/id in that stack also works.

```bash
brain stop --project-id my-app-brain
brain stop --project-id my-app-brain --reset
brain stop --all
```

| Option | Description |
|---|---|
| `--project-id <id>` | Compose project id, or a container name/id in that stack. |
| `--reset` | Also remove the local database volume (wipes local brains and data). |
| `--all` | Stop every local Brain Compose project on this machine. |

### `brain deploy --env local`

```bash
brain deploy --env local --instance local-brain
brain deploy --env local --instance my-app      # creates my-app on local if missing
brain deploy --env local --instance local-brain --dry-run
```

`--env local` loads `.env.local`. `--local` is a thin helper: if no URL/key is
set, use `brain.config.toml` / the well-known local org key.

`brain start` creates one instance (default `local-brain`). Additional instance
names are created on first `brain deploy --env local --instance <name>` — the
CLI asks the local API whether the instance exists and creates it when missing.
A name that already exists in `brain.lock` from a cloud deploy does **not**
block local create. After `brain stop --reset` (wiped database), deploy
recreates missing local instances instead of failing.

Deployment order is **brain → skills → workflows → tools → memory**.

If the live brain has moved on (inbox training, remote edits), snapshot first:

```bash
brain snapshot --env local --instance local-brain
brain deploy --env local --instance local-brain
```

Version conflicts (`409`) are warnings, not failures — the CLI still applies
everything else and exits `0`.

---

## 7. Calling a host app from Brain-in-Docker

The Brain server runs **inside** a container. `localhost` and `127.0.0.1`
inside that container are the container itself — not your machine.

Compose adds `extra_hosts: host.docker.internal:host-gateway` so tools can
reach APIs on the developer machine. Docker Desktop already injects this on
Mac/Windows; the extra host is required on Linux.

**Use this from the Brain container:**

```
http://host.docker.internal:<PORT>
```

Examples:

| Host app listens on | Connector / `url-env` value |
|---|---|
| `http://localhost:5183` | `http://host.docker.internal:5183` |
| `http://127.0.0.1:5000` | `http://host.docker.internal:5000` |

The host app must listen on an address the Docker host-gateway can reach
(typically `0.0.0.0` or `127.0.0.1` on the host). The Brain ports themselves
bind to `127.0.0.1` only; that does not stop outbound calls from the container
to `host.docker.internal`.

### Connector `url` / `url-env`

Prefer `url-env` so the same schema can point at a local host in `.env.local`
and a real HTTPS host in `.env.prod` (**BRA209** §4.4, **BRA202**).

```yaml
# connectors/squeeze-api.yml
name: squeeze-api
url-env: SQUEEZE_API_URL
auth-type: api-key
scope: brain
```

```bash
# .env.local
SQUEEZE_API_URL=http://host.docker.internal:5183
```

```bash
# .env.prod
SQUEEZE_API_URL=https://api.example.com
```

Connector base URLs must be **HTTPS**, except HTTP is allowed for these local
harness hosts (any port):

- `http://localhost`
- `http://127.0.0.1`
- `http://host.docker.internal`

`http://localhost` and `http://127.0.0.1` pass validation but **do not reach
the host app** from inside the container. Use `host.docker.internal`.

Remote HTTP URLs (e.g. `http://insecure.example.com`) are rejected.

After changing `.env.local`, redeploy so the brain environment variable is
upserted:

```bash
brain deploy --env local --instance local-brain
```

---

## 8. Local stack behaviour

| Topic | What to expect |
|---|---|
| Auth | No Clerk. Synthetic SuperAdmin (`local@localhost`) and the well-known org key. Admin UI needs no sign-in. |
| Bind address | API, admin UI, and SQL are bound to `127.0.0.1` only — not reachable on the LAN. |
| Admin UI | `http://127.0.0.1:60061` (or `--port`). |
| Health | `http://127.0.0.1:60061/api/health` |
| SQL | `127.0.0.1:1433` by default (`sql_port` in `brain.config.toml`). Change it if that port is already taken. |
| MCP | `Mcp__AllowAnyHost=true` on the local stack so host checks are relaxed. |
| Image | `supporttelosready/telos-brain:latest`. Recreate after changing image: `brain stop --project-id <id> --reset`, then `brain start`. |

`brain.config.toml` (created by `brain start`):

```toml
[local]
project_id = "brain"
instance = "local-brain"
api_port = 60061
sql_port = 1433
image = "supporttelosready/telos-brain:latest"
```

To run a locally built platform image (Brain repo developers):

```bash
brain start --image telos-brain:local
```

---

## 9. Common pitfalls

- **Tool call fails with "must resolve to a valid HTTPS URL"** — the
  `url-env` value is missing, blank, or a remote `http://` URL. Local HTTP is
  only valid for localhost, `127.0.0.1`, and `host.docker.internal`.
- **Tool call reaches the Brain container, not your app** — the URL used
  `localhost` or `127.0.0.1`. Switch to `http://host.docker.internal:<PORT>`
  and redeploy.
- **Deploy fails on embeddings** — `VOYAGE_API_KEY` (or `OPENAI_API_KEY` for
  an OpenAI embedding model) is blank. Fill it in `.env.local`.
- **Port already allocated** — change `api_port` / `sql_port` in
  `brain.config.toml`, or pass `--port`, then `brain start` again.
- **`brain init` fails** — GitHub SSH is not set up for the private starter
  repo. Override with `--template <url>` if you have another remote.
- **`--reset` wipes data** — the local SQL volume is deleted. Use it when you
  want a clean database, not for a routine stop.

---

## Related skills

- **BRA104** — guided interview to configure a new schema after `brain init`
- **BRA211** — auto-build a schema from an existing application
- **BRA201** — brain schema file format
- **BRA202** — `.env` upload, system keys, secret injection
- **BRA209** — connectors, `url` vs `url-env`
- **BRA210** — LLM providers and model codes
- **BRA401** — Execution API authentication once the brain is running
