---
name: build-orchestrator
description: Multi-task parallel dispatch with quality gates. Decomposes large features into bounded task contracts, dispatches to isolated worktree agents in parallel, reviews, merges, and iterates. Supports two modes — single-feature (default) and roadmap-driven (continuous). Use when the user says "编排", "dispatch", "parallel build", "拆开做", "并行推", or wants multiple independent slices built and landed simultaneously.
user-invocable: true
argument-hint: "[--roadmap <path>] <feature, module, or milestone to build>"
---

# Build Orchestrator

**User's request:** $ARGUMENTS

## Core Idea

Use this skill when the best move is not one big implementation in the current thread, but a controlled batch of small independent agents, each in its own worktree. You own decomposition, dispatch, review, merge, and cleanup.

Best for early development or large-module buildout where many slices can move in parallel. Poor for hardware-heavy acceptance, production deploys, payment tests, or steps requiring frequent human observation — keep those serialized and explicit.

Treat this skill as a living runbook. When orchestration reveals a better rule, update the skill so future sessions inherit the correction.

## Two Modes

### Single-feature (default)

```
/build-orchestrator <feature description>
```

Build one feature and stop. Use when you know exactly what to build.

### Roadmap-driven (continuous)

```
/build-orchestrator --roadmap docs/roadmap.md
```

Read a roadmap document, pick the next feature by priority, build it, then loop: done → re-read roadmap → pick next → continue, until a stop condition is hit.

The orchestrator loops automatically: done → re-read roadmap → pick next → continue, until a stop condition is hit. See the "Roadmap-Driven Mode" section below for details.

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

### Worktree Base Trap (learned the hard way)

`isolation: "worktree"` creates the worktree from **`origin/main`**, not local `main`. If you just merged a serial batch to local main but didn't push, parallel batch agents **won't see the serial batch's changes**.

