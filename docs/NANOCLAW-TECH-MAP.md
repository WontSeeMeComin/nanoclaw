# NanoClaw Technical Ecosystem Map

> **Purpose:** Reference document for Claude agents planning integrations (e.g., Elves/Workshop).
> **Generated:** 2026-03-29 | **NanoClaw version:** 1.2.42 | **Agent SDK:** 0.2.76

---

## Architecture Overview

NanoClaw is a single Node.js orchestrator that receives messages from channels (Discord, Gmail, etc.), routes them to Claude Agent SDK instances running inside Docker containers, and streams responses back. Each registered group gets an isolated container with its own filesystem, session state, and IPC namespace.

```
User ──► Channel (Discord/Gmail) ──► Orchestrator (src/index.ts)
              ▲                            │
              │                            ▼
              │                      GroupQueue (concurrency control)
              │                            │
              │                            ▼
              └──── Response ◄──── Docker Container
                                   ├── Agent Runner (Claude Agent SDK)
                                   ├── MCP Servers (nanoclaw, gmail, ollama)
                                   ├── Skills (agent-browser, etc.)
                                   └── /workspace/{group,global,extra,ipc}
```

---

## 1. Container Architecture

### Image
- **Base:** `node:22-slim`
- **Key packages:** Chromium (headless), `agent-browser`, `@anthropic-ai/claude-code`
- **Runs as:** `node` user (UID 1000)
- **Entrypoint:** `container/entrypoint.sh` — compiles TS, reads JSON from stdin, executes agent
- **Build:** `./container/build.sh` → `nanoclaw-agent:latest`
- **Dockerfile:** `container/Dockerfile`

### Input/Output Protocol

**Input (stdin):** JSON `ContainerInput`
```typescript
{
  prompt: string;           // Formatted user messages
  sessionId?: string;       // Resume existing session
  groupFolder: string;      // Group identity
  chatJid: string;          // Chat/channel JID
  isMain: boolean;          // Elevated privileges?
  isScheduledTask?: boolean;
  assistantName?: string;   // Bot name (e.g., "Aegis")
  script?: string;          // Optional pre-agent bash script
}
```

**Output (stdout):** JSON wrapped in `---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---` markers. Multiple results possible (agent teams).

### Mount Points

| Host Path | Container Path | Access | Notes |
|-----------|---------------|--------|-------|
| Project root | `/workspace/project` | read-only | **Main group only** |
| `groups/{folder}/` | `/workspace/group` | read-write | Per-group workspace |
| `groups/global/` | `/workspace/global` | read-only | Non-main groups only |
| `data/sessions/{folder}/.claude` | `/home/node/.claude` | read-write | Settings, skills, sessions |
| `container/skills/*` | `/home/node/.claude/skills/` | read-write | Synced at startup |
| `~/.gmail-mcp` | `/home/node/.gmail-mcp` | read-write | Gmail OAuth creds |
| `data/ipc/{folder}` | `/workspace/ipc` | read-write | IPC namespace |
| `data/sessions/{folder}/agent-runner-src` | `/app/src` | read-write | Per-group agent-runner copy |
| Custom mounts | `/workspace/extra/*` | varies | Via `containerConfig.additionalMounts` |
| `.env` | `/workspace/project/.env` | shadowed `/dev/null` | Secrets never exposed |

**Key file:** `src/container-runner.ts` (lines 63-236)

---

## 2. Agent SDK Query Configuration

**File:** `container/agent-runner/src/index.ts` (lines 394-477)

```typescript
query({
  prompt: messageStream,     // AsyncIterable — keeps session alive for follow-ups
  options: {
    cwd: '/workspace/group',
    additionalDirectories: ['/workspace/extra/*'],
    resume: sessionId,
    systemPrompt: { type: 'preset', preset: 'claude_code', append: globalClaudeMd },
    allowedTools: [
      'Bash', 'Read', 'Write', 'Edit', 'Glob', 'Grep',
      'WebSearch', 'WebFetch',
      'Task', 'TaskOutput', 'TaskStop',
      'TeamCreate', 'TeamDelete', 'SendMessage',
      'TodoWrite', 'ToolSearch', 'Skill', 'NotebookEdit',
      'mcp__nanoclaw__*',    // IPC tools
      'mcp__gmail__*',       // Gmail
      'mcp__ollama__*'       // Local models
    ],
    permissionMode: 'bypassPermissions',
    mcpServers: { nanoclaw, gmail, ollama },
    hooks: { PreCompact: [archiveTranscriptHook] }
  }
})
```

**No explicit model or thinking budget is set.** The SDK defaults are used. To customize, modify the `query()` call.

**Agent Teams enabled** via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in per-group settings.json.

---

## 3. MCP Servers

