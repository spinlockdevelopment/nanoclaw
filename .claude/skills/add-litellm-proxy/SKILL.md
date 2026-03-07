---
name: add-litellm-proxy
description: Add LiteLLM API proxy for secret isolation and usage tracking. Agents get virtual keys instead of real API credentials. Supports local Docker container or external LiteLLM instance. Triggers on "litellm", "api proxy", "secret isolation", "virtual keys".
---

# Add LiteLLM API Proxy

This skill adds a LiteLLM proxy between NanoClaw agents and the Anthropic API. Agents never see real API keys — they get virtual keys that the proxy validates and routes. This provides:

- **Secret isolation**: Real API keys live only in the proxy, never in agent containers
- **Usage tracking**: Per-group cost and token usage via LiteLLM dashboard
- **Rate limiting**: Per-group rate limits to prevent runaway agents
- **Model routing**: Route different groups to different models or providers
- **Key revocation**: Instantly revoke a group's access by invalidating its virtual key

## Architecture

```
┌────────────────────────────────────────────┐
│  Docker Network: nanoclaw-internal         │
│                                            │
│  ┌──────────┐    ┌──────────────────┐      │
│  │ Agent A  │─vk→│  LiteLLM Proxy   │     │
│  │ (group)  │    │                  │      │
│  └──────────┘    │  Real API keys   │──────┼──→ api.anthropic.com
│  ┌──────────┐    │  Virtual keys    │      │
│  │ Agent B  │─vk→│  Usage tracking  │──────┼──→ api.openai.com
│  │ (group)  │    │  Rate limiting   │      │
│  └──────────┘    └──────────────────┘      │
└────────────────────────────────────────────┘
```

## Phase 1: Pre-flight

### Choose deployment mode

Use `AskUserQuestion`:

> How would you like to run LiteLLM?
>
> **Option A: Local container** — LiteLLM runs as a Docker container alongside your agents on a private Docker network. Best for single-machine setups.
>
> **Option B: External instance** — Point to an existing LiteLLM instance running elsewhere on your network. Best for multi-machine setups or if you already run LiteLLM.

Wait for the user's choice.

### For external mode: collect configuration

Ask the user for:
1. LiteLLM proxy URL (e.g., `http://192.168.1.100:4000`)
2. LiteLLM API key (the proxy key or master key)

Skip to Phase 3.

### For local mode: proceed to Phase 2.

## Phase 2: Local Container Setup

### Step 1: Create LiteLLM configuration directory

```bash
mkdir -p litellm
```

### Step 2: Create LiteLLM config

Read the user's current `.env` to determine which providers they use. Create `litellm/config.yaml`:

```yaml
model_list:
  # Anthropic models (primary)
  - model_name: claude-sonnet-4-20250514
    litellm_params:
      model: anthropic/claude-sonnet-4-20250514
      api_key: os.environ/ANTHROPIC_API_KEY

  - model_name: claude-sonnet-4-20250514
    litellm_params:
      model: anthropic/claude-sonnet-4-20250514
      api_key: os.environ/ANTHROPIC_API_KEY

  - model_name: claude-haiku-4-20250414
    litellm_params:
      model: anthropic/claude-haiku-4-20250414
      api_key: os.environ/ANTHROPIC_API_KEY

  # Add more models as needed. LiteLLM supports 100+ providers:
  # https://docs.litellm.ai/docs/providers

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  # Store virtual keys in SQLite (persists across restarts)
  database_url: "sqlite:///data/litellm.db"

litellm_settings:
  # Log all requests for observability
  success_callback: ["langfuse"]  # Optional: remove if not using Langfuse
  # Drop params not supported by the provider
  drop_params: true
  # Set default max tokens
  max_tokens: 8192
```

If the user has `ANTHROPIC_AUTH_TOKEN` or `CLAUDE_CODE_OAUTH_TOKEN` instead of `ANTHROPIC_API_KEY`, adjust the config accordingly. Note: LiteLLM works with API keys, not OAuth tokens. If the user only has OAuth, tell them they'll need an API key for the proxy.

