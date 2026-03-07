---
name: add-group-checkpointing
description: Add git-based checkpointing for agent config files (CLAUDE.md, memories, session settings). Every container exit auto-commits changes to a tamper-proof repo outside the project root. Optional remote push for off-site audit trail. Triggers on "checkpoint", "config versioning", "rollback", "group history", "agent versioning".
---

# Add Group Checkpointing

This skill adds automatic git-based versioning of agent-modifiable files. After every container exit, changed files are committed to a separate git repository stored outside the project root. This gives you:

- **Full history** of every change an agent makes to its own identity, memories, and settings
- **Instant rollback** if an agent corrupts its CLAUDE.md or rewrites its personality
- **Tamper-proof audit trail** when pushed to a remote the agent cannot reach
- **Blame tracking** — which container run changed which line

## What Gets Checkpointed

| Source (host path) | Tracked As | Why |
|--------------------|-----------|-----|
| `groups/{folder}/CLAUDE.md` | `groups/{folder}/CLAUDE.md` | Agent identity, instructions, memories |
| `groups/{folder}/**` | `groups/{folder}/**` | Any files agent creates in its group dir |
| `data/sessions/{folder}/.claude/settings.json` | `sessions/{folder}/settings.json` | Claude Code environment settings |
| `groups/global/CLAUDE.md` | `groups/global/CLAUDE.md` | Shared global memory |

**What is NOT checkpointed** (too large, too ephemeral, or already logged elsewhere):
- Container logs (`groups/{folder}/logs/`) — already written per-run
- IPC files (`data/ipc/`) — ephemeral, consumed and deleted
- Agent runner source (`data/sessions/{folder}/agent-runner-src/`) — copied from template each run
- Claude session data (`data/sessions/{folder}/.claude/projects/`) — SDK internal state

## Architecture

```
Container Exit (container-runner.ts)
        │
        └── checkpointGroup(group, containerName, exitCode, duration)
                │
                ├── rsync group files → ~/.config/nanoclaw/group-checkpoints/
                ├── git add -A
                ├── git commit (with structured message)
                └── git push (if remote configured, async, best-effort)
```

The checkpoint repo lives at `~/.config/nanoclaw/group-checkpoints/` — **outside the project root**:
- Agents cannot read, modify, or rewrite history (not mounted into containers)
- Survives project directory deletion
- Remote push credentials live in system git config, never in `.env`

## Phase 1: Create the Checkpoint Module

Create `src/checkpoint.ts`:

```typescript
/**
 * Git-based checkpointing for agent-modifiable files.
 * After each container exit, changed files are committed to a separate
 * git repo outside the project root (tamper-proof from containers).
 *
 * Commit messages include structured metadata:
 *   group, container name, exit code, duration, timestamp
 *
 * Optional remote push for off-site audit trail.
 */
import fs from 'fs';
import path from 'path';
import os from 'os';
import { execSync } from 'child_process';

import { DATA_DIR, GROUPS_DIR } from './config.js';
import { logger } from './logger.js';

const CHECKPOINT_DIR = process.env.NANOCLAW_CHECKPOINT_DIR || path.join(
  os.homedir(),
  '.config',
  'nanoclaw',
  'group-checkpoints',
);

const CHECKPOINT_REMOTE = process.env.NANOCLAW_CHECKPOINT_REMOTE || '';

let initialized = false;

// ─── Initialization ─────────────────────────────────────────────────────────

/** Initialize the checkpoint git repo if it doesn't exist. */
export function initCheckpointRepo(): void {
  if (initialized) return;

  try {
    fs.mkdirSync(CHECKPOINT_DIR, { recursive: true });

    const gitDir = path.join(CHECKPOINT_DIR, '.git');
    if (!fs.existsSync(gitDir)) {
      execSync('git init', { cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 10000 });
      // Configure repo-local identity (doesn't affect user's global git config)
      execSync('git config user.name "NanoClaw Checkpoint"', {
        cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 5000,
      });
      execSync('git config user.email "checkpoint@nanoclaw.local"', {
        cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 5000,
      });

      // Initial commit so we have a HEAD to diff against
      const readmePath = path.join(CHECKPOINT_DIR, 'README.md');
      fs.writeFileSync(readmePath, [
        '# NanoClaw Group Checkpoints',
        '',
        'Auto-generated git history of agent-modifiable files.',
        'Each commit represents one container exit.',
        '',
        'DO NOT EDIT — this repo is managed by NanoClaw.',
      ].join('\n'));
      execSync('git add -A && git commit -m "init: checkpoint repo"', {
        cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 10000,
      });

      logger.info({ path: CHECKPOINT_DIR }, 'Checkpoint repo initialized');
    }

    // Configure remote if specified and not already set
    if (CHECKPOINT_REMOTE) {
      try {
        const currentRemote = execSync('git remote get-url origin', {
          cwd: CHECKPOINT_DIR, stdio: 'pipe', encoding: 'utf-8', timeout: 5000,
        }).trim();
        if (currentRemote !== CHECKPOINT_REMOTE) {
          execSync(`git remote set-url origin "${CHECKPOINT_REMOTE}"`, {
            cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 5000,
          });
        }
      } catch {
        execSync(`git remote add origin "${CHECKPOINT_REMOTE}"`, {
          cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 5000,
        });
      }
      logger.info({ remote: CHECKPOINT_REMOTE }, 'Checkpoint remote configured');
    }

    initialized = true;
  } catch (err) {
    logger.error({ err }, 'Failed to initialize checkpoint repo');
  }
}

// ─── File Syncing ───────────────────────────────────────────────────────────

/**
 * Copy tracked files from source into the checkpoint repo.
 * Mirrors the directory structure so git diffs are meaningful.
 */
function syncGroupFiles(groupFolder: string): void {
  const srcGroupDir = path.join(GROUPS_DIR, groupFolder);
  const dstGroupDir = path.join(CHECKPOINT_DIR, 'groups', groupFolder);

  if (!fs.existsSync(srcGroupDir)) return;

  // Ensure destination exists
  fs.mkdirSync(dstGroupDir, { recursive: true });

  // Copy group files (excluding logs/ which are large and already captured)
  copyDirRecursive(srcGroupDir, dstGroupDir, ['logs']);

  // Copy session settings if they exist
  const sessionSettings = path.join(DATA_DIR, 'sessions', groupFolder, '.claude', 'settings.json');
  if (fs.existsSync(sessionSettings)) {
    const dstSettings = path.join(CHECKPOINT_DIR, 'sessions', groupFolder, 'settings.json');
    fs.mkdirSync(path.dirname(dstSettings), { recursive: true });
    fs.copyFileSync(sessionSettings, dstSettings);
  }
}

function syncGlobalFiles(): void {
  const globalDir = path.join(GROUPS_DIR, 'global');
  const dstGlobalDir = path.join(CHECKPOINT_DIR, 'groups', 'global');

  if (!fs.existsSync(globalDir)) return;
  fs.mkdirSync(dstGlobalDir, { recursive: true });
  copyDirRecursive(globalDir, dstGlobalDir, ['logs']);
}

/** Recursive copy with exclusion list. */
function copyDirRecursive(src: string, dst: string, exclude: string[] = []): void {
  const entries = fs.readdirSync(src, { withFileTypes: true });
  for (const entry of entries) {
    if (exclude.includes(entry.name)) continue;
    // Skip hidden dirs except .claude-related
    if (entry.name.startsWith('.') && entry.name !== '.claude') continue;

    const srcPath = path.join(src, entry.name);
    const dstPath = path.join(dst, entry.name);

    if (entry.isDirectory()) {
      fs.mkdirSync(dstPath, { recursive: true });
      copyDirRecursive(srcPath, dstPath, exclude);
    } else if (entry.isFile()) {
      // Skip binary files and large files (> 1MB)
      try {
        const stat = fs.statSync(srcPath);
        if (stat.size > 1_048_576) continue; // Skip files > 1MB
        fs.copyFileSync(srcPath, dstPath);
      } catch {
        // Skip files we can't read
      }
    }
  }
}

// ─── Checkpoint Logic ───────────────────────────────────────────────────────

export interface CheckpointMeta {
  group: string;
  groupFolder: string;
  containerName: string;
  exitCode: number | null;
  durationMs: number;
  timedOut: boolean;
}

/**
 * Checkpoint a group's files after container exit.
 * Copies files to the checkpoint repo, commits changes (if any),
 * and optionally pushes to the remote.
 *
 * This function never throws — checkpoint failures must not affect
 * the main process.
 */
export function checkpointGroup(meta: CheckpointMeta): void {
  try {
    initCheckpointRepo();

    // Sync files from live locations into checkpoint repo
    syncGroupFiles(meta.groupFolder);
    syncGlobalFiles();

    // Check if there are any changes
    execSync('git add -A', {
      cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 10000,
    });

    const status = execSync('git status --porcelain', {
      cwd: CHECKPOINT_DIR, stdio: 'pipe', encoding: 'utf-8', timeout: 5000,
    }).trim();

    if (!status) {
      // No changes — skip commit
      logger.debug({ group: meta.group }, 'Checkpoint: no changes');
      return;
    }

    // Build structured commit message
    const changedFiles = status.split('\n').length;
    const subject = `checkpoint: ${meta.group} (${meta.exitCode === 0 ? 'ok' : `exit ${meta.exitCode}`})`;
    const body = [
      '',
      `group: ${meta.group}`,
      `folder: ${meta.groupFolder}`,
      `container: ${meta.containerName}`,
      `exit_code: ${meta.exitCode}`,
      `duration_ms: ${meta.durationMs}`,
      `timed_out: ${meta.timedOut}`,
      `files_changed: ${changedFiles}`,
      `timestamp: ${new Date().toISOString()}`,
    ].join('\n');

    const message = subject + '\n' + body;
    const msgFile = path.join(CHECKPOINT_DIR, '.commit-msg');
    fs.writeFileSync(msgFile, message);

    execSync('git commit -F .commit-msg', {
      cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 15000,
    });

    // Clean up temp file
    try { fs.unlinkSync(msgFile); } catch { /* ignore */ }

    logger.info(
      { group: meta.group, changedFiles, containerName: meta.containerName },
      'Checkpoint committed',
    );

    // Push to remote (async, best-effort)
    if (CHECKPOINT_REMOTE) {
      pushToRemote();
    }
  } catch (err) {
    // Checkpoint failures must NEVER crash the process
    logger.error({ err, group: meta.group }, 'Failed to create checkpoint');
  }
}

/**
 * Push to remote asynchronously. Best-effort — failures are logged
 * but do not affect the main process.
 */
function pushToRemote(): void {
  try {
    // Use spawn instead of execSync to avoid blocking
    const { exec: execAsync } = require('child_process');
    execAsync(
      'git push origin HEAD 2>&1',
      { cwd: CHECKPOINT_DIR, timeout: 30000 },
      (err: Error | null, _stdout: string, stderr: string) => {
        if (err) {
          logger.warn({ err: stderr || err.message }, 'Checkpoint push failed (will retry next commit)');
        } else {
          logger.debug('Checkpoint pushed to remote');
        }
      },
    );
  } catch (err) {
    logger.warn({ err }, 'Checkpoint push failed');
  }
}

// ─── Manual Operations ──────────────────────────────────────────────────────

/** Get recent checkpoint history for a group. */
export function getCheckpointHistory(
  groupFolder: string,
  count: number = 10,
): string {
  try {
    initCheckpointRepo();
    return execSync(
      `git log --oneline -${count} -- "groups/${groupFolder}/"`,
      { cwd: CHECKPOINT_DIR, encoding: 'utf-8', stdio: 'pipe', timeout: 5000 },
    ).trim();
  } catch {
    return '';
  }
}

/** Show diff between current state and last checkpoint for a group. */
export function getCheckpointDiff(groupFolder: string): string {
  try {
    initCheckpointRepo();
    syncGroupFiles(groupFolder);
    execSync('git add -A', {
      cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 10000,
    });
    return execSync(
      `git diff --cached -- "groups/${groupFolder}/"`,
      { cwd: CHECKPOINT_DIR, encoding: 'utf-8', stdio: 'pipe', timeout: 5000 },
    ).trim();
  } catch {
    return '';
  }
}

/**
 * Rollback a group's files to a specific commit.
 * Returns true if rollback succeeded.
 */
export function rollbackGroup(groupFolder: string, commitHash: string): boolean {
  try {
    initCheckpointRepo();

    // Restore files from the specified commit into the checkpoint repo
    execSync(`git checkout ${commitHash} -- "groups/${groupFolder}/"`, {
      cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 10000,
    });

    // Copy restored files back to the live group directory
    const checkpointGroupDir = path.join(CHECKPOINT_DIR, 'groups', groupFolder);
    const liveGroupDir = path.join(GROUPS_DIR, groupFolder);

    if (fs.existsSync(checkpointGroupDir)) {
      copyDirRecursive(checkpointGroupDir, liveGroupDir);
    }

    // Restore session settings
    const checkpointSettings = path.join(CHECKPOINT_DIR, 'sessions', groupFolder, 'settings.json');
    if (fs.existsSync(checkpointSettings)) {
      const liveSettings = path.join(DATA_DIR, 'sessions', groupFolder, '.claude', 'settings.json');
      fs.copyFileSync(checkpointSettings, liveSettings);
    }

    // Commit the rollback itself
    execSync('git add -A', {
      cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 10000,
    });
    execSync(
      `git commit -m "rollback: ${groupFolder} to ${commitHash.slice(0, 7)}"`,
      { cwd: CHECKPOINT_DIR, stdio: 'pipe', timeout: 10000 },
    );

    logger.info({ groupFolder, commitHash }, 'Group rolled back');
    return true;
  } catch (err) {
    logger.error({ err, groupFolder, commitHash }, 'Rollback failed');
    return false;
  }
}
```

