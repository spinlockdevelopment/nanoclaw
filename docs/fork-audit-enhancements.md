# NanoClaw Audit & Observability Enhancements — Conversation Fork

This document summarizes all audit, observability, and checkpointing work from the original conversation. Three skills were written; additional ideas are included at the end. Use this as context to continue the observability track.

## Codebase Context

NanoClaw uses Pino for logging (pretty-print only, no JSON output, no file transport). IPC is filesystem-based: JSON files written to `data/ipc/{group}/messages/` and `data/ipc/{group}/tasks/`, polled every 1 second, processed and deleted. There is no telemetry, no trace correlation, no structured event taxonomy, and no audit trail.

### Key Files for Observability Work

| File | Role |
|------|------|
| `src/logger.ts` | Pino logger — currently pretty-print only |
| `src/ipc.ts` | IPC watcher — polls per-group namespaces, processes JSON files |
| `src/container-runner.ts` | Container lifecycle — spawn, stream output, exit handling |
| `src/index.ts` | Main orchestrator — message loop, agent invocation, shutdown |
| `src/config.ts` | Configuration constants and `.env` reading |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | MCP server inside container — sends IPC messages/tasks |

### What Already Exists

- Pino logger with `pino-pretty` transport (dev-friendly, not machine-readable)
- Container logs written to `groups/{folder}/logs/container-{timestamp}.log` after each run
- No structured events, no trace IDs, no metrics, no alerting

---

## Skill 1: `/add-ipc-observability` (Written)

**Location**: `.claude/skills/add-ipc-observability/SKILL.md`

**Problem**: IPC operations are fire-and-forget with no visibility. When something goes wrong (message lost, task stuck, authorization denied), there's no way to trace what happened.

**Solution**: Structured telemetry that piggybacks on the existing Pino logger, adding JSON file transport and OpenTelemetry-compatible trace contexts.

### What It Creates

**`src/telemetry.ts`** — The telemetry module with:

- **Trace contexts**: 128-bit trace IDs, 64-bit span IDs, parent trace propagation
- **Event taxonomy**: Typed events for every IPC/container/message/task operation
- **Metrics**: Counters (messages processed, containers spawned, errors) and gauges (active containers, pending tasks)
- **Timer helpers**: `startTrace()` / `endTrace()` for duration measurement

### Trace Context

```typescript
interface TraceContext {
  traceId: string;   // 128-bit hex, propagated across IPC boundaries
  spanId: string;    // 64-bit hex, unique per operation
  operation: string; // e.g., "ipc.message.process", "container.spawn"
  startTime: number; // Date.now()
}

function startTrace(operation: string, parentTraceId?: string): TraceContext {
  return {
    traceId: parentTraceId || crypto.randomBytes(16).toString('hex'),
    spanId: crypto.randomBytes(8).toString('hex'),
    operation,
    startTime: Date.now(),
  };
}
```

### Event Types

| Event | Source | Fields |
|-------|--------|--------|
| `ipc.message.received` | `src/ipc.ts` | group, sender, messageType, traceId |
| `ipc.message.authorized` | `src/ipc.ts` | group, sender, authorized (bool) |
| `ipc.message.processed` | `src/ipc.ts` | group, durationMs, traceId |
| `ipc.task.received` | `src/ipc.ts` | group, taskType, taskId |
| `ipc.task.processed` | `src/ipc.ts` | group, taskType, durationMs |
| `container.spawn` | `src/container-runner.ts` | group, containerName, image, network |
| `container.output` | `src/container-runner.ts` | group, containerName, outputLength |
| `container.exit` | `src/container-runner.ts` | group, containerName, exitCode, durationMs |
| `container.timeout` | `src/container-runner.ts` | group, containerName, timeoutMs |
| `container.error` | `src/container-runner.ts` | group, containerName, error |
| `message.inbound` | `src/index.ts` | channel, group, sender, messageLength |
| `message.outbound` | `src/index.ts` | channel, group, recipientCount |

