# NanoClaw Security Enhancements — Conversation Fork

This document summarizes all security-related work, findings, and design decisions from the original conversation. Three skills were written; several more were identified but not implemented. Use this as context to continue the security hardening track.

## Codebase Context

NanoClaw is a single Node.js process that routes messages from channels (WhatsApp, Telegram, Slack, Discord, Gmail) to Claude agents running in Docker containers. Each "group" gets its own isolated filesystem namespace. The key files for security work are:

| File | Role |
|------|------|
| `src/container-runner.ts` | Spawns Docker containers with mounts, passes secrets via stdin |
| `src/config.ts` | Reads `.env` via `readEnvFile()` (never `process.env`) |
| `src/env.ts` | `readEnvFile(keys)` — parses `.env` without loading into environment |
| `container/agent-runner/src/index.ts` | Agent execution inside container; `createSanitizeBashHook()` strips API keys from subprocesses |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | MCP server for IPC; atomic writes via temp+rename |
| `container/Dockerfile` | Node.js 22-slim, Chromium, non-root `node` user |
| `src/container-runtime.ts` | Docker binary path, readonly mounts, orphan cleanup |

### Current Secret Flow (Pre-LiteLLM)

```
.env (on disk)
  → readEnvFile(['ANTHROPIC_API_KEY', ...])  (parsed, never process.env)
    → JSON stringified to stdin pipe
      → container reads /dev/stdin → /tmp/input.json
        → sdkEnv = { ...secrets }  (never process.env inside container either)
          → createSanitizeBashHook() unsets ANTHROPIC_API_KEY in Bash subprocesses
```

Real API keys travel through stdin and exist in container memory. LiteLLM eliminates this.

### Current Container Isolation (Pre-Network Policy)

- Containers run on Docker's default `bridge` network — **full internet access**
- No egress filtering, no firewall rules, no network policy
- Filesystem isolation: per-group mounts, project root is read-only, `.env` shadowed with `/dev/null`
- Non-root `node` user inside container
- No seccomp profile, no read-only rootfs, no capability dropping

---

## Skill 1: `/add-litellm-proxy` (Written)

**Location**: `.claude/skills/add-litellm-proxy/SKILL.md`

**Problem**: Real API keys (`ANTHROPIC_API_KEY`) are passed into every container. A compromised agent could exfiltrate them. There's also no per-group usage tracking or model routing.

**Solution**: LiteLLM proxy sits between agents and the Anthropic API. Agents get virtual keys that are worthless outside the proxy. Real keys stay in the proxy container only.

### Two Deployment Modes

1. **Local Docker container** — LiteLLM runs as a sibling container, managed by `litellm/manage.sh`
2. **External instance** — Point at an existing LiteLLM deployment (set `LITELLM_PROXY_URL` and `LITELLM_API_KEY`)

### Key Architecture

```
Agent Container                    LiteLLM Proxy                  Anthropic API
┌─────────────┐                   ┌──────────────┐               ┌─────────────┐
│ Virtual key  │──HTTP──────────▶│ Real API key  │──HTTPS──────▶│             │
│ sk-vk-group1 │  nanoclaw-net   │ sk-ant-...    │  internet    │  claude.ai  │
└─────────────┘                   └──────────────┘               └─────────────┘
```

### What It Modifies

- **`src/config.ts`**: Adds `LITELLM_PROXY_URL`, `LITELLM_API_KEY`, `LITELLM_MASTER_KEY`, `LITELLM_MODE`, `NANOCLAW_NETWORK`
- **`src/container-runner.ts`**: Overrides `readSecrets()` — when proxy is configured, sets `ANTHROPIC_BASE_URL` to proxy URL and `ANTHROPIC_API_KEY` to the group's virtual key. Adds `--network` flag to container args.
- **Creates `litellm/config.yaml`**: Model list configuration (`os.environ/ANTHROPIC_API_KEY` syntax)
- **Creates `litellm/manage.sh`**: Start/stop/restart/status/key generation/key listing

### Per-Group Virtual Keys

Each group can have its own virtual key stored in the DB via `container_config.litellmApiKey`. This enables:
- Per-group usage tracking and spend limits
- Per-group model access policies
- Revoking a single group's access without affecting others

### Modified readSecrets()

```typescript
function readSecrets(group?: RegisteredGroup): Record<string, string> {
  if (LITELLM_PROXY_URL) {
    const secrets: Record<string, string> = {};
    secrets.ANTHROPIC_BASE_URL = LITELLM_PROXY_URL;
    const groupKey = group?.containerConfig?.litellmApiKey;
    const proxyKey = groupKey || LITELLM_API_KEY || LITELLM_MASTER_KEY;
    if (proxyKey) secrets.ANTHROPIC_API_KEY = proxyKey;
    return secrets;
  }
  return readEnvFile(['CLAUDE_CODE_OAUTH_TOKEN', 'ANTHROPIC_API_KEY', ...]);
}
```

