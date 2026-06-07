# thcloud-mcp

Custom MCP server Docker images for services not yet in the [official Docker MCP catalog](https://hub.docker.com/mcp). Uses [Docker MCP Gateway](https://github.com/docker/mcp-gateway) to expose all servers as a single streamable HTTP endpoint — compatible with OpenWebUI, Claude Desktop, and any MCP client.

## Current Servers

| Server | Package | Tools |
|--------|---------|-------|
| `meta-ads` | [meta-ads-mcp](https://pypi.org/project/meta-ads-mcp/) | 37 — manage Facebook/Instagram ad campaigns |
| `line-bot` | [@line/line-bot-mcp-server](https://github.com/line/line-bot-mcp-server) | 12 — send messages, manage rich menus |
| `coolify` | [@masonator/coolify-mcp](https://github.com/StuMason/coolify-mcp) | 42 — manage self-hosted deployments |

## Structure

```
thcloud-mcp/
├── servers/
│   ├── meta-ads/
│   │   ├── Dockerfile        # Python (uv) — installs meta-ads-mcp
│   │   └── server.yaml       # server definition (source of truth)
│   ├── line-bot/
│   │   ├── Dockerfile        # Node.js — installs @line/line-bot-mcp-server
│   │   └── server.yaml
│   └── coolify/
│       ├── Dockerfile        # Node.js — installs @masonator/coolify-mcp
│       └── server.yaml
├── catalog.yaml              # prod: references GHCR images
├── catalog.yaml          # dev: references locally built images
├── docker-compose.yml        # prod
├── docker-compose.dev.yml    # dev (standalone)
├── .env.example
└── .github/workflows/build.yml
```

## Setup

```bash
git clone https://github.com/THCloudAI/thcloud-mcp
cd thcloud-mcp
cp .env.example .env
# fill in tokens in .env
```

### `.env` configuration

```bash
# Which servers to run (comma-separated, no spaces)
MCP_SERVERS=meta-ads,line-bot,coolify

# Meta Ads — https://developers.facebook.com/tools/explorer/
META_ACCESS_TOKEN=your_meta_access_token

# LINE Bot — https://developers.line.biz/console/
CHANNEL_ACCESS_TOKEN=your_line_channel_access_token
DESTINATION_USER_ID=your_line_user_or_group_id

# Coolify — Settings > API Keys in your Coolify dashboard
COOLIFY_ACCESS_TOKEN=your_coolify_access_token
COOLIFY_BASE_URL=https://coolify.example.com

PORT=8599
```

Only include tokens for servers listed in `MCP_SERVERS`.

## Running

### Prod — pulls images from GHCR

```bash
docker compose up -d
```

### Dev — builds images locally

```bash
# Build all server images
docker compose -f docker-compose.dev.yml --profile build build

# Start gateway
docker compose -f docker-compose.dev.yml up -d
```

Build a single server after updating its Dockerfile:

```bash
docker compose -f docker-compose.dev.yml --profile build build meta-ads-mcp
docker compose -f docker-compose.dev.yml restart mcp-gateway
```

### Dry run (debug — lists tools without starting the server)

```bash
docker mcp gateway run \
  --transport streaming \
  --port 8599 \
  --servers meta-ads,line-bot \
  --catalog ./catalog.yaml \
  --secrets ./.env \
  --dry-run
```

## Connecting Clients

### OpenWebUI (running in Docker)

Settings → Tools → Add Tool Server:
```
http://host.docker.internal:8599/mcp
```

### Claude Desktop

`~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "meta-ads": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "-e", "META_ACCESS_TOKEN", "ghcr.io/thcloudai/meta-ads-mcp:latest"],
      "env": { "META_ACCESS_TOKEN": "your_token" }
    },
    "line-bot": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "-e", "CHANNEL_ACCESS_TOKEN", "-e", "DESTINATION_USER_ID", "ghcr.io/thcloudai/line-bot-mcp:latest"],
      "env": {
        "CHANNEL_ACCESS_TOKEN": "your_token",
        "DESTINATION_USER_ID": "your_id"
      }
    }
  }
}
```

Each MCP server image also works standalone via `docker run` in stdio mode — no gateway needed for Claude Desktop.

## Selecting Which Servers to Run

Set `MCP_SERVERS` in `.env`:

```bash
MCP_SERVERS=meta-ads              # only meta-ads
MCP_SERVERS=meta-ads,line-bot     # two servers
MCP_SERVERS=meta-ads,line-bot,coolify  # all three
```

To run different sets on different ports, add more gateway services in `docker-compose.yml`:

```yaml
services:
  gateway-all:
    image: docker/mcp-gateway
    ports: ["8599:8599"]
    command: [--servers=meta-ads,line-bot,coolify, --port=8599, ...]

  gateway-ads-only:
    image: docker/mcp-gateway
    ports: ["8600:8600"]
    command: [--servers=meta-ads, --port=8600, ...]
```

## Adding a New MCP Server

### 1. Create the server directory

```bash
mkdir servers/<name>
```

### 2. Write the Dockerfile

**Python package (uvx):**
```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-alpine
RUN uv tool install <package-name>
ENV PATH="/root/.local/bin:$PATH"
ENTRYPOINT ["<binary-name>"]
```

**Node.js package (npm):**
```dockerfile
FROM node:22-alpine
RUN npm install -g <package-name>
ENTRYPOINT ["<binary-name>"]
```

### 3. Write `servers/<name>/server.yaml`

```yaml
<name>:
  type: server
  title: Display Name
  description: What it does
  image: ghcr.io/thcloudai/<name>-mcp:latest
  secrets:
    - name: MY_API_KEY          # must match the key name in .env
      env: MY_API_KEY
      example: your_key_here
  allowHosts:
    - api.example.com:443
```

### 4. Add to `catalog.yaml`

**`catalog.yaml`** Copy entry from `server.yaml`.

**`catalog.yaml`** (dev) — same entry but image is `<name>-mcp:local`.

### 5. Add secrets to `.env.example` and `.env`

```bash
MY_API_KEY=your_key_here
```

### 6. Add to `MCP_SERVERS` in `.env`

```bash
MCP_SERVERS=meta-ads,line-bot,coolify,<name>
```

### 7. Push — CI/CD handles the rest

GitHub Actions detects the new `servers/<name>/` directory and builds + pushes the image to GHCR automatically.

## Using Official Docker MCP Catalog Servers

The [official catalog](http://desktop.docker.com/mcp/catalog/v2/catalog.yaml) includes brave, github, notion, stripe, playwright, and 50+ more. To use them alongside custom servers:

```yaml
# docker-compose.yml
command:
  - --catalog=/catalog.yaml
  - --additional-catalog=http://desktop.docker.com/mcp/catalog/v2/catalog.yaml
  - --servers=${MCP_SERVERS}
```

Official servers don't need a Dockerfile or `server.yaml` — just add their name to `MCP_SERVERS`.

## CI/CD

GitHub Actions (`.github/workflows/build.yml`) triggers on push to `main` when files under `servers/` change. Only the changed server is rebuilt — path-filtered matrix build.

Images pushed to GHCR:
- `ghcr.io/thcloudai/<name>-mcp:latest`
- `ghcr.io/thcloudai/<name>-mcp:<git-sha>`

PRs build images but do not push.

## Resources

- [Docker MCP Gateway docs](https://docs.docker.com/ai/mcp-catalog-and-toolkit/mcp-gateway/)
- [docker/mcp-gateway repo](https://github.com/docker/mcp-gateway)
- [docker/mcp-registry repo](https://github.com/docker/mcp-registry)
- [Gateway examples](https://github.com/docker/mcp-gateway/tree/main/examples)
- [Custom catalog example](https://github.com/docker/mcp-gateway/tree/main/examples/custom-catalog)
- [Official Docker MCP catalog](https://hub.docker.com/mcp)
- [Official catalog YAML](http://desktop.docker.com/mcp/catalog/v2/catalog.yaml)