### Logger Upgrade

Modifies `src/logger.ts` to add dual transport:

```typescript
// Development: pretty-print to stderr (existing behavior)
// Production: JSON to file at data/logs/nanoclaw.ndjson
const transports = pino.transport({
  targets: [
    { target: 'pino-pretty', level: 'info', options: { destination: 2 } },
    { target: 'pino/file', level: 'debug', options: { destination: logFilePath } },
  ],
});
```

The JSON transport enables Grafana Alloy ingestion. The pretty transport preserves the existing dev experience.

### Instrumentation Points

The skill instruments three files:

1. **`src/ipc.ts`**: Wraps `processIpcFiles()` with trace context; logs every message receive, authorization check, and processing completion
2. **`src/container-runner.ts`**: Logs container spawn, output streaming, exit (with duration), timeout, and error events
3. **`src/index.ts`**: Logs inbound/outbound message events with channel and group context

### Grafana Alloy Integration

Creates `alloy/config.alloy` template:

```hcl
local.file_match "nanoclaw_logs" {
  path_targets = [{ __path__ = "/path/to/nanoclaw/data/logs/nanoclaw.ndjson" }]
}

loki.source.file "nanoclaw" {
  targets    = local.file_match.nanoclaw_logs.targets
  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

The user's stated goal was to log all IPC between agents to an external server using Grafana Alloy. This config achieves that by tailing the JSON log file and shipping entries to Loki.

### Metrics Counters

```typescript
const metrics = {
  messagesProcessed: 0,
  messagesErrors: 0,
  containersSpawned: 0,
  containersErrors: 0,
  containersTimedOut: 0,
  tasksProcessed: 0,
  ipcAuthDenied: 0,
};
```

Exposed for future Prometheus endpoint or periodic log emission.

---

## Skill 2: `/add-audit-log` (Written)

**Location**: `.claude/skills/add-audit-log/SKILL.md`

**Problem**: Observability logs are great for debugging but they're in the project directory (agents could theoretically modify them) and they're not designed for compliance/security review.

**Solution**: Tamper-proof, append-only audit log stored outside the project root.

### Key Design Decisions

1. **Location**: `~/.config/nanoclaw/audit.log` — outside project root, never mounted into containers
2. **Format**: NDJSON (one JSON object per line) — greppable, parseable, Loki-compatible
3. **Write mode**: `O_APPEND` flag — atomic appends, no seek/truncate possible
4. **Failure mode**: Log errors to Pino but never crash the process

### What It Creates

**`src/audit.ts`** — The audit module with:

```typescript
interface AuditEntry {
  timestamp: string;    // ISO 8601
  event: string;        // e.g., "container.spawn", "ipc.auth.denied"
  level: 'info' | 'warn' | 'error' | 'critical';
  group?: string;
  details: Record<string, unknown>;
}
```

### Audit Functions

| Function | When Called | What It Records |
|----------|-----------|----------------|
| `auditAuth()` | Channel authentication | Channel, success/failure, identity |
| `auditContainerSpawn()` | Container starts | Group, image, network, mounts |
| `auditContainerExit()` | Container stops | Group, exit code, duration, timed out |
| `auditIpcMessage()` | IPC message processed | Group, sender, target, message type |
| `auditTaskOperation()` | Task scheduled/paused/etc. | Group, task ID, operation type |
| `auditGroupRegistration()` | New group registered | Group name, folder, channel |
| `auditSecretAccess()` | Secrets read from .env | Which keys were accessed (not values) |
| `auditBlocked()` | Authorization denied | Group, attempted action, reason |
| `auditPromptInjection()` | Injection pattern detected | Group, pattern matched, action taken |

### Core Writer

```typescript
function writeAuditEntry(entry: AuditEntry): void {
  try {
    const fd = ensureAuditLog(); // Opens with O_APPEND, caches fd
    const line = JSON.stringify(entry) + '\n';
    fs.writeSync(fd, line);
  } catch (err) {
    logger.error({ err, event: entry.event }, 'Failed to write audit entry');
  }
}
```

### Integration Points

- **`src/ipc.ts`**: `auditIpcMessage()` on every processed message, `auditBlocked()` on authorization denial
- **`src/container-runner.ts`**: `auditContainerSpawn()` on launch, `auditContainerExit()` on close
- **`src/index.ts`**: `auditAuth()` on channel connection, shutdown handler to flush/close fd
- **`src/prompt-injection.ts`** (if applied): `auditPromptInjection()` on pattern match

### Relationship to IPC Observability

| IPC Observability | Audit Log |
|-------------------|-----------|
| JSON logs in project dir (`data/logs/`) | NDJSON at `~/.config/nanoclaw/audit.log` |
| Debug-level detail (trace IDs, durations) | Security-focused (auth, access, blocked) |
| Grafana/Loki for dashboards | Compliance review, incident response |
| Can be modified by project processes | Tamper-proof (outside project, O_APPEND) |
| High volume, all operations | Lower volume, security events only |

They complement each other — observability for debugging, audit for security review.

---

## Skill 3: `/add-group-checkpointing` (Written)

**Location**: `.claude/skills/add-group-checkpointing/SKILL.md`

**Problem**: Agents can modify their own CLAUDE.md, memories, and settings. If an agent corrupts its own identity (accidentally or via prompt injection), there's no history and no rollback mechanism.

**Solution**: Git-based versioning of agent-modifiable files. After every container exit, changed files are committed to a separate git repo outside the project root.

### What Gets Checkpointed

| Source | Tracked As | Why |
|--------|-----------|-----|
| `groups/{folder}/CLAUDE.md` | `groups/{folder}/CLAUDE.md` | Agent identity and memories |
| `groups/{folder}/**` | `groups/{folder}/**` | Any files agent creates |
| `data/sessions/{folder}/.claude/settings.json` | `sessions/{folder}/settings.json` | Claude Code settings |
| `groups/global/CLAUDE.md` | `groups/global/CLAUDE.md` | Shared global memory |

**Excluded**: Container logs (already captured per-run), IPC files (ephemeral), agent-runner source (copied from template), Claude session internals (SDK state).

### Architecture

```
Container Exit (container-runner.ts)
  └── setImmediate(() => checkpointGroup(...))
        ├── rsync group files → ~/.config/nanoclaw/group-checkpoints/
        ├── git add -A
        ├── git commit (structured message with metadata)
        └── git push (async, best-effort, if remote configured)
```

### Key Design Decisions

1. **`setImmediate()`**: Checkpoint runs after `resolve()` fires — never delays message delivery
2. **Never throws**: All checkpoint code wrapped in try/catch; failures log but don't crash
3. **Separate repo**: `~/.config/nanoclaw/group-checkpoints/` — not inside the project, not mounted into containers
4. **Remote push**: Async, best-effort — credentials in system SSH config, never in `.env`
5. **Files > 1MB skipped**: Prevents the repo from bloating with binary artifacts

### Commit Message Format

```
checkpoint: GroupName (ok|exit N)

group: GroupName
folder: group-folder-slug
container: nanoclaw-group-1709726400
exit_code: 0
duration_ms: 15234
timed_out: false
files_changed: 2
timestamp: 2026-03-07T12:00:00.000Z
```

Searchable with `git log --grep`:
- `git log --grep="exit_code: [^0]"` — find error exits
- `git log --grep="timed_out: true"` — find timeouts
- `git log --grep="folder: main"` — history for a specific group

### Rollback Capability

```typescript
function rollbackGroup(groupFolder: string, commitHash: string): boolean {
  // Restores files from the specified commit
  // Copies them back to the live group directory
  // Commits the rollback itself (so the rollback is also in history)
}
```

Manual rollback is also simple:
```bash
cd ~/.config/nanoclaw/group-checkpoints
git log --oneline -- groups/main/  # find commit
git checkout a1b2c3d -- groups/main/  # restore
cp -r groups/main/* /path/to/nanoclaw/groups/main/  # copy back
```

### How It Complements the Audit Log

| Audit Log | Checkpoint |
|-----------|------------|
| Records WHAT happened (operations, events) | Records the RESULT (file changes) |
| "Container spawned, IPC sent" | "CLAUDE.md line 42 changed from X to Y" |
| Real-time stream | Point-in-time snapshots |
| Loki/Grafana queryable | Git diff/blame/bisect |

Together: full forensic capability. The audit log shows the sequence of operations, checkpointing shows the state changes those operations produced.

---

## Not Implemented: Real-Time Alerting

Mentioned but not designed. Natural extension of IPC observability:

- **Container failure alerts**: Notify (via Telegram/Slack/webhook) when a container exits non-zero N times in a row
- **IPC anomaly detection**: Alert when message volume spikes or drops to zero (agent may be stuck or spamming)
- **Prompt injection alerts**: When the sanitizer flags a message, send an alert to the admin group
- **Resource usage alerts**: CPU/memory thresholds per container (requires metrics collection)

Could be implemented as a Grafana alert rule (if using Loki/Alloy pipeline) or as a lightweight in-process monitor.

---

## Not Implemented: Dashboard Templates

The Grafana Alloy config ships the logs to Loki, but there are no pre-built dashboards. Future work:

- **IPC Overview**: Message volume by group, authorization denials, error rates
- **Container Lifecycle**: Spawn rate, average duration, timeout frequency, exit code distribution
- **Agent Activity**: Messages per group over time, active hours, conversation lengths
- **Security Events**: Prompt injection attempts, blocked operations, audit log entries
- **Group Checkpoints**: Change frequency per group, file change volume, rollback events

These would be Grafana JSON dashboard exports, included in an `alloy/dashboards/` directory.

---

## Not Implemented: Structured IPC Protocol Upgrade

Originally proposed as a full IPC overhaul but the user redirected to observability instead. The original idea was:

- Replace filesystem polling with a proper message queue (Redis, NATS, or SQLite WAL)
- Add message schemas with validation
- Add delivery guarantees (at-least-once, acknowledgments)
- Add message ordering and deduplication

The user's position: the existing filesystem IPC works fine for the scale; just add visibility into what's happening. This was the right call — observability provides immediate value without the risk of rewriting a working system.

---

## Not Implemented: A2A Protocol

Initially considered for inter-agent communication (Google's Agent-to-Agent protocol). Dropped because:
- NanoClaw's IPC is internal (same host, same Docker network) — A2A is designed for cross-organization agent discovery
- The filesystem IPC with MCP tools already provides what's needed
- A2A would add HTTP server overhead inside each container
- Better suited for a multi-host deployment (not current architecture)

---

## Summary of Audit/Observability Skills

| Skill | Status | What It Does |
|-------|--------|-------------|
| `/add-ipc-observability` | Written | Structured telemetry, trace contexts, Grafana Alloy, JSON logging |
| `/add-audit-log` | Written | Tamper-proof NDJSON audit trail at `~/.config/nanoclaw/` |
| `/add-group-checkpointing` | Written | Git-based versioning of agent config files, remote push, rollback |
| Real-time alerting | Not designed | Alert on failures, anomalies, injection attempts |
| Dashboard templates | Not designed | Pre-built Grafana dashboards for all event types |
| IPC protocol upgrade | Rejected by user | Keep filesystem IPC, add visibility instead |
| A2A protocol | Dropped | Overkill for single-host architecture |

### Recommended Application Order

1. `/add-ipc-observability` — foundation (structured logging, trace IDs)
2. `/add-audit-log` — security layer (requires observability patterns)
3. `/add-group-checkpointing` — state history (independent, but complements audit log)
