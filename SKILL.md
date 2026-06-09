---
name: build-orchestrator
description: Multi-task parallel dispatch with quality gates. Decomposes large features into bounded task contracts, dispatches to isolated worktree agents in parallel, reviews, merges, and iterates. Complements paseo-epic (single-task depth) with multi-task width. Use when the user says "编排", "dispatch", "parallel build", "拆开做", "并行推", or wants multiple independent slices built and landed simultaneously.
user-invocable: true
argument-hint: "<feature, module, or milestone to build>"
---

# Build Orchestrator

**User's request:** $ARGUMENTS

## Core Idea

Use this skill when the best move is not one big implementation in the current thread, but a controlled batch of small independent agents, each in its own worktree. You own decomposition, dispatch, review, merge, and cleanup.

Best for early development or large-module buildout where many slices can move in parallel. Poor for hardware-heavy acceptance, production deploys, payment tests, or steps requiring frequent human observation — keep those serialized and explicit.

Treat this skill as a living runbook. When orchestration reveals a better rule, update the skill so future sessions inherit the correction.

## When To Use This Skill

| Approach | When | Difference |
|---|---|---|
| **Single agent** | One focused task, sequential phases | Simple, no coordination overhead |
| **Manual multi-agent** | Ad-hoc parallel agents, you track everything | Flexible but error-prone, no merge safety |
| **build-orchestrator** | Multiple independent tasks in parallel with quality gates | Bounded task contracts, worktree isolation, anti-shallow-slice enforcement, batch dispatch-review-merge cycles |

Use a single agent for depth (one complex task). Use this skill for width (many independent tasks in parallel).

## Execution Model

Claude Code agents are **synchronous**: they run and return results within the current session. No fire-and-forget, no heartbeat polling.

```
Dispatch (parallel Agent calls with isolation:"worktree")
    ↓ agents run concurrently, each in own worktree
All agents return results (worktree paths, branches, commits)
    ↓
Review each branch
    ↓
Merge accepted / reject failed → cleanup
    ↓
Next batch (or stop)
```

Primary tool: `Agent` with `isolation: "worktree"`. Multiple Agent calls in one message run concurrently. For complex multi-phase orchestration with schemas, use `Workflow` instead.

---

## Operating Loop

### Step 1: Reconnaissance

Before decomposing, read the real repo state:

```bash
git status --short --branch
git log --oneline -10
git worktree list
```

Read project progress/roadmap docs if present (e.g. `PROGRESS.md`, roadmap files). Understand what's done, what's blocked, what's next. Do not decompose from memory.

If shared contract surfaces exist (proto definitions, DB migrations, API contracts, command/event handlers), identify them — they must be serialized before parallel work.

### Step 2: Decompose

Choose the next **feature package** before choosing individual tasks:

- When a domain already has multiple partial closures, define the module-level milestone first.
- Break that milestone into the fewest parallel worker contracts that can safely merge.
- Only dispatch a tiny standalone task when it removes a named blocker or lands a shared contract needed by the larger milestone.

