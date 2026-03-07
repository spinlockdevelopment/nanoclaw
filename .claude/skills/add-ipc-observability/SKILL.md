---
name: add-ipc-observability
description: Add structured telemetry and observability to all IPC operations. OpenTelemetry-compatible traces and metrics, Grafana Alloy integration, per-operation structured logging. Piggybacks on existing Pino logger. Triggers on "observability", "telemetry", "grafana", "alloy", "tracing", "ipc logging", "audit log".
---

# Add IPC Observability

This skill adds structured telemetry to NanoClaw's IPC system, container lifecycle, and message pipeline. Every operation gets a trace ID, structured fields, and duration — ready for Grafana Alloy ingestion without touching the IPC protocol.

**What it instruments:**
- IPC file processing (messages, tasks, authorization decisions)
- Container lifecycle (spawn, output, close, timeout, errors)
- Message pipeline (channel receive → agent invoke → response send)
- Task scheduler (due check, execution, completion)
- All authorization decisions (allowed/blocked)

**What it does NOT change:**
- IPC protocol (still filesystem-based, still ephemeral files)
- Container networking
- Agent behavior

## Architecture

```
NanoClaw Process
├── Channel → onMessage()        ──┐
├── IPC Watcher → processIpc()     ├── Pino structured JSON ──→ log file
├── Container Runner → spawn()     │                             │
├── Task Scheduler → runTask()   ──┘                             │
                                                                 ↓
                                                          Grafana Alloy
                                                          (tail log file)
                                                                 │
                                                    ┌────────────┼────────────┐
                                                    ↓            ↓            ↓
                                                  Loki       Prometheus    Tempo
                                                 (logs)      (metrics)   (traces)
```

## Phase 1: Pre-flight

### Ask about Grafana Alloy

Use `AskUserQuestion`:

> Would you like to configure Grafana Alloy for log shipping now?
>
> **Option A: Yes, configure Alloy** — I'll create an Alloy config that tails the NanoClaw log file and ships to your Grafana stack.
>
> **Option B: Just add telemetry** — I'll add structured logging and you can configure Alloy later.

If Option A, ask for:
1. Loki push URL (e.g., `http://loki:3100/loki/api/v1/push`)
2. Optional: Prometheus remote write URL
3. Optional: Tempo OTLP endpoint

## Phase 2: Add Telemetry Module

### Step 1: Create the telemetry module

Create `src/telemetry.ts`:

```typescript
/**
 * Telemetry module for NanoClaw
 * Adds structured fields, trace IDs, and duration tracking to operations.
 * Compatible with OpenTelemetry semantic conventions for Grafana Alloy ingestion.
 *
 * Design: wraps existing Pino logger with operation-scoped context.
 * No external dependencies — uses crypto.randomUUID() for trace IDs.
 */

import crypto from 'crypto';
import { logger } from './logger.js';

// ─── Trace Context ───────────────────────────────────────────────────────────

export interface TraceContext {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  operation: string;
  startTime: number;
}

/**
 * Create a new trace context for an operation.
 * Uses OpenTelemetry-compatible 32-char hex trace IDs and 16-char span IDs.
 */
export function startTrace(operation: string, parentTraceId?: string): TraceContext {
  return {
    traceId: parentTraceId || crypto.randomBytes(16).toString('hex'),
    spanId: crypto.randomBytes(8).toString('hex'),
    parentSpanId: parentTraceId ? crypto.randomBytes(8).toString('hex') : undefined,
    operation,
    startTime: Date.now(),
  };
}

/**
 * End a trace and log the completion with duration.
 */
export function endTrace(
  ctx: TraceContext,
  fields: Record<string, unknown> = {},
  level: 'info' | 'warn' | 'error' = 'info',
): void {
  const duration = Date.now() - ctx.startTime;
  const logData = {
    traceId: ctx.traceId,
    spanId: ctx.spanId,
    parentSpanId: ctx.parentSpanId,
    operation: ctx.operation,
    durationMs: duration,
    ...fields,
  };

  logger[level](logData, `${ctx.operation} completed`);
}

// ─── IPC Telemetry ───────────────────────────────────────────────────────────

export interface IpcEvent {
  eventType: 'ipc_message' | 'ipc_task' | 'ipc_auth';
  sourceGroup: string;
  isMain: boolean;
  ipcType: string;        // message, schedule_task, pause_task, etc.
  chatJid?: string;
  taskId?: string;
  authorized: boolean;
  error?: string;
}

/**
 * Log an IPC event with full structured context.
 */
export function logIpcEvent(ctx: TraceContext, event: IpcEvent): void {
  const level = event.authorized ? 'info' : 'warn';
  logger[level](
    {
      traceId: ctx.traceId,
      spanId: ctx.spanId,
      operation: ctx.operation,
      durationMs: Date.now() - ctx.startTime,
      ...event,
    },
    event.authorized
      ? `IPC ${event.ipcType} processed`
      : `IPC ${event.ipcType} BLOCKED (unauthorized)`,
  );
}

// ─── Container Telemetry ─────────────────────────────────────────────────────

export interface ContainerEvent {
  eventType: 'container_spawn' | 'container_output' | 'container_close' | 'container_error' | 'container_timeout';
  group: string;
  groupFolder: string;
  containerName: string;
  isMain: boolean;
  isScheduledTask?: boolean;
  exitCode?: number | null;
  hasResult?: boolean;
  mountCount?: number;
  error?: string;
}

/**
 * Log a container lifecycle event.
 */
export function logContainerEvent(ctx: TraceContext, event: ContainerEvent): void {
  const level = event.eventType === 'container_error' || event.eventType === 'container_timeout'
    ? 'error'
    : 'info';
  logger[level](
    {
      traceId: ctx.traceId,
      spanId: ctx.spanId,
      operation: ctx.operation,
      durationMs: Date.now() - ctx.startTime,
      ...event,
    },
    `Container ${event.eventType.replace('container_', '')}`,
  );
}

// ─── Message Pipeline Telemetry ──────────────────────────────────────────────

export interface MessageEvent {
  eventType: 'message_received' | 'message_queued' | 'message_sent' | 'message_error';
  channel: string;
  chatJid: string;
  groupFolder?: string;
  sender?: string;
  messageLength?: number;
  triggered?: boolean;
  error?: string;
}

/**
 * Log a message pipeline event.
 */
export function logMessageEvent(ctx: TraceContext, event: MessageEvent): void {
  const level = event.eventType === 'message_error' ? 'error' : 'info';
  logger[level](
    {
      traceId: ctx.traceId,
      spanId: ctx.spanId,
      operation: ctx.operation,
      durationMs: Date.now() - ctx.startTime,
      ...event,
    },
    `Message ${event.eventType.replace('message_', '')}`,
  );
}

// ─── Task Scheduler Telemetry ────────────────────────────────────────────────

export interface TaskEvent {
  eventType: 'task_due' | 'task_started' | 'task_completed' | 'task_error';
  taskId: string;
  groupFolder: string;
  scheduleType: string;
  scheduleValue: string;
  contextMode?: string;
  nextRun?: string | null;
  error?: string;
}

/**
 * Log a task scheduler event.
 */
export function logTaskEvent(ctx: TraceContext, event: TaskEvent): void {
  const level = event.eventType === 'task_error' ? 'error' : 'info';
  logger[level](
    {
      traceId: ctx.traceId,
      spanId: ctx.spanId,
      operation: ctx.operation,
      durationMs: Date.now() - ctx.startTime,
      ...event,
    },
    `Task ${event.eventType.replace('task_', '')}`,
  );
}

// ─── Metrics Helpers ─────────────────────────────────────────────────────────

/** Counters stored in-memory, logged periodically for Prometheus scraping */
const counters: Record<string, number> = {};
const gauges: Record<string, number> = {};

export function incrementCounter(name: string, value = 1): void {
  counters[name] = (counters[name] || 0) + value;
}

export function setGauge(name: string, value: number): void {
  gauges[name] = value;
}

/**
 * Flush metrics as a structured log line.
 * Call this periodically (e.g., every 60s) from the main loop.
 */
export function flushMetrics(): void {
  if (Object.keys(counters).length === 0 && Object.keys(gauges).length === 0) return;

  logger.info(
    {
      eventType: 'metrics_flush',
      counters: { ...counters },
      gauges: { ...gauges },
      timestamp: new Date().toISOString(),
    },
    'Metrics flush',
  );

  // Reset counters (gauges keep their values)
  for (const key of Object.keys(counters)) {
    counters[key] = 0;
  }
}
```

### Step 2: Update the logger for JSON output

