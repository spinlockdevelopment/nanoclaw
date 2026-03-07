---
name: add-audit-log
description: Add tamper-proof security audit logging for all IPC operations, authorization decisions, container lifecycle, and agent actions. Append-only log stored outside project root. Requires /add-ipc-observability. Triggers on "audit log", "audit trail", "security log", "compliance log".
---

# Add Audit Log

This skill adds a tamper-proof, append-only audit log for security-relevant operations. It piggybacks on the telemetry module from `/add-ipc-observability` — filtering for security events and writing them to a separate, external log file.

**Prerequisite**: `/add-ipc-observability` must be applied first. If `src/telemetry.ts` does not exist, tell the user to run `/add-ipc-observability` first.

## What Gets Audited

| Event | Details Logged |
|-------|---------------|
| **Authorization decisions** | Allowed and blocked IPC operations, source group, target, reason |
| **Container spawns** | Group, mounts, isMain, network, image |
| **Container exits** | Exit code, duration, timeout status |
| **IPC messages** | Source group, target JID, message type (not content) |
| **Task operations** | Create, pause, resume, cancel, update — who did what |
| **Group registration** | JID, folder, who registered it |
| **Secret access** | Which secrets were requested (not values), proxy mode |
| **Blocked actions** | Unauthorized IPC, invalid group folders, mount rejections |

**What is NOT logged** (privacy):
- Message content (only metadata: length, sender, target)
- API keys or secrets (only access patterns)
- Agent output (only success/error status)

## Architecture

```
Telemetry Module (src/telemetry.ts)
         │
         ├── Pino Logger ──→ logs/nanoclaw-telemetry.json (all events)
         │
         └── Audit Logger ──→ ~/.config/nanoclaw/audit.log (security events only)
                                    │
                                    └── Grafana Alloy (separate Loki stream)
```

The audit log is stored at `~/.config/nanoclaw/audit.log` — **outside the project root**. This means:
- Agents cannot read, modify, or delete it (not mounted into containers)
- It survives project directory deletion
- It can be on a different filesystem/volume for retention

## Phase 1: Create the Audit Module

Create `src/audit.ts`:

