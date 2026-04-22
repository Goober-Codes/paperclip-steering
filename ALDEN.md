# ALDEN.md — Your Guide to This Project

## Why this project exists

Paperclip's `pi_local` adapter runs Pi agents in fire-and-forget mode. Three problems:

1. **No mid-flight steering.** When you want to redirect a running agent, Paperclip
   kills the process and restarts from scratch. Context is lost. It's like hanging up a
   phone call and calling back just to change what you were saying.

2. **No sandboxing.** Pi tools (bash, write, edit) run directly on the host machine.
   No network policy, no isolation. An agent can make any HTTP request, write to any
   path, read any file.

3. **No durable execution.** If the Paperclip server restarts mid-run, the agent's
   progress is lost. The heartbeat scheduler starts fresh with no memory of what was
   happening.

This project fixes all three by adding the missing layers:

| Problem | Current | This Project | Key gain |
|---|---|---|---|
| Agent steering | Kill & restart | Pi RPC `steer` command | Mid-flight redirect, zero context loss |
| Sandboxing | Direct host execution | **Gondolin** micro-VM | HTTP egress policy, secret injection |
| Durability | Ephemeral processes | **Absurd** + Postgres | Runs survive crashes, resume from checkpoint |

**After all three phases:**
- A human can type a redirect into the UI and the agent pivots mid-task without restarting
- Pi tools run inside an isolated Linux VM — the agent only reaches hosts you allow
- Agent runs are checkpointed in Postgres — a server restart doesn't lose work
- Everything is upstream-contributable to a 57K-star open-source repo

---

## Hi Alden

