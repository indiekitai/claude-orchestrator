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

**A living runbook prunes, not just grows.** Rules only get appended here; nothing forces them out. That is itself a failure mode (ORC-17): a bloated runbook stops fitting in dispatch prompts and the orchestrator can't hold it all (worsens ORC-06 constraint amnesia). Periodically — every long run, or whenever syncing codex-orchestrator ↔ claude-orchestrator — walk each gate/anti-pattern and ask "did this actually catch a bug in a recent run?" A rule with no evidence it's working gets demoted to a docs/ appendix or deleted. Syncing means importing new discipline *and* trimming dead discipline.

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

This is a **deliberate tradeoff**, not a limitation to route around: synchronous dispatch is what makes the review→merge gates enforceable and keeps merge risk bounded. The cost is no async fan-out. When Claude Code's background/async agents mature, re-evaluate this section — but the gain to protect is the Batch Status Report (below), which is a cheap context-rebuild checkpoint. Async multi-agent work trades throughput for context-switch overload; the structured per-batch report is the antidote, so it becomes *more* valuable under async, not less.

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

If the project has a **concepts file** (e.g. `concepts.md`, `GLOSSARY.md`), read it — it contains stable project terms, rules, prior decisions, and historical pitfalls. This reduces constraint amnesia (ORC-06) after context compression.

If the project has an **inbox file** (e.g. `inbox.md`), read it — it collects untriaged issues, user feedback, external review findings, and run observations that haven't been turned into tasks yet. Check if any inbox item should influence the current batch.

If shared contract surfaces exist (proto definitions, DB migrations, API contracts, command/event handlers), identify them — they must be serialized before parallel work.

**Contract sync gate**: if the repo has a contract sync gate that requires same-change consumers (e.g. a proto sync check requiring multiple consumers to update after `.proto` changes), do not dispatch an unmergeable "contract-only" branch. Keep the work serialized, but include the minimal required consumer compile/wiring updates in that same serial task, or stop with a blocker before editing.

### Step 2: Decompose

Choose the next **feature package** before choosing individual tasks:

- When a domain already has multiple partial closures, define the module-level milestone first.
- Break that milestone into the fewest parallel worker contracts that can safely merge.
- Only dispatch a tiny standalone task when it removes a named blocker or lands a shared contract needed by the larger milestone.

