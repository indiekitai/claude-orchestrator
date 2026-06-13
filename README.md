[English](README.md) | [中文](README.zh-CN.md)

# claude-orchestrator

**Parallel AI agents building your feature while you review.** A Claude Code skill that decomposes large features into bounded task contracts, dispatches them to isolated worktree agents in parallel, enforces quality gates, and merges the results — all in one orchestration cycle.

## 🤔 The Problem

Building a feature that spans multiple subsystems with Claude Code usually means:

- Running one agent at a time, waiting, reviewing, then starting the next
- Manually tracking which files each agent is allowed to touch
- Hoping two agents don't edit the same shared contract simultaneously
- No systematic way to enforce "shared contract first, consumers after"
- Shallow slices that look done but don't connect end-to-end

You end up as an ad-hoc project manager, juggling agent outputs in your head.

## 🔧 How It Works

```
┌─────────────────────────────────────────────────────┐
│                   ORCHESTRATOR                      │
│                                                     │
│  1. Reconnaissance   — read repo state              │
│  2. Decompose        — identify task contracts      │
│  3. Anti-shallow gate — reject placeholder slices   │
│         │                                           │
│         ▼                                           │
│  4. Serialize shared contracts (if any)             │
│         │                                           │
│         ▼                                           │
│  5. Parallel fan-out ─┬─ Agent A (worktree)         │
│                       ├─ Agent B (worktree)         │
│                       └─ Agent C (worktree)         │
│         │                                           │
│         ▼                                           │
│  6. Review each branch (boundary + gates + evidence)│
│         │                                           │
│         ▼                                           │
│  7. Merge + integration test                        │
│         │                                           │
│         ▼                                           │
│  8. Next batch (or stop)                            │
└─────────────────────────────────────────────────────┘
```

## ✨ Key Features

| Feature | Description |
|---------|-------------|
| **Serial → Parallel** | Shared contracts (proto, migrations, APIs) are built first, then consumers fan out in parallel |
| **Worktree Isolation** | Each agent works in its own git worktree — no merge conflicts during development |
| **Anti-Shallow-Slice Gate** | Rejects placeholder pages, read-only shells, and duplicate first-slices. Every dispatched task must be a `vertical-completion`, `runtime-proof`, `blocked-removal`, or `owner-gated` |
| **Bounded Task Contracts** | Each agent gets explicit allowed/forbidden paths, gates (test/typecheck/build), and evidence requirements |
| **4-Level Evidence Discipline** | Proof is labeled `direct`, `proxy`, `local`, or `blocked` — no upgrading between levels (local test ≠ staging test ≠ production proof) |
| **Claim Verification** | Agent self-reports ("tests pass", "no forbidden paths touched") are verified against actual command output and diff — not blindly trusted |
| **Quality Gates** | Every branch is reviewed for boundary violations, gate results, evidence strength, and idempotency before merge |
| **Post-Merge Integration Test** | After merging a batch, cross-layer tests run to catch issues that individual gates miss |
| **Concurrency Control** | Smart limits: 2 agents default, 3 max when safe, 1 when shared contracts are in play |
| **State Transition Verification** | Dispatch prompts require agents to verify ALL consumers handle ALL state transitions, not just their own layer |
| **Package Lane Guard** | Prevents slot-filling with unrelated backlog items — keeps work focused on one feature package at a time |
| **Decision Brief Discipline** | When blocked, produces structured consultation briefs with evidence, options, and recommendations — not bare blocker text |
| **Anti-Pattern Checklist** | 10 named orchestration mistakes (ORC-01 through ORC-10) for systematic review — evidence upgrade, slot-filling, constraint amnesia, etc. |
| **Roadmap-Driven Mode** | Continuously build features from a roadmap doc — done → pick next → continue, until stop condition |

## 🚀 Quick Start

### 1. Install the skill

```bash
# Copy to your Claude Code skills directory
cp -r . ~/.claude/skills/build-orchestrator/

# Or symlink it
ln -s "$(pwd)" ~/.claude/skills/build-orchestrator
```

### 2. Use it

**Single feature** (build one thing and stop):

```
/build-orchestrator <describe the feature or milestone to build>
```

**Roadmap-driven** (continuously build features from a priority list):