## Phase 2: Integrate with Container Runner

### Step 1: Add the import

Read `src/container-runner.ts` and add:

```typescript
import { checkpointGroup } from './checkpoint.js';
```

### Step 2: Add post-exit checkpoint hook

In `runContainerAgent()`, inside the `container.on('close', ...)` handler, add the checkpoint call **at the very end of the handler**, right before the closing `});` of the close handler, after all resolve paths.

The checkpoint runs in every exit path (success, error, timeout-with-output). Add it as a separate block at the bottom of the close handler:

```typescript
    // ── Post-exit checkpoint ──────────────────────────────────────────
    // Runs after resolve() so it never blocks response delivery.
    // Placed at the end of the close handler, outside all if/return blocks.
    // We use setImmediate so the checkpoint runs after the current
    // event loop tick (after resolve/promise callbacks settle).
    setImmediate(() => {
      checkpointGroup({
        group: group.name,
        groupFolder: group.folder,
        containerName,
        exitCode: code,
        durationMs: Date.now() - startTime,
        timedOut,
      });
    });
```

**Important**: This must be the very last statement in the `close` handler, outside all `if (timedOut)` / `if (code !== 0)` blocks. It runs regardless of the exit path because every exit — normal, error, or timeout — may have changed files. Using `setImmediate` ensures the `resolve()` callbacks fire first, so checkpoint I/O never delays message delivery.

