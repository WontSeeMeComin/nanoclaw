# Aegis as the Primary Elves PM Surface

## Summary

Use the **Mac Studio host** as the Elves backend authority and make **Aegis in NanoClaw/Discord** the new PM/operator chat surface. Do **not** try to run legacy Elves setup or the Cowork-shaped Workshop flow inside the NanoClaw container. Instead:

- Elves on the Mac Studio owns project truth, Forge sessions, jobs, inbox, harness control, and dashboard.
- NanoClaw gets a thin **Elves adapter** that talks to the host control plane over HTTP.
- Aegis gets a new **Discord-native PM skill** derived from Workshop flows, but without Cowork bridge assumptions.
- Discord topology becomes **`#primary` for overview + one project thread per project** for focused PM work.

This is a phased migration. Phase 1 reaches **PM control parity first**: status, tasks/tickets, job submission/status/result, inbox, Forge visibility, and harness control. Richer Workshop flows come next.

## Key Changes

### 1. Make the Mac Studio the canonical Elves host

- Install Elves normally on the Mac Studio host and keep using `elves link` there during development.
- Run `elves control-plane` and `elves dashboard` on the Mac Studio host; they are the only services Aegis calls.
- Do not run `elves init` or skill install logic inside NanoClaw containers.
- Keep all project repos cloned on the Mac Studio host at stable absolute paths under one root (for example `~/Dev/...`).

**Host defaults**
- Control plane base URL: `http://host.docker.internal:9051` from the container.
- Dashboard remains host-side and optional for Aegis, but still the best visual observability surface.
- Phase 1 may keep host Elves credentials in current host config if needed; phase 4 modernizes this.

### 2. Add a NanoClaw-side Elves adapter

Implement a new NanoClaw MCP/tool layer that maps Discord/Aegis requests to the existing Elves control-plane HTTP API.

**Tool surface**
- `elves_list_projects()`
- `elves_get_project_status(project_alias)`
- `elves_get_project_config(project_alias)`
- `elves_list_tickets(project_alias)`
- `elves_set_ticket(project_alias, ticket_id)`
- `elves_list_tasks(project_alias, ticket_id?, run_id?, include_closed?, scope_current_ticket?)`
- `elves_list_jobs(project_alias, limit?)`
- `elves_submit_job(project_alias, kind, ticket_id?, params)`
- `elves_get_job(job_id)`
- `elves_get_job_result(job_id)`
- `elves_list_inbox(project_alias?, limit?)`
- `elves_launch_harness(project_alias, watch?)`
- `elves_stop_harness(project_alias)`
- `elves_stop_after_task(project_alias)`
- `elves_get_forge_history(project_alias, ticket_id, limit?)`
- `elves_get_forge_sessions(project_alias)`

**Project binding config**
Use a NanoClaw-owned config file, not Elves global config, to resolve Discord project aliases to host/container paths.

Required record shape:

```json
{
  "alias": "covenant-portal",
  "displayName": "covenant-portal-replit",
  "hostProjectDir": "/Users/wontseemecomin/Dev/Projects/crossroads/covenant/covenant-portal-replit",
  "containerReadPath": "/workspace/extra/covenant-portal-replit",
  "defaultTicketId": "browser-recorder",
  "threadChannelId": "optional-discord-thread-id"
}
```

**Rules**
- `project_alias` is the public identifier Aegis uses everywhere; raw absolute paths are hidden from chat.
- The adapter resolves `project_alias -> hostProjectDir` for all control-plane calls.
- `containerReadPath` is used only for repo reading inside the container.
- If Discord threads are not already distinct NanoClaw group registrations, add that support before project-thread rollout.

### 3. Replace Workshop-with-bridge with an Aegis PM skill

Do not port [workshop/SKILL.md](/Users/wontseemecomin/Dev/AI%20Agents/elves/workshop/SKILL.md) unchanged. Build a new NanoClaw skill, for example `container/skills/aegis-pm/`, using Workshop as source material.

**What to keep from Workshop**
- catch-up / status synthesis
- project triage and “what next?”
- PRD and planning workflows
- task generation orchestration
- Forge research / follow-up flow
- result and inbox handling
- test/verification intent and checklists

**What to remove**
- Cowork VM bridge detection
- file-based bridge instructions
- host `~/.claude/skills` assumptions
- manual workshop ZIP upload workflow
- direct references that assume the skill is installed by `elves update --skills`

**How Aegis behaves**
- `#primary` is the overview/orchestration surface.
- Each project thread is the authoritative PM conversation for one project.
- In a project thread, Aegis assumes the bound `project_alias` unless the user explicitly switches.
- Long-running work is always job-backed.
- If a job outlives the active turn, Aegis schedules its own follow-up and posts the final result back into the same Discord thread.

### 4. Use NanoClaw’s scheduler/message loop for “don’t babysit this”

This is the main improvement over Workshop.

**Job follow-up behavior**
- When Aegis submits a long-running Elves job, it immediately posts:
  - what was queued
  - which project/ticket it targets
  - the job id
  - whether Aegis will keep watching automatically
- For jobs expected to finish quickly, Aegis may poll inline within the current turn.
- For jobs expected to run longer, Aegis creates a NanoClaw scheduled follow-up task that:
  - checks `elves_get_job(job_id)`
  - posts the summary/result into the same Discord thread when complete
  - reschedules itself until completion/failure or timeout budget is exceeded