### 3.1 NanoClaw IPC (`mcp__nanoclaw__*`)
**File:** `container/agent-runner/src/ipc-mcp-stdio.ts`

| Tool | Purpose | Auth |
|------|---------|------|
| `send_message` | Send message to user mid-execution | All groups |
| `schedule_task` | Create cron/interval/once task | Own group (main can target others) |
| `list_tasks` | List scheduled tasks | Own tasks (main sees all) |
| `pause_task` / `resume_task` / `cancel_task` | Task lifecycle | Own tasks only |
| `update_task` | Modify prompt, script, schedule | Own tasks only |
| `register_group` | Register new chat/channel | **Main only** |

### 3.2 Ollama (`mcp__ollama__*`)
**File:** `container/agent-runner/src/ollama-mcp-stdio.ts`

| Tool | Purpose | Requires |
|------|---------|----------|
| `ollama_list_models` | List installed models | Always |
| `ollama_generate` | Run inference on local model | Always |
| `ollama_pull_model` | Download model | `OLLAMA_ADMIN_TOOLS=true` |
| `ollama_delete_model` | Remove model | `OLLAMA_ADMIN_TOOLS=true` |
| `ollama_show_model` | Model details | `OLLAMA_ADMIN_TOOLS=true` |
| `ollama_list_running` | Memory usage | `OLLAMA_ADMIN_TOOLS=true` |

Connects to `http://host.docker.internal:11434` (Docker Desktop) with localhost fallback.

### 3.3 Gmail (`mcp__gmail__*`)
**Package:** `@gongrzhe/server-gmail-autoauth-mcp` (installed via npm)
Credentials at `~/.gmail-mcp/` mounted into container.

---

## 4. Container Skills

**Location:** `container/skills/` — synced to `/home/node/.claude/skills/` at container startup.

| Skill | Purpose |
|-------|---------|
| `agent-browser` | Chromium automation (open, click, fill, screenshot) |
| `capabilities` | System report (main-only) |
| `slack-formatting` | Slack mrkdwn reference |
| `status` | Health check (main-only) |

**To add a new skill:** Create `container/skills/{name}/SKILL.md` (and optional code files). Rebuild container. Auto-discovered by Agent SDK.

---

## 5. Message Flow

```
1. Channel receives message (e.g., Discord MessageCreate)
2. OnInboundMessage callback → stored in SQLite
3. startMessageLoop polls every 2000ms (POLL_INTERVAL)
4. getNewMessages() per registered group
5. GroupQueue.enqueueMessageCheck() → concurrency control (max 5 containers)
6. processGroupMessages():
   a. Session command check (/compact)
   b. Trigger check (main = no trigger, others = @Aegis prefix)
   c. Format messages with timezone
   d. Advance cursor
   e. runContainerAgent() with formatted prompt
7. Container spawned → JSON stdin → Agent SDK query()
8. Streaming output → channel.sendMessage()
9. IPC watcher processes send_message/schedule_task files
```

**Key files:**
- `src/index.ts` — orchestrator, message loop, agent invocation
- `src/group-queue.ts` — concurrency control, idle management
- `src/ipc.ts` — host-side IPC processing

---

## 6. Group Isolation & Privileges

### Main Group (`isMain: true`)
- Sees project root at `/workspace/project` (read-only)
- Can register new groups, refresh metadata
- Can see/control all tasks across groups
- Can schedule tasks targeting other groups
- No trigger word required

### Non-Main Groups
- Only see own group folder + global memory (read-only)
- Can only manage own tasks
- Require trigger prefix (unless `requiresTrigger: false`)
- Cannot register groups or cross-group schedule
- IPC authorization enforced in `src/ipc.ts`

### Per-Group Identity
- `groups/{folder}/CLAUDE.md` — system prompt, personality, instructions
- `groups/global/CLAUDE.md` — shared context appended for non-main groups
- `data/sessions/{folder}/.claude/settings.json` — environment overrides

---

## 7. Task Scheduling

**File:** `src/task-scheduler.ts`

```typescript
interface ScheduledTask {
  id: string;
  group_folder: string;
  chat_jid: string;
  prompt: string;
  script?: string;                          // Optional pre-agent bash script
  schedule_type: 'cron' | 'interval' | 'once';
  schedule_value: string;                   // Cron expr, ms, or ISO timestamp
  context_mode: 'group' | 'isolated';      // Chat history access
  next_run: string | null;
  status: 'active' | 'paused' | 'completed';
}
```

**Script pattern:** Script runs first (30s timeout), outputs `{ "wakeAgent": bool, "data": {...} }`. Agent only invoked if `wakeAgent: true`.

Scheduler polls every 60 seconds. Tasks stored in SQLite.

---

## 8. Configuration

