# SkogAI MCPHub — podman deployment

Runs MCPHub as a rootless podman Quadlet service (`skogai-mcphub.service`), backed by
Postgres instead of `mcp_settings.json`.

## Why a local image instead of `samanhappy/mcphub:latest`

The published image doesn't run cleanly under podman yet:

- `procps` is missing from the base image → `spawn ps ENOENT` during MCP subprocess introspection.
- Podman doesn't create `/.dockerenv`, so the app's trust-proxy auto-detection never fires
  (`TRUST_PROXY=1` is set explicitly in the unit instead).

Both fixes are patched into this repo's `Dockerfile` but aren't in any upstream release tag,
so the image is built locally: `mise run container:build` → `localhost/mcphub:local`.

## Why DB mode instead of the config file

`mcp_settings.json` is git-ignored and no longer part of the deployment. The container runs
with `DB_URL` set (in the env file below), so `DaoFactory` switches to the database-backed
DAOs on boot and never touches a config file. See [AGENTS.md](../../AGENTS.md#4-architecture-invariants)
for how the dual-datasource migration works and its one-shot, empty-table-only trigger.

Postgres (`mcphub-postgres`, `pgvector/pgvector:pg17`) is managed separately via
[`docker-compose.db.yml`](../../docker-compose.db.yml), which creates the
`mcphub_mcphub-network` podman network. `skogai-mcphub.container` joins that same network
(`Network=mcphub_mcphub-network`) and reaches Postgres via its compose service alias `postgres`
— not via a published host port. Bring Postgres up first (`podman-compose -f docker-compose.db.yml up -d`
or `podman start mcphub-postgres`); if it's down, the app will fail to connect on boot.

## Secrets

`skogai-mcphub.container` loads `~/.config/containers/systemd/skogai-mcphub.env`
(untracked, not in this repo) for two keys:

- `JWT_SECRET` — dashboard session signing key.
- `DB_URL` — `postgresql://mcphub:<password>@postgres:5432/mcphub`, matching
  `docker-compose.db.yml`'s Postgres credentials.

`mise run container:install` ensures both keys exist without clobbering either one — it
checks each key independently (`grep -q '^KEY=' || append`), so regenerating the unit never
silently drops `DB_URL` back to file mode.

## mise tasks

| Task                 | Does |
| -------------------- | ---- |
| `container:build`    | `podman build -t localhost/mcphub:local .` |
| `container:install`  | Copy the Quadlet unit into `~/.config/containers/systemd/`, ensure the env file has `JWT_SECRET` + `DB_URL`, `daemon-reload` |
| `container:deploy`   | build + install, then restart the service |
| `container:start/stop/restart/status/logs` | `systemctl --user <verb> skogai-mcphub.service` |

## If the DB ever gets wiped

Migration only runs when the `users` table is empty. If you truncate/reset `mcphub-postgres`,
the next `skogai-mcphub` boot will auto-migrate — but only if `mcp_settings.json` exists at
build time to seed it from (it's git-ignored, so it must be present locally, not just in git
history). Restore it from a backup, or accept fresh defaults (randomized `admin` password
unless `ADMIN_PASSWORD` is set).