## Phase 3: Optional — Startup Initialization

Read `src/index.ts` and add:

```typescript
import { initCheckpointRepo } from './checkpoint.js';
```

After other initialization calls (e.g., `ensureContainerRuntimeRunning()`):

```typescript
// Initialize checkpoint repo (creates if needed, configures remote)
initCheckpointRepo();
```

This is optional — `checkpointGroup()` calls `initCheckpointRepo()` lazily. But initializing at startup catches configuration errors early and logs the repo path.

## Phase 4: Environment Configuration

Add to `.env` (both are optional):

```bash
# Git-based checkpointing (optional)
# Directory for the checkpoint git repo (default: ~/.config/nanoclaw/group-checkpoints/)
# NANOCLAW_CHECKPOINT_DIR=~/.config/nanoclaw/group-checkpoints

# Remote to push checkpoints to (default: none — local only)
# Use SSH URL for key-based auth, HTTPS for token-based
# NANOCLAW_CHECKPOINT_REMOTE=git@github.com:yourorg/nanoclaw-checkpoints.git
```

### Remote setup

If the user wants remote push:

```bash
# Create a private repo on GitHub/GitLab/Gitea
# Then configure the remote:
cd ~/.config/nanoclaw/group-checkpoints
git remote add origin git@github.com:yourorg/nanoclaw-checkpoints.git

# SSH keys must be in the HOST user's ~/.ssh/ (not in .env, not in containers)
# Test: ssh -T git@github.com
```

**Security**: Remote credentials (SSH keys, git credential helpers) live in the host user's home directory. They are never in `.env`, never mounted into containers, and never accessible to agents.

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

### Trigger a checkpoint

Send a message to any agent. After it responds, check:

```bash
cd ~/.config/nanoclaw/group-checkpoints
git log --oneline
```

You should see:

```
a1b2c3d checkpoint: Main (ok)
f4e5d6c init: checkpoint repo
```

### Inspect changes

