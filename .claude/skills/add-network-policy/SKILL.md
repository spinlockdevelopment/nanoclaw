---
name: add-network-policy
description: Add container network isolation and egress control. Creates a private Docker network for agents with configurable egress allowlist. Detects LiteLLM proxy and auto-configures. Triggers on "network policy", "egress", "firewall", "network isolation", "container network".
---

# Add Network Policy (Container Egress Control)

This skill adds network isolation for NanoClaw agent containers. By default, agents have full internet access via Docker's bridge network. This skill restricts egress to only allowlisted destinations.

**What it does:**
- Creates an isolated Docker network (`nanoclaw-internal`) with no default internet access
- Agent containers are placed on this network — they can only reach allowlisted services
- If LiteLLM proxy is detected, it's automatically allowlisted
- Configurable egress allowlist for additional services (stored outside project root)

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  nanoclaw-internal (--internal = no default egress)  │
│                                                      │
│  ┌──────────┐    ┌──────────────────┐                │
│  │ Agent A  │───→│  LiteLLM Proxy   │────→ Internet  │
│  └──────────┘    │  (dual-homed)    │  (via bridge)  │
│  ┌──────────┐    └──────────────────┘                │
│  │ Agent B  │───→ (blocked)                          │
│  └──────────┘                                        │
│  ┌──────────┐    ┌──────────────────┐                │
│  │ Agent C  │───→│  Egress Proxy    │────→ Allowlist  │
│  └──────────┘    │  (optional)      │  (filtered)    │
│                  └──────────────────┘                │
└─────────────────────────────────────────────────────┘
```

**Two isolation levels:**

1. **Basic (no egress proxy)**: Agents can only reach other containers on `nanoclaw-internal`. No internet. LiteLLM handles API calls. WebSearch/WebFetch tools won't work.

2. **With egress proxy (future enhancement)**: A Squid/Envoy proxy on the internal network filters outbound requests against an allowlist. WebSearch/WebFetch route through this proxy.

This skill implements Level 1 (basic). Level 2 can be added later as `/add-egress-proxy`.

## Phase 1: Pre-flight

### Detect existing configuration

Check for existing network and LiteLLM setup:

```bash
docker network inspect nanoclaw-internal 2>/dev/null && echo "Network exists" || echo "Network does not exist"
docker ps --filter name=nanoclaw-litellm --format '{{.Names}}' 2>/dev/null
```

Read `.env` to check if `LITELLM_PROXY_URL` and `LITELLM_MODE` are configured.

### Ask the user about egress policy

Use `AskUserQuestion`:

> How should I configure network egress for agent containers?
>
> **Option A: Full lockdown** — Agents can ONLY reach the LiteLLM proxy (requires `/add-litellm-proxy`). WebSearch and WebFetch tools will not work. Best security.
>
> **Option B: Allowlisted egress** — Agents can reach the LiteLLM proxy AND specific allowlisted domains (e.g., api.anthropic.com, google.com for web search). You'll configure the allowlist.
>
> **Option C: Proxy-only API, open web** — API calls go through the proxy, but agents retain full internet access for WebSearch/WebFetch. Moderate security — isolates API keys but not network access.

Wait for the user's choice.

If they choose Option A and LiteLLM is not set up, tell them:

> Full lockdown requires the LiteLLM proxy to be configured first. Would you like me to set up LiteLLM now (run `/add-litellm-proxy`), or would you prefer a different egress policy?

## Phase 2: Create Network Infrastructure

### Step 1: Create the Docker network

For Option A (full lockdown) or Option B (allowlisted):

```bash
# Remove existing network if it exists (no containers attached)
docker network rm nanoclaw-internal 2>/dev/null || true

# Create internal network (no internet gateway)
docker network create --internal nanoclaw-internal
```

The `--internal` flag is key — it creates a network with no default route to the internet. Containers on this network can only reach each other.

For Option C (proxy API, open web):

```bash
# Regular network (has internet via Docker gateway)
docker network create nanoclaw-internal 2>/dev/null || true
```

### Step 2: Connect LiteLLM proxy to the network

If LiteLLM is running in local mode, it needs to be on both networks:
- `nanoclaw-internal` — so agents can reach it
- `bridge` (default) — so it can reach external APIs

```bash
# Connect LiteLLM to the internal network (if not already)
docker network connect nanoclaw-internal nanoclaw-litellm 2>/dev/null || true

