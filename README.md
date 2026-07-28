# Honcho on Dockhand

This overlay is for a **Git-backed Dockhand stack** built from the official Honcho source repository.

- Upstream: https://github.com/plastic-labs/honcho
- Tested source revision when this overlay was authored: `3ee890fa6f55388abd23b7660fb726e14d83459d`
- Dockhand host/LXC: `192.168.1.113`
- Compose file: `compose.dockhand.yaml`
- API: `http://192.168.1.113:8000`
- Health: `http://192.168.1.113:8000/health`
- OpenAPI/Swagger: `http://192.168.1.113:8000/docs`

Honcho does not include a conventional end-user dashboard at `/`.

## 1. Add this overlay to a fork

Fork `plastic-labs/honcho`, add these files at the repository root, and use a deployment branch such as `dockhand`:

- `compose.dockhand.yaml`
- `.env.example`
- `.gitignore` additions

Do **not** commit `.env`. Store its real values in Dockhand's stack environment editor.

## 2. Confirm Docker Compose

On the Debian 13 Docker LXC:

```bash
docker compose version
```

If the subcommand is missing and Docker's official Debian repository is already configured:

```bash
sudo apt update
sudo apt install docker-compose-plugin
```

## 3. Prepare bind-mount directories

The PostgreSQL directory name below intentionally preserves the requested spelling `postgressql`.

```bash
sudo install -d -m 0750 \
  /srv/docker/dockhand/dockhand_data/stacks/homelab/honcho-self-hosted/postgressql \
  /srv/docker/dockhand/dockhand_data/stacks/homelab/honcho-self-hosted/redis \
  /srv/docker/dockhand/dockhand_data/stacks/homelab/honcho-self-hosted/backups
```

The official PostgreSQL and Redis entrypoints normally initialize ownership themselves. If either container reports `permission denied`, discover the image UID/GID and apply it to only that service's directory:

```bash
docker run --rm --entrypoint id pgvector/pgvector:pg15 postgres
docker run --rm --entrypoint id redis:8.2 redis
```

Then use the reported numeric UID/GID with `sudo chown -R UID:GID <directory>`.

## 4. Generate secrets

Generate two independent values:

```bash
openssl rand -hex 32   # POSTGRES_PASSWORD
openssl rand -hex 32   # AUTH_JWT_SECRET
```

Create a dedicated OpenAI Platform project/key for Honcho. A ChatGPT subscription does not provide OpenAI API billing.

In Dockhand's environment editor, paste the variables from `.env.example` and replace:

- `REPLACE_WITH_64_HEX_POSTGRES_PASSWORD`
- `REPLACE_WITH_64_HEX_JWT_SECRET`
- `REPLACE_WITH_OPENAI_API_KEY`

Do not paste credentials into Compose YAML or commit them to Git.

## 5. Create the Git-backed Dockhand stack

Use:

```text
Repository: your fork of https://github.com/plastic-labs/honcho
Branch: dockhand (recommended)
Compose path: compose.dockhand.yaml
Build: enabled / BuildKit
```

The first source build can take several minutes. Only `192.168.1.113:8000` is published. PostgreSQL `5432` and Redis `6379` are not exposed.

## 6. Verify Honcho

From the Docker LXC:

```bash
docker compose -f compose.dockhand.yaml ps
curl -fsS http://192.168.1.113:8000/health
```

Expected health response:

```json
{"status":"ok"}
```

Check startup and background-worker logs:

```bash
docker compose -f compose.dockhand.yaml logs api --tail 100
docker compose -f compose.dockhand.yaml logs deriver --tail 100
```

## 7. Generate a Hermes workspace token

Authentication requires a JWT signed by `AUTH_JWT_SECRET`. Generate a workspace-scoped token inside the API container:

```bash
docker compose -f compose.dockhand.yaml exec -T api \
  /app/.venv/bin/python scripts/generate_jwt.py \
  --workspace hermes --print-only
```

Treat the output as a secret. It is the local Honcho bearer token used by Hermes; it is not the JWT signing secret.

For initial administrative troubleshooting only, an unrestricted token can be generated with:

```bash
docker compose -f compose.dockhand.yaml exec -T api \
  /app/.venv/bin/python scripts/generate_jwt.py --admin --print-only
```

Prefer the workspace-scoped token for normal Hermes use.

## 8. Database-backed smoke test

Put the workspace token in your shell temporarily:

```bash
read -rsp 'Honcho workspace JWT: ' HONCHO_JWT; echo
```

Create or retrieve the `hermes` workspace:

```bash
curl -fsS -X POST http://192.168.1.113:8000/v3/workspaces \
  -H "Authorization: Bearer $HONCHO_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"name":"hermes"}'
unset HONCHO_JWT
```

A returned workspace object proves authentication, API routing, PostgreSQL connectivity, and migrations are working.

## 9. Connect Hermes Agent

On the machine running Hermes:

```bash
hermes memory setup
```

Choose Honcho and local/self-hosted mode, then enter:

```text
Base URL: http://192.168.1.113:8000
Local JWT / bearer token: the workspace-scoped JWT generated above
Workspace: hermes
```

The wizard writes profile-local configuration under `$HERMES_HOME/honcho.json`.

A conservative personal-use profile is:

```json
{
  "recallMode": "hybrid",
  "saveMessages": true,
  "writeFrequency": "async",
  "contextCadence": 5,
  "dialecticCadence": 10,
  "dialecticDepth": 1,
  "dialecticReasoningLevel": "minimal",
  "dialecticDynamic": false,
  "contextTokens": 800,
  "sessionStrategy": "per-session",
  "observationMode": "unified",
  "pinUserPeer": true
}
```

Use the setup wizard rather than hand-editing Hermes configuration. After setup, run:

```bash
hermes memory status
```

## 10. Backup

Do not rely on copying the live PostgreSQL data directory as a complete logical backup. Create a dump:

```bash
mkdir -p /srv/docker/dockhand/dockhand_data/stacks/homelab/honcho-self-hosted/backups
docker compose -f compose.dockhand.yaml exec -T database \
  pg_dump -U postgres -d postgres -Fc \
  > /srv/docker/dockhand/dockhand_data/stacks/homelab/honcho-self-hosted/backups/honcho-$(date +%F).dump
```

Redis can be recreated from PostgreSQL-backed state if needed, but its AOF is persisted in the requested Redis directory.

## Notes

- LAN-only HTTP is suitable only on a trusted network. Do not port-forward `8000` or `11434` through the router.
- CORS is set to `[]` because Hermes is a server-side client and does not need browser cross-origin access. Swagger at `/docs` remains same-origin.
- No resource limits were added because LXC CPU/RAM values were not supplied.
- Summary and Dream processing are disabled, raw-message embeddings are disabled, and Deriver batching is increased to reduce OpenAI usage.
- Derived observations still use `text-embedding-3-small`.
