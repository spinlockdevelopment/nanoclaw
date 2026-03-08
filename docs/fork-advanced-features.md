# NanoClaw Advanced Features & Ideas — Conversation Fork

This document captures all feature ideas discussed but not implemented during the original conversation. These range from BastionClaw ports to entirely new capabilities. Use this as a backlog for future development conversations.

## Codebase Context

NanoClaw is a personal Claude assistant with a channel-based architecture. Messages arrive from channels (WhatsApp, Telegram, Slack, Discord, Gmail), route to Claude agents running in Docker containers, each with per-group isolated filesystems. BastionClaw is included as a submodule and contains several features not present in NanoClaw.

### What NanoClaw Already Has

- Channel system with self-registration (`registerChannel()`)
- Telegram swarm (bot pool, round-robin assignment, group chat with all bots)
- Docker containers with per-group mounts and filesystem isolation
- Filesystem-based IPC with MCP tools (send_message, schedule_task, etc.)
- Task scheduler (cron-like scheduling via IPC)
- Ollama integration (local model tool)
- Voice transcription (Whisper API and local whisper.cpp)
- PDF reader, image vision
- Agent browser (Chromium inside container)

### What BastionClaw Has That NanoClaw Doesn't

Discovered during codebase comparison:

| BastionClaw Feature | NanoClaw Equivalent | Gap |
|---------------------|-------------------|-----|
| Insight engine | None | Full port needed |
| Semantic memory (qmd) | None | Full port needed |
| WebUI control panel | None | Full build needed |
| Prompt injection defense | Now written as skill | Closed |
| Security hooks | Now written as skill | Closed |
| YouTube pipeline | None | Full build needed |
| Discord channel | Already exists | None |
| Image generation (Gemini) | None | Full build needed |

---

## Feature 1: Insight Engine (From BastionClaw)

### What It Does

The insight engine in BastionClaw performs semantic deduplication of information across conversations. When an agent learns something new, the insight engine checks if it already knows a semantically similar fact. If so, it merges or updates rather than creating duplicates.

### BastionClaw Implementation

Found in BastionClaw's codebase:
- Uses embedding vectors to compare new information against existing knowledge
- Maintains a searchable index of facts/insights
- Deduplication threshold is configurable (cosine similarity)
- Integrates with the agent's memory system

### Why It's Interesting

NanoClaw agents accumulate knowledge in their CLAUDE.md files over time. Without dedup:
- The same fact gets recorded multiple times in different phrasings
- CLAUDE.md files grow unbounded
- Context window gets wasted on redundant information
- Agent "forgets" things because they're buried in duplicate noise

### Implementation Approach for NanoClaw

1. **Embedding generation**: Use a local model (Ollama) or API call to generate embeddings for each "insight" (paragraph or fact in CLAUDE.md)
2. **Vector storage**: SQLite with a vector extension (sqlite-vec), or a simple JSON file with cosine similarity comparison
3. **Integration point**: Post-container hook (like checkpointing) — after each run, extract new facts from CLAUDE.md, compare against existing index, merge duplicates
4. **Skill name**: `/add-insight-engine`

### Complexity

Medium-high. Requires:
- Embedding model access (Ollama or API)
- Vector similarity computation
- CLAUDE.md parsing (extracting structured facts from freeform text)
- Merge strategy (which version of a fact to keep)

### Dependencies

- Would benefit from `/add-ollama-tool` for local embeddings (avoids API costs)
- Would benefit from `/add-litellm-proxy` for API-based embeddings (usage tracking)

---

## Feature 2: Semantic Memory / QMD (From BastionClaw)

### What It Does

QMD (Query-Memory-Document) is BastionClaw's structured memory system. Instead of dumping everything into a flat CLAUDE.md, it organizes agent knowledge into:

- **Queries**: Questions the agent has been asked
- **Memories**: Facts, preferences, and learned information
- **Documents**: Reference material and long-form content

### How It Differs from CLAUDE.md

