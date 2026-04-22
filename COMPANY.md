# Project: Paperclip + Pi Agent Integration

## Context

[Paperclip](https://github.com/paperclipai/paperclip) is an open-source orchestration platform for AI agent companies (57K+ stars). It provides a task board, org charts, budgets, governance, and agent coordination via a web UI and API.

[Pi](https://github.com/badlogic/pi-mono) (`@mariozechner/pi-coding-agent`) is a minimal terminal coding agent with read, bash, edit, write tools, session management, and an extensible skill/plugin system.

Paperclip already ships a `pi_local` adapter (`packages/adapters/pi-local/`) that runs Pi as a local agent. This project takes that integration further — making Pi a first-class steerable employee inside Paperclip, operated from a human-driven kanban board.

---

## What Already Exists

### Paperclip side

- **Kanban board** (`ui/src/components/KanbanBoard.tsx`) — drag-and-drop board with columns for backlog, todo, in_progress, in_review, blocked, done, cancelled. Uses `@dnd-kit/core`. Toggle between list and board views in `IssuesList.tsx`.
- **Issue system** — full task lifecycle modeled after Linear. Custom workflow states per team. Priorities, assignees, projects, milestones, labels.
- **Execution policies** — per-issue review and approval stages. E.g. "after engineer finishes, QA reviews, then human approves."
- **`pi_local` adapter** (`packages/adapters/pi-local/`) — spawns `pi --mode json -p "prompt"` as a child process. Handles session persistence across heartbeats via `~/.pi/paperclips/`, skills injection into `~/.pi/agent/skills/`, cost/token tracking, model validation, and session recovery on failure.
- **Paperclip skill** (`skills/paperclip/SKILL.md`) — injected into agents. Teaches them the 9-step heartbeat protocol: check identity → get assignments → pick work → checkout → understand context → do work → update status → delegate.
- **Governance** — board approvals for hiring and strategy. Budget caps per agent (hard stop at 100%, conserve at 80%). Pause/resume/terminate any agent. Full activity audit trail.
- **Interrupt mechanism** — board users can post a comment with `interrupt: true` on a task with an active run. Paperclip cancels the running agent process and re-wakes the agent with the comment in the next heartbeat context.
- **Deferred comment wakes** — comments posted during an active run are batched and forwarded to the next heartbeat via `PAPERCLIP_WAKE_PAYLOAD_JSON`.
- **Feedback voting** — thumbs up/down on agent outputs with trace bundles for offline review.

### Pi side

- **RPC mode** (`pi --mode rpc`) — headless JSON-over-stdin/stdout protocol. Supports `prompt`, `steer`, `follow_up`, `abort`, `get_state`, `get_messages`, `compact`, session management, and extension UI sub-protocol.
- **Steering** — `steer` messages are delivered after current tool calls finish, before the next LLM call. The agent doesn't restart — it incorporates new instructions mid-flight. `follow_up` messages are delivered only when the agent finishes all work.
- **SDK** (`createAgentSession()`) — programmatic Node.js/TypeScript API. Create sessions, subscribe to events, send prompts, steer, manage tools/extensions/skills. Both Pi and Paperclip are TypeScript/Node.js.
- **Session persistence** — JSONL tree-structured session files with branching. Resume via `--session <path>`.
- **Multi-provider** — Anthropic, OpenAI, Google, xAI, Mistral, Groq, etc. Each agent can use a different model.
- **Skills & extensions** — auto-discovered from `~/.pi/agent/skills/` and `~/.pi/agent/extensions/`. The Paperclip skill is injected here by the adapter.
- **Cost tracking** — per-session token/cost reporting parsed from JSONL output.
- **No permission popups** — designed for headless execution.

---

## The Gap: True Mid-Execution Steering

### Current state

The `pi_local` adapter runs Pi in **one-shot print mode**:

```
pi --mode json -p "prompt" --session ~/.pi/paperclips/{session}.jsonl
```

This is fire-and-forget. No stdin pipe is kept open. When a board user posts an interrupt comment, Paperclip **kills the process** and re-wakes the agent in a fresh heartbeat. The agent loses its current execution context within the heartbeat (though the session file preserves conversation history).

### What Pi supports but the adapter doesn't use

Pi's RPC mode allows **true mid-execution steering**:

```json
{"type": "steer", "message": "Stop working on the CSS, focus on the API endpoint instead"}
```

- Delivered after current tool calls finish, before the next LLM call
- Agent incorporates the instruction without restarting
- No context loss — same session, same conversation thread
- Also supports `follow_up` (delivered when agent finishes current work) and `abort`

**Pi is the only agent runtime with a structured steering protocol.** Claude Code, Codex, Gemini CLI — none of them have this. This is a unique differentiator.

### What needs to change

Upgrade the `pi_local` adapter to optionally run Pi in RPC mode:

1. Spawn `pi --mode rpc --session <path>` instead of `pi --mode json -p`
2. Keep the stdin pipe open during execution
3. Send the initial prompt via `{"type": "prompt", "message": "..."}` over stdin
4. Stream events from stdout (text deltas, tool executions, agent lifecycle)
5. When Paperclip receives an interrupt-comment from a board user, instead of killing the process, pipe a steer command: `{"type": "steer", "message": "Board says: ..."}`
6. When the heartbeat timeout approaches or work is done, send `{"type": "abort"}` or let the agent finish naturally
7. Parse the final state for cost/usage reporting

This gives Paperclip something no other adapter has: **a human can redirect an agent mid-task without losing context.**

---

## Project Scope

### For the junior dev

#### Phase 1: Get it running (Week 1)

- [ ] Clone and set up Paperclip locally (`pnpm install && pnpm dev`)
- [ ] Verify Pi is installed and working (`pi --help`)
- [ ] Create a company in the Paperclip UI
- [ ] Create an agent using the `pi_local` adapter
- [ ] Configure a working directory and model (e.g. `anthropic/claude-sonnet-4-20250514`)
- [ ] Assign a simple task (e.g. "Create a hello world Express server")
- [ ] Watch the agent pick it up, do the work, and update status
- [ ] Use the kanban board view to track progress

#### Phase 2: Human-in-the-loop workflows (Week 2)

- [ ] Create tasks with execution policies — e.g. a review stage where the human approves before done
- [ ] Test the `in_review` flow: agent does work → parks at in_review → human reviews in UI → approves or sends back
- [ ] Test the interrupt flow: agent is working → human posts interrupt comment → agent is cancelled and re-woken with comment context
- [ ] Set up budget caps and verify agents respect them
- [ ] Test board approval flow: agent requests approval → human approves/denies in UI
- [ ] Document the HITL patterns that work well

#### Phase 3: Steerable adapter upgrade (Week 3-4)

- [ ] Study the existing `pi_local` adapter (`packages/adapters/pi-local/src/server/execute.ts`)
- [ ] Study Pi's RPC protocol (`tmp/pi-coding-agent/packages/coding-agent/docs/rpc.md`)
- [ ] Study Pi's RPC client implementation (`tmp/pi-coding-agent/packages/coding-agent/src/modes/rpc/rpc-client.ts`)
- [ ] Implement an alternative execution path in the adapter that uses `--mode rpc` instead of `--mode json -p`
- [ ] Wire Paperclip's interrupt-comment flow to send `steer` commands via stdin instead of killing the process
- [ ] Handle the event stream from Pi's stdout (map to Paperclip's run log format)
- [ ] Ensure session persistence still works across heartbeats
- [ ] Ensure cost/token tracking still works (parse from RPC events instead of JSONL output)
- [ ] Add adapter config flag: `"mode": "rpc"` vs `"mode": "print"` (default: `"print"` for backward compatibility)
- [ ] Test: human steers agent mid-task → agent adjusts without restart → task completes correctly

#### Phase 4: Polish and document (Week 4+)

- [ ] Document the steerable adapter configuration
- [ ] Create an example company setup with recommended HITL patterns
- [ ] Write up what works, what doesn't, and lessons learned
- [ ] Consider: should follow-up messages map to non-interrupt comments?
- [ ] Consider: should the UI surface a "steer" action distinct from "interrupt"?

---

## Architecture

```
┌─────────────────────────────────────┐
│         Paperclip Web UI            │
│  ┌─────────┐  ┌──────────────────┐  │
│  │ Kanban   │  │ Issue Detail     │  │
│  │ Board    │  │ + Comments       │  │
│  │ (drag &  │  │ + Steer button   │  │
│  │  drop)   │  │ + Review/Approve │  │
│  └─────────┘  └──────────────────┘  │
└──────────────┬──────────────────────┘
               │ REST API
┌──────────────▼──────────────────────┐
│         Paperclip Server            │
│  ┌───────────────────────────────┐  │
│  │ Heartbeat Scheduler           │  │
│  │  - Wake agents on schedule    │  │
│  │  - Wake on task assignment    │  │
│  │  - Wake on @-mention/comment  │  │
│  │  - Interrupt + re-wake        │  │
│  │  - Budget enforcement         │  │
│  └──────────────┬────────────────┘  │
│                 │                    │
│  ┌──────────────▼────────────────┐  │
│  │ pi_local Adapter              │  │
│  │                               │  │
│  │  Print mode (current):        │  │
│  │   pi --mode json -p "prompt"  │  │
│  │   → fire and forget           │  │
│  │                               │  │
│  │  RPC mode (upgrade):          │  │
│  │   pi --mode rpc --session x   │  │
│  │   → stdin: prompt, steer      │  │
│  │   ← stdout: events stream     │  │
│  │   → stdin: steer mid-flight   │  │
│  └──────────────┬────────────────┘  │
└─────────────────┼────────────────────┘
                  │ child process
┌─────────────────▼────────────────────┐
│            Pi Agent                   │
│  ┌─────────────────────────────────┐ │
│  │ Session (JSONL tree)            │ │
│  │ Tools: read, bash, edit, write  │ │
│  │ Skills: paperclip skill         │ │
│  │ Model: any provider/model       │ │
│  └─────────────────────────────────┘ │
│                                       │
│  Heartbeat protocol:                  │
│  1. GET /api/agents/me                │
│  2. GET /api/agents/me/inbox-lite     │
│  3. Pick work (priority order)        │
│  4. POST /api/issues/{id}/checkout    │
│  5. GET /api/issues/{id}/heartbeat-context │
│  6. Do the work (tools)              │
│  7. PATCH /api/issues/{id} + comment │
│  8. Create subtasks if needed        │
└───────────────────────────────────────┘
```

---

## Key Files

### Paperclip

| File | What |
|------|------|
| `packages/adapters/pi-local/src/server/execute.ts` | Core adapter execution — **the main file to modify for RPC mode** |
| `packages/adapters/pi-local/src/server/parse.ts` | Pi JSONL output parser — needs RPC event parser equivalent |
| `packages/adapters/pi-local/src/index.ts` | Adapter metadata and config documentation |
| `skills/paperclip/SKILL.md` | The skill injected into agents — the heartbeat protocol |
| `skills/paperclip/references/api-reference.md` | Full API reference for the Paperclip REST API |
| `ui/src/components/KanbanBoard.tsx` | Kanban board UI component |
| `ui/src/components/IssuesList.tsx` | Issue list with board/list toggle |
| `server/src/routes/issues.ts` | Issue routes — includes interrupt logic |
| `server/src/services/heartbeat.ts` | Heartbeat scheduler — wake, cancel, deferred comment batching |
| `doc/SPEC.md` | System specification — company/agent/governance model |
| `doc/execution-semantics.md` | How runs, wakes, and recovery work |
| `docs/adapters/creating-an-adapter.md` | Guide for building/modifying adapters |
| `docs/adapters/overview.md` | Adapter architecture overview |
| `docs/deploy/overview.md` | Deployment modes (local_trusted, authenticated) |

### Pi (in `tmp/pi-coding-agent/`)

| File | What |
|------|------|
| `packages/coding-agent/docs/rpc.md` | Full RPC protocol documentation |
| `packages/coding-agent/docs/sdk.md` | SDK documentation for programmatic usage |
| `packages/coding-agent/src/modes/rpc/rpc-mode.ts` | RPC mode implementation |
| `packages/coding-agent/src/modes/rpc/rpc-client.ts` | TypeScript RPC client — reference for building the adapter |
| `packages/coding-agent/src/modes/rpc/rpc-types.ts` | RPC command/response/event type definitions |
| `packages/agent/src/agent.ts` | Core agent — steering queue implementation |
| `packages/coding-agent/src/core/agent-session.ts` | AgentSession — prompt, steer, followUp methods |

---

## HITL Patterns in Paperclip

### Pattern 1: Human creates task, agent executes, human reviews

1. Human creates a task on the kanban board
2. Assigns it to a Pi agent
3. Agent picks it up on next heartbeat, does the work
4. Agent sets status to `in_review`
5. Human reviews in the UI, approves (→ done) or sends back (→ in_progress with comment)

### Pattern 2: Execution policy with review gate

1. Human creates a task with an execution policy:
   ```json
   {
     "mode": "normal",
     "stages": [
       { "type": "review", "participants": [{ "type": "user", "userId": "..." }] }
     ]
   }
   ```
2. Agent does work → Paperclip automatically parks at review stage
3. Human reviews, approves or requests changes
4. If changes requested, agent gets re-woken with feedback

### Pattern 3: Interrupt and redirect (current)

1. Agent is working on a task
2. Human posts a comment with `interrupt: true`
3. Paperclip kills the agent process, logs the cancellation
4. Agent re-wakes in new heartbeat, sees the comment, adjusts course

### Pattern 4: Mid-flight steering (upgrade goal)

1. Agent is working on a task
2. Human posts a comment (or clicks "steer" in UI)
3. Paperclip sends `{"type": "steer", "message": "..."}` to Pi's stdin
4. Pi incorporates the instruction after current tool calls finish
5. Agent continues with adjusted direction — no restart, no context loss

### Pattern 5: Budget-gated work

1. Company/agent budgets set in Paperclip UI
2. Agent tracks costs natively (Pi reports token/cost per session)
3. At 80% budget: agent focuses on critical tasks only
4. At 100%: agent is paused automatically

### Pattern 6: Board approval for high-stakes decisions

1. Agent encounters something that needs human sign-off
2. Agent calls `POST /api/companies/{id}/approvals` with a summary
3. Human gets notified in UI, approves or denies
4. Agent is re-woken with `PAPERCLIP_APPROVAL_ID` and `PAPERCLIP_APPROVAL_STATUS`

---

## Open Questions

- Should `steer` be a separate UI action from `interrupt`? (Steer = redirect without restart, Interrupt = kill and re-wake)
- Should non-interrupt comments during a run be sent as `follow_up` messages to the RPC pipe?
- What happens if the RPC pipe breaks mid-execution? Fall back to session resume on next heartbeat?
- Should the adapter support both modes simultaneously (some agents in print, some in RPC)?
- Is there a latency concern with RPC mode vs print mode for short tasks?
- How should the kanban board visually distinguish "agent is working" vs "agent is waiting for steer response"?

---

## Local Setup

```bash
# Paperclip
cd /Users/haziqazizi/paperclip
pnpm install
pnpm dev          # Starts API at http://localhost:3100

# Pi is already installed globally
pi --version      # Verify Pi is available

# Pi source (for reference/development)
ls tmp/pi-coding-agent/
```

## Resources

- [Paperclip README](https://github.com/paperclipai/paperclip)
- [Paperclip Docs](https://paperclip.ing/docs)
- [Paperclip Discord](https://discord.gg/m4HZY7xNG3)
- [Pi README](https://github.com/badlogic/pi-mono)
- [Pi RPC Docs](tmp/pi-coding-agent/packages/coding-agent/docs/rpc.md)
- [Pi SDK Docs](tmp/pi-coding-agent/packages/coding-agent/docs/sdk.md)
- [Awesome Paperclip](https://github.com/gsxdsm/awesome-paperclip) — plugins and community resources
