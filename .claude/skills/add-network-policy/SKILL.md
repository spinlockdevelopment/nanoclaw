---
name: add-network-policy
description: Add container network isolation and egress control. Creates a private Docker network with an allowlist model. Empty allowlist = air-gapped, add LiteLLM = proxy-only, add domains = selective access, wildcard = open. Triggers on "network policy", "egress", "firewall", "network isolation", "container network".
---

# Add Network Policy (Container Egress Control)

This skill adds network isolation for NanoClaw agent containers using a single, unified allowlist model.

## How It Works

One concept: **an allowlist of what agents can reach.**

| Allowlist | Docker Network | Effect | Squid Needed? |
|-----------|---------------|--------|---------------|
| `[]` (empty) | `--internal` | Air-gapped. No internet, no proxy. | No |
| `["nanoclaw-litellm"]` | `--internal` | Proxy-only. API via LiteLLM, no web. | No |
| `["nanoclaw-litellm", "google.com", ...]` | `--internal` | Filtered web. Needs egress proxy. | **Yes** (`/add-egress-proxy`) |
| `["*"]` | regular | Full internet. Default Docker behavior. | No |

**Docker's `--internal` flag** creates a network with zero internet gateway. Containers can only talk to other containers on the same network. Services that need internet (LiteLLM, Squid) are "dual-homed" — connected to both the internal network and the default bridge.

Without `/add-egress-proxy`, domain-level filtering is not available — the allowlist entries are informational for containers and dual-homed services. The isolation is binary: `--internal` (no internet) or regular (full internet). Squid adds the gradient between those two states.

## Phase 1: Pre-flight

### Detect existing setup

```bash
docker network inspect nanoclaw-internal 2>/dev/null && echo "Network exists" || echo "Network does not exist"
docker ps --filter name=nanoclaw-litellm --format '{{.Names}}' 2>/dev/null
```

Read `.env` to check for `LITELLM_PROXY_URL`.

### Collect the allowlist

Use `AskUserQuestion`:

> What should agent containers be allowed to reach?
>
> **Option A: Nothing (air-gapped)** — Containers have zero network access. Agents can only read/write local files and use IPC. Use if agents don't need API calls at all.
>
> **Option B: LiteLLM proxy only** — Containers can reach the LiteLLM proxy for API calls but nothing else. WebSearch/WebFetch won't work. Best security for most setups.
>
> **Option C: Full internet** — Containers retain full internet access (default Docker behavior). API calls still route through LiteLLM if configured. Least restrictive.

Wait for the user's choice.

If they choose Option B and LiteLLM is not set up:

> Proxy-only mode requires the LiteLLM proxy. Would you like me to set it up now (`/add-litellm-proxy`)?

Build the allowlist based on their choice:
- Option A: `[]`
- Option B: `["nanoclaw-litellm"]`
- Option C: `["*"]`

## Phase 2: Create Network Infrastructure

### Step 1: Create the Docker network

Determine network type from allowlist:

```bash
# If allowlist is ["*"], create a regular network (full internet)
# Otherwise, create an --internal network (no internet gateway)
```

For `["*"]` (full internet):

```bash
docker network create nanoclaw-internal 2>/dev/null || true
```

For anything else (empty or specific entries):

```bash
# Remove existing network if type needs to change
docker network rm nanoclaw-internal 2>/dev/null || true

# Create internal network (no internet gateway)
docker network create --internal nanoclaw-internal
```

### Step 2: Dual-home allowlisted services

Any container in the allowlist that needs internet access must be on both networks:

```bash
# If LiteLLM is in the allowlist and running locally
docker network connect nanoclaw-internal nanoclaw-litellm 2>/dev/null || true
docker network connect bridge nanoclaw-litellm 2>/dev/null || true
```

This is what makes the allowlist work without Squid for container-to-container services: the dual-homed service acts as a bridge. Agents on `--internal` can reach LiteLLM. LiteLLM (also on bridge) can reach the internet.

### Step 3: Write the egress policy

Create `~/.config/nanoclaw/egress-policy.json`:

```json
{
  "allowlist": ["nanoclaw-litellm"],
  "network": "nanoclaw-internal",
  "dualHomedServices": ["nanoclaw-litellm"]
}
```

The `dualHomedServices` array lists containers that should be connected to both networks at startup. This is auto-populated from the allowlist (only entries that match running container names).