# Ensure it's also on bridge for API access
docker network connect bridge nanoclaw-litellm 2>/dev/null || true
```

### Step 3: Create egress allowlist configuration

Create `~/.config/nanoclaw/egress-policy.json`:

For Option A:
```json
{
  "mode": "lockdown",
  "network": "nanoclaw-internal",
  "networkInternal": true,
  "allowlist": [],
  "notes": "Full lockdown: agents can only reach containers on nanoclaw-internal (LiteLLM proxy)"
}
```

For Option B — ask the user which domains to allow, then:
```json
{
  "mode": "allowlist",
  "network": "nanoclaw-internal",
  "networkInternal": true,
  "allowlist": [
    "api.anthropic.com",
    "api.openai.com",
    "www.google.com",
    "*.googleapis.com"
  ],
  "notes": "Allowlisted egress: agents can reach listed domains and containers on nanoclaw-internal"
}
```

For Option C:
```json
{
  "mode": "proxy-only",
  "network": "nanoclaw-internal",
  "networkInternal": false,
  "allowlist": ["*"],
  "notes": "Proxy-only mode: API calls through LiteLLM, agents retain internet access for WebSearch/WebFetch"
}
```

## Phase 3: Integrate with Container Runner

### Step 1: Update configuration

Read `src/config.ts`. If the LiteLLM skill hasn't already added `NANOCLAW_NETWORK`, add it:

```typescript
export const NANOCLAW_NETWORK = process.env.NANOCLAW_NETWORK || 'nanoclaw-internal';
```

Add the egress policy path:

```typescript
export const EGRESS_POLICY_PATH = path.join(
  HOME_DIR,
  '.config',
  'nanoclaw',
  'egress-policy.json',
);
```

### Step 2: Create network policy module

Create `src/network-policy.ts`:

```typescript
/**
 * Network policy enforcement for NanoClaw containers.
 * Reads egress policy from external config (tamper-proof from containers).
 */
import fs from 'fs';
import { execSync } from 'child_process';

import { EGRESS_POLICY_PATH, NANOCLAW_NETWORK } from './config.js';
import { logger } from './logger.js';

export interface EgressPolicy {
  mode: 'lockdown' | 'allowlist' | 'proxy-only' | 'open';
  network: string;
  networkInternal: boolean;
  allowlist: string[];
}

const DEFAULT_POLICY: EgressPolicy = {
  mode: 'open',
  network: '',
  networkInternal: false,
  allowlist: ['*'],
};

let cachedPolicy: EgressPolicy | null = null;

export function loadEgressPolicy(): EgressPolicy {
  if (cachedPolicy) return cachedPolicy;

  try {
    if (fs.existsSync(EGRESS_POLICY_PATH)) {
      const raw = JSON.parse(fs.readFileSync(EGRESS_POLICY_PATH, 'utf-8'));
      cachedPolicy = {
        mode: raw.mode || 'open',
        network: raw.network || NANOCLAW_NETWORK,
        networkInternal: raw.networkInternal ?? false,
        allowlist: raw.allowlist || ['*'],
      };
      logger.info(
        { mode: cachedPolicy.mode, network: cachedPolicy.network },
        'Egress policy loaded',
      );
      return cachedPolicy;
    }
  } catch (err) {
    logger.warn({ err }, 'Failed to load egress policy, using defaults');
  }

  cachedPolicy = DEFAULT_POLICY;
  return cachedPolicy;
}

/**
 * Get Docker network args for container based on egress policy.
 * Returns empty array if no network policy is configured.
 */
export function getNetworkArgs(): string[] {
  const policy = loadEgressPolicy();
  if (!policy.network) return [];
  return ['--network', policy.network];
}

/**
 * Ensure the Docker network exists, creating it if needed.
 * Called once at startup.
 */
export function ensureNetwork(): void {
  const policy = loadEgressPolicy();
  if (!policy.network) return;

  try {
    execSync(`docker network inspect ${policy.network}`, {
      stdio: 'pipe',
      timeout: 5000,
    });
    logger.debug({ network: policy.network }, 'Docker network exists');
  } catch {
    logger.info({ network: policy.network, internal: policy.networkInternal }, 'Creating Docker network');
    const internalFlag = policy.networkInternal ? '--internal' : '';
    execSync(`docker network create ${internalFlag} ${policy.network}`, {
      stdio: 'pipe',
      timeout: 10000,
    });
    logger.info({ network: policy.network }, 'Docker network created');
  }
}

/**
 * Reload the egress policy (e.g., after config change).
 */