```typescript
/**
 * Security audit logging for NanoClaw.
 * Append-only log stored outside project root (tamper-proof from containers).
 *
 * Records security-relevant events:
 * - All authorization decisions (allowed and blocked)
 * - Container lifecycle (spawn, exit, timeout)
 * - IPC operations (message, task, registration)
 * - Secret access patterns
 *
 * Format: NDJSON (newline-delimited JSON) for Alloy/Loki ingestion.
 */
import fs from 'fs';
import path from 'path';
import os from 'os';

import { logger } from './logger.js';

const AUDIT_LOG_PATH = process.env.NANOCLAW_AUDIT_LOG || path.join(
  os.homedir(),
  '.config',
  'nanoclaw',
  'audit.log',
);

// Open file descriptor in append mode — survives process crashes
let auditFd: number | null = null;

function ensureAuditLog(): number {
  if (auditFd !== null) return auditFd;

  const dir = path.dirname(AUDIT_LOG_PATH);
  fs.mkdirSync(dir, { recursive: true });

  // O_APPEND is atomic on POSIX — concurrent writes don't interleave
  auditFd = fs.openSync(AUDIT_LOG_PATH, 'a');
  logger.info({ path: AUDIT_LOG_PATH }, 'Audit log initialized');
  return auditFd;
}

// ─── Core Audit Writer ───────────────────────────────────────────────────────

export interface AuditEntry {
  timestamp: string;
  event: string;
  severity: 'info' | 'warn' | 'critical';
  sourceGroup?: string;
  isMain?: boolean;
  details: Record<string, unknown>;
}

function writeAuditEntry(entry: AuditEntry): void {
  try {
    const fd = ensureAuditLog();
    const line = JSON.stringify(entry) + '\n';
    fs.writeSync(fd, line);
  } catch (err) {
    // Audit failures should never crash the process
    logger.error({ err, event: entry.event }, 'Failed to write audit entry');
  }
}

// ─── Audit Events ────────────────────────────────────────────────────────────

/** Log an authorization decision (allowed or blocked). */
export function auditAuth(
  allowed: boolean,
  operation: string,
  sourceGroup: string,
  isMain: boolean,
  details: Record<string, unknown> = {},
): void {
  writeAuditEntry({
    timestamp: new Date().toISOString(),
    event: allowed ? 'auth.allowed' : 'auth.blocked',
    severity: allowed ? 'info' : 'warn',
    sourceGroup,
    isMain,
    details: { operation, ...details },
  });
}

/** Log a container spawn. */
export function auditContainerSpawn(
  group: string,
  groupFolder: string,
  containerName: string,
  isMain: boolean,
  mounts: string[],
  network?: string,
): void {
  writeAuditEntry({
    timestamp: new Date().toISOString(),
    event: 'container.spawn',
    severity: 'info',
    sourceGroup: groupFolder,
    isMain,
    details: {
      group,
      containerName,
      mountCount: mounts.length,
      mounts, // Mount paths (not content) for auditing what was exposed
      network: network || 'default',
    },
  });
}

/** Log a container exit. */
export function auditContainerExit(
  groupFolder: string,
  containerName: string,
  exitCode: number | null,
  durationMs: number,
  timedOut: boolean,
): void {
  writeAuditEntry({
    timestamp: new Date().toISOString(),
    event: timedOut ? 'container.timeout' : (exitCode === 0 ? 'container.exit' : 'container.error'),
    severity: timedOut || (exitCode !== null && exitCode !== 0) ? 'warn' : 'info',
    sourceGroup: groupFolder,
    details: { containerName, exitCode, durationMs, timedOut },
  });
}

/** Log an IPC message (metadata only, not content). */
export function auditIpcMessage(
  sourceGroup: string,
  isMain: boolean,
  targetJid: string,
  messageType: string,
  authorized: boolean,
): void {
  writeAuditEntry({
    timestamp: new Date().toISOString(),
    event: authorized ? 'ipc.message' : 'ipc.message.blocked',
    severity: authorized ? 'info' : 'warn',
    sourceGroup,
    isMain,
    details: { targetJid, messageType },
  });
}

/** Log a task operation (create, pause, resume, cancel, update). */
export function auditTaskOperation(
  operation: string,
  sourceGroup: string,
  isMain: boolean,
  taskId?: string,
  targetFolder?: string,
  authorized?: boolean,
): void {
  writeAuditEntry({
    timestamp: new Date().toISOString(),
    event: `task.${operation}`,
    severity: authorized === false ? 'warn' : 'info',
    sourceGroup,
    isMain,
    details: {
      taskId,
      targetFolder,
      authorized: authorized ?? true,
    },
  });
}

/** Log a group registration. */
export function auditGroupRegistration(
  sourceGroup: string,
  jid: string,
  folder: string,
  name: string,
): void {
  writeAuditEntry({
    timestamp: new Date().toISOString(),
    event: 'group.registered',
    severity: 'info',
    sourceGroup,
    isMain: true, // Only main can register
    details: { jid, folder, name },
  });
}

/** Log secret access pattern (never log values). */
export function auditSecretAccess(
  groupFolder: string,
  secretsRequested: string[],
  proxyMode: boolean,
): void {
  writeAuditEntry({
    timestamp: new Date().toISOString(),
    event: 'secrets.accessed',
    severity: 'info',
    sourceGroup: groupFolder,
    details: {
      keys: secretsRequested, // Key names only, never values
      proxyMode,
    },
  });
}

/** Log a blocked action (mount rejection, invalid folder, etc). */
export function auditBlocked(
  event: string,
  sourceGroup: string,
  reason: string,
  details: Record<string, unknown> = {},
): void {
  writeAuditEntry({
    timestamp: new Date().toISOString(),
    event: `blocked.${event}`,
    severity: 'critical',
    sourceGroup,
    details: { reason, ...details },
  });
}

/** Log prompt injection detection. */
export function auditPromptInjection(
  sourceGroup: string,
  chatJid: string,
  sender: string,
  action: 'neutralized' | 'blocked',
  reason: string,
): void {
  writeAuditEntry({
    timestamp: new Date().toISOString(),
    event: `injection.${action}`,
    severity: action === 'blocked' ? 'critical' : 'warn',
    sourceGroup,
    details: { chatJid, sender, reason },
  });
}

/** Flush the audit log file descriptor. */
export function flushAuditLog(): void {
  if (auditFd !== null) {
    try {
      fs.fsyncSync(auditFd);
    } catch {
      // Best effort
    }
  }
}

/** Close the audit log (for graceful shutdown). */
export function closeAuditLog(): void {
  if (auditFd !== null) {
    try {
      fs.fsyncSync(auditFd);
      fs.closeSync(auditFd);
    } catch {
      // Best effort
    }
    auditFd = null;
  }
}
```

