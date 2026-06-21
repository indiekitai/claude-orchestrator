[English](README.md) | [中文](README.zh-CN.md)

# claude-orchestrator

**Parallel AI agents building your feature while you review.** A Claude Code skill that decomposes large features into bounded task contracts, dispatches them to isolated worktree agents, enforces quality gates, and merges the results.

The point is not to let agents write forever. The point is to make every worker branch reviewable, rejectable, mergeable, and cleanable.

## Why It Exists

One Claude Code session is enough for small edits. Larger work gets messy:

- agents share the same worktree and step on each other;
- nobody tracks which files each agent is allowed to touch;
- shallow slices look done but don't connect end-to-end;
- evidence gets exaggerated — local test ≠ staging test ≠ production proof;
- the orchestrator drifts into random backlog cleanup instead of finishing one feature package.

`claude-orchestrator` is the operating discipline around that workflow.

## What It Includes

- **Claude Code skill**: installed into `~/.claude/skills/build-orchestrator/`, used as the orchestration runbook when you invoke `/build-orchestrator`.
- **Docs**: full guide, lessons learned, comparison tables, and concurrency rules.

It is not a daemon, a CLI tool, or an autonomous coding bot. Claude Code still runs the agents.

## Quick Start

### Install

```bash
# Clone and symlink
git clone https://github.com/indiekitai/claude-orchestrator.git
ln -s "$(pwd)/claude-orchestrator" ~/.claude/skills/build-orchestrator
```

### Use

**Single feature** (build one thing and stop):

```
/build-orchestrator <describe the feature or milestone to build>
```

**Roadmap-driven** (continuously build features from a priority list):

```
/build-orchestrator --roadmap docs/roadmap.md
```

Or trigger it naturally:

> "I need to build the payment integration across 3 subsystems. Dispatch parallel agents."

## How It Works

```
Plan feature package
    → Dispatch bounded workers (each in own worktree)
    → Review diff, boundaries, gates, evidence
    → Merge / push / cleanup
    → Cross-model review (Codex or Pi)
    → Continue or stop
```

The loop is intentionally conservative:

- repo/worktree truth beats agent self-reports;
- shared contracts, migrations, and APIs are serialized before parallel work;
- same-module tasks are serialized (parallel only across different modules);
- `direct`, `proxy`, `local`, and `blocked` evidence stay separate;
- cross-model review is mandatory after each batch, not optional;
- spare concurrency is not a reason to start unrelated work.

## Production Results

| Metric | Overnight Run | Full-Day Run |
|--------|--------------|--------------|
| Batches | 14 | 53 |
| Agents | 23 | ~95 |
| Code output | ~15,000 lines | ~70,000 lines |
| Commit rate | 40%→100% (after rule) | 100% |
| Merge conflicts | 43% (parallel) → 0% (serial) | 0% |
| Cross-model findings | 2×P1 + 8×P2 | 15×P1 + 37×P2 |

> Read these as "the gates hold at scale," not "more is better." Lines and agent counts measure activity, not delivered capability; a high P1 catch rate means review is working *and* that upstream design left bugs to catch. The goal is fewer bugs needing a fix, not more bugs caught late — invest in decomposition, contracts, and per-task gates so the finding count trends *down* over a run.

## Documentation

- [Full guide](docs/full-guide.md): features, execution model, real examples, lessons learned, comparison tables, and concurrency rules.

## Related Projects

- **[codex-orchestrator](https://github.com/indiekitai/codex-orchestrator)** — The sibling project for Codex App users. Same orchestration philosophy, but adapted for Codex App's async session model with a Go CLI helper for persistent ledger, heartbeat, routines, and policy/eval.

## License

MIT — see [LICENSE](LICENSE)

---

Built by [IndieKit.ai](https://indiekit.ai) — open-source developer tools for the AI-native workflow.