### Environment Variables (src/config.ts)

| Variable | Default | Purpose |
|----------|---------|---------|
| `ASSISTANT_NAME` | `Andy` | Bot display name |
| `CONTAINER_IMAGE` | `nanoclaw-agent:latest` | Docker image |
| `CONTAINER_TIMEOUT` | 30 min | Hard timeout |
| `MAX_CONCURRENT_CONTAINERS` | 5 | Parallel containers |
| `IDLE_TIMEOUT` | 30 min | Keep idle container alive |
| `MAX_MESSAGES_PER_PROMPT` | 10 | Messages per batch |
| `POLL_INTERVAL` | 2000ms | Message check interval |
| `SCHEDULER_POLL_INTERVAL` | 60000ms | Task scheduler interval |
| `OLLAMA_ADMIN_TOOLS` | false | Ollama management tools |
| `ONECLI_URL` | `http://localhost:10254` | Credential gateway |

### Current .env (variable names only)
```
TZ, ONECLI_URL, DISCORD_BOT_TOKEN, ASSISTANT_NAME, OLLAMA_ADMIN_TOOLS
```

### Per-Group Settings
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD": "1",
    "CLAUDE_CODE_DISABLE_AUTO_MEMORY": "0"
  }
}
```

---

## 9. Credential System

**OneCLI** manages API keys/tokens — injects credentials into container HTTP requests via a gateway proxy. No secrets are ever passed as environment variables to containers.

- Gateway: `http://127.0.0.1:10254` (Docker container)
- CLI: `onecli secrets list/create`
- Dashboard: `http://127.0.0.1:10254`

The `.env` file is shadowed with `/dev/null` inside containers to prevent secret access.

---

## 10. Integration Points

### Adding a Custom MCP Server
1. Create stdio server in `container/agent-runner/src/your-server.ts`
2. Add to `mcpServers` in `runQuery()` (agent-runner/src/index.ts, line 420)
3. Add `mcp__yourserver__*` to `allowedTools` (line 412)
4. Rebuild container: `./container/build.sh`

### Adding a Container Skill
1. Create `container/skills/{name}/SKILL.md` (+ optional code)
2. Rebuild container
3. Auto-synced to all groups on next container startup

### Adding a Channel
Channel skills are on separate repos/branches, merged via git. Each implements the `Channel` interface and self-registers via `registerChannel()` in `src/channels/registry.ts`.

### Extending Group Capabilities
- Mount additional directories via `containerConfig.additionalMounts` in DB registration
- They appear at `/workspace/extra/{name}` and are auto-added to `additionalDirectories`

### Customizing the Agent Model/Thinking
Currently no model is specified in the `query()` call — SDK defaults are used. To set a specific model or enable extended thinking, modify `container/agent-runner/src/index.ts` in the `query()` options.

---

## 11. Current Installation State

- **Channel:** Discord (`Aegis#5231` in `aegis-nanobot #primary`)
- **JID:** `dc:1487657391297269887`
- **Group folder:** `discord_main` (main group, no trigger required)
- **Gmail:** Tool-only mode (`matt@cascadialogic.io`)
- **Ollama:** Enabled with admin tools (qwen3-coder:30b, devstral, mistral:7b, etc.)
- **Credentials:** OneCLI with Claude subscription token
- **Mount allowlist:** `~/Dev` (read-only for non-main groups)
- **Session commands:** `/compact` enabled
- **Status bar:** macOS menu bar indicator running
- **Timezone:** America/Los_Angeles

---

## 12. Key File Index

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator, message loop, agent invocation |
| `src/container-runner.ts` | Mount construction, container spawn, output parsing |
| `src/group-queue.ts` | Concurrency control, idle management |
| `src/ipc.ts` | Host-side IPC processing |
| `src/task-scheduler.ts` | Scheduled task execution |
| `src/config.ts` | Environment and config loading |
| `src/types.ts` | Type definitions |
| `src/channels/registry.ts` | Channel self-registration |
| `src/channels/discord.ts` | Discord channel implementation |
| `src/channels/gmail.ts` | Gmail channel implementation |
| `src/session-commands.ts` | `/compact` command handling |
| `container/Dockerfile` | Container image definition |
| `container/build.sh` | Build script |
| `container/agent-runner/src/index.ts` | Agent runner, SDK query, message stream |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | NanoClaw MCP server (IPC tools) |
| `container/agent-runner/src/ollama-mcp-stdio.ts` | Ollama MCP server |
| `container/agent-runner/package.json` | SDK versions |
| `groups/discord_main/CLAUDE.md` | Main group system prompt |
| `groups/global/CLAUDE.md` | Shared context for non-main groups |
| `data/sessions/{folder}/.claude/settings.json` | Per-group agent settings |