| CLAUDE.md (Current) | QMD (BastionClaw) |
|---------------------|-------------------|
| Flat markdown file | Structured database |
| Agent writes freeform | Categorized entries (query/memory/doc) |
| Linear growth | Indexed, searchable |
| No dedup | Deduplication via insight engine |
| Full file in context window | Selective retrieval (only relevant memories) |

### Why It's Interesting

The biggest limitation of flat CLAUDE.md files is context window waste. A group that's been running for months might have a 50KB CLAUDE.md — most of which is irrelevant to the current conversation. QMD would enable:

- **Selective retrieval**: Only load memories relevant to the current query
- **Categorization**: "This is a user preference" vs "this is a learned fact" vs "this is reference material"
- **Expiry**: Memories can have TTLs (useful for time-sensitive information)
- **Cross-group knowledge**: Global memories that all groups can access (already partially supported via `groups/global/CLAUDE.md`)

### Implementation Approach

1. **Storage**: SQLite table with columns for type, content, embedding vector, created_at, expires_at, group
2. **Retrieval**: Before each agent invocation, query the memory DB for relevant entries based on the incoming message embedding
3. **Injection**: Add relevant memories to the agent's system prompt or CLAUDE.md dynamically
4. **Write-back**: Agent uses an MCP tool (`remember`, `forget`, `update_memory`) instead of editing CLAUDE.md directly
5. **Skill name**: `/add-semantic-memory`

### Complexity

High. This is the most architecturally significant feature in this document:
- Changes how agents store and retrieve knowledge (fundamental)
- Requires embedding infrastructure (Ollama or API)
- Needs careful migration from existing CLAUDE.md content
- MCP tool additions inside the container

### Relationship to Insight Engine

The insight engine is the dedup layer on top of semantic memory. You could implement semantic memory without the insight engine (just accept duplicates), but the insight engine without semantic memory is less useful (deduping a flat file is harder than deduping structured entries).

**Recommended order**: Semantic memory first, insight engine second.

---

## Feature 3: YouTube Pipeline

### What It Does (In BastionClaw)

Processes YouTube videos for agent consumption:
1. Downloads video (or audio only) via yt-dlp
2. Transcribes audio via Whisper
3. Optionally extracts key frames
4. Summarizes content and stores as a document in the agent's memory

### Why It's Interesting

Agents can't watch videos. But users frequently share YouTube links in chat. Without this pipeline, the agent either ignores the link or gives a generic "I can't watch videos" response. With it, the agent can:
- Summarize the video content
- Answer questions about what was said
- Extract action items or key points
- Cross-reference video content with other knowledge

### Implementation Approach

1. **Container addition**: Add `yt-dlp` and `ffmpeg` to the Dockerfile
2. **MCP tool**: `process_youtube(url)` — downloads audio, transcribes, returns text
3. **Integration**: Agent detects YouTube URLs in messages, calls the tool, gets transcript
4. **Storage**: Transcript stored as a document in the group's memory (or QMD if available)
5. **Skill name**: `/add-youtube-pipeline`

### Complexity

Medium. The individual components are well-established (yt-dlp, Whisper, ffmpeg). The main challenges are:
- Container image size increase (yt-dlp + ffmpeg are large)
- Transcription time (long videos take minutes)
- Storage (transcripts of long videos are large)
- Rate limiting (YouTube may block aggressive downloading)

### Dependencies

- Requires voice transcription infrastructure (Whisper API or local whisper.cpp)
- Would benefit from semantic memory (to store transcripts as searchable documents)
- Network policy must allow YouTube access if egress filtering is enabled

---

## Feature 4: WebUI Control Panel (From BastionClaw)

### What It Does

Web-based dashboard for managing NanoClaw:
- View all groups and their status
- Read conversation history
- Edit CLAUDE.md files through a web editor
- Monitor container status (running, idle, errored)
- View IPC message flow in real-time
- Trigger manual agent invocations
- Manage task schedules

### BastionClaw Implementation

BastionClaw has a full web interface built with a frontend framework. It provides administrative control that's currently only possible via the terminal or messaging channels.

