# Hey Alden 👋

This is your onboarding guide for this project. Read it before touching any code.

---

## What is this?

[Paperclip](https://github.com/paperclipai/paperclip) is an open-source platform for running a business with AI agents. Think of it as a company operating system — you define goals, hire AI agents, set budgets, and manage everything from a kanban board. Instead of babysitting 20 terminal tabs, you have an org chart.

This repo is a fork/local clone of Paperclip. The work here is about making one specific thing work better: **mid-flight steering**.

Right now, when you want to redirect an AI agent that's already working on a task, Paperclip kills it and restarts it from scratch. Context is lost. It's like hanging up a phone call and calling back just to change what you were saying.

[Pi](https://github.com/badlogic/pi-mono) — the coding agent Paperclip uses here — actually supports a `steer` command. You can send it a message mid-task and it pivots without restarting. No other agent runtime (Claude Code, Codex, Gemini CLI) has this. The problem is Paperclip's `pi_local` adapter doesn't use it yet.

**Your job: make it use it.**

When it's done, a human can watch an agent working on a task, type a redirect into the UI, and the agent adjusts course in real time. That's the demo. That's the thesis.

---

## Why this is interesting (and who cares)

The hard part of running AI agents isn't starting them. It's staying in control while they run.

The current solution everywhere is "kill and restart." It works, but it's lossy and jarring. Pi's RPC protocol solves this properly — there's a clean steering interface designed for exactly this. Nobody has wired it up to a real orchestration platform yet.

**Who cares:**
- Developers running multi-agent systems who know the kill-restart pain
- Founders who want to delegate to AI but stay in control
- The Paperclip community — this is a natural upstream contribution, and they'll want it

The steering feature is small in code surface area but big in product impact. One Steer button in the UI, wired correctly, changes the relationship between human and agent.

---

## Your learning objectives

This project is also a structured way to learn real software engineering. Not tutorials. Production code in a 57K-star repo.

Haziq is working through [*Web Scalability for Startup Engineers*](https://www.oreilly.com/library/view/web-scalability-for/9780071843584/) and each phase of this project maps to a chapter. Your job is to learn these concepts by applying them — not by reading about them in isolation.

| Phase | What you're building | Concept you're learning |
|-------|---------------------|------------------------|
| **0** | Read the codebase. Draw the architecture diagram. | Ch 2: understand design before touching anything. Loose coupling, single responsibility. |
| **1** | Get Paperclip running. Find and fix a small bug or doc gap. | Ch 2: reading production code. Making your first upstream PR. |
| **2** | Test the human-in-the-loop workflows. Document what you find. | Ch 7: async processing. The heartbeat scheduler is a message queue — learn it from the inside. |
| **3** | Implement `mode: "rpc"` in the Pi adapter. Add the Steer button to the UI. | Ch 3 + 4: stateless services, SOA. The adapter is a stateless service layer — understand why that matters. |
| **4** | Polish. Write the example config. Clean up docs. | Ch 9: automation, monitoring. Add tests to the Phase 3 PR. |

By the end you'll have read a real codebase, made real contributions to an open-source project, and applied scalability principles to actual production decisions — not homework.

---

## The contribution strategy

Every phase ends in an upstream PR to the real Paperclip repo.

This matters because:
- **Merged PRs are a public portfolio.** Better than any side project.
- **Upstream review is honest feedback.** Maintainers will push back on your PR if it's wrong. That's the learning.
- **It's real.** Your code will run for thousands of people.

The pattern: learn a concept → apply it to a real problem → open a PR → get reviewed → merge → repeat.

Phase 0 deliverable: an architecture diagram committed to `docs/`. Phase 1 deliverable: a small PR merged. Phase 3 deliverable: the RPC steering PR — the big one.

Don't open the Phase 3 PR until you've run a real task end-to-end through the steer demo. The demo gates the PR.

---

## The design doc

The full design for the RPC steering feature is here:

👉 [`docs/design-rpc-steering.md`](docs/design-rpc-steering.md)

Read it after you've finished Phase 0. It covers:
- Why the current adapter is fire-and-forget and what needs to change
- The exact implementation plan (specific files, endpoint shapes, data flow)
- Open questions that must be answered before writing Phase 3 code
- What the reviewer found and what was deferred

The one thing to verify before touching Phase 3 code: read `tmp/pi-coding-agent/packages/coding-agent/docs/rpc.md` and confirm Pi accepts a `steer` message mid-flight. The whole thesis depends on this.

---

## How to use gstack

This project uses [gstack](https://github.com/garryslist/gstack) — a set of AI-powered engineering skills that help you think through design, review code, and plan implementations.

You have access to it via Claude Code. Key skills:

| Skill | What it does | When to use it |
|-------|-------------|----------------|
| `/office-hours` | YC-style design session. Works through what you're building, surfaces alternatives, writes a design doc. | When you're about to start a new phase or a non-obvious decision comes up. |
| `/plan-eng-review` | Engineering architecture review. Finds gaps in a design before you implement it. | Before writing Phase 3 code. Run it on `docs/design-rpc-steering.md`. |
| `/investigate` | Debugging and root cause analysis. | When something is broken and you don't know why. |
| `/review` | Code review on your current diff. | Before opening any upstream PR. |
| `/ship` | Pre-PR checklist. Makes sure the branch is clean and the PR is ready. | When you're about to push. |

To run a skill, type it in the Claude Code prompt. Example: `/office-hours` or `/plan-eng-review`.

---

## Setup

```bash
# Clone and install
cd /Users/haziqazizi/paperclip
pnpm install

# Start the dev server
pnpm dev          # API at http://localhost:3100, UI at http://localhost:5173

# Verify Pi is installed
pi --version
```

Requirements: Node.js 20+, pnpm 9.15+

---

## Key files

| File | What it is |
|------|-----------|
| `packages/adapters/pi-local/src/server/execute.ts` | **The main file to modify for Phase 3.** Core adapter execution. |
| `packages/adapters/pi-local/src/server/parse.ts` | Pi output parser — needs an RPC equivalent. |
| `server/src/routes/issues.ts` | Issue routes — interrupt logic lives here. |
| `server/src/services/heartbeat.ts` | Heartbeat scheduler — how agents wake up. |
| `tmp/pi-coding-agent/packages/coding-agent/docs/rpc.md` | Pi's RPC protocol docs. **Read this before Phase 3.** |
| `docs/design-rpc-steering.md` | The design doc for this project. |
| `skills/paperclip/SKILL.md` | The skill injected into Pi agents — the heartbeat protocol. |

---

## Phase 0 checklist (start here)

- [ ] Read this document end-to-end
- [ ] Read `README.md` — understand what Paperclip is
- [ ] Read `docs/design-rpc-steering.md` — understand what you're building
- [ ] Run `pnpm dev` and get the stack running locally
- [ ] Create a company in the UI
- [ ] Create an agent using the `pi_local` adapter
- [ ] Assign it a simple task and watch it run
- [ ] Read `packages/adapters/pi-local/src/server/execute.ts` end-to-end
- [ ] Draw the architecture diagram (Paperclip → adapter → Pi, with heartbeat and RPC paths labeled)
- [ ] Commit the diagram to `docs/architecture.md`

That's Phase 0 done. Now you're ready to find the Phase 1 PR.

---

## Questions

If you're stuck, the best places:

- **Paperclip Discord**: https://discord.gg/m4HZY7xNG3 — the community is active
- **GitHub Issues**: https://github.com/paperclipai/paperclip/issues
- **Run `/investigate`** in Claude Code — describe what's broken, it'll dig in with you
- **Ask Haziq** — he's been through Phases 0 and 1 already

Good luck. The steer demo is worth it.

---

## The blog post

Once you've shipped the Phase 3 PR, write a post about it. Not a tutorial. Not a "here's what I learned" listicle. A real account of what it was like.

**Inspiration:** [Maggie Appleton — One Developer, Two Dozen Agents, Zero Alignment](https://maggieappleton.com) — a staff research engineer at GitHub talking about collaborative AI engineering. Great voice, digital garden format, no corporate polish.

Your angle is better. Maggie is presenting a research prototype from inside GitHub. You're a developer who was handed a real open-source project, learned scalability engineering by reading production code, and made upstream PRs to a 57K-star repo. That's a first-person account, not a talk.

**What the post should cover:**

- What `paperclip-steering` is and why mid-flight agent steering matters
- What it's actually like to onboard onto a production codebase as a junior dev
- How the scalability book chapters mapped to real decisions (the heartbeat scheduler as a message queue, the adapter as a stateless service layer)
- The ColeMurray pattern — learning via upstream contributions, merged PRs as the artifact
- The steer demo: what it looks like when it works

**The demo recording is required.** Screen record the moment an agent is mid-task writing something real, you type a redirect into the Steer button, and it pivots in-place. That's the lede. Put it at the top of the post.

**Format:** Digital garden style — [Maggie's site](https://maggieappleton.com) is the reference. Evergreen, not time-stamped. "Planted" date, not a publication date. Personal voice.

**Where to publish:** Wherever you already write, or start a garden. The post should link back to the PR so people can see the real thing.

---

## Using this stack to build your own projects

Once `background-agents-pi` is set up, use it for your own projects — not just this one.

[`Goober-Codes/background-agents-pi`](https://github.com/Goober-Codes/background-agents-pi) is a fork of Open-Inspect that replaces the cloud execution layer with Pi (as the coding agent) running inside Daytona dev environments. [`Goober-Codes/daytona-pi`](https://github.com/Goober-Codes/daytona-pi) is the Daytona fork wired up with Gondolin (VM-level network security) and Absurd (durable workflows on Postgres).

The setup gives you: Pi coding agents running in isolated cloud VMs, with durable workflows so work survives interruptions. It's the same Pi you're working with in `paperclip-steering` — just in a background-agent setup instead of an orchestrated one.

Use it on your own projects alongside this one. The best way to understand what you're building is to use the tools you're building around.