- `allowlist: []` → `--internal` network, nothing reachable
- `allowlist: ["nanoclaw-litellm"]` → `--internal` network, LiteLLM dual-homed
- `allowlist: ["*"]` → regular network, full internet

## Phase 3: Integrate with Container Runner

### Step 1: Update configuration

Read `src/config.ts`. If not already present from the LiteLLM skill, add:

```typescript
export const NANOCLAW_NETWORK = process.env.NANOCLAW_NETWORK || 'nanoclaw-internal';
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
 * Single model: an allowlist of reachable services/domains.
 *
 * - Empty allowlist → --internal Docker network (air-gapped)
 * - ["nanoclaw-litellm"] → --internal + LiteLLM dual-homed (proxy-only)
 * - ["*"] → regular Docker network (full internet)
 * - Specific domains → --internal + Squid proxy (requires /add-egress-proxy)
 *
 * Config stored outside project root (tamper-proof from containers):
 *   ~/.config/nanoclaw/egress-policy.json
 */
import fs from 'fs';
import { execSync } from 'child_process';

import { EGRESS_POLICY_PATH, NANOCLAW_NETWORK } from './config.js';
import { logger } from './logger.js';

export interface EgressPolicy {
  allowlist: string[];
  network: string;
  dualHomedServices: string[];
}

const DEFAULT_POLICY: EgressPolicy = {
  allowlist: ['*'],
  network: '',
  dualHomedServices: [],
};

let cachedPolicy: EgressPolicy | null = null;

export function loadEgressPolicy(): EgressPolicy {
  if (cachedPolicy) return cachedPolicy;

  try {
    if (fs.existsSync(EGRESS_POLICY_PATH)) {
      const raw = JSON.parse(fs.readFileSync(EGRESS_POLICY_PATH, 'utf-8'));
      cachedPolicy = {
        allowlist: raw.allowlist || ['*'],
        network: raw.network || NANOCLAW_NETWORK,
        dualHomedServices: raw.dualHomedServices || [],
      };
      logger.info(
        {
          network: cachedPolicy.network,
          allowlistSize: cachedPolicy.allowlist.length,
          isOpen: cachedPolicy.allowlist.includes('*'),
        },
        'Egress policy loaded',
      );
      return cachedPolicy;
    }
  } catch (err) {
    logger.warn({ err }, 'Failed to load egress policy, using defaults (open)');
  }

  cachedPolicy = DEFAULT_POLICY;
  return cachedPolicy;
}

/** Whether the policy uses --internal (no internet gateway). */
function isInternal(policy: EgressPolicy): boolean {
  return !policy.allowlist.includes('*');
}

/**
 * Get Docker network args for container launch.
 * Returns empty array if no network policy is configured.
 */
export function getNetworkArgs(): string[] {
  const policy = loadEgressPolicy();
  if (!policy.network) return [];
  return ['--network', policy.network];
}

/**
 * Ensure the Docker network and dual-homed services are configured.
 * Called once at startup.
 */
export function ensureNetwork(): void {
  const policy = loadEgressPolicy();
  if (!policy.network) return;

  // Check if network exists
  let networkExists = false;
  try {
    execSync(`docker network inspect ${policy.network}`, {
      stdio: 'pipe',
      timeout: 5000,
    });
    networkExists = true;
  } catch {
    // Network doesn't exist, create it
  }

  if (!networkExists) {
    const internalFlag = isInternal(policy) ? '--internal' : '';
    logger.info(
      { network: policy.network, internal: isInternal(policy) },
      'Creating Docker network',
    );
    execSync(
      `docker network create ${internalFlag} ${policy.network}`.trim(),
      { stdio: 'pipe', timeout: 10000 },
    );
  }

  // Dual-home listed services (connect to both internal + bridge)
  for (const service of policy.dualHomedServices) {
    try {
      // Check if service container is running
      const running = execSync(
        `docker ps --filter name=^${service}$ --format '{{.Names}}'`,
        { stdio: 'pipe', encoding: 'utf-8', timeout: 5000 },
      ).trim();
      if (!running) continue;

      // Connect to internal network
      execSync(`docker network connect ${policy.network} ${service}`, {
        stdio: 'pipe',
        timeout: 5000,
      });
      logger.info({ service, network: policy.network }, 'Dual-homed service connected');
    } catch {
      // Already connected or not running — both are fine
    }
  }

  logger.info(
    {
      network: policy.network,
      internal: isInternal(policy),
      dualHomed: policy.dualHomedServices,
    },
    'Network policy active',
  );
}

/** Reload the egress policy (e.g., after config change). */
export function reloadEgressPolicy(): void {
  cachedPolicy = null;
  loadEgressPolicy();
}
```