### Why It's Interesting

Currently, managing NanoClaw requires either:
- Sending messages through a channel (limited to what IPC supports)
- SSH into the server and editing files directly
- Reading logs via terminal

A WebUI would make NanoClaw accessible to non-technical users and provide a better overview of system state.

### Implementation Approach

1. **Backend**: Express.js HTTP server running alongside the main NanoClaw process (or as a separate process)
2. **Frontend**: Simple HTML/JS (no heavy framework needed for an admin panel)
3. **WebSocket**: Real-time updates for container status, IPC messages, logs
4. **Auth**: Basic auth or token-based (this is a local/personal tool)
5. **Skill name**: `/add-webui`

### Complexity

Medium-high. It's a full web application:
- HTTP server setup and routing
- Frontend UI (even simple HTML needs design work)
- WebSocket for real-time updates
- File editing with conflict detection
- Security (the WebUI has full system access)

### Why It Was Deprioritized

The user's current workflow is channel-based (Telegram/WhatsApp). Adding a WebUI is useful but not critical — the existing channels provide sufficient control for daily use. The WebUI becomes more valuable as the system scales to many groups.

---

## Feature 5: Image Generation (Via Gemini)

### What It Does

Allows agents to generate images in response to user requests. Uses Google's Gemini model which has native image generation capabilities.

### Implementation Approach

1. **API integration**: Gemini API client in the agent runner
2. **MCP tool**: `generate_image(prompt, options)` — calls Gemini, returns image
3. **Channel delivery**: Route generated image back through the channel (WhatsApp, Telegram, etc.)
4. **Skill name**: `/add-image-generation`

### Complexity

Low-medium. The main challenges are:
- Gemini API authentication (another secret to manage, benefits from LiteLLM proxy)
- Image delivery through different channels (each has different APIs for sending images)
- Cost management (image generation is more expensive than text)

---

## Feature 6: Bidirectional IPC Responses

### What It Does

Currently, IPC is fire-and-forget: an agent writes a JSON file to the IPC directory, and the host process picks it up. There's no response channel — the agent can't wait for a result.

### Why It Matters

Some operations naturally need responses:
- "Send a message to group X" → "Message sent successfully" or "Group X doesn't exist"
- "Schedule a task" → "Task ID: abc123" or "Invalid cron expression"
- "Query another agent" → Wait for the other agent's response

### Implementation Approach

1. **Response files**: After processing an IPC message, write a response to `data/ipc/{group}/responses/{messageId}.json`
2. **Polling in container**: The MCP tool writes the request, then polls for the response file
3. **Timeout**: If no response within N seconds, return error
4. **Skill name**: `/add-ipc-responses`

### Complexity

Low-medium. The filesystem IPC pattern extends naturally to responses. The main design decision is synchronous (agent blocks waiting for response) vs asynchronous (agent continues, response arrives later via callback).

### Why It Was Deprioritized

The existing fire-and-forget model works for most use cases. The agent trusts that its IPC messages will be processed. Adding responses adds complexity and latency. Best reserved for when a specific use case demands it.

---

## Feature 7: A2A Protocol (Considered, Then Dropped)

### What It Is

Google's Agent-to-Agent (A2A) protocol — a standard for AI agents to discover and communicate with each other over HTTP. Defines agent cards (capabilities), task lifecycle, and message format.

### Why It Was Dropped

NanoClaw's architecture is single-host with filesystem IPC. A2A is designed for:
- Cross-organization agent discovery
- Multi-host deployments
- Agents from different vendors communicating

For NanoClaw's use case (all agents on the same machine, managed by the same orchestrator), A2A adds HTTP server overhead inside each container without meaningful benefit over the existing MCP-based IPC.

### When It Would Make Sense