Read `src/logger.ts` and update it to support both pretty (development) and JSON (production) output:

```typescript
import pino from 'pino';

const isProduction = process.env.NODE_ENV === 'production';
const logFile = process.env.NANOCLAW_LOG_FILE || '';

// JSON mode for Grafana Alloy ingestion; pretty mode for development
const transport = isProduction || logFile
  ? logFile
    ? {
        targets: [
          // Pretty output to stderr for human readability
          { target: 'pino-pretty', options: { colorize: true, destination: 2 } },
          // JSON output to file for Alloy ingestion
          { target: 'pino/file', options: { destination: logFile, mkdir: true } },
        ],
      }
    : undefined  // Raw JSON to stdout in production (for Alloy to capture)
  : { target: 'pino-pretty', options: { colorize: true } };

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  ...(transport ? { transport } : {}),
  // Add service metadata for Grafana
  base: {
    service: 'nanoclaw',
    version: process.env.npm_package_version || 'dev',
  },
});

// Route uncaught errors through pino so they get timestamps in stderr
process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'Uncaught exception');
  process.exit(1);
});

process.on('unhandledRejection', (reason) => {
  logger.error({ err: reason }, 'Unhandled rejection');
});
```

## Phase 3: Instrument the IPC System

### Step 1: Instrument IPC watcher

Read `src/ipc.ts` and add the import:

```typescript
import {
  startTrace,
  endTrace,
  logIpcEvent,
  incrementCounter,
} from './telemetry.js';
```

Wrap the message processing in `processIpcFiles`:

In the message processing loop, after `const data = JSON.parse(...)`, add tracing:

```typescript
const ctx = startTrace('ipc.message');
```

After `await deps.sendMessage(data.chatJid, data.text);`, add:

```typescript
logIpcEvent(ctx, {
  eventType: 'ipc_message',
  sourceGroup,
  isMain,
  ipcType: 'message',
  chatJid: data.chatJid,
  authorized: true,
});
incrementCounter('ipc_messages_sent');
```

For the unauthorized block, add:

```typescript
logIpcEvent(ctx, {
  eventType: 'ipc_auth',
  sourceGroup,
  isMain,
  ipcType: 'message',
  chatJid: data.chatJid,
  authorized: false,
});
incrementCounter('ipc_messages_blocked');
```

In the task processing, wrap `processTaskIpc`:

```typescript
const taskCtx = startTrace('ipc.task');
await processTaskIpc(data, sourceGroup, isMain, deps);
endTrace(taskCtx, {
  eventType: 'ipc_task',
  sourceGroup,
  isMain,
  ipcType: data.type,
  taskId: data.taskId,
});
incrementCounter('ipc_tasks_processed');
```

### Step 2: Instrument processTaskIpc

In the `processTaskIpc` function, add tracing for authorization decisions. For each `case` block that has an authorization check, log the decision:

For the `schedule_task` case, after the authorization check:

```typescript
if (!isMain && targetFolder !== sourceGroup) {
  logIpcEvent(startTrace('ipc.auth'), {
    eventType: 'ipc_auth',
    sourceGroup,
    isMain,
    ipcType: 'schedule_task',
    authorized: false,
  });
  incrementCounter('ipc_tasks_blocked');
  // ... existing logger.warn
  break;
}
```

Apply the same pattern to `pause_task`, `resume_task`, `cancel_task`, `update_task`, `register_group`, and `refresh_groups`.

## Phase 4: Instrument the Container Runner

### Step 1: Add tracing to container lifecycle

Read `src/container-runner.ts` and add the import:

```typescript
import {
  startTrace,
  logContainerEvent,
  incrementCounter,
  setGauge,
} from './telemetry.js';
```

In `runContainerAgent()`, create a trace at the start:

```typescript
export async function runContainerAgent(
  group: RegisteredGroup,
  input: ContainerInput,
  onProcess: (proc: ChildProcess, containerName: string) => void,
  onOutput?: (output: ContainerOutput) => Promise<void>,
): Promise<ContainerOutput> {
  const startTime = Date.now();
  const ctx = startTrace('container.run');

  // ... existing code ...
```

After container spawns successfully, log the event:

```typescript
logContainerEvent(ctx, {
  eventType: 'container_spawn',
  group: group.name,
  groupFolder: group.folder,
  containerName,
  isMain: input.isMain,
  isScheduledTask: input.isScheduledTask,
  mountCount: mounts.length,
});
incrementCounter('containers_spawned');
```

