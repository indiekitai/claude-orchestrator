[English](README.md) | [中文](README.zh-CN.md)

# claude-orchestrator

**并行 AI Agent 帮你写功能，你来 review。** 一个 Claude Code skill，把大功能拆成有边界的任务契约，派发到隔离 worktree 的 agent 并行执行，强制质量门禁，合并结果。

核心不是让 agent 无限写代码，而是让每个 worker 分支都可 review、可拒绝、可合并、可清理。

## 为什么需要它

一个 Claude Code session 做小改动够用。大功能就乱了：

- agent 共享同一个 worktree，互相踩脚；
- 没有人追踪每个 agent 允许改哪些文件；
- 浅切片看着完成了但端到端没打通；
- 证据被夸大——本地测试 ≠ staging 测试 ≠ 生产证明；
- 编排器跑着跑着就从功能包跑偏到随机 backlog 清理。

`claude-orchestrator` 就是这个工作流的操作纪律。

## 包含什么

- **Claude Code skill**：安装在 `~/.claude/skills/build-orchestrator/`，通过 `/build-orchestrator` 调用。
- **文档**：完整指南、踩坑记录、对比表和并发规则。

不是 daemon，不是 CLI 工具，不是自动编码机器人。Claude Code 仍然负责运行 agent。

## 快速开始

### 安装

```bash
# 克隆并符号链接
git clone https://github.com/indiekitai/claude-orchestrator.git
ln -s "$(pwd)/claude-orchestrator" ~/.claude/skills/build-orchestrator
```

### 使用

**单功能模式**（做完一个就停）：

```
/build-orchestrator <描述要做的功能或里程碑>
```

**路线图模式**（从路线图连续推进）：

```
/build-orchestrator --roadmap docs/roadmap.md
```

或者在对话中自然触发：

> "拆开做，并行推这个功能"

## 工作原理

```
规划功能包
    → 派发有边界的 worker（各自在独立 worktree）
    → 审查 diff、边界、门禁、证据
    → 合并 / 推送 / 清理
    → 异模型 review（Codex 或 Pi）
    → 继续或停止
```

循环是刻意保守的：

- repo/worktree 事实 > agent 自报；
- 共享契约、migration、API 在并行前先串行；
- 同模块任务串行（只在不同模块间并行）；
- `direct`、`proxy`、`local`、`blocked` 四级证据严禁升级；
- 异模型 review 是每批次强制的，不是可选的；
- 有空槽不代表应该抓无关任务。

## 生产数据

| 指标 | 夜跑 | 全天跑 |
|------|------|--------|
| Batch 数 | 14 | 53 |
| Agent 数 | 23 | ~95 |
| 代码产出 | ~15,000 行 | ~70,000 行 |
| Commit 率 | 40%→100%（规则加入后） | 100% |
| Merge 冲突 | 43%（并行）→ 0%（串行） | 0% |
| 异模型发现 | 2×P1 + 8×P2 | 15×P1 + 37×P2 |

## 文档

- [完整指南](docs/full-guide.md)：功能列表、执行模型、真实案例、踩坑记录、对比表和并发规则。

## 相关项目

- **[codex-orchestrator](https://github.com/indiekitai/codex-orchestrator)** — 面向 Codex App 用户的姐妹项目。相同的编排哲学，但适配了 Codex App 的异步 session 模型，附带 Go CLI helper 提供持久化 ledger、heartbeat、routine 和 policy/eval 能力。

## 许可证

MIT — 见 [LICENSE](LICENSE)

---

由 [IndieKit.ai](https://indiekit.ai) 构建 — 面向 AI 原生工作流的开源开发者工具。