```bash
# See what the agent changed in its last run
git show HEAD --stat

# Full diff of last change
git show HEAD

# History for a specific group
git log --oneline -- groups/main/

# Diff between any two points
git diff HEAD~5 HEAD -- groups/main/CLAUDE.md
```

### Test rollback

```bash
# Find the commit to rollback to
git log --oneline -- groups/main/

# From the NanoClaw process (programmatic):
# rollbackGroup('main', 'a1b2c3d')

# Manual rollback:
cd ~/.config/nanoclaw/group-checkpoints
git checkout a1b2c3d -- groups/main/
cp -r groups/main/* /path/to/nanoclaw/groups/main/
git add -A && git commit -m "rollback: main to a1b2c3d"
```

## Commit Message Format

Every checkpoint commit has this structure:

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

This makes commits searchable:

```bash
# Find all error exits
git log --grep="exit_code: [^0]" --oneline

# Find all checkpoints for a group
git log --grep="folder: main" --oneline

# Find timed-out runs
git log --grep="timed_out: true" --oneline

# Find runs that changed many files (possible corruption)
git log --grep="files_changed:" --oneline | sort -t: -k2 -rn
```

## Integration with /add-audit-log

If `/add-audit-log` is applied, the checkpoint module complements it:

| Audit Log | Checkpoint |
|-----------|------------|
| Records WHAT happened (operations, events) | Records the RESULT (file changes) |
| "Container spawned, IPC sent, task scheduled" | "CLAUDE.md line 42 changed from X to Y" |
| Real-time stream | Point-in-time snapshots |
| Loki/Grafana queryable | Git diff/blame/bisect |

Together they give you the full picture: the audit log shows the sequence of operations, and the checkpoint shows the state changes those operations produced.

## Gitignore for Checkpoint Repo

The checkpoint module automatically excludes:
- Files > 1MB (binary assets, large outputs)
- Hidden directories (except `.claude`)
- `logs/` directories

If you need additional exclusions, create `~/.config/nanoclaw/group-checkpoints/.gitignore`:

```gitignore
# Ignore large or binary files
*.sqlite
*.db
*.png
*.jpg
*.pdf
*.zip
```

## Troubleshooting

### Checkpoints not appearing

1. Check the repo exists: `ls ~/.config/nanoclaw/group-checkpoints/.git`
2. Check logs: `grep "checkpoint" logs/nanoclaw.log`
3. Verify git is installed: `git --version`
4. Check permissions: `touch ~/.config/nanoclaw/group-checkpoints/test && rm $_`

### Push failures

1. Test SSH auth: `ssh -T git@github.com`
2. Check remote URL: `cd ~/.config/nanoclaw/group-checkpoints && git remote -v`
3. Push manually: `cd ~/.config/nanoclaw/group-checkpoints && git push origin HEAD`
4. Push failures are non-blocking — changes accumulate locally and push on next success

### Repo growing too large

```bash
# Check size
du -sh ~/.config/nanoclaw/group-checkpoints/.git

# Aggressive gc
cd ~/.config/nanoclaw/group-checkpoints && git gc --aggressive

# Nuclear option: squash old history (keeps last N commits)
cd ~/.config/nanoclaw/group-checkpoints
git checkout --orphan temp HEAD~50  # Keep last 50
git commit -m "squash: historical checkpoints"
git branch -M temp main
git gc --aggressive
```

### Agents cannot access the checkpoint repo

This is by design. The checkpoint repo is at `~/.config/nanoclaw/group-checkpoints/` which is never mounted into containers. Agents see:
- `/workspace/group/` — their own group folder (read-write)
- `/workspace/project/` — the project root (read-only, main only)
- `/workspace/global/` — global memory (read-only, non-main)

None of these include the checkpoint repo.

## Removal

1. Delete `src/checkpoint.ts`
2. Remove `import { checkpointGroup } from './checkpoint.js'` from `src/container-runner.ts`
3. Remove the `setImmediate(() => { checkpointGroup(...) })` block from the close handler
4. Remove `import { initCheckpointRepo } from './checkpoint.js'` from `src/index.ts` (if added)
5. Remove the `initCheckpointRepo()` call from `src/index.ts` (if added)
6. Rebuild: `npm run build`
7. Optionally remove the checkpoint repo: `rm -rf ~/.config/nanoclaw/group-checkpoints`
8. Restart NanoClaw