In the `close` handler, log completion:

```typescript
container.on('close', (code) => {
  clearTimeout(timeout);
  const duration = Date.now() - startTime;

  logContainerEvent(ctx, {
    eventType: code !== 0 ? 'container_error' : 'container_close',
    group: group.name,
    groupFolder: group.folder,
    containerName,
    isMain: input.isMain,
    exitCode: code,
    hasResult: hadStreamingOutput,
  });
  incrementCounter(code !== 0 ? 'container_errors' : 'containers_completed');

  // ... rest of existing close handler
```

On timeout, log the event:

```typescript
const killOnTimeout = () => {
  timedOut = true;
  logContainerEvent(ctx, {
    eventType: 'container_timeout',
    group: group.name,
    groupFolder: group.folder,
    containerName,
    isMain: input.isMain,
  });
  incrementCounter('container_timeouts');
  // ... existing timeout handling
```

## Phase 5: Add Metrics Flushing

### Step 1: Add periodic metrics flush

Read `src/index.ts` and add the import:

```typescript
import { flushMetrics, setGauge } from './telemetry.js';
```

In the main loop (or as a separate interval), add metrics flushing:

```typescript
// Flush telemetry metrics every 60 seconds
setInterval(() => {
  setGauge('active_containers', /* get count from GroupQueue */);
  setGauge('registered_groups', Object.keys(registeredGroups).length);
  flushMetrics();
}, 60_000);
```

## Phase 6: Grafana Alloy Configuration (Optional)

If the user wants Alloy configuration, create `alloy/config.alloy`:

```hcl
// NanoClaw Telemetry Pipeline
// Grafana Alloy configuration for ingesting NanoClaw structured logs

// ─── Log Source ──────────────────────────────────────────────────────────────

local.file_match "nanoclaw_logs" {
  path_targets = [{
    __path__ = "/path/to/nanoclaw/logs/nanoclaw-telemetry.json",
    job       = "nanoclaw",
    instance  = "nanoclaw-host",
  }]
}

loki.source.file "nanoclaw" {
  targets    = local.file_match.nanoclaw_logs.targets
  forward_to = [loki.process.nanoclaw.receiver]
}

// ─── Log Processing ─────────────────────────────────────────────────────────

loki.process "nanoclaw" {
  // Parse JSON structured logs
  stage.json {
    expressions = {
      level     = "level",
      msg       = "msg",
      traceId   = "traceId",
      spanId    = "spanId",
      operation = "operation",
      eventType = "eventType",
      group     = "group",
      duration  = "durationMs",
    }
  }

  // Add labels for efficient querying
  stage.labels {
    values = {
      level     = "",
      operation = "",
      eventType = "",
      group     = "",
    }
  }

  // Extract metrics from log lines
  stage.metrics {
    metric.counter {
      name        = "nanoclaw_ipc_messages_total"
      description = "Total IPC messages processed"
      match_all   = false
      source      = "eventType"
      value       = "ipc_message"
      action      = "inc"
    }
    metric.counter {
      name        = "nanoclaw_containers_total"
      description = "Total containers spawned"
      match_all   = false
      source      = "eventType"
      value       = "container_spawn"
      action      = "inc"
    }
    metric.histogram {
      name        = "nanoclaw_container_duration_ms"
      description = "Container execution duration in milliseconds"
      source      = "duration"
      buckets     = [1000, 5000, 10000, 30000, 60000, 300000, 900000, 1800000]
    }
    metric.counter {
      name        = "nanoclaw_auth_blocked_total"
      description = "Total unauthorized IPC attempts blocked"
      match_all   = false
      source      = "eventType"
      value       = "ipc_auth"
      action      = "inc"
    }
  }

  forward_to = [loki.write.default.receiver]
}

// ─── Loki Output ─────────────────────────────────────────────────────────────

loki.write "default" {
  endpoint {
    url = "LOKI_PUSH_URL_HERE"
  }
}

// ─── Prometheus Metrics (Optional) ───────────────────────────────────────────
// Uncomment to expose metrics endpoint for Prometheus scraping

// prometheus.exporter.unix "default" {}
//
// prometheus.scrape "nanoclaw_metrics" {
//   targets = prometheus.exporter.unix.default.targets
//   forward_to = [prometheus.remote_write.default.receiver]
// }
//
// prometheus.remote_write "default" {
//   endpoint {
//     url = "PROMETHEUS_REMOTE_WRITE_URL_HERE"
//   }
// }
```