## Phase 2: Integrate with IPC System

### Step 1: Instrument IPC message processing

Read `src/ipc.ts` and add the import:

```typescript
import { auditIpcMessage, auditTaskOperation, auditGroupRegistration, auditAuth } from './audit.js';
```

In `processIpcFiles`, after the authorization check for messages:

For **authorized** messages (after `await deps.sendMessage(...)`:

```typescript
auditIpcMessage(sourceGroup, isMain, data.chatJid, data.type || 'message', true);
```

For **blocked** messages (in the `else` branch):

```typescript
auditIpcMessage(sourceGroup, isMain, data.chatJid, data.type || 'message', false);
```

### Step 2: Instrument processTaskIpc

In each `case` block of `processTaskIpc`, add audit calls:

**schedule_task** (after successful creation):
```typescript
auditTaskOperation('schedule', sourceGroup, isMain, taskId, targetFolder, true);
```

**schedule_task** (blocked):
```typescript
auditTaskOperation('schedule', sourceGroup, isMain, undefined, targetFolder, false);
```

**pause_task / resume_task / cancel_task** (success):
```typescript
auditTaskOperation('pause', sourceGroup, isMain, data.taskId, undefined, true);
```

**pause_task / resume_task / cancel_task** (blocked):
```typescript
auditTaskOperation('pause', sourceGroup, isMain, data.taskId, undefined, false);
```

**register_group** (success):
```typescript
auditGroupRegistration(sourceGroup, data.jid!, data.folder!, data.name!);
```

**register_group** (blocked — non-main attempt):
```typescript
auditAuth(false, 'register_group', sourceGroup, false);
```

## Phase 3: Instrument Container Runner

### Step 1: Add audit to container spawning

Read `src/container-runner.ts` and add the import:

```typescript
import {
  auditContainerSpawn,
  auditContainerExit,
  auditSecretAccess,
} from './audit.js';
```

In `runContainerAgent()`, after building mounts and before spawning:

```typescript
auditContainerSpawn(
  group.name,
  group.folder,
  containerName,
  input.isMain,
  mounts.map(m => `${m.containerPath}${m.readonly ? ' (ro)' : ''}`),
  // Include network if network-policy is applied
);
```

In `readSecrets()`, add secret access audit:

```typescript
function readSecrets(group?: RegisteredGroup): Record<string, string> {
  const proxyMode = !!LITELLM_PROXY_URL;
  // ... existing logic ...

  // Audit which secrets were requested (key names only, never values)
  const secretKeys = Object.keys(secrets);
  if (group) {
    auditSecretAccess(group.folder, secretKeys, proxyMode);
  }

  return secrets;
}
```

In the container `close` handler:

```typescript
auditContainerExit(group.folder, containerName, code, duration, timedOut);
```

## Phase 4: Graceful Shutdown

Read `src/index.ts` and add:

```typescript
import { closeAuditLog } from './audit.js';
```

In the shutdown handler (or add one if it doesn't exist):

```typescript
process.on('SIGTERM', () => {
  closeAuditLog();
  process.exit(0);
});

process.on('SIGINT', () => {
  closeAuditLog();
  process.exit(0);
});
```

## Phase 5: Build and Verify

### Build

```bash
npm run build
```

### Restart

```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

### Check audit log

```bash
# Send a message to the agent, then:
cat ~/.config/nanoclaw/audit.log | python3 -m json.tool --no-ensure-ascii
```

You should see entries like:

```json
{"timestamp":"2026-03-06T12:00:00.000Z","event":"container.spawn","severity":"info","sourceGroup":"main","isMain":true,"details":{"group":"Main","containerName":"nanoclaw-main-1709726400","mountCount":6,"mounts":["/workspace/project (ro)","/workspace/group","/home/node/.claude","/workspace/ipc","/workspace/global (ro)","/app/src"],"network":"nanoclaw-internal"}}
{"timestamp":"2026-03-06T12:00:01.000Z","event":"secrets.accessed","severity":"info","sourceGroup":"main","details":{"keys":["ANTHROPIC_BASE_URL","ANTHROPIC_API_KEY"],"proxyMode":true}}
{"timestamp":"2026-03-06T12:00:05.000Z","event":"ipc.message","severity":"info","sourceGroup":"main","isMain":true,"details":{"targetJid":"tg:12345","messageType":"message"}}
{"timestamp":"2026-03-06T12:00:10.000Z","event":"container.exit","severity":"info","sourceGroup":"main","details":{"containerName":"nanoclaw-main-1709726400","exitCode":0,"durationMs":10000,"timedOut":false}}
```

### Verify tamper protection

```bash
# Confirm the audit log is OUTSIDE the project root
ls -la ~/.config/nanoclaw/audit.log

# Confirm it's NOT mounted into containers
docker inspect <agent-container> | grep -i audit  # Should return nothing
```

## Grafana Alloy Integration

If `/add-ipc-observability` configured Alloy, add a second file source to `alloy/config.alloy`:

```hcl
// Audit log source (separate from telemetry)
local.file_match "nanoclaw_audit" {
  path_targets = [{
    __path__ = "/home/<user>/.config/nanoclaw/audit.log",
    job       = "nanoclaw-audit",
    instance  = "nanoclaw-host",
  }]
}

loki.source.file "audit" {
  targets    = local.file_match.nanoclaw_audit.targets
  forward_to = [loki.process.audit.receiver]
}

loki.process "audit" {
  stage.json {
    expressions = {
      event    = "event",
      severity = "severity",
      source   = "sourceGroup",
    }
  }
  stage.labels {
    values = {
      event    = "",
      severity = "",
      source   = "",
    }
  }
  forward_to = [loki.write.default.receiver]
}
```

Replace `<user>` with the actual home directory path.

## Log Rotation

The audit log grows indefinitely. Set up rotation:

### logrotate (Linux)

Create `/etc/logrotate.d/nanoclaw-audit`:

```
/home/<user>/.config/nanoclaw/audit.log {
    daily
    rotate 90
    compress
    missingok
    notifempty
    copytruncate
}
```

### newsyslog (macOS)

Add to `/etc/newsyslog.d/nanoclaw-audit.conf`:

```
/Users/<user>/.config/nanoclaw/audit.log  644  90  *  @T00  J
```

## Troubleshooting

### Audit log not being written

1. Check permissions: `ls -la ~/.config/nanoclaw/`
2. Check the process can write: `touch ~/.config/nanoclaw/audit.log`
3. Verify build is clean: `npm run build`
4. Check for errors: `grep "audit" logs/nanoclaw.log`

### Audit log too large

Set up log rotation (see above). Or manually: `cp audit.log audit.log.bak && : > audit.log`

### Events missing

The audit module never throws — failures are logged to pino instead. Check `logs/nanoclaw.log` for "Failed to write audit entry" messages.

## Removal

1. Delete `src/audit.ts`
2. Remove all `audit*` imports and calls from `src/ipc.ts`, `src/container-runner.ts`, `src/index.ts`
3. Rebuild: `npm run build`
4. Optionally remove the audit log: `rm ~/.config/nanoclaw/audit.log`
5. Restart NanoClaw