Mitigations (pick one):
- **Recommended: push after serial batch merge**, then dispatch parallel batch. This keeps `origin/main` in sync with local main
- Fallback: don't push, but explicitly tell parallel agents "your base is missing XYZ changes that are on local main — do not redo them, only build your own part." Less reliable than pushing
- If an agent goes out of bounds and redoes serial batch work: use `git cherry-pick <commit> --no-commit` to extract only that agent's unique changes, discard duplicates

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
# Must be on main before merging (learned the hard way: merging on a stale branch loses changes)
git checkout main
git merge --no-ff <task-branch> -m "merge: <scope>"
git worktree remove <worktree-path>
git branch -d <task-branch>
```

If the agent went out of bounds (modified forbidden paths or has stale base):
```bash
git checkout main
git cherry-pick <commit> --no-commit
# Inspect staged changes, unstage out-of-bounds files
git reset HEAD <forbidden-file>
git checkout -- <forbidden-file>
# Commit only the in-scope changes
git commit -m "<scope>"
# Clean up the out-of-bounds agent's worktree and branch
git worktree remove <worktree-path>
git branch -D <task-branch>
```

If rejected: report blocking findings to the user. Leave the branch/worktree for targeted fix. To fix, send a follow-up to the same agent via `SendMessage` (preserves context), or dispatch a new agent with the rejection findings.

Resolve simple doc conflicts by preserving both entries. If the conflict is in a shared contract, migration, core aggregate, or protocol — stop and review manually.

### Cleanup Verification (learned the hard way)

After all merges and cleanups in a batch, verify:

```bash
git branch --show-current          # Must be main
git branch                         # Only main (and long-lived branches) should remain
git worktree list                  # Only the main working tree should remain
```

**Do not skip this.** Leftover branches and worktrees pollute the next batch's agents (worktree lock conflicts, branch name collisions, accidental merge into a stale branch).

### Post-Merge Integration Test (learned the hard way)

After merging all branches in a batch, run a **cross-layer integration test** — don't rely solely on each agent's individual gate results. Individual layers passing doesn't mean they work together.

```bash
# Example: run tests across all affected subsystems after merge
<project-specific test command covering all modified subsystems>
```

### Step 7: Push + Next Batch

After merging accepted branches and resolving rejected ones:
- **Push to origin/main** (both single-feature and roadmap modes) — keeps origin/main in sync with local main, avoids the worktree base trap for the next batch
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
- Do not revert, delete, or modify code you did not write in this task.
  You are not alone in the codebase — other agents may have added files
  or changed code that looks unfamiliar. Adapt to current state, don't undo it.
- Do not start subagents or second-level delegation.
- Self-review your diff before final commit: check boundaries, forbidden paths,
  shared contracts, evidence strength, and gates.
- If the feature involves state transitions (e.g. PENDING→ACTIVE→COMPLETED),
  verify that ALL consumers handle ALL transitions — not just the happy path
  in your own layer. Check: does the other layer receive the event that triggers
  this transition? Does it have code to apply it? What happens if the state
  change is never received?
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

## Roadmap-Driven Mode

When the user passes `--roadmap <path>`, enter continuous orchestration mode. The orchestrator autonomously selects the next feature from the roadmap document — no need for the user to specify what to build each time.

### Execution Loop

```
1. Read roadmap document, understand tier/priority structure
2. Read repo state (git log, PROGRESS.md), confirm what's done and in-progress
3. Filter buildable features from current tier:
   - Skip blocked (needs hardware/payment/credentials/human action)
   - Skip completed
   - Prefer features that unblock other tasks
   - Stay within current tier (don't jump ahead)
4. Decide serial vs parallel:
   - Only 1 buildable → execute directly
   - Multiple buildable → check Concurrency Rules:
     Write sets fully disjoint + no shared contract conflict → batch in parallel
     Any overlap → serialize, one at a time
5. Execute standard Operating Loop for selected feature(s) (decompose → dispatch → review → merge)
6. After feature completion:
   - Push to origin/main
   - Update PROGRESS.md
   - Return to Step 1, re-read roadmap, pick next
7. Stop when a stop condition is hit
```

**Prerequisites for cross-feature parallelism** (reusing Concurrency Rules):
- Each feature's write set (allowed paths) is fully disjoint
- No feature modifies shared contracts (proto/DDL/API definitions)
- No feature requires hardware/payment/credentials
- Max 2 features in parallel, never more than 3

### Selection Rules

- **Don't jump tiers**: if the current tier still has buildable features, don't skip to the next tier
- **Don't guess priorities**: among features without explicit priority ordering, pick the smallest write set / lowest risk first
- **Don't do owner-gated tasks**: tasks needing product decisions, payment credentials, or human approval → mark skipped with reason
- **Don't get stuck on one feature**: if an agent returns unsolvable issues (compile errors needing design decisions, tests needing external deps), mark blocked and move on
- **Push after every feature**: keep origin/main in sync with local main to avoid the worktree base trap for the next feature

### Progress Reporting

After completing each feature, output a one-line progress summary:

```
[Tier 1] ✅ Feature A (3 commits, 450 lines) → ⏭️ Next: Feature B
[Tier 1] ⏭️ Feature B → skipped: blocked (needs payment backend)
```

### Stop Conditions (roadmap-specific)

In addition to the general stop conditions:

- Current tier has no remaining buildable features (all blocked or completed) — report tier status, wait for user decision on next tier
- 2 consecutive features fail (agent results rejected at review) — likely a systemic issue, stop and investigate
- Roadmap document structure is unclear, can't determine next feature — ask user to update roadmap

---

## Independent Cross-Agent Review

After all batches for a feature are merged, run an independent review of the entire feature diff using a separate review tool or agent. This catches blind spots shared by the orchestrator and its agents — they all use the same model and may share the same biases.

In real-world testing, an independent review caught 2 critical bugs (missing state transition handling across layers + silent cancellation edge case) that both agent self-review and orchestrator review missed.

**Recommendation**: for features involving **state machines** or **cross-layer event flows**, an independent post-merge review is not optional. Fix findings before marking the feature complete.

---

## Using /loop With This Skill

`/loop` is not needed for the core dispatch-review-merge cycle (agents are synchronous). Use `/loop` only when:

- A dispatched agent kicked off a long-running background process (build, soak test, benchmark) and you want to monitor its output
- You want periodic progress reports while agents are running in a `Workflow`
- You're waiting for an external event (CI, deploy) before the next batch

For the main orchestration loop, just iterate: dispatch → wait for results → review → merge → next batch.