For each candidate task, determine:
- Allowed paths (directories/files this agent may edit)
- Forbidden paths (shared contracts, other agents' domains)
- Required gates (tests, typecheck, build, lint)
- Evidence type needed (`direct` / `proxy` / `blocked`)
- Whether it needs hardware, payment, or human action (if yes → serialize, don't parallel-dispatch)

### Step 3: Anti-Shallow-Slice Gate

Before dispatching a new task in a domain that already has a partial closure, classify it as one of:

| Classification | Meaning | Example |
|---|---|---|
| `vertical-completion` | Connects already-landed pieces into a more complete end-to-end flow | UI action → API → persistence → readback |
| `runtime-proof` | Proves an existing local/proxy path in a real runtime | Browser test, device smoke, LAN proof |
| `blocked-removal` | Removes a named blocker preventing the next complete flow | Missing write API, stale device path, missing auth seam |
| `owner-gated` | Records the exact human/product/payment decision that blocks progress | Needs product priority call, payment backend access, credentials |

**Reject** if the candidate is only another read-only shell, placeholder page, static review, copy checklist, local fixture summary, or first guard in an already-partial domain — unless it clearly removes a named blocker.

If the same domain has two or more merged partial closures and no single blocker prevents progress, stop dispatching standalone slices. Promote to a feature-package plan (Step 2).

### Step 4: Dispatch

Send multiple `Agent` calls **in a single message** — they run concurrently:

```
Agent({
  isolation: "worktree",
  prompt: "<task contract — see template below>",
  description: "<3-5 word label>"
})
Agent({
  isolation: "worktree",
  prompt: "<task contract>",
  description: "<3-5 word label>"
})
```

Each agent gets its own git worktree automatically. When it finishes, the result includes the worktree path and branch if changes were made.

**Do not dispatch and then start doing your own implementation work.** You are the orchestrator. Wait for agent results, then review.

### Step 5: Review

For every completed agent, inspect:

```bash
# In the agent's worktree
git -C <worktree> status --short --branch
git -C <worktree> log --oneline -5
git -C <worktree> diff --name-status main..HEAD
git -C <worktree> diff --check main..HEAD
```

Check:
- Changed files match the task boundary (allowed paths only)
- No forbidden shared contracts changed
- Self-review present in agent output
- Gates ran and passed (tests, typecheck, build)
- Evidence labels honest (`direct` / `proxy` / `blocked`)
- No evidence exaggeration (local unit test claimed as direct proof, TCP reachable claimed as payment proof)
- Docs/progress updates factual

### Step 6: Merge and Cleanup

If accepted:

```bash
git merge --no-ff <task-branch> -m "merge: <scope>"
# Run post-merge gates
git worktree remove <worktree-path>
git branch -d <task-branch>
```

If rejected: report blocking findings to the user. Leave the branch/worktree for targeted fix. To fix, send a follow-up to the same agent via `SendMessage` (preserves context), or dispatch a new agent with the rejection findings.

Resolve simple doc conflicts by preserving both entries. If the conflict is in a shared contract, migration, core aggregate, or protocol — stop and review manually.

### Step 7: Next Batch

After merging accepted branches and resolving rejected ones:
- Update progress docs if the project uses them
- Re-read repo state (Step 1)
- Decompose next batch (Step 2)
- Repeat until the feature package is complete or a stop condition is hit

---

## Dispatch Prompt Template

Each agent prompt should include:

```text
## Task
<plain-language outcome — what, not how>

## Classification
<vertical-completion | runtime-proof | blocked-removal | owner-gated>
Why this is not repeating an already-completed first slice: <explanation>

## Scope
- Base: origin/main (current HEAD)
- Allowed paths: <list>
- Forbidden paths: <list>
- Branch name: build/<task-slug>

## Gates
- <test command>
- <typecheck command>
- git diff --check

## Evidence
Label all evidence as `direct`, `proxy`, or `blocked`.
Do not upgrade local/unit/integration tests, TCP reachability, screenshots,
or SENT status into direct proof.

## Rules
- Read project CLAUDE.md and relevant rules before editing.
- Create branch `build/<task-slug>` and commit only scoped changes.
- Do not touch files outside allowed paths.
- Do not start subagents or second-level delegation.
- Self-review your diff before final commit: check boundaries, forbidden paths,
  shared contracts, evidence strength, and gates.
- If you need a human physical action (swipe card, plug USB, etc.), state the
  exact action, device, risk, and what the user should reply. Then stop and wait.

## Handoff
Report: branch, final commit(s), changed files, gate results, evidence labels,
residual risks.
```

Adapt the template per task. Hardware/payment tasks need explicit device ownership, mutual exclusions, and human-action notification instructions.

---

## Concurrency Rules

Default to **two** parallel agents. Allow **three** only when all of:
- No shared contract, migration, or API branch is active
- No hardware/payment task is active
- Main is clean
- Each task has a separate module and disjoint write set

Use **one** (serialize) when:
- A shared contract is being edited (proto, migration, API definition)
- Hardware or payment device ownership is involved
- The next step depends on a result from the current task
- Two tasks would edit the same shared contract / migration / core aggregate / review file

Do not open more agents just because capacity exists. Parallelism should reduce calendar time without increasing merge risk.

---

## Evidence Discipline

For hardware, payment, deploy, and environment work:

- Label proof as `direct`, `proxy`, or `blocked`
- `SENT` / TCP reachable / local test / screenshot / user oral confirmation = **proxy at best**
- `direct` requires: device readback, callback-backed ACKED, processor response, DB/API artifact, or equivalent physical evidence
- If human action is needed, pause at a safe checkpoint, state the exact action, and wait for confirmation before continuing

If the project has its own evidence rules, those take precedence.

---

## Feature-Package Planning Gate

Use when the user asks for larger functional work, or when a domain has accumulated several partial closures.

Answer before dispatching:

1. What is the feature package in user/operator terms?
2. Which existing closures does it build on?
3. What is the smallest coherent end-to-end capability to deliver?
4. Which work must be serial (shared contracts, migrations, APIs)?
5. Which work can be parallel (disjoint write sets, independent evidence)?
6. What evidence will prove the package, and what remains `blocked` or `owner-gated`?

Prefer package-sized outcomes:
- UI action → API/client → persistence/projection → readback/audit
- Operational flow across list/detail/write/readback states
- Local runtime proof for a previously source-only flow

Do not use package planning as permission for a huge unreviewable branch. Split into mergeable worker contracts, but each must be tied to the package outcome.

---

## When To Stop

Stop dispatching and report status when:

- All remaining tasks need hardware, payment backend, credentials, or production access
- A shared contract must be decided before more consumers can proceed
- Current agents are running and new work would compete for the same files/resources
- The roadmap is stale enough that choosing work would be speculation
- Multiple candidates require product priority decisions

When stopping, report: completed/merged tasks, active/blocked tasks, clean repo state, next best candidates, and blockers.

---

## Using /loop With This Skill

`/loop` is not needed for the core dispatch-review-merge cycle (agents are synchronous). Use `/loop` only when:

- A dispatched agent kicked off a long-running background process (build, soak test, benchmark) and you want to monitor its output
- You want periodic progress reports while agents are running in a `Workflow`
- You're waiting for an external event (CI, deploy) before the next batch

For the main orchestration loop, just iterate: dispatch → wait for results → review → merge → next batch.