### Step 3: Update container-runner.ts

Read `src/container-runner.ts` and add the import:

```typescript
import { getNetworkArgs } from './network-policy.js';
```

In `buildContainerArgs()`, after the `--name` argument, add:

```typescript
// Apply network policy (egress control)
args.push(...getNetworkArgs());
```

If the LiteLLM skill already added a `LITELLM_MODE` network check, **replace it** with this line. The network policy module handles all cases now.

### Step 4: Initialize at startup

Read `src/index.ts` and add the import:

```typescript
import { ensureNetwork } from './network-policy.js';
```

After `ensureContainerRuntimeRunning()` and before channel connections, add:

```typescript
ensureNetwork();
```

### Step 5: Build

```bash
npm run build
```

## Phase 4: Verify

### Test isolation (internal network)

```bash
# Should FAIL — no internet on --internal network
docker run --rm --network nanoclaw-internal curlimages/curl \
  curl -sf --connect-timeout 5 http://api.anthropic.com 2>&1 || echo "BLOCKED (expected)"

# Should SUCCEED — LiteLLM is dual-homed
docker run --rm --network nanoclaw-internal curlimages/curl \
  curl -sf --connect-timeout 5 http://nanoclaw-litellm:4000/health
```

### Test agent

Send a message to your agent. It should respond normally via LiteLLM.

### Verify network state

```bash
docker network inspect nanoclaw-internal --format '{{range .Containers}}{{.Name}} {{end}}'
cat ~/.config/nanoclaw/egress-policy.json
```

## Changing the Policy

Edit `~/.config/nanoclaw/egress-policy.json` and restart NanoClaw. Examples:

**Lockdown → Open:**

```json
{ "allowlist": ["*"], "network": "nanoclaw-internal", "dualHomedServices": [] }
```

Then recreate the network without `--internal`:

```bash
docker network rm nanoclaw-internal && docker network create nanoclaw-internal
```

**Open → Lockdown:**

```json
{ "allowlist": ["nanoclaw-litellm"], "network": "nanoclaw-internal", "dualHomedServices": ["nanoclaw-litellm"] }
```

Then recreate with `--internal`:

```bash
docker network rm nanoclaw-internal && docker network create --internal nanoclaw-internal
```

## Future: /add-egress-proxy

For selective domain filtering (the middle ground between proxy-only and open), a Squid HTTP proxy skill would:

1. Run Squid as a Docker container, dual-homed on `nanoclaw-internal` + bridge
2. Read domain allowlist from `egress-policy.json`
3. Pass `HTTP_PROXY=http://nanoclaw-squid:3128` to agent containers
4. Squid filters outbound requests — only allowlisted domains pass
5. WebSearch/WebFetch work through the proxy

This is a separate skill because it adds a new container and complexity. The base network policy (this skill) gives you the binary choice that covers most use cases.

## Troubleshooting

### Agent can't reach LiteLLM

1. Verify LiteLLM is on the internal network: `docker network inspect nanoclaw-internal`
2. Reconnect if needed: `docker network connect nanoclaw-internal nanoclaw-litellm`
3. Check dual-homing: `docker inspect nanoclaw-litellm --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool`

### Changing network type requires recreation

Docker doesn't allow changing a network between `--internal` and regular. You must delete and recreate it (stop all containers on it first).

### WebSearch stopped working

Expected when `allowlist` doesn't include `"*"`. Either:
- Add `"*"` to the allowlist (opens internet)
- Wait for `/add-egress-proxy` (filtered web access)
- Use LiteLLM's built-in web search pass-through

## Removal

1. Remove policy: `rm ~/.config/nanoclaw/egress-policy.json`
2. Remove `src/network-policy.ts`
3. Remove `EGRESS_POLICY_PATH` from `src/config.ts`
4. Remove `ensureNetwork()` from `src/index.ts`
5. Remove `getNetworkArgs()` from `src/container-runner.ts`
6. Remove Docker network: `docker network rm nanoclaw-internal`
7. Rebuild and restart