### Step 3: Create proxy management script

Create `litellm/manage.sh`:

```bash
#!/bin/bash
# LiteLLM Proxy Management for NanoClaw
set -euo pipefail

NETWORK="nanoclaw-internal"
CONTAINER_NAME="nanoclaw-litellm"
LITELLM_IMAGE="ghcr.io/berriai/litellm:main-latest"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PROJECT_DIR="$(dirname "$SCRIPT_DIR")"

# Load environment
if [ -f "$PROJECT_DIR/.env" ]; then
  set -a
  source "$PROJECT_DIR/.env"
  set +a
fi

ensure_network() {
  if ! docker network inspect "$NETWORK" >/dev/null 2>&1; then
    echo "Creating Docker network: $NETWORK"
    docker network create "$NETWORK"
  fi
}

start() {
  ensure_network

  if docker ps --format '{{.Names}}' | grep -q "^${CONTAINER_NAME}$"; then
    echo "LiteLLM proxy already running"
    return 0
  fi

  # Stop any existing stopped container
  docker rm -f "$CONTAINER_NAME" 2>/dev/null || true

  echo "Starting LiteLLM proxy..."
  docker run -d \
    --name "$CONTAINER_NAME" \
    --network "$NETWORK" \
    --restart unless-stopped \
    -v "$SCRIPT_DIR/config.yaml:/app/config.yaml:ro" \
    -v "$SCRIPT_DIR/data:/data" \
    -e "ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY:-}" \
    -e "LITELLM_MASTER_KEY=${LITELLM_MASTER_KEY:-}" \
    -e "DATABASE_URL=sqlite:///data/litellm.db" \
    -p "${LITELLM_PORT:-4000}:4000" \
    "$LITELLM_IMAGE" \
    --config /app/config.yaml --port 4000

  # Also connect to default bridge so it can reach external APIs
  docker network connect bridge "$CONTAINER_NAME" 2>/dev/null || true

  echo "LiteLLM proxy started on port ${LITELLM_PORT:-4000}"
  echo "Dashboard: http://localhost:${LITELLM_PORT:-4000}/ui"

  # Wait for healthy
  echo -n "Waiting for proxy to be ready"
  for i in $(seq 1 30); do
    if curl -sf "http://localhost:${LITELLM_PORT:-4000}/health" >/dev/null 2>&1; then
      echo " ready!"
      return 0
    fi
    echo -n "."
    sleep 1
  done
  echo " timeout (check: docker logs $CONTAINER_NAME)"
  return 1
}

stop() {
  echo "Stopping LiteLLM proxy..."
  docker stop "$CONTAINER_NAME" 2>/dev/null || true
  docker rm "$CONTAINER_NAME" 2>/dev/null || true
  echo "LiteLLM proxy stopped"
}

status() {
  if docker ps --format '{{.Names}}' | grep -q "^${CONTAINER_NAME}$"; then
    echo "LiteLLM proxy: running"
    docker ps --filter "name=$CONTAINER_NAME" --format "  Image: {{.Image}}\n  Status: {{.Status}}\n  Ports: {{.Ports}}"
  else
    echo "LiteLLM proxy: stopped"
  fi
}

generate_key() {
  local team_id="${1:-default}"
  local budget="${2:-100.0}"
  local proxy_url="http://localhost:${LITELLM_PORT:-4000}"

  if [ -z "${LITELLM_MASTER_KEY:-}" ]; then
    echo "Error: LITELLM_MASTER_KEY not set in .env"
    exit 1
  fi

  echo "Generating virtual key for team: $team_id (budget: \$${budget})"
  curl -sf -X POST "$proxy_url/key/generate" \
    -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"team_id\": \"$team_id\", \"max_budget\": $budget, \"metadata\": {\"nanoclaw_group\": \"$team_id\"}}" \
    | python3 -m json.tool 2>/dev/null || \
    curl -sf -X POST "$proxy_url/key/generate" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
      -H "Content-Type: application/json" \
      -d "{\"team_id\": \"$team_id\", \"max_budget\": $budget, \"metadata\": {\"nanoclaw_group\": \"$team_id\"}}"
}

list_keys() {
  local proxy_url="http://localhost:${LITELLM_PORT:-4000}"
  curl -sf "$proxy_url/key/list" \
    -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
    | python3 -m json.tool 2>/dev/null || echo "Failed to list keys. Is the proxy running?"
}

case "${1:-help}" in
  start)   start ;;
  stop)    stop ;;
  restart) stop; start ;;
  status)  status ;;
  key)     generate_key "${2:-}" "${3:-100.0}" ;;
  keys)    list_keys ;;
  help)
    echo "Usage: $0 {start|stop|restart|status|key <team_id> [budget]|keys}"
    echo ""
    echo "Commands:"
    echo "  start              Start LiteLLM proxy container"
    echo "  stop               Stop and remove proxy container"
    echo "  restart            Restart proxy container"
    echo "  status             Show proxy status"
    echo "  key <team> [budget] Generate a virtual key for a team/group"
    echo "  keys               List all virtual keys"
    ;;
  *) echo "Unknown command: $1. Run '$0 help' for usage." ;;
esac
```