```
/build-orchestrator --roadmap docs/roadmap.md
```

Or trigger it naturally in conversation:

> "I need to build the payment integration across 3 subsystems. Dispatch parallel agents."

> "拆开做，并行推这个功能"

## 📦 Real Example

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

The orchestrator handled the entire dependency graph: the shared contract had to land first (serial) **and be pushed** (so parallel agents' worktrees would include it), then the three consumer agents ran simultaneously (parallel), each in its own worktree with disjoint file sets. After all returned, each branch was reviewed against its task contract, merged, and a cross-layer integration test validated the combined result.

## ⚠️ Lessons Learned (from production use)

These are baked into the skill, but worth knowing:

| Trap | What happened | Rule added |
|------|---------------|------------|
| **Worktree base trap** | Parallel agents' worktrees are created from `origin/main`, not local `main`. After merging a serial batch locally but not pushing, parallel agents didn't see the serial changes and redid the work | **Push after serial batch merge, before dispatching parallel batch** |
| **Agent self-review misses cross-layer bugs** | 3 agents all self-reviewed, but an independent review found 2 critical state transition bugs across layers | **Independent cross-agent review required for state machine / cross-layer features** |
| **Out-of-bounds agent can't merge** | An agent modified forbidden paths due to stale base. `git merge` would include the out-of-bounds changes | **Use `cherry-pick --no-commit` to extract only in-scope changes** |
| **Individual gates pass but integration fails** | Each agent's tests passed, but combined code had cross-layer gaps | **Post-merge integration test after every batch** |
| **Blind self-report trust** | Agent claimed "all tests pass" in handoff but no command output was provided. Orchestrator merged without verifying | **Claim verification: verify self-reported completion against actual command output and diff** |
| **Slot-filling scatters progress** | Available concurrency slot → grabbed unrelated backlog item → daily progress read as random cleanup instead of coherent feature advance | **Package lane guard: stay in current feature package, don't scatter** |
| **Evidence upgrade** | Local unit test passing was labeled as `proxy` proof in review docs. Over time, `proxy` drifted to be treated as `direct` | **4-level evidence discipline with strict no-upgrade rule** |
| **Contract-only branch unmergeable** | Dispatched a proto-only branch, but the repo's sync check required consumers to update in the same change. Branch could never merge alone | **Contract sync gate: include minimal consumer updates in serial contract task** |

## ⚙️ Execution Model

Claude Code agents are **synchronous** — they run within the current session and return results. This is not fire-and-forget.

```
Dispatch → Agents run concurrently → All return → Review → Merge → Integration test → Next batch
```

Each agent call uses `isolation: "worktree"`, which gives it a dedicated git worktree. The orchestrator waits for all agents in a batch to complete, then reviews and merges before dispatching the next batch.

## 📊 vs Other Approaches

| | build-orchestrator | Single agent | Manual multi-agent | CI/CD workflow |
|---|---|---|---|---|
| **Parallelism** | Serial deps → parallel fan-out | Sequential phases | Ad-hoc, hope for the best | Pipeline stages |
| **Isolation** | Git worktrees per agent | Single worktree | Shared worktree (conflict risk) | Separate repos/branches |
| **Quality gates** | Per-branch review + evidence + integration test | Self-review only | Manual review | Automated tests only |
| **Anti-shallow-slice** | Built-in classification gate | N/A | N/A | N/A |
| **Shared contracts** | Serialized first, then fan-out | N/A | Manual coordination | N/A |
| **Continuous mode** | Roadmap-driven auto-selection | N/A | N/A | N/A |
| **Best for** | Width (many tasks) | Depth (one complex task) | Simple changes | Post-merge validation |

## 🔢 Concurrency Rules

| Agents | When |
|--------|------|
| **1** (serialize) | Shared contract being edited, hardware/payment involved, next step depends on current result, overlapping write sets |
| **2** (default) | Standard parallel work with disjoint modules |
| **3** (max) | All clean: no shared contracts active, main is clean, fully disjoint write sets, no hardware/payment tasks |

The goal is to reduce calendar time without increasing merge risk. Don't open more agents just because you can.

## 📄 License

MIT — see [LICENSE](LICENSE)

---

Built by [IndieKit.ai](https://indiekit.ai) — open-source developer tools for the AI-native workflow.