You're going to build this. Each phase is a self-contained PR to
[paperclipai/paperclip](https://github.com/paperclipai/paperclip) that makes the system
meaningfully better. As a side effect, each phase teaches you something concrete from
[Web Scalability for Startup Engineers](https://www.amazon.com/Scalability-Startup-Engineers-Artur-Ejsmont/dp/0071843655)
— the book maps almost perfectly onto the three layers you're changing.

By the end, you'll have:
- Real merged PRs on a 57K-star public repo
- Deep understanding of stateless services, async messaging, and durable data layers
- A blog post per phase with a demo recording

---

## Use This System for Your Own Projects

Don't study this in isolation. **Use it.**

Once Phase 1 ships (RPC steering working), use Paperclip + Pi to work on your own
projects from the kanban board. Assign real tasks from your backlog to the agent.
That's the real test — does it do useful work on something you care about?

Every time you use it on your own project, you're experiencing the system from both
sides: as the developer building it and the user depending on it. That dual perspective
is where the real learning happens, and it's what makes the blog post worth reading.

---

## Writing About What You Learn

After each phase, write about it. Not a tutorial — a reflection. What did you expect?
What surprised you? What broke?

Model your writing on [Maggie Appleton's digital garden](https://maggieappleton.com).
Personal, well-researched, genuinely readable. Her "Ace" post is a good example: starts
from a lived experience, explains the mechanism, lands on a clear argument.

### Post format

```
Title: [Something specific, not generic]
  Good: "The agent redirected mid-task. Here's what that took."
  Bad:  "What I learned about async messaging"

Planted: [date]
Tags:    [2-3 topic tags]

---

[Opening: one concrete moment — not "in this post I will..."]
[The mechanism: what actually happened, with code snippets]
[The surprise: what you didn't expect]
[The argument: what this teaches about the broader concept]
[The demo: embed your screen recording here]
```

**Suggested titles per phase:**

| Phase | Title angle |
|---|---|
| Phase 1 | "Replacing kill-and-restart with a steer command (and what that took)" |
| Phase 2 | "Running AI tools inside a micro-VM — the 30-line change and the week of reading" |
| Phase 3 | "The data layer is the hardest part. Here's why." |

### The demo recording (required for every post)

Without it the post is just words. With it, someone understands in 60 seconds what
takes 10 minutes to read.

**Structure — keep it under 60 seconds:**

```
0:00–0:01  HOOK    Bold text: what you built. No talking yet.
0:01–0:05  SHOW    Open Paperclip — let the product appear before you explain it
0:05–0:10  SETUP   "Agent is mid-task. I want to change direction."
0:10–0:50  DEMO    One complete workflow — steer the agent, it pivots, it keeps working
0:50–1:00  CLOSE   PR link. Your repo. Where to find the code.
```

What to capture for Phase 1: agent writing something real, you type a redirect into the
Steer button, agent pivots without restarting. The pivot is the whole demo.

Tools: QuickTime (Mac) → upload to YouTube unlisted → embed in post. No narration needed.

### Promoting each post (X/Twitter thread)

```
Tweet 1 — HOOK (95% of your success)
  Line 1: Why this matters
  Line 2: The challenge or surprise
  Line 3: Tease what's coming

  Example:
  "I just gave a running AI agent new instructions without restarting it.
  Every other agent runtime kills the process and starts over. This one doesn't.
  Here's what it took to build that:"

Tweet 2 — INTRO: what you'll cover (3–4 bullets)
Tweets 3–7 — BODY: one concept per tweet, code snippet or screenshot each
Tweet 8 — CONCLUSION: revisit the hook, stronger ending
Tweet 9 — CTA: "Read the full post: [link]. Retweet tweet 1."
```

Hook test: read your first line. If the reaction isn't "I need to keep reading" — rewrite it.

### Where to publish

| Platform | Best for |
|---|---|
| dev.to | Discovery — tech audience already there |
| Personal site | Ownership — your garden, your rules |
| Substack | Email list — worth starting from post 1 even with 0 readers |

Post the thread to X the same day. Cross-post to dev.to and your site. Costs 5 minutes,
doubles reach.

**The writing is for you first. The audience is a bonus. But ship it publicly every time.**

---

## What You're Going to Build — Three Phases

### Phase 1: RPC steering — mid-flight agent redirection
**Size:** Medium (adapter plumbing + UI button)
**Where:** `packages/adapters/pi-local/src/server/execute.ts` + issue detail UI

The current adapter spawns Pi like this:
```bash
pi --mode json -p "prompt" --session ~/.pi/paperclips/{session}.jsonl
```
Fire and forget. No stdin pipe kept open. Interrupt = kill process.

You'll add an alternative execution path:
```bash
pi --mode rpc --session ~/.pi/paperclips/{session}.jsonl
```
Stdin stays open. Initial prompt sent as `{"type":"prompt","message":"..."}`. When a
board user posts a steer comment, Paperclip writes `{"type":"steer","message":"..."}` to
stdin instead of killing the process. No restart. No context loss.

**What to build:**
- In `execute.ts`: add `mode: "rpc"` code path. Call `spawn("pi", [...])` directly
  (not via `runChildProcess` — it doesn't expose raw stdin). Store `ChildProcess` in a
  `Map<runId, ChildProcess>` in the adapter.
- New route: `POST /api/runs/:runId/steer` — looks up live process, writes steer to stdin.
  Body: `{ message: string }`. Response: `{ ok: true }` or 404.
- UI: "Steer" text input + button in issue detail, next to the comment box. Calls new endpoint.
- Cleanup: on run completion or error, delete from Map and close stdin.

**Before writing any code:** read `tmp/pi-coding-agent/packages/coding-agent/docs/rpc.md`
end-to-end and confirm Pi accepts `steer` mid-flight. The whole phase depends on this.

**Why this matters to learn:**
Ch 3 + Ch 4 — stateless services and service isolation. The adapter is a stateless
service layer between Paperclip and Pi. `Map<runId, ChildProcess>` is in-memory per
process — understand why that's acceptable here (local dev) and where it breaks down
(server restart). That's the statelessness lesson.

**Decision made:** comment box = `follow_up` (delivered when agent finishes). Steer button
= `steer` (delivered mid-flight). Two distinct affordances, both wired to the RPC pipe.

---

### Phase 2: Gondolin — Pi tools run inside a micro-VM
**Size:** Medium (wrap the Pi execution layer, add network policy config)
**Where:** New integration in the adapter, using the Pi + Gondolin extension

Currently Pi's bash/write/edit tools run directly on the host. With Gondolin:
```ts
import { VM, createHttpHooks } from "@earendil-works/gondolin";

const { httpHooks, env } = createHttpHooks({
  allowedHosts: ["api.github.com", "npm.registry.com"],
  secrets: { GITHUB_TOKEN: { hosts: ["api.github.com"], value: process.env.GITHUB_TOKEN } },
});

const vm = await VM.create({ httpHooks, env });
// Pi tools now run inside the VM — agent only reaches allowedHosts
```

Pi tools execute in an isolated Linux VM. The agent gets a placeholder token; the real
secret is injected only for allowed hosts. An agent can't exfiltrate credentials or make
requests to arbitrary endpoints.

**Gondolin is local-only:** QEMU runs as a child process on your machine. Use it for
local dev. Not for remote/production use.

**Read before writing code:**
- `tmp/gondolin/README.md` — SDK overview
- `tmp/gondolin/host/examples/pi-gondolin.ts` — the Pi extension pattern, start here
- Gondolin docs: https://earendil-works.github.io/gondolin/

**Why this matters to learn:**
Ch 7 — async processing and isolation. A sandboxed VM is an execution boundary. This
phase teaches you why isolation matters in async systems: an agent making unexpected
network calls is a bug you'll never see until it's a security incident.

---

### Phase 3: Absurd — agent runs become durable tasks
**Size:** Large (multi-week, stretch goal)
**Where:** Heartbeat scheduler integration, Paperclip server

Currently when Paperclip runs an agent heartbeat, it's an ephemeral process. Server
restarts, power cuts, or OOM kills mean lost work. With Absurd:

```ts
import { Absurd } from 'absurd-sdk';
const app = new Absurd();

app.registerTask({ name: 'agent-heartbeat' }, async (params, ctx) => {
  const context = await ctx.step('fetch-context', async () => {
    return await getHeartbeatContext(params.issueId);
  });

  const result = await ctx.step('run-agent', async () => {
    return await runPiWithRpc(context);  // your Phase 1 code
  });

  await ctx.step('update-issue', async () => {
    await updateIssueStatus(params.issueId, result);
  });
});
```

Each step is checkpointed. If the server crashes between steps, the task resumes from
the last completed checkpoint — not from the beginning.

**Absurd just needs Postgres.** Paperclip already has Postgres. Zero new infrastructure.

**Before writing code:**
- `tmp/absurd/README.md` — concepts and TypeScript SDK
- `server/src/services/heartbeat.ts` — the current scheduler you're augmenting
- Map: every agent run → an Absurd task. Every logical unit of work → a step.

**Why this matters to learn:**
Ch 5 — the data layer. Absurd is a practical introduction to durable execution:
idempotency, checkpoints, exactly-once semantics, and why "just retry it" isn't good
enough. This is the hardest chapter in the book and the hardest phase.

---

## What You'll Learn — Book Chapter Map

| Book Chapter | Concept | When You Learn It |
|---|---|---|
| Ch 2: Design Principles | Loose coupling, single responsibility, open-closed | Phase 0 — read `execute.ts` and understand why the adapter interface is designed the way it is |
| Ch 3: Front-End Layer | Stateless services | Phase 1 — the adapter is stateless; the Map is in-memory per process |
| Ch 4: Web Services | Service isolation, SOA | Phase 1 — Pi and Paperclip as decoupled services, adapter as the glue |
| Ch 7: Async Processing | Event-driven arch, isolation boundaries | Phase 2 — the VM is an execution boundary; tools run async inside it |
| Ch 5: Data Layer | Durable execution, idempotency, checkpoints | Phase 3 — Absurd makes this concrete |
| Ch 9: Automation | CI/CD, testing | Every phase — add tests to each PR before opening it |

---

## How to Navigate the Codebase

```
packages/adapters/pi-local/
  src/server/execute.ts       ← Phase 1 main file
  src/server/parse.ts         ← Pi output parser (needs RPC equivalent)
  src/index.ts                ← Adapter metadata and config

server/src/
  routes/issues.ts            ← Interrupt logic, add steer route here
  services/heartbeat.ts       ← Heartbeat scheduler (Phase 3 entry point)

ui/src/components/
  IssueDetail.tsx (or equiv)  ← Add Steer button here

skills/paperclip/SKILL.md     ← The skill injected into Pi agents

tmp/                          ← Reference repos (read, don't edit)
  pi-coding-agent/            ← Pi agent — RPC docs at docs/rpc.md
  gondolin/                   ← Gondolin micro-VMs
  absurd/                     ← Absurd durable execution
```

Key files to read first, in order:
1. This file
2. `docs/design-paperclip-pi.md` — the design doc from the planning session
3. `tmp/pi-coding-agent/packages/coding-agent/docs/rpc.md` — verify steer works mid-flight
4. `packages/adapters/pi-local/src/server/execute.ts` — understand before touching

---

## Phase 0 Checklist (start here)

- [ ] Read this file end-to-end
- [ ] Read `README.md` — understand what Paperclip is
- [ ] Read `docs/design-paperclip-pi.md` — the planning doc
- [ ] Run `pnpm install && pnpm dev` — get the stack running locally
- [ ] Create a company in the UI, create a Pi agent, assign a task, watch it run
- [ ] Read `execute.ts` end-to-end before touching anything
- [ ] Read `tmp/pi-coding-agent/packages/coding-agent/docs/rpc.md` — verify steer
- [ ] Draw the architecture diagram (Paperclip → adapter → Pi, heartbeat + RPC paths)
- [ ] Commit the diagram to `docs/architecture.md` — this is your Phase 0 deliverable

---

## How to Work on This (Using gstack)

We use [gstack](https://github.com/garryslist/gstack) to structure sessions. Each time
you sit down to plan or hit a non-obvious decision, run:

```
/office-hours
```

This starts a session that challenges the approach and produces a design doc.

Other useful commands:
- `/plan-eng-review` — before writing Phase 2 or 3 code, review the design
- `/investigate` — when something is broken and you don't know why
- `/review` — before opening any upstream PR
- `/ship` — when you're ready to push

The planning doc for this project is at `docs/design-paperclip-pi.md`. Read it before
starting Phase 1.

---

## Reference: The Three Libraries

### Pi — Coding Agent Runtime
- Repo: [badlogic/pi-mono](https://github.com/badlogic/pi-mono) — cloned to `tmp/pi-coding-agent/`
- RPC mode docs: `tmp/pi-coding-agent/packages/coding-agent/docs/rpc.md`
- Key commands: `{"type":"prompt"}`, `{"type":"steer"}`, `{"type":"follow_up"}`, `{"type":"abort"}`

### Gondolin — Micro-VM Sandbox
- Repo: [earendil-works/gondolin](https://github.com/earendil-works/gondolin) — cloned to `tmp/gondolin/`
- Pi extension example: `tmp/gondolin/host/examples/pi-gondolin.ts`
- **Local only** — QEMU runs as child process. Not for remote use.

### Absurd — Durable Execution
- Repo: [earendil-works/absurd](https://github.com/earendil-works/absurd) — cloned to `tmp/absurd/`
- TypeScript SDK: `npm install absurd-sdk`
- Postgres only, no extra services needed

---

## Questions

- **Paperclip Discord**: https://discord.gg/m4HZY7xNG3
- **GitHub Issues**: https://github.com/paperclipai/paperclip/issues
- **Run `/investigate`** in Claude Code — describe what's broken
- **Ask Haziq** — he's read the codebase and been through Phase 0