**Default polling policy**
- Inline wait budget: up to 60 seconds
- Scheduled follow-up cadence: every 2 minutes
- Final completion post includes a concise summary plus `job_result` details
- Failed jobs post error text and the recommended next action

### 5. Keep repo access read-only inside the container

Aegis is a PM/operator surface, not the place where project files are mutated directly.

**Mount policy**
- `#primary`: no broad codebase write access; only project metadata and optional overview mounts
- Project threads: mount the bound project repo read-only at `containerReadPath`
- Optional shared read-only mount for the Elves repo/docs if Aegis needs local reference material
- All code changes, harness runs, task creation, Forge work, and source sync happen via the Elves host control plane

### 6. Modernize host bootstrap and secrets after Phase 1 works

Phase 1 does not require fully replacing Elves host credential loading, but the plan includes it explicitly.

**Required host-side follow-up**
- Change Elves credential loading to be **environment-first**, file-fallback:
  - if needed vars already exist in process env, do not require `~/.elves/credentials.env`
  - keep `~/.elves/credentials.env` as backward-compatible fallback
- Update health/doctor output so env-provided credentials count as valid
- Add a non-interactive host bootstrap path so Mac Studio setup does not require `elves init`
- Keep `elves link/unlink` usable without a prior full interactive init; accept explicit repo path or current repo inference if needed

## Phases

### Phase 1: Aegis Operator Core
Build the NanoClaw Elves adapter and basic Discord PM operation.

Deliverables:
- host control plane reachable from NanoClaw container
- project alias config
- Discord `#primary` + project thread model
- Aegis can:
  - list projects
  - catch up on one project
  - inspect tasks/tickets/jobs/inbox
  - submit `forge.prompt`, `project.summary`, `taskgen.*`, and `source.sync`
  - launch/stop harness
  - summarize Forge history and recent results
- automatic job follow-up posts in Discord

### Phase 2: Aegis PM Skill Parity
Build `aegis-pm` skill from Workshop flows.

Deliverables:
- status/catch-up flow
- “what should we do next?” flow
- PRD authoring / refinement flow
- task generation orchestration flow
- Forge guidance flow
- inbox/result interpretation flow
- browser-verification intent flow that can delegate to existing browser capability when needed

### Phase 3: Skill Packaging and Drift Prevention
Eliminate manual Workshop ZIP style deployment.

Deliverables:
- one repeatable export/sync path from Elves Workshop source material to NanoClaw `container/skills/aegis-pm`
- no manual upload step
- versioned PM skill deployment tied to repo changes
- update docs so Aegis skill deployment is explicit and separate from `elves update --skills`

### Phase 4: Secrets and Headless Host Bootstrap
Remove the remaining host-centric friction.

Deliverables:
- env-first credential loading in Elves host services
- host setup path that does not depend on interactive `elves init`
- optional OneCLI-fed env bridge for host-launched Elves services
- docs for Mac Studio-first deployment

## Test Plan

### Phase 1 acceptance
1. From `#primary`, ask Aegis for available Elves projects; it returns aliases, not raw paths.
2. Open/create a project thread for one project; Aegis binds that thread to the project alias.
3. In the project thread, ask for a catch-up; Aegis summarizes current ticket, run state, tasks, recent completions, Forge activity, and inbox.
4. Ask Aegis to queue a Forge research task; it acknowledges immediately, tracks the job, and posts the result back without manual prompting.
5. Ask Aegis to run task generation; it submits the job, reports progress, and posts the resulting task summary.
6. Ask Aegis to launch the harness, then stop after task; Discord updates reflect both requests and later state changes.
7. Ask Aegis for the latest Forge status/history after a run; it shows recent exchanges or active-session state correctly.

### Phase 2 acceptance
1. Give Aegis raw notes/requirements and have it organize them into a project understanding.
2. Ask Aegis to draft/refine a PRD for a ticket using Workshop-derived structure.
3. Ask Aegis what the highest-value next item is for a project; it reasons from project state and recent results.
4. Ask Aegis to verify a completed feature using the browser/testing flow and report pass/fail clearly.

### Phase 3 acceptance
1. Updating the Elves Workshop source material and running the sync/export step updates the NanoClaw PM skill with no manual zip upload.
2. New NanoClaw containers pick up the PM skill automatically on restart/startup.

### Phase 4 acceptance
1. Elves host services start and run correctly with credentials supplied via environment only.
2. `doctor` or equivalent reports env-backed credentials as healthy.
3. Mac Studio host setup can be completed without running interactive `elves init`.

## Assumptions and Defaults

- Current Discord starting point is one server with one `#primary` channel and a `discord_main` bot; v1 evolves this into `#primary` plus project threads.
- `#primary` remains the overview/orchestration surface; each project thread is the focused PM surface for one project.
- The Mac Studio host is the only Elves execution authority.
- Aegis does not mutate project repos directly; it delegates all project actions to the Elves host backend.
- Phase 1 prioritizes PM control parity, not full Workshop parity.
- Phase 1 may temporarily keep host Elves credential fallback behavior; env-first/OneCLI host cleanup is Phase 4.
- Dashboard remains supported and useful, but Discord/Aegis becomes the primary PM chat surface.