Make it executable:

```bash
chmod +x litellm/manage.sh
```

### Step 4: Create data directory for LiteLLM persistence

```bash
mkdir -p litellm/data
```

### Step 5: Generate master key

Generate a random master key and add to `.env`:

```bash
MASTER_KEY="sk-nanoclaw-$(openssl rand -hex 16)"
```

Add to `.env`:

```bash
# LiteLLM Proxy
LITELLM_MASTER_KEY=<generated-master-key>
LITELLM_PROXY_URL=http://nanoclaw-litellm:4000
LITELLM_MODE=local
```

### Step 6: Start the proxy

```bash
./litellm/manage.sh start
```

Verify it's healthy:

```bash
curl -sf http://localhost:4000/health
```

## Phase 3: Integrate with Container Runner

### Step 1: Update configuration

Read `src/config.ts` and add after the existing env config reads:

```typescript
const proxyConfig = readEnvFile(['LITELLM_PROXY_URL', 'LITELLM_API_KEY', 'LITELLM_MASTER_KEY', 'LITELLM_MODE']);

// LiteLLM proxy: when configured, agents connect to the proxy instead of Anthropic directly
export const LITELLM_PROXY_URL = process.env.LITELLM_PROXY_URL || proxyConfig.LITELLM_PROXY_URL || '';
export const LITELLM_API_KEY = process.env.LITELLM_API_KEY || proxyConfig.LITELLM_API_KEY || '';
export const LITELLM_MASTER_KEY = process.env.LITELLM_MASTER_KEY || proxyConfig.LITELLM_MASTER_KEY || '';
export const LITELLM_MODE = process.env.LITELLM_MODE || proxyConfig.LITELLM_MODE || ''; // 'local' or 'external'

// Docker network for container-to-proxy communication
export const NANOCLAW_NETWORK = process.env.NANOCLAW_NETWORK || 'nanoclaw-internal';
```

### Step 2: Update container-runner.ts — secret override

Read `src/container-runner.ts` and modify the `readSecrets()` function. Import the new config values at the top:

```typescript
import {
  CONTAINER_IMAGE,
  CONTAINER_MAX_OUTPUT_SIZE,
  CONTAINER_TIMEOUT,
  DATA_DIR,
  GROUPS_DIR,
  IDLE_TIMEOUT,
  LITELLM_API_KEY,
  LITELLM_MASTER_KEY,
  LITELLM_MODE,
  LITELLM_PROXY_URL,
  NANOCLAW_NETWORK,
  TIMEZONE,
} from './config.js';
```