export function reloadEgressPolicy(): void {
  cachedPolicy = null;
  loadEgressPolicy();
}
```

### Step 3: Update container-runner.ts — network args

Read `src/container-runner.ts` and add the import:

```typescript
import { getNetworkArgs } from './network-policy.js';
```

In `buildContainerArgs()`, add network args. If the LiteLLM skill already added a network check, replace it with the generalized version:

Find the section in `buildContainerArgs` that builds the args array. After the `--name` argument (and removing any existing LiteLLM-specific network logic), add:

```typescript
// Apply network policy (egress control)
args.push(...getNetworkArgs());
```

This replaces any existing `LITELLM_MODE` network check. The network policy module handles all cases:
- If no policy is configured → no network args (default Docker bridge)
- If policy exists → uses the configured network

### Step 4: Initialize network at startup

Read `src/index.ts` and add the import:

```typescript
import { ensureNetwork } from './network-policy.js';
```

In the startup sequence (inside `main()`), after `ensureContainerRuntimeRunning()` and before the channel connections, add:

```typescript
ensureNetwork();
```

### Step 5: Update .env.example

Add to `.env.example` if not already present:

```bash
# === Network Policy ===
# Docker network for container isolation (created automatically if egress policy exists)
# NANOCLAW_NETWORK=nanoclaw-internal
```

### Step 6: Build

```bash
npm run build
```

## Phase 4: Verify

### Test network isolation

Start a test container on the internal network:

```bash
docker run --rm --network nanoclaw-internal curlimages/curl curl -sf --connect-timeout 5 http://api.anthropic.com 2>&1 || echo "BLOCKED (expected)"
```

This should fail with a connection error — the `--internal` network has no internet gateway.

### Test proxy reachability (if LiteLLM is set up)

```bash
docker run --rm --network nanoclaw-internal curlimages/curl curl -sf --connect-timeout 5 http://nanoclaw-litellm:4000/health
```

This should succeed — containers on the same network can reach each other.

### Test agent functionality

Tell the user:

> Send a message to your agent. It should respond normally via the LiteLLM proxy.
>
> Then try: "Search the web for today's news"
>
> In lockdown mode, WebSearch will fail (no internet). In proxy-only mode, it will work normally.
>
> Check logs: `tail -f logs/nanoclaw.log | grep -i network`

### Verify isolation

```bash
# Check the network configuration
docker network inspect nanoclaw-internal

# Check which containers are on the network
docker network inspect nanoclaw-internal --format '{{range .Containers}}{{.Name}} {{end}}'
```

## Phase 5: WebSearch/WebFetch Considerations

### Current behavior by mode

| Mode | API Calls | WebSearch | WebFetch |
|------|-----------|-----------|----------|
| Lockdown | Via LiteLLM | Blocked | Blocked |
| Allowlist | Via LiteLLM | Depends on allowlist | Depends on allowlist |
| Proxy-only | Via LiteLLM | Works | Works |
| Open (default) | Direct | Works | Works |

### Future enhancement: /add-egress-proxy

For allowlisted web access in lockdown mode, a future skill could add a Squid HTTP proxy on the internal network:

```
Agent → Squid Proxy (on nanoclaw-internal) → Allowlisted domains only
```

The proxy would:
1. Run as a Docker container on `nanoclaw-internal` with dual-homing
2. Filter outbound requests against the domain allowlist
3. Agents' WebSearch/WebFetch would use `HTTP_PROXY` environment variable

This is noted here for future implementation. LiteLLM can also proxy web search if the model supports it.

### WebSearch through LiteLLM

LiteLLM supports web search as a model capability. If using Claude models that support web search, the search requests go through LiteLLM's API — no direct internet access needed. This is the recommended approach for lockdown mode.

## Troubleshooting

### Agent can't connect to anything

1. Check network exists: `docker network inspect nanoclaw-internal`
2. Check agent is on the network: look for `--network nanoclaw-internal` in container logs
3. Check egress policy: `cat ~/.config/nanoclaw/egress-policy.json`
4. Verify LiteLLM is on the network: `docker inspect nanoclaw-litellm | grep -A10 Networks`

### WebSearch stopped working after enabling network policy

Expected in lockdown mode. Options:
1. Switch to proxy-only mode (edit `~/.config/nanoclaw/egress-policy.json`, set `"mode": "proxy-only"`, `"networkInternal": false`)
2. Use LiteLLM's built-in web search capability
3. Wait for `/add-egress-proxy` skill

### Network already exists error

```bash
docker network rm nanoclaw-internal
# Then re-run setup
```

Note: all containers on the network must be stopped first.

### LiteLLM not reachable from agents

1. Verify LiteLLM is on the internal network:
   ```bash
   docker network inspect nanoclaw-internal --format '{{range .Containers}}{{.Name}} {{end}}'
   ```
2. If not, reconnect:
   ```bash
   docker network connect nanoclaw-internal nanoclaw-litellm
   ```
3. Restart NanoClaw to pick up changes

## Removal

To remove the network policy:

1. Remove egress policy: `rm ~/.config/nanoclaw/egress-policy.json`
2. Remove `src/network-policy.ts`
3. Remove `EGRESS_POLICY_PATH` from `src/config.ts`
4. Remove `ensureNetwork()` call from `src/index.ts`
5. Remove `getNetworkArgs()` import and call from `src/container-runner.ts`
6. Remove the Docker network: `docker network rm nanoclaw-internal`
7. Rebuild: `npm run build`
8. Restart: `launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `systemctl --user restart nanoclaw` (Linux)