Replace `LOKI_PUSH_URL_HERE` with the user's Loki URL. Replace the log file path with the actual path.

### Update .env for telemetry output

Add to `.env`:

```bash
# === Observability ===
# Log file for structured JSON output (Grafana Alloy ingests this)
NANOCLAW_LOG_FILE=logs/nanoclaw-telemetry.json
# Set to 'production' for JSON-only output (no pretty printing)
# NODE_ENV=production
```

### Alloy as a Docker container (optional)

If the user wants Alloy running as a container:

```bash
mkdir -p alloy

docker run -d \
  --name nanoclaw-alloy \
  --restart unless-stopped \
  -v "$(pwd)/alloy/config.alloy:/etc/alloy/config.alloy:ro" \
  -v "$(pwd)/logs:/logs:ro" \
  -p 12345:12345 \
  grafana/alloy:latest \
  run /etc/alloy/config.alloy
```

## Phase 7: Build and Verify

### Build

```bash
npm run build
```

### Restart

```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

### Test telemetry output

Send a message to the agent and check the log output:

```bash
tail -f logs/nanoclaw-telemetry.json | python3 -m json.tool
```

Look for entries with:
- `traceId` — 32-character hex trace ID
- `operation` — e.g., `ipc.message`, `container.run`
- `durationMs` — operation duration
- `eventType` — e.g., `ipc_message`, `container_spawn`

### Verify trace correlation

A single user message should produce a chain of events sharing the same `traceId`:

```
traceId=abc123  operation=message.receive    eventType=message_received
traceId=abc123  operation=container.run      eventType=container_spawn
traceId=abc123  operation=ipc.message        eventType=ipc_message
traceId=abc123  operation=container.run      eventType=container_close
traceId=abc123  operation=message.send       eventType=message_sent
```

### Verify metrics flush

Wait 60 seconds and check for metrics:

```bash
grep "metrics_flush" logs/nanoclaw-telemetry.json | tail -1 | python3 -m json.tool
```

## Future: /add-audit-log

The audit log skill piggybacks on this observability foundation:

1. **Subscribe to telemetry events** — filter for security-relevant operations
2. **Write to append-only audit log** — separate file, never truncated
3. **Events to audit**:
   - All authorization decisions (allowed and blocked)
   - Container spawns with mount details
   - IPC task operations (schedule, pause, cancel)
   - Group registration
   - Secret access patterns
4. **Tamper protection** — write to external path (`~/.config/nanoclaw/audit.log`)
5. **Feed into Alloy** — separate Loki stream with retention policy

This is a thin layer on top of the telemetry module — the hard part (structured events, trace IDs, operation context) is done by this skill.

## Troubleshooting

### No telemetry output

1. Check `NANOCLAW_LOG_FILE` is set in `.env`
2. Check the log directory exists: `ls -la logs/`
3. Check permissions: the NanoClaw process must be able to write to the log file
4. Verify build is clean: `npm run build`

### Alloy not ingesting logs

1. Check Alloy is running: `docker logs nanoclaw-alloy`
2. Verify log file path in Alloy config matches actual path
3. Check Loki endpoint is reachable from Alloy container
4. Test with: `curl -s http://localhost:12345/ready`

### Missing trace IDs

Trace IDs are generated per-operation. If you see log lines without `traceId`, they're from code paths that haven't been instrumented yet. The main paths (IPC, containers, messages) are all instrumented.

### Performance impact

The telemetry module adds:
- ~1ms per IPC operation (JSON serialization + write)
- ~50KB/hour of log output (varies with traffic)
- No network calls (all file-based)
- No blocking operations

## Removal

To remove IPC observability:

1. Delete `src/telemetry.ts`
2. Revert `src/logger.ts` to original (remove JSON transport, base fields)
3. Remove telemetry imports and calls from `src/ipc.ts`, `src/container-runner.ts`, `src/index.ts`
4. Remove `NANOCLAW_LOG_FILE` from `.env`
5. Remove `alloy/` directory (if created)
6. Rebuild: `npm run build`
7. Restart: `launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `systemctl --user restart nanoclaw` (Linux)
