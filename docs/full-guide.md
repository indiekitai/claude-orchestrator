# claude-orchestrator Full Guide

The detailed reference for the claude-orchestrator skill. For a quick overview, see [README.md](../README.md).

## Key Features

| Feature | Description |
|---------|-------------|
| **Serial → Parallel** | Shared contracts (proto, migrations, APIs) are built first, then consumers fan out in parallel |
| **Worktree Isolation** | Each agent works in its own git worktree — no merge conflicts during development |
| **Anti-Shallow-Slice Gate** | Rejects placeholder pages, read-only shells, and duplicate first-slices. Every dispatched task must be a `vertical-completion`, `runtime-proof`, `blocked-removal`, or `owner-gated` |
| **Bounded Task Contracts** | Each agent gets explicit allowed/forbidden paths, gates (test/typecheck/build), and evidence requirements |
| **4-Level Evidence Discipline** | Proof is labeled `direct`, `proxy`, `local`, or `blocked` — no upgrading between levels |
| **Claim Verification** | Agent self-reports are verified against actual command output and diff — not blindly trusted |
| **Quality Gates** | Every branch is reviewed for boundary violations, gate results, evidence strength, and idempotency before merge |
| **Post-Merge Integration Test** | After merging a batch, cross-layer tests run to catch issues that individual gates miss |
| **Same-Module Serialize** | Tasks in the same module default to serial dispatch — parallel only across different modules (43% conflict rate → 0%) |
| **State Transition Verification** | Dispatch prompts require agents to verify ALL consumers handle ALL state transitions |
| **Package Lane Guard** | Prevents slot-filling with unrelated backlog items — keeps work focused on one feature package |
| **Review Gate** | Cross-model review must be launched before dispatching the next batch. Skipping 2 consecutive reviews stops orchestration |
| **P2 Finding Tracker** | Unfixed P2 findings escalate to P1 after 3 batches |
| **Decision Brief Discipline** | When blocked, produces structured consultation briefs — not bare blocker text |
| **Anti-Pattern Checklist** | 16 named orchestration mistakes (ORC-01 through ORC-16) for systematic review |
| **Roadmap-Driven Mode** | Continuously build features from a roadmap doc — done → pick next → continue, until stop condition |

## Execution Model

Claude Code agents are **synchronous** — they run within the current session and return results. This is not fire-and-forget.

```
Dispatch → Agents run concurrently → All return → Review → Merge → Cross-model review → Next batch
```

Each agent call uses `isolation: "worktree"`, which gives it a dedicated git worktree. The orchestrator waits for all agents in a batch to complete, then reviews and merges before dispatching the next batch.

## Real Example

A real-world feature requiring **4 agents across 3 subsystems**, producing **1,400+ lines of production code** in one orchestration cycle:

```
Phase 1 — Serial (shared contract)
┌──────────────────────────────────┐
│  Agent 0: Proto Contract         │
│  Define shared types & interfaces│
│  ✓ merged to main               │
│  ✓ pushed (worktree base trap!)  │
└──────────────┬───────────────────┘
               │
Phase 2 — Parallel fan-out (3 consumers)
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐┌────────┐┌────────┐
│Agent 1 ││Agent 2 ││Agent 3 │
│Subsys A││Subsys B││Subsys C│
│        ││        ││        │
│✓ review││✓ review││✓ review│
│✓ merge ││✓ merge ││✓ merge │
└────────┘└────────┘└────────┘
               │
Phase 3 — Post-merge
┌──────────────────────────────────┐
│  Integration test across all     │
│  subsystems                      │
│  Independent cross-agent review  │
│  → caught 2 critical bugs       │
└──────────────────────────────────┘

Result: 4 agents, 1,400+ lines, one cycle
```

A larger overnight run: **53 batches, ~95 agents, ~70,000 lines** in 32 hours. Cross-model review (Codex) caught **15×P1 + 37×P2** findings that agent self-review and orchestrator mechanical review both missed.

## Lessons Learned (from production use)

| Trap | What happened | Rule added |
|------|---------------|------------|
| **Worktree base trap** | Parallel agents' worktrees are created from `origin/main`, not local `main`. Agents didn't see unpshed serial changes | **Push after serial batch merge, before dispatching parallel batch** |
| **Agent self-review misses cross-layer bugs** | 3 agents all self-reviewed, but independent review found 2 critical state transition bugs | **Independent cross-agent review required for state machine / cross-layer features** |
| **Out-of-bounds agent can't merge** | An agent modified forbidden paths due to stale base | **Use `cherry-pick --no-commit` to extract only in-scope changes** |
| **Individual gates pass but integration fails** | Each agent's tests passed, but combined code had cross-layer gaps | **Post-merge integration test after every batch** |
| **Blind self-report trust** | Agent claimed "all tests pass" but no command output was provided | **Claim verification: verify self-reported completion against actual evidence** |
| **Slot-filling scatters progress** | Available slot → grabbed unrelated backlog item → scattered daily progress | **Package lane guard: stay in current feature package, don't scatter** |
| **Evidence upgrade** | Local unit test labeled as `proxy` → over time treated as `direct` | **4-level evidence discipline with strict no-upgrade rule** |
| **Contract-only branch unmergeable** | Proto-only branch couldn't merge because sync check required consumer updates | **Contract sync gate: include minimal consumer updates in serial task** |
| **Same-module parallelism** | 2 agents in same app, both touched strings/navigation/DI. 43% batches had merge conflicts | **Serialize same-module tasks. Parallel only across different modules** |
| **Phantom commit** | Agent reported "done" but never ran `git commit`. Merge got "Already up to date" | **Commit enforcement in dispatch prompt (#1 failure mode, 40%→100% fix rate)** |
| **Codex review catches what orchestrator can't** | 15×P1 + 37×P2 found by cross-model review. Zero by agent self-review or orchestrator diff review | **Cross-model review is mandatory. Budget 600s timeout for large diffs** |

## vs Other Approaches

| | build-orchestrator | Single agent | Manual multi-agent | CI/CD workflow |
|---|---|---|---|---|
| **Parallelism** | Serial deps → parallel fan-out | Sequential phases | Ad-hoc, hope for the best | Pipeline stages |
| **Isolation** | Git worktrees per agent | Single worktree | Shared worktree (conflict risk) | Separate repos/branches |
| **Quality gates** | Per-branch review + evidence + integration test | Self-review only | Manual review | Automated tests only |
| **Anti-shallow-slice** | Built-in classification gate | N/A | N/A | N/A |
| **Shared contracts** | Serialized first, then fan-out | N/A | Manual coordination | N/A |
| **Cross-model review** | Mandatory after each batch | N/A | Manual | N/A |
| **Continuous mode** | Roadmap-driven auto-selection | N/A | N/A | N/A |
| **Best for** | Width (many tasks) | Depth (one complex task) | Simple changes | Post-merge validation |

## Concurrency Rules

| Agents | When |
|--------|------|
| **1** (serialize) | Same module (new pages sharing navigation/strings/DI), shared contract being edited, hardware/payment involved, next step depends on current result |
| **2** (default) | Different modules with disjoint write sets |
| **3** (max) | All clean: no shared contracts active, main is clean, fully disjoint modules, no hardware/payment tasks |

In practice, same-module parallelism almost never saves time — merge conflict resolution (10-15 min per batch) exceeds the wall-clock gain.