Replace the `readSecrets()` function:

```typescript
/**
 * Read allowed secrets from .env for passing to the container via stdin.
 * Secrets are never written to disk or mounted as files.
 *
 * When LiteLLM proxy is configured, agents get the proxy URL and a proxy key
 * instead of direct API credentials. The real API keys stay in the proxy.
 */
function readSecrets(group?: RegisteredGroup): Record<string, string> {
  if (LITELLM_PROXY_URL) {
    // Proxy mode: agents connect to LiteLLM, never see real API keys
    const secrets: Record<string, string> = {};

    // Point the SDK at the proxy
    secrets.ANTHROPIC_BASE_URL = LITELLM_PROXY_URL;

    // Use per-group virtual key if configured, otherwise use shared key
    const groupKey = group?.containerConfig?.litellmApiKey;
    const proxyKey = groupKey || LITELLM_API_KEY || LITELLM_MASTER_KEY;
    if (proxyKey) {
      secrets.ANTHROPIC_API_KEY = proxyKey;
    }

    // Do NOT pass CLAUDE_CODE_OAUTH_TOKEN — proxy uses API keys
    return secrets;
  }

  // Direct mode: pass real credentials
  return readEnvFile([
    'CLAUDE_CODE_OAUTH_TOKEN',
    'ANTHROPIC_API_KEY',
    'ANTHROPIC_BASE_URL',
    'ANTHROPIC_AUTH_TOKEN',
  ]);
}
```

Update the call site in `runContainerAgent` to pass the group:

```typescript
input.secrets = readSecrets(group);
```

### Step 3: Update container-runner.ts — network flag

Read `src/container-runner.ts` and in the `buildContainerArgs()` function, add the network flag after the `--name` argument:

```typescript
function buildContainerArgs(
  mounts: VolumeMount[],
  containerName: string,
): string[] {
  const args: string[] = ['run', '-i', '--rm', '--name', containerName];

  // Connect to isolated Docker network when proxy is configured (local mode)
  if (LITELLM_MODE === 'local' && LITELLM_PROXY_URL) {
    args.push('--network', NANOCLAW_NETWORK);
  }

  // Pass host timezone so container's local time matches the user's
  args.push('-e', `TZ=${TIMEZONE}`);
  // ... rest of function unchanged
```

### Step 4: Update the sanitize bash hook

Read `container/agent-runner/src/index.ts` and update the `SECRET_ENV_VARS` array to also strip any LiteLLM-related vars from Bash subprocess environments:

```typescript
const SECRET_ENV_VARS = [
  'ANTHROPIC_API_KEY',
  'CLAUDE_CODE_OAUTH_TOKEN',
  'ANTHROPIC_AUTH_TOKEN',
  'LITELLM_MASTER_KEY',
  'LITELLM_API_KEY',
];
```

### Step 5: Update .env.example

Add to `.env.example`:

```bash
# === LiteLLM API Proxy ===
# When configured, agents connect to LiteLLM instead of Anthropic directly.
# Real API keys stay in the proxy; agents get virtual keys.
#
# Mode: 'local' = Docker container alongside agents, 'external' = remote instance
# LITELLM_MODE=local
#
# Proxy URL (for local: use container name; for external: use IP/hostname)
# LITELLM_PROXY_URL=http://nanoclaw-litellm:4000
#
# Master key for the LiteLLM admin API (generate with: openssl rand -hex 16)
# LITELLM_MASTER_KEY=sk-nanoclaw-<random>
#
# Shared API key for all agents (or use per-group virtual keys via container_config)
# LITELLM_API_KEY=
#
# Docker network name (shared between proxy and agents)
# NANOCLAW_NETWORK=nanoclaw-internal
```

### Step 6: Sync environment and rebuild

```bash
mkdir -p data/env && cp .env data/env/env
npm run build
```

## Phase 4: Verify