For each candidate task, determine:
- Allowed paths (directories/files this agent may edit)
- Forbidden paths (shared contracts, other agents' domains)
- Required gates (tests, typecheck, build, lint)
- Evidence type needed (`direct` / `proxy` / `local` / `blocked`)
- Whether it needs hardware, payment, or human action (if yes → serialize, don't parallel-dispatch)

**Package lane guard**: do not fill idle slots by grabbing unrelated "safe" tasks from the global backlog merely because their write sets don't conflict. After a worker finishes, first ask "what is the next useful task inside the current feature package?" Only switch packages when the current package is blocked, its local scope is genuinely drained, or the user explicitly asks to change focus.

**Package ledger** (lightweight): for multi-batch features, keep a brief mental or written record of: milestone outcome, active worker contracts, merge order, gates, and what evidence remains blocked. This prevents the orchestrator from losing track of the package shape across batches — especially important after context compression.

### Step 3: Anti-Shallow-Slice Gate

Before dispatching a new task in a domain that already has a partial closure, classify it as one of:

| Classification | Meaning | Example |
|---|---|---|
| `vertical-completion` | Connects already-landed pieces into a more complete end-to-end flow | UI action → API → persistence → readback |
| `runtime-proof` | Proves an existing local/proxy path in a real runtime | Browser test, device smoke, LAN proof |
| `blocked-removal` | Removes a named blocker preventing the next complete flow | Missing write API, stale device path, missing auth seam |
| `owner-gated` | Records the exact human/product/payment decision that blocks progress | Needs product priority call, payment backend access, credentials |

**Reject** if the candidate is only another read-only shell, placeholder page, static review, copy checklist, local fixture summary, or first guard in an already-partial domain — unless it clearly removes a named blocker. The task prompt must answer: what complete feature path does this advance, what previous partial closure does it build on, and what will still remain after this slice.

For domains with several partial closures, **prefer fewer larger vertical tasks over many small horizontal tasks**. A vertical task that stays `local`/`proxy` (because hardware or production is unavailable) is fine, as long as it exercises a coherent local flow rather than adding another isolated surface.

If the same domain has two or more merged partial closures and no single blocker prevents progress, stop dispatching standalone slices. Promote to a feature-package plan (Step 2). The package plan should name the user-visible capability, list the minimum worker branches needed to make it coherent, and define the merge order.

**Roadmap-driven simplification**: when the roadmap document explicitly lists the next item, anti-shallow-slice classification can be reduced to two questions: (1) Is this the next item in the roadmap? (2) Does it belong to the same feature package as the previous batch? Full four-way classification is only needed when the roadmap is ambiguous or the orchestrator is ad-hoc selecting tasks.

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

**If dispatch fails** (agent errors, worktree setup fails, etc.), stay in the orchestration layer: report the dispatch/tooling blocker, fix the dispatch method, or ask the user. Do not implement the task yourself just because dispatch failed (ORC-07). Direct implementation is only acceptable as a takeover when an agent already produced a useful partial diff that needs finishing, or when the user explicitly asks you to do the task yourself.

### Step 5: Review

For every completed agent, first check commit status:

```bash
# Check if agent actually committed (the #1 failure mode)
git -C <worktree> status --short --branch
git -C <worktree> log --oneline -3
```

**If the agent has uncommitted changes** (modified files in `git status` but no new commit on branch): this is common with Claude Code subagents. The orchestrator must commit on behalf of the agent before reviewing:

```bash
git -C <worktree> add -A
git -C <worktree> commit -m "<task-slug>: <brief description>"
```

Then inspect the committed diff:

```bash
git -C <worktree> diff --name-status main..HEAD
git -C <worktree> diff --check main..HEAD
```

Check:
- Changed files match the task boundary (allowed paths only)
- No forbidden shared contracts changed
- Self-review present in agent output
- Gates ran and passed (tests, typecheck, build)
- Evidence labels honest (`direct` / `proxy` / `local` / `blocked`)
- No evidence exaggeration (local unit test claimed as direct proof, TCP reachable claimed as payment proof, `local` claimed as `proxy`)
- Docs/progress updates factual
- **Claim verification**: if the agent claims "tests passed" / "build succeeded" / "no forbidden paths touched", verify that actual command output or diff supports the claim. Don't blindly trust self-reported completion.
- **Idempotency check**: if the task changes cleanup, retry, event/outbox writes, lifecycle APIs, migrations, aggregate versioning, or unique constraints — verify the self-review discusses repeated execution safety. Running the action twice must either be safe or produce a documented `blocked` / product decision.
- **Authorization awareness**: review evidence does not by itself authorize merge, push, cleanup, release, or deploy. Keep these decisions separate.
- **Live proof gate**: if the task changes a runtime, production, device, payment, hardware, provider, or external-service boundary, `direct` proof or an explicit item-specific waiver is required before landing. `local` gate passing is not sufficient for these boundaries.

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

### Step 7: Push + Cross-Model Review + Next Batch

After merging accepted branches and resolving rejected ones:
- **Push to origin/main** (both single-feature and roadmap modes) — keeps origin/main in sync with local main, avoids the worktree base trap for the next batch
- **Run cross-model review** (Codex CLI or Pi). Can run in the background, but **must confirm review is launched before entering Step 4 Dispatch for the next batch**. Cannot skip, cannot forget, cannot "rush ahead without review".
- Update progress docs if the project uses them
- Re-read repo state (Step 1)
- Decompose next batch (Step 2)
- Repeat until the feature package is complete or a stop condition is hit

**Review Gate (mandatory)**: before dispatching the next batch, check:
1. Has the current batch's cross-model review been launched? (running in background = OK, not started = cannot dispatch)
2. Has the previous batch's review result been processed? (P1/HIGH must be fixed, P2/MEDIUM must be recorded)
3. **Review must leave a trace**: record findings count in progress docs (e.g. "Codex: 1×P1 + 3×P2, P1 fixed"). No record = not run. "Claimed to have run but no findings table" does not count.
If 2 consecutive batches skip review, **stop orchestration and report** — this is systemic discipline breakdown, not an occasional miss.

**P2 Finding Tracker**: P2 findings don't block the current batch but cannot be deferred forever.
- After each batch review, append unfixed P2s to an open P2 list in progress docs (or a standalone file)
- Check the open P2 list at the start of each batch
- **A P2 unfixed for 3+ batches escalates to P1** — either fix immediately or give a concrete reason to mark it deferred

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
Label all evidence as `direct`, `proxy`, `local`, or `blocked`.
- `local` = dev/test environment only (unit test, local build, static analysis)
- `proxy` = substitute environment or intermediary (staging, TCP check, screenshot)
- `direct` = real target environment (device readback, production log, payment confirmation)
Do not upgrade between levels.

## Rules
- **⚠️ YOU MUST COMMIT before reporting done.** Run `git add` + `git commit`
  on your task branch. If you report "done" without a commit, the orchestrator
  has to do it for you, which wastes a review cycle. This is the #1 agent
  failure mode — do not skip it.
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
- If the task changes cleanup, retry, event/outbox writes, lifecycle APIs,
  migrations, aggregate/versioning, or unique constraints, explicitly self-review
  repeated execution and idempotency: running the action twice must either be
  safe or produce a documented blocked/product decision.

## Common P1 Patterns (self-check before commit)
Before committing, verify you have NOT introduced these recurring bugs:
- DTO field names match the actual API/Cloud response (e.g. `businessDay` vs `businessDate`)
- State machine covers ALL legal states, not just the happy path
- Multi-tenant queries include `tenant_id` / `store_id` filter
- Money uses integer cents, never floating-point
- Nullable fields from external APIs have null handling
These patterns account for ~40% of P1 findings in cross-model review.

## Minimalism Ladder (self-check before commit)
For every new function, class, or file you created, walk down this ladder.
Stop at the first "yes":
1. Does this need to exist at all? → delete it
2. Does the stdlib / platform already provide it? → use that
3. Does an installed dependency already do it? → use that
4. Can it be a one-liner instead of a function? → inline it
5. None of the above → keep it, but write the minimum
Do NOT simplify security boundaries, error handling at trust boundaries,
or accessibility code — lazy, not negligent.

## Shared Resource Files
The following files are commonly edited by multiple parallel agents. When
editing them, ONLY APPEND new entries — do not reorder, reformat, or modify
existing lines. This minimizes merge conflicts when the orchestrator combines
branches:
<list — e.g. strings.xml, navigation graph, DI module, route registry>

## Handoff
Report: branch, final commit(s), changed files, gate results, evidence labels,
residual risks. If you did not commit, say so explicitly — do not imply a
commit exists.
```

Adapt the template per task. Hardware/payment tasks need explicit device ownership, mutual exclusions, and human-action notification instructions.

---

## Concurrency Rules

**Default to one agent** when tasks are in the **same module** and share navigation/resource/config files (e.g. multiple new pages in the same Android app or web app). Serialize: dispatch A → merge → push → dispatch B.

**Allow two parallel agents** only when write sets are in **different modules or different projects** (e.g. backend service + frontend app, or two independent libraries).

Allow **three** only when all of:
- No shared contract, migration, or API branch is active
- No hardware/payment task is active
- Main is clean
- Each task has a separate module and fully disjoint write set

Use **one** (serialize) when:
- A shared contract is being edited (proto, migration, API definition)
- Hardware or payment device ownership is involved
- The next step depends on a result from the current task
- Two tasks would edit the same shared contract / migration / core aggregate / review file
- **Two tasks are in the same module and both add new pages/screens** (they will both touch navigation, strings, DI, and config — merge conflicts are guaranteed and cost more time than serialization saves)

Do not open more agents just because capacity exists. Parallelism should reduce calendar time without increasing merge risk. **In practice, same-module parallelism almost never saves time** — merge conflict resolution (10-15 min per batch) exceeds the wall-clock gain from concurrent agents.

**Shared resource files**: files that every new page/feature touches within a module (e.g. `strings.xml`, navigation graphs, DI modules, route registries). When these overlap between two tasks, **serialize, don't parallelize**. The "only append" strategy helps git auto-merge (~57% success rate in testing), but the 43% failure rate means manual conflict resolution is frequent enough to negate parallelism benefits.

If you must run two agents in the same module (user explicitly requests it), mitigate with:
- Tell each agent to only append, not reformat or reorder shared files
- Use scoped naming for new types (e.g. `OdPaymentStatus` vs `PaymentStatus`)
- Budget 10-15 minutes per batch for manual conflict resolution

**Available slots ≠ dispatch permission.** Before dispatching a new agent, check:
- Is the current feature package still the right focus? (don't scatter across unrelated work)
- Are there unreviewed/unmerged branches from the current batch? (review first, then dispatch)
- Does the new task genuinely belong to the same package lane?
If an agent just finished and freed a slot, the default is to review/merge/cleanup first, not to immediately fill the slot.

---

## Evidence Discipline

Label all proof with one of four levels:

| Level | Meaning | Examples |
|---|---|---|
| `direct` | Observed in the real target environment | Device readback, callback-backed ACKED, processor response, DB/API artifact, production log, real payment confirmation |
| `proxy` | Observed in a substitute environment or through an intermediary | Staging test, TCP reachability, screenshot, user oral confirmation, `SENT` status, mock service response |
| `local` | Observed only in the local dev/test environment | Unit test passing, local build succeeding, `git diff --check` clean, local integration test, static analysis passing |
| `blocked` | Cannot be observed — needs human action, credentials, hardware, or production access | Payment backend unavailable, device not connected, credentials missing, product decision pending |

Rules:

- Do not upgrade `local` to `proxy` or `proxy` to `direct`. Each level means what it means.
- `SENT` / TCP reachable / local test / screenshot / user oral confirmation = **proxy at best**.
- `direct` requires: device readback, callback-backed ACKED, processor response, DB/API artifact, or equivalent physical evidence.
- If human action is needed, pause at a safe checkpoint, state the exact action, and wait for confirmation before continuing.

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

**Package lane continuity**: in continuous/roadmap mode, keep the next-batch decision package-scoped. After a worker is merged and cleaned, first ask "what is the next useful worker inside the current package?" Only switch packages when:
- Current package is blocked by owner/hardware/provider dependencies
- Its local scope is genuinely drained (all buildable tasks done)
- The user explicitly asks to change focus

Record the switch reason concretely: `package-closed`, `local-scope-drained`, `blocked`, `owner-gated`, or `shared-blocker-removal`. Do not switch packages just because there is an available slot or another safe task exists.

---

## When To Stop

Stop dispatching and report status when:

- All remaining tasks need hardware, payment backend, credentials, or production access
- A shared contract must be decided before more consumers can proceed
- Current agents are running and new work would compete for the same files/resources
- The roadmap is stale enough that choosing work would be speculation
- Multiple candidates require product priority decisions

When stopping, report: completed/merged tasks, active/blocked tasks, clean repo state, next best candidates, and blockers.

### Content Filter Recovery

If an agent is blocked by content filtering (safety filter), do not waste a full review cycle:
1. Record the blocked task slug and the approximate prompt that triggered it
2. In the next batch, retry with adjusted wording — remove or rephrase the content that likely triggered the filter
3. If the retry also fails, mark the task as `blocked` with reason "content filter" and move on
4. Do not stop the entire orchestration for a single content filter hit

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

**Multi-model review**: when available, prefer a reviewer from a different model family (e.g. use `/pi-review` or Codex for a Claude-orchestrated feature). Same-model review catches formatting and logic issues but shares the same reasoning blind spots. Cross-model review is `proxy`/advisory evidence — it can block acceptance or inform the orchestrator, but does not by itself authorize merge, push, or deploy.

### Codex Review (recommended cross-model review method)

After each batch merge + push, run Codex review via the **companion script** — do NOT use the standalone `codex review` CLI (it has parameter conflicts, no `--wait`, and times out on large diffs). See `~/.claude/rules/codex-review.md` for full details.

```bash
PLUGIN_ROOT="$HOME/.claude/plugins/marketplaces/openai-codex/plugins/codex"

# Recommended: blocking wait for result
node "$PLUGIN_ROOT/scripts/codex-companion.mjs" review --wait --base <pre-merge-commit>

# Or background (don't block next batch dispatch)
# Use Bash run_in_background, handle results when they arrive
```

Review results:
- P1 findings → fix immediately, in current or next batch
- P2 findings → record to open P2 list, fix within current feature package
- P3 findings → record but don't block

**Review the whole invariant, not one patch at a time (avoid review ping-pong).** When a finding exposes a bug in a shared invariant/contract (a partition that must sum, a wage/total computed from parts, a state machine, a multi-place sync), do NOT just patch the one line the reviewer pointed at and immediately re-review that patch in isolation. First reason through the *complete* invariant and fix every face at once, then run ONE cross-model review of the whole feature diff. Patching a single face and re-reviewing it alone makes the reviewer see only the latest diff — so each fix exposes the next adjacent face and you ping-pong across many rounds (real case: 4 consecutive Codex rounds, each a real P1, all faces of one labor-seconds partition invariant: regular=worked−Σpremium AND wage must price every bucket AND every bucket non-null AND both projection paths — should have been one fix + one review). Symptom you're in this trap: 2+ review rounds where each new finding is in the *same* subsystem the previous fix just touched. Stop, write out the full invariant, fix all of it, review once. Sediment each face as an assertion (mechanical guard), not as another review round.

In real-world testing (53-batch run, ~95 agents), Codex review caught 15×P1 + 37×P2 findings that agent self-review and orchestrator mechanical review both missed: DTO field name mismatches, state machine gaps, tenant_id filter omissions, floating-point cents bugs, fake success toasts, and missing null handling.

Good triggers for independent review:
- 3-5 related worker branches merged into one feature package
- Shared contract / API / DB schema changes
- Payment / security / hardware / production boundary changes
- A package that will be described as one user-facing outcome

---

## Decision Brief Discipline

When a task is blocked and needs user input, do not ask with only a task ID, vague blocker, or bare URL. Produce a decision-ready brief:

- **What** changes or is blocked
- **Why** the decision is needed now (not later)
- **Evidence gathered**: what local/proxy/direct evidence already exists
- **Evidence missing**: what proof is still needed
- **Options**: the exact choices available and the tradeoff of each
- **Recommendation**: what the orchestrator thinks is best and why
- **Branch/worktree**: whether to keep, retry, or clean up later

This also applies when requesting human physical actions (device, payment, deploy). State the exact action, device/resource, what NOT to do, and what the user should reply.

---

## Orchestrator Anti-Patterns

Named common mistakes to check against during review and dispatch. These are distilled from real orchestration failures.

| ID | Anti-Pattern | What Goes Wrong |
|---|---|---|
| ORC-01 | **Shallow-slice disguise** | Renaming a shallow slice doesn't make it vertical. "Add placeholder page" → "Create initial UI surface" is still shallow |
| ORC-02 | **Evidence upgrade** | Claiming `local` test passing as `proxy` or `direct` proof. Unit test ≠ staging test ≠ production proof |
| ORC-03 | **Blind self-report trust** | Agent says "all tests pass" but no command output in the handoff. Verify claims against actual evidence |
| ORC-04 | **Slot-filling dispatch** | Available slot → grab unrelated backlog item. Breaks package lane continuity, scatters daily progress |
| ORC-05 | **Review-as-authorization** | Treating a code review as permission to merge + push + deploy + cleanup. These are separate decisions |
| ORC-06 | **Constraint amnesia** | After context compression, forgetting the original allowed/forbidden paths and dispatching out-of-scope work |
| ORC-07 | **Orchestrator writes worker code** | Dispatch fails → orchestrator implements the task itself instead of fixing the dispatch. Stay in the orchestration layer |
| ORC-08 | **Merge before review** | Merging a branch because the agent said "done" without actually inspecting the diff, boundaries, and gates |
| ORC-09 | **Silent stop** | Finishing one batch and stopping without reporting status, next candidates, or blockers. Always report when stopping |
| ORC-10 | **Idempotency blindness** | Merging cleanup/retry/event/lifecycle code without checking what happens on repeated execution |
| ORC-11 | **Phantom commit** | Assuming the agent committed because it said "done". Always `git status` the worktree first — uncommitted work is the #1 Claude Code subagent failure mode |
| ORC-12 | **Parallel name collision** | Two agents independently create the same type/enum/class name. Caught only after merge when build fails. Mitigate with scoped naming in dispatch prompts |
| ORC-13 | **Same-module parallelism** | Two new-page tasks in the same module dispatched in parallel. Both touch strings/navigation/DI → guaranteed merge conflicts costing 10-15 min each. Serialize instead |
| ORC-14 | **Stale worktree base** | Local main has unpushed merge commits. `isolation: "worktree"` creates from `origin/main`, not local main → agent's base is stale → merge produces "Already up to date" or duplicate work. Always push before dispatching |
| ORC-15 | **Review skip under pressure** | "Rush ahead without review" → 3 consecutive batches skip cross-model review → findings pile up until all work is done. Review gate in Step 7 prevents this |
| ORC-16 | **Role drift** | Orchestrator starts doing research, answering unrelated questions, or implementing code instead of orchestrating. Stay in your role: decompose, dispatch, review, merge, cleanup. Route unrelated input back to the user |
| ORC-17 | **Runbook bloat** | Every run appends a rule; nothing ever removes one. The skill grows past what fits in dispatch prompts and what the orchestrator can hold (feeds ORC-06). Prune on every sync: a gate with no evidence it caught a real bug gets demoted or deleted |
| ORC-18 | **Motion mistaken for progress** | Reporting batch/agent/commit/line counts (or token spend) as if they were outcomes. They measure activity, not delivered capability. Lead the Batch Status Report with what a user can now do; counts are secondary |

Use these IDs in review comments and rejection reasons for clarity.

---

## Batch Status Report

After completing each dispatch-review-merge cycle, output a structured status report:

```
## Batch Status

### Delivered (user-visible capability — the headline, not the line count)
- <what a human can now do that they couldn't before this batch>
  e.g. "Checkout now completes order → payment → readback end-to-end (was UI shell only)"
- If this batch delivered no new user-visible capability, say so plainly — that is a signal, not a formatting gap.

### Completed
- [task-slug]: merged (3 commits, 120 lines) — evidence: local
- [task-slug]: merged (1 commit, 45 lines) — evidence: proxy

### Rejected
- [task-slug]: forbidden path violation (touched shared proto) — branch kept for fix

### Blocked
- [task-slug]: needs payment backend credentials — owner-gated

### Repo State
- Branch: main, clean
- Pushed: yes/no
- Worktrees: only main remaining

### Next
- Next candidate: [description] (package: [current package])
- Or: stop reason — [why]
```

Keep it factual and brief. The user should be able to glance at this and know exactly where things stand.

---

## Continuous Run Discipline

When running in roadmap-driven mode or long multi-batch sessions:

**Context compression resilience**: long orchestration sessions will hit context compression. To survive it:
- Keep the package ledger (milestone, active workers, merge order, blocked evidence) in a progress doc or IMPL.md, not only in chat
- The Batch Status Report at each cycle serves as a checkpoint — if context is compressed, the last report is the recovery point
- After compression, re-read repo state (Step 1) and progress docs before dispatching — do not rely on compressed memory for allowed/forbidden paths or package decisions

**Phase transitions**: when a phase changes (e.g. from many small evidence closures to a larger feature module), consider starting a fresh orchestrator session rather than stretching the current one indefinitely. Treat repository docs and merged commits as the handoff surface, not compressed chat history. Use `/handoff` to create a durable handoff document if needed.

**Progress coherence**: in continuous runs, optimize for a coherent product/module story that a human can summarize in a daily report — not for "safe and mergeable". Three related workers advancing one feature package > three unrelated workers touching three different modules.

**Role discipline**: the orchestrator is one role, not all roles. During an orchestration session:
- **Do**: decompose, dispatch, review, merge, cleanup, report status, route blockers to user
- **Don't**: answer unrelated questions, do research tangents, implement code yourself (ORC-07/ORC-16), handle personal/work inbox items that aren't part of the current feature package
- **P1 hotfix exception**: P1 fix that is ≤5 lines, single file, does not change public interfaces, may be fixed directly by the orchestrator without dispatching an agent. Anything larger must be dispatched. Record the fix in the batch status report with "orchestrator hotfix" label.
If the user asks something outside the orchestration scope, acknowledge it and suggest handling it in a separate session — don't let the orchestrator context get polluted with unrelated work.

**Local knowledge files**: for long-running or multi-session orchestration, maintain two lightweight files in the project:
- **`concepts.md`** (or `GLOSSARY.md`): stable project terms, rules, prior decisions, historical pitfalls, and blocked concepts. Read this at Step 1 of every batch to prevent constraint amnesia. Update it when orchestration reveals a new rule or decision.
- **`inbox.md`**: untriaged issues, user feedback, external review findings (from Codex/Pi/human), run observations, and pulse outputs. Items here are not yet tasks — they're intake. During Step 2 Decompose, check if any inbox item should influence the current batch. After an item becomes a task or is dismissed, remove it from inbox.

These files survive context compression and session handoffs. They are local/static coordination state, not direct proof.

---

## Using /loop With This Skill

`/loop` is not needed for the core dispatch-review-merge cycle (agents are synchronous). Use `/loop` only when:

- A dispatched agent kicked off a long-running background process (build, soak test, benchmark) and you want to monitor its output
- You want periodic progress reports while agents are running in a `Workflow`
- You're waiting for an external event (CI, deploy) before the next batch

For the main orchestration loop, just iterate: dispatch → wait for results → review → merge → next batch.