- Multi-host NanoClaw deployment (agents distributed across servers)
- Integrating with external agent systems (other people's agents)
- Publishing agent capabilities for third-party discovery

---

## Feature 8: Telegram Swarm Enhancements

### What Already Exists

The `/add-telegram-swarm` skill provides:
- Bot pool (multiple Telegram bots)
- Round-robin assignment (each subagent gets a stable bot identity)
- `setMyName` for dynamic bot naming
- Group chat with all bots visible

### Potential Enhancements

1. **Skill-based routing**: Instead of round-robin, route messages to the bot whose specialization matches the query (e.g., "code bot" for programming, "research bot" for web searches)
2. **Swarm coordination**: Bots can discuss among themselves before responding to the user
3. **Dynamic scaling**: Spin up additional bots when workload increases
4. **Cross-channel swarms**: Same swarm concept but across Telegram + Discord + Slack

### Why Not Implemented

The existing swarm works well for the user's needs. These enhancements are optimizations rather than new capabilities.

---

## Feature 9: BastionClaw's Security Hooks (Partially Ported)

### What Was Ported

The `/add-prompt-injection` skill ported:
- Input sanitization patterns (regex-based)
- Bash environment stripping hook
- Secret path blocking hook (PreToolUse for Read/Bash)

### What Was NOT Ported

BastionClaw has additional security mechanisms:
- **Output sanitization**: Checks agent responses for leaked secrets before sending to the channel
- **Token budget enforcement**: Limits per-conversation token usage to prevent runaway costs
- **Command allowlisting**: Only specific Bash commands are permitted (not just env stripping)
- **File write restrictions**: Agents can only write to specific directories, even within their read-write mounts

These were noted for a future `/add-container-hardening` skill but not designed or written.

---

## Prioritization Matrix

| Feature | Value | Complexity | Dependencies | Recommendation |
|---------|-------|-----------|--------------|----------------|
| Semantic Memory | High | High | Embeddings (Ollama/API) | Build when ready for a major architecture change |
| Insight Engine | High | Medium-High | Semantic Memory | Build after semantic memory |
| YouTube Pipeline | Medium | Medium | Whisper, ffmpeg | Quick win if Whisper already configured |
| WebUI | Medium | Medium-High | None | Build when managing many groups |
| Image Generation | Low-Medium | Low-Medium | Gemini API, LiteLLM | Quick win, nice-to-have |
| IPC Responses | Low-Medium | Low-Medium | None | Build when a use case demands it |
| A2A Protocol | Low | High | HTTP server in containers | Skip unless multi-host |
| Swarm Enhancements | Low | Medium | Existing swarm | Optimize when current swarm limits are hit |
| Output Sanitization | Medium | Low | Prompt injection skill | Natural follow-on to prompt injection |
| Token Budgets | Medium | Low | LiteLLM proxy | Natural follow-on to LiteLLM |

### Recommended Next Tracks

**Track A — Knowledge Management**: Semantic Memory → Insight Engine → YouTube Pipeline
- This is the highest-impact track. Agents that can efficiently store, retrieve, and deduplicate knowledge are fundamentally more capable.

**Track B — User Interface**: WebUI → Dashboard Templates (from observability)
- Becomes important as the system scales. Not urgent for personal use.

**Track C — Security Deepening**: Output Sanitization → Token Budgets → Container Hardening
- Natural continuation of the security work already done. Each is relatively small and self-contained.

---

## BastionClaw Submodule Reference

The BastionClaw codebase is at `bastionclaw/` (git submodule). Key files for future porting work:

| BastionClaw File | Contains |
|-----------------|----------|
| `src/insight-engine/` | Semantic dedup, embedding comparison, merge strategies |
| `src/qmd/` | Query-Memory-Document structured storage |
| `src/security/prompt-injection.ts` | Pattern-based input sanitization (already ported) |
| `src/security/security-hooks.ts` | PreToolUse hooks for Bash/Read (already ported) |
| `src/webui/` | Express.js + frontend for admin panel |
| `src/youtube/` | yt-dlp + Whisper pipeline |
| `src/image-gen/` | Gemini image generation integration |

When starting any of these features, read the BastionClaw implementation first — it's the reference architecture, even if NanoClaw's channel-based design requires adaptation.