### Test the proxy

For local mode:

```bash
./litellm/manage.sh status
```

Test a direct API call through the proxy:

```bash
curl -sf http://localhost:4000/v1/messages \
  -H "x-api-key: $LITELLM_MASTER_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model": "claude-sonnet-4-20250514", "max_tokens": 50, "messages": [{"role": "user", "content": "Say hi"}]}'
```

### Test agent with proxy

Tell the user:

> Send a message to your agent and check the logs for proxy routing:
>
> ```bash
> tail -f logs/nanoclaw.log | grep -i "proxy\|litellm\|secret"
> ```
>
> The agent should respond normally. Verify in LiteLLM dashboard (http://localhost:4000/ui) that the request was logged.

### Generate per-group virtual keys (optional)

```bash
./litellm/manage.sh key main 50.0
./litellm/manage.sh key telegram_general 25.0
```

Store per-group keys in the database by updating the group's `container_config`:

```sql
UPDATE registered_groups
SET container_config = json_set(COALESCE(container_config, '{}'), '$.litellmApiKey', 'sk-generated-key-here')
WHERE folder = 'group-folder-name';
```

## Phase 5: Service Integration

### Add proxy to startup/shutdown

For macOS, update the launchd plist or create a separate one for LiteLLM. The simplest approach is to add proxy startup to a wrapper script.

Create `scripts/start-with-proxy.sh`:

```bash
#!/bin/bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PROJECT_DIR="$(dirname "$SCRIPT_DIR")"

# Start LiteLLM proxy if in local mode
if grep -q 'LITELLM_MODE=local' "$PROJECT_DIR/.env" 2>/dev/null; then
  "$PROJECT_DIR/litellm/manage.sh" start
fi

# Start NanoClaw
cd "$PROJECT_DIR"
exec node dist/index.js
```

```bash
chmod +x scripts/start-with-proxy.sh
```

Update the launchd plist or systemd service to use this wrapper.

## Troubleshooting

### Agent gets "authentication error"

1. Check proxy is running: `./litellm/manage.sh status`
2. Verify API key in proxy config: `cat litellm/config.yaml`
3. Test proxy directly: `curl http://localhost:4000/health`
4. Check if using virtual key — verify it exists: `./litellm/manage.sh keys`

### Agent can't reach proxy (connection refused)

1. For local mode: verify network exists: `docker network inspect nanoclaw-internal`
2. Check proxy container is on the network: `docker inspect nanoclaw-litellm | grep -A5 Networks`
3. Check agent container is on the network: `docker inspect <agent-container> | grep -A5 Networks`
4. For external mode: verify URL is reachable from host: `curl $LITELLM_PROXY_URL/health`

### Per-group keys not working

1. Verify key was stored: `sqlite3 store/messages.db "SELECT container_config FROM registered_groups WHERE folder='<group>'"`
2. Check the key is valid: `curl -H "Authorization: Bearer <key>" http://localhost:4000/key/info`
3. Rebuild after DB changes: `npm run build && launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `systemctl --user restart nanoclaw` (Linux)

## Removal

To remove the LiteLLM proxy integration:

1. Stop the proxy: `./litellm/manage.sh stop`
2. Remove the Docker network: `docker network rm nanoclaw-internal`
3. Remove LiteLLM env vars from `.env`: `LITELLM_PROXY_URL`, `LITELLM_API_KEY`, `LITELLM_MASTER_KEY`, `LITELLM_MODE`, `NANOCLAW_NETWORK`
4. Revert `src/config.ts`: remove LiteLLM config exports
5. Revert `src/container-runner.ts`: restore original `readSecrets()` and `buildContainerArgs()`
6. Remove `litellm/` directory
7. Remove `scripts/start-with-proxy.sh`
8. Sync and rebuild: `cp .env data/env/env && npm run build`
9. Restart: `launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `systemctl --user restart nanoclaw` (Linux)
