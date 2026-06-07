# thcloud-mcp — Claude Context

Monorepo of custom MCP server Docker images exposed via Docker MCP Gateway as a single streamable HTTP endpoint. Servers not yet available in the official Docker MCP catalog are built here, pushed to GHCR via CI/CD, and served through the gateway.

## What This Repo Does

- Builds Docker images for MCP servers not in the [official Docker MCP catalog](https://hub.docker.com/mcp)
- Uses `docker/mcp-gateway` as the central proxy
- Exposes all active servers as one HTTP endpoint (`/mcp`) for OpenWebUI
- Each image also works standalone via `docker run` for Claude Desktop (stdio mode)
- CI/CD: push to `servers/<name>/` → GitHub Actions builds only that image → pushes to GHCR

## Current Servers

| Name | Runtime | Package |
|------|---------|---------|
| `meta-ads` | Python/uv | `meta-ads-mcp` (pip) |
| `line-bot` | Node.js | `@line/line-bot-mcp-server` (npm) |
| `coolify` | Node.js | `@masonator/coolify-mcp` (npm) |

## Key Architecture Concepts

**One endpoint, all tools aggregated.** The gateway bundles all active servers into `http://host:8599/mcp`. There is no per-server path routing. To serve different tool sets on different ports, run multiple gateway containers.

**catalog.yaml** is the menu of available servers — `registry:` is just a YAML key inside it. There is no separate `registry.yaml`. The `--registry` gateway flag is internal/legacy — ignore it.

- `catalog.yaml` — prod, images reference `ghcr.io/thcloudai/<name>-mcp:latest`

**`server.yaml`** inside each `servers/<name>/` is the source of truth for that server's definition. Copy its content into both catalog files when adding a server.

**`MCP_SERVERS` env var** controls which servers from the catalog are active. Set in `.env`. No fallback default — if empty, no servers run.

**Secrets injection:** `.env` is mounted as a Docker secret. The gateway reads it and injects values into each server container. The `name` field in the catalog's `secrets:` definition must match the key in `.env` exactly.

**`--pull never`:** The gateway always passes this flag when spawning server containers. This means locally built images are used without pulling from the registry — how dev mode works without needing to push first.

**Gateway spawns containers on demand.** Each MCP server runs in its own isolated container, started when the gateway receives a tool call for it. Not persistent unless `longLived: true` is set in the catalog.

## File Structure

```
servers/<name>/Dockerfile     # builds the MCP image
servers/<name>/server.yaml    # server definition — source of truth
catalog.yaml                  # prod catalog: GHCR image refs
docker-compose.yml            # prod — pulls from GHCR
docker-compose.dev.yml        # dev — standalone, builds + runs local images
.env.example                  # commit this; .env is gitignored
.github/workflows/build.yml   # path-filtered matrix CI/CD
```

## Common Commands

```bash
# Prod
docker compose up -d
docker compose down

# Dev — build all servers
docker compose -f docker-compose.dev.yml --profile build build

# Dev — build one server (after changing its Dockerfile)
docker compose -f docker-compose.dev.yml --profile build build meta-ads-mcp

# Dev — run
docker compose -f docker-compose.dev.yml up -d

# Dev — rebuild and reload one server
docker compose -f docker-compose.dev.yml --profile build build meta-ads-mcp
docker compose -f docker-compose.dev.yml restart mcp-gateway

# Dry run — verify config, list tools, no actual listening
docker mcp gateway run \
  --transport streaming \
  --port 8599 \
  --servers meta-ads,line-bot \
  --secrets ./.env \
  --dry-run

# Test a tool via HTTP (get a session first)
SESSION=$(curl -si -X POST http://localhost:8599/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}},"id":1}' \
  | grep -i "mcp-session-id" | awk '{print $2}' | tr -d '\r')

curl -s -X POST http://localhost:8599/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"get_ad_accounts","arguments":{}},"id":2}'

# Test a server directly in stdio mode (no gateway needed)
echo '{"jsonrpc":"2.0","method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}},"id":1}' \
  | docker run -i --rm -e META_ACCESS_TOKEN meta-ads-mcp:local
```

## Adding a New MCP Server

1. `mkdir servers/<name>`

2. Write `servers/<name>/Dockerfile`:
   - **Python:** `FROM ghcr.io/astral-sh/uv:python3.12-alpine` → `uv tool install <pkg>` → `ENTRYPOINT ["<bin>"]`
   - **Node.js:** `FROM node:22-alpine` → `npm install -g <pkg>` → `ENTRYPOINT ["<bin>"]`

3. Write `servers/<name>/server.yaml` (copy from an existing one, update fields)

4. Add entry to `catalog.yaml` (GHCR image ref — same file used for dev and prod)


6. Add secrets to `.env.example` and `.env`

7. Add `<name>` to `MCP_SERVERS` in `.env`

8. Push — CI builds only the changed server (path-filtered matrix)

## .env Keys Reference

```bash
MCP_SERVERS=meta-ads,line-bot,coolify   # required — which servers to activate

META_ACCESS_TOKEN=...                    # meta-ads
CHANNEL_ACCESS_TOKEN=...                 # line-bot
DESTINATION_USER_ID=...                  # line-bot (default send target)
COOLIFY_ACCESS_TOKEN=...                 # coolify
COOLIFY_BASE_URL=https://...            # coolify

PORT=8599                                # gateway HTTP port
```

## Catalog Format

```yaml
# catalog.yaml
registry:
  <name>:
    type: server
    title: Display Name
    description: What it does
    image: ghcr.io/thcloudai/<name>-mcp:latest   # GHCR in prod, 
    secrets:
      - name: MY_KEY      # must match .env key exactly
        env: MY_KEY       # env var injected into the container
        example: value
    allowHosts:
      - api.example.com:443
```

## Mixing with Official Docker Catalog

The official catalog (brave, github, notion, stripe, 50+) lives at:
`http://desktop.docker.com/mcp/catalog/v2/catalog.yaml`

To mix official + custom, add `--additional-catalog` to the compose command:
```yaml
- --catalog=/catalog.yaml
- --additional-catalog=http://desktop.docker.com/mcp/catalog/v2/catalog.yaml
- --servers=${MCP_SERVERS}
```

Official servers don't need a Dockerfile or server.yaml — just their name in `MCP_SERVERS`.

## CI/CD

`.github/workflows/build.yml` — triggers on push to `main` for changes under `servers/`:
- Detects which server subdirectories changed
- Matrix build — only builds changed servers
- Pushes to `ghcr.io/thcloudai/<name>-mcp:latest` and `ghcr.io/thcloudai/<name>-mcp:<sha>`
- PRs: build only, no push

## Client Connections

**OpenWebUI** (Docker): `http://host.docker.internal:8599/mcp`

**Claude Desktop** (stdio, per server):
```json
{
  "command": "docker",
  "args": ["run", "-i", "--rm", "-e", "META_ACCESS_TOKEN", "ghcr.io/thcloudai/meta-ads-mcp:latest"],
  "env": { "META_ACCESS_TOKEN": "..." }
}
```

## Why Docker MCP Gateway Over Supergateway

Supergateway wraps a single stdio MCP process as HTTP. It works but requires `uvx`/`npx` available in the container — needed a custom Dockerfile just for the runtime.

Docker MCP Gateway:
- Spawns each MCP server as its own isolated container on demand
- Native secrets injection — no plaintext env vars in container definitions
- Works with official Docker catalog servers out of the box
- Each image also works standalone via `docker run` (Claude Desktop)
- `--pull never` means local builds are used instantly without pushing

Supergateway setup preserved at `~/Claude/supergateway/` as reference.

## Key Resources

- https://docs.docker.com/ai/mcp-catalog-and-toolkit/mcp-gateway/
- https://github.com/docker/mcp-gateway
- https://github.com/docker/mcp-registry
- https://github.com/docker/mcp-gateway/tree/main/examples
- https://github.com/docker/mcp-gateway/tree/main/examples/custom-catalog
- https://hub.docker.com/mcp
- http://desktop.docker.com/mcp/catalog/v2/catalog.yaml