### Model Selection (Not Implemented — Future Skill)

The user asked about a model selection skill that would work with LiteLLM. The idea: different task types (summarization, code generation, translation) could route to different models (Haiku for cheap tasks, Opus for hard ones, local Ollama for private data). LiteLLM already supports this via its model list and routing config. A future `/add-model-routing` skill could:
- Define task-type-to-model mappings in config
- Set per-group model preferences
- Use LiteLLM's fallback chains (try Sonnet, fall back to Haiku)
- Track cost per model per group

---

## Skill 2: `/add-network-policy` (Written)

**Location**: `.claude/skills/add-network-policy/SKILL.md`

**Problem**: Containers have full internet access on Docker's default bridge network. A compromised agent could exfiltrate data to any endpoint.

**Solution**: Private Docker network with `--internal` flag (no internet gateway) plus a configurable allowlist. The allowlist determines behavior:

| Allowlist Contents | Behavior |
|-------------------|----------|
| Empty `[]` | Air-gapped — no network at all |
| Proxy only `["litellm"]` | Can only reach LiteLLM proxy |
| Selective `["litellm", "api.github.com"]` | Proxy + specific domains |
| Wildcard `["*"]` | Full internet (like today's default) |

### Single Allowlist Model

The user specifically requested simplification from three modes to one. The allowlist is the single source of truth:

```json
// ~/.config/nanoclaw/egress-policy.json
{
  "allowlist": ["litellm"],
  "network": "nanoclaw-internal",
  "dualHomedServices": ["litellm-proxy"]
}
```

### Docker `--internal` Networks

Key insight discovered during analysis: Docker `--internal` is binary — containers either have an internet gateway or they don't. There is no built-in "allow these domains" mechanism. The allowlist in nanoclaw controls which *Docker containers/services* agents can reach on the internal network, not which internet domains they can access.

To get selective domain filtering (e.g., "allow api.github.com but block everything else"), you need Squid. See the egress proxy section below.

### Dual-Homing Pattern

LiteLLM must be on **two networks** simultaneously:
- `nanoclaw-internal` (so agents can reach it)
- `bridge` or host network (so it can reach the Anthropic API)

```bash
docker network connect nanoclaw-internal litellm-proxy
```

This is the dual-homing pattern — the proxy bridges the air gap.

### What It Creates

- **`src/network-policy.ts`**: `loadEgressPolicy()`, `getNetworkArgs()`, `ensureNetwork()`
- **Config at `~/.config/nanoclaw/egress-policy.json`** (outside project root)
- **Modifies `src/container-runner.ts`**: Uses `getNetworkArgs()` in `buildContainerArgs()`

---

## Skill 3: `/add-prompt-injection` (Written)

**Location**: `.claude/skills/add-prompt-injection/SKILL.md`

**Problem**: Agents receive untrusted input from messaging channels. Malicious messages could attempt to override system prompts, extract secrets, or execute destructive commands.

**Solution**: Three-layer defense-in-depth, ported from BastionClaw's `prompt-injection.ts` and `security-hooks.ts`.

### Layer 1: Input Sanitization (Host-Side)

Runs in `src/prompt-injection.ts` before the message reaches the container:

- **Neutralize patterns** (wrapped in brackets, not blocked):
  - System prompt overrides: `ignore previous instructions`, `you are now`, `new system prompt`
  - Identity overrides: `forget who you are`, `act as`, `pretend to be`
  - Jailbreaks: `DAN`, `developer mode`, `no restrictions`
  - HTML/markdown injection: `<script>`, `javascript:`, `data:text/html`
  - Destructive commands: `rm -rf`, `DROP TABLE`, `format c:`
  - SQL injection patterns

- **Block patterns** (message rejected entirely):
  - Obfuscated payloads: >70% special characters
  - Spam padding: <15% unique words (token dilution attacks)

- **Truncation**: Messages over `MAX_MESSAGE_LENGTH` are truncated

### Layer 2: Container Sandbox (OS-Level)

Already exists — Docker provides process isolation. The container runs as non-root `node` user. Filesystem mounts are per-group with read-only project root.

### Layer 3: Runtime Hooks (Inside Container)

Created in `container/agent-runner/src/security-hooks.ts`:

- **`createSanitizeBashHook()`**: PreToolUse hook for Bash — strips `ANTHROPIC_API_KEY` and `CLAUDE_CODE_OAUTH_TOKEN` from subprocess environments via `unset` prefix
- **`createSecretPathBlockHook()`**: PreToolUse hook for Read/Bash — blocks access to:
  - `/proc/self/environ` (would leak all env vars)
  - `/proc/<pid>/environ` (same via PID)
  - `/tmp/input.json` (contains the stdin secrets payload)

### Sanitizer Return Type

```typescript
interface SanitizeResult {
  safe: boolean;      // true if no patterns matched
  blocked: boolean;   // true if message should be rejected entirely
  sanitized: string;  // cleaned version of the message
  reason?: string;    // why it was flagged (for audit log)
  matches?: string[]; // which patterns matched
}
```

---

## Not Implemented: `/add-egress-proxy` (Squid)

This was discussed in detail but deliberately deferred. Here's the full design rationale:

### Why Squid Is Needed

Docker `--internal` networks are binary: no internet gateway at all, or full internet. There's no built-in way to say "allow api.github.com but block evil.com." Squid fills this gap:

```
Agent Container → HTTP_PROXY=squid:3128 → Squid (domain allowlist) → Internet
```

### What It Would Do

- Deploy Squid as a Docker container on the `nanoclaw-internal` network
- Configure domain-based ACLs: `acl allowed_domains dstdomain .github.com .npmjs.org`
- Set `HTTP_PROXY` and `HTTPS_PROXY` in container environment
- All agent HTTP traffic goes through Squid — unauthorized domains are blocked
- Squid logs provide full HTTP audit trail (URLs, response codes, bytes transferred)

### Why It Was Deferred

- The network policy skill provides the critical first step (no internet vs full internet)
- Squid adds operational complexity (certificate handling for HTTPS, performance overhead)
- Most use cases are served by "proxy only" (agents talk to LiteLLM, nothing else)
- Can be layered on top of the network policy later without changing existing architecture

### Integration with Network Policy

The `/add-network-policy` skill was designed to be Squid-aware. When Squid is added:
1. Add `squid-proxy` to `dualHomedServices` in `egress-policy.json`
2. Agent containers get `HTTP_PROXY=squid-proxy:3128` in their environment
3. Squid container is dual-homed (internal network + bridge)
4. Domain allowlist moves from conceptual to enforced

### Future Consideration: Web Sandbox

The user mentioned wanting a web sandbox for safe browsing. The agent-browser tool (in `container/skills/agent-browser.md`) launches Chromium inside the container. Currently it has full internet access. With Squid, Chromium traffic would also be filtered through the domain allowlist. A more advanced approach would be an isolated Chromium instance in its own container with even tighter restrictions.

---

## Not Implemented: `/add-container-hardening`

Mentioned during discussion, covers OS-level container security beyond network isolation:

### Read-Only Root Filesystem

```bash
docker run --read-only --tmpfs /tmp --tmpfs /home/node ...
```

Prevents agents from modifying container binaries or system files. Only `/tmp` and `/home/node` are writable.

### Seccomp Profiles

Custom seccomp profile that drops dangerous syscalls:
- `ptrace` (process debugging/inspection)
- `mount`/`umount` (filesystem manipulation)
- `reboot`, `sethostname`, `setdomainname`
- `init_module`, `delete_module` (kernel modules)

### Capability Dropping

```bash
docker run --cap-drop=ALL --cap-add=CHOWN --cap-add=SETUID --cap-add=SETGID ...
```

Drops all Linux capabilities except the minimum needed for Node.js + Chromium.

### Ephemeral Subagents for WebFetch

Instead of the main agent doing web fetches (which exposes it to prompt injection from web content), spawn a separate ephemeral container for each web request. The subagent fetches the content, sanitizes it, and returns only the cleaned text. If the web content is malicious, the subagent is destroyed — the main agent is never exposed.

### Resource Limits

```bash
docker run --memory=2g --cpus=1.5 --pids-limit=256 ...
```

Prevents resource exhaustion attacks (fork bombs, memory allocation).

---

## Design Principles Established

1. **Secrets never in `process.env`** — already true, validated during analysis
2. **Secrets passed via stdin, not env vars** — Docker env vars are visible in `docker inspect`
3. **Defense in depth** — multiple independent layers, each effective alone
4. **Config outside project root** — security config at `~/.config/nanoclaw/`, not in the project directory agents can access
5. **Non-blocking failures** — security module errors log but never crash the main process
6. **Existing hooks preserved** — the `createSanitizeBashHook()` in agent-runner was already stripping API keys; the prompt injection skill extends this pattern rather than replacing it

---

## Summary of Security Skills

| Skill | Status | What It Does |
|-------|--------|-------------|
| `/add-litellm-proxy` | Written | Virtual keys, secret isolation, usage tracking |
| `/add-network-policy` | Written | Docker `--internal` network, allowlist model |
| `/add-prompt-injection` | Written | Input sanitization, secret path blocking, env stripping |
| `/add-egress-proxy` | Designed, not written | Squid-based domain filtering for selective internet access |
| `/add-container-hardening` | Discussed, not designed | Read-only rootfs, seccomp, capabilities, resource limits |
| `/add-model-routing` | Mentioned | Per-task-type model selection via LiteLLM |
