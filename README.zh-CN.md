[English](README.md) | [中文](README.zh-CN.md)

# claude-orchestrator

**多个 AI Agent 并行构建你的功能，你只负责 review。** 一个 Claude Code skill，将大型功能拆解为有边界的任务契约，派发到隔离的 worktree agent 并行执行，强制质量门禁，最后合并结果——一个编排周期搞定。

## 🤔 痛点

用 Claude Code 构建跨多个子系统的功能，通常意味着：

- 一次跑一个 agent，等它跑完 review，再启动下一个
- 手动记录每个 agent 能碰哪些文件
- 祈祷两个 agent 不会同时改同一个共享契约
- 没有系统化的方式保证"先做共享契约，再做消费者"
- 一堆看起来完成了但实际没有端到端贯通的浅切片

你变成了一个临时项目经理，在脑子里 juggle agent 的输出。

## 🔧 工作原理

```
┌─────────────────────────────────────────────────────┐
│                     编排器                           │
│                                                     │
│  1. 侦察         — 读取仓库状态                      │
│  2. 拆解         — 识别任务契约                      │
│  3. 反浅切片门禁  — 拒绝占位式切片                    │
│         │                                           │
│         ▼                                           │
│  4. 串行执行共享契约（如有）                          │
│         │                                           │
│         ▼                                           │
│  5. 并行扇出  ───┬── Agent A（独立 worktree）        │
│                  ├── Agent B（独立 worktree）        │
│                  └── Agent C（独立 worktree）        │
│         │                                           │
│         ▼                                           │
│  6. 逐个 review（边界 + 门禁 + 证据）                │
│         │                                           │
│         ▼                                           │
│  7. 合并 + 集成测试                                  │
│         │                                           │
│         ▼                                           │
│  8. 下一批（或停止）                                 │
└─────────────────────────────────────────────────────┘
```

## ✨ 核心能力

| 能力 | 说明 |
|------|------|
| **串行→并行** | 共享契约（proto、迁移、API 定义）先串行完成，消费者再并行扇出 |
| **Worktree 隔离** | 每个 agent 在独立的 git worktree 中工作，开发期间零冲突 |
| **反浅切片门禁** | 拒绝占位页面、只读 shell、重复的首切片。每个任务必须是 `vertical-completion`、`runtime-proof`、`blocked-removal` 或 `owner-gated` |
| **有边界的任务契约** | 每个 agent 收到明确的允许/禁止路径、门禁（测试/类型检查/构建）和证据要求 |
| **四级证据纪律** | 证据标记为 `direct`（直接）、`proxy`（间接）、`local`（本地）或 `blocked`（阻塞）——不允许跨级升级 |
| **声明验证** | Agent 自报的"测试通过"、"没碰禁止路径"必须有实际命令输出或 diff 佐证——不盲信 |
| **质量门禁** | 每个分支合并前都要 review：边界违规、门禁结果、证据强度、幂等性 |
| **合并后集成测试** | 每批分支合并后跑跨层集成测试，各层单独通过不代表组合能工作 |
| **并发控制** | 智能限制：默认 2 个 agent，安全时最多 3 个，共享契约活跃时降为 1 个 |
| **状态转换验证** | Dispatch prompt 要求 agent 验证所有消费者处理所有状态转换，不只是自己这层的 happy path |
| **功能包主线守卫** | 阻止用无关 backlog 任务填空槽——让工作聚焦在同一个功能包上 |
| **决策咨询包纪律** | 阻塞时生成结构化咨询包（证据、选项、建议），不是丢裸 blocker 文本 |
| **反模式检查表** | 10 个命名编排错误（ORC-01 到 ORC-10）：证据升级、填槽派活、约束遗忘等 |
| **Roadmap 连续模式** | 从路线图文档中连续构建功能——做完一个 → 自动选下一个 → 继续 |

## 🚀 快速开始

### 1. 安装 skill

```bash
# 复制到 Claude Code skills 目录
cp -r . ~/.claude/skills/build-orchestrator/

# 或者用符号链接
ln -s "$(pwd)" ~/.claude/skills/build-orchestrator
```

### 2. 使用

**单功能模式**（做完一个就停）：

```
/build-orchestrator <描述要构建的功能或里程碑>
```

**Roadmap 连续模式**（从路线图中连续推进）：

```
/build-orchestrator --roadmap docs/roadmap.md
```

或者在对话中自然触发：

> "这个功能涉及 3 个子系统，拆开做，并行推"

> "dispatch parallel agents for the payment integration"

## 📦 实战案例

一个真实功能：**4 个 agent 横跨 3 个子系统**，一个编排周期产出 **1,400+ 行生产代码**：

```
阶段 1 — 串行（共享契约）
┌──────────────────────────────────┐
│  Agent 0: 共享契约               │
│  定义共享类型和接口               │
│  ✓ 合并到 main                   │
│  ✓ push（worktree 基线陷阱！）   │
└──────────────┬───────────────────┘
               │
阶段 2 — 并行扇出（3 个消费者）
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐┌────────┐┌────────┐
│Agent 1 ││Agent 2 ││Agent 3 │
│子系统 A││子系统 B││子系统 C│
│        ││        ││        │
│✓ review││✓ review││✓ review│
│✓ 合并  ││✓ 合并  ││✓ 合并  │
└────────┘└────────┘└────────┘
               │
阶段 3 — 合并后
┌──────────────────────────────────┐
│  跨子系统集成测试                 │
│  独立异 agent review              │
│  → 抓到 2 个 critical bug        │
└──────────────────────────────────┘

结果：4 个 agent，1,400+ 行代码，一个周期
```

编排器处理了整个依赖图：共享契约必须先落地（串行）**且 push**（否则并行 agent 的 worktree 看不到），然后三个消费者 agent 同时运行（并行），各自在独立 worktree 中工作。全部返回后，逐个 review，合并，跑跨层集成测试验证组合结果。

## ⚠️ 实战踩坑（已写入 skill 规则）

| 坑 | 发生了什么 | 加的规则 |
|-----|-----------|---------|
| **Worktree 基线陷阱** | 并行 agent 的 worktree 基于 `origin/main` 创建，不是本地 `main`。串行 batch merge 到本地但没 push，并行 agent 看不到串行改动，重做了一遍 | **串行 batch merge 后先 push，再 dispatch 并行 batch** |
| **自审抓不住跨层 bug** | 3 个 agent 都做了 self-review，但独立 review 发现了 2 个 critical 的跨层状态转换 bug | **状态机/跨层功能合并后必须跑独立 review** |
| **越界 agent 无法直接 merge** | agent 因为 stale base 改了禁区文件，`git merge` 会把越界改动一起带进来 | **用 `cherry-pick --no-commit` 只择取合规改动** |
| **各层单独通过但组合有 bug** | 每个 agent 的测试都过了，但合并后跨层有缺口 | **每批合并后跑跨层集成测试** |
| **盲信自报完成** | Agent 在 handoff 里写"所有测试通过"但没有命令输出。编排器没验证就 merge 了 | **声明验证：核对自报完成是否有实际命令输出和 diff 佐证** |
| **填槽导致进度散乱** | 有空闲并发槽 → 从全局 backlog 抓了无关任务 → 日报看起来像随机清理而不是功能推进 | **功能包主线守卫：留在当前功能包，不散射** |
| **证据升级** | 本地单元测试通过被标记为 `proxy`。随时间推移 `proxy` 被当成 `direct` 处理 | **四级证据纪律，严禁跨级升级** |
| **纯契约分支无法合并** | 派发了只改 proto 的分支，但 repo 的同步检查要求消费者在同一个 change 里更新。分支永远无法独立合并 | **契约同步门禁：串行任务要包含最小的消费者更新** |
| **同模块并行** | 2 个 agent 给同一个 Android app 加新页面。都改了 strings.xml + 导航 + DI。43% 的 batch 有合并冲突，每次 10-15 分钟——超过并行省的时间 | **同模块任务串行。只在不同模块间并行** |
| **幽灵提交** | Agent 报"完成"但没执行 `git commit`。编排器以为有 commit 直接 merge——"Already up to date" | **merge 前必 `git status`。dispatch prompt 第一条就是 commit 强制（修复率 40%→100%）** |
| **Codex review 抓到编排器抓不到的** | 4 次 Codex review 发现 10 个问题（2×P1 + 8×P2）：版本冲突、回归、日期 crash、浮点截断、假成功 toast。agent 自审和编排器 diff review 零发现 | **异模型 review 是强制的，不是可选的。大 diff 需要 600 秒 timeout** |

## ⚙️ 执行模型

Claude Code agent 是**同步**的——在当前 session 中运行并返回结果。不是 fire-and-forget。

```
派发 → Agent 并发运行 → 全部返回 → Review → 合并 → 集成测试 → 下一批
```

每个 agent 调用使用 `isolation: "worktree"`，获得独立的 git worktree。编排器等待当前批次所有 agent 完成，review 并合并后，再派发下一批。

## 📊 方案对比

| | build-orchestrator | 单 agent | 手动管多 agent | CI/CD 流水线 |
|---|---|---|---|---|
| **并行能力** | 串行依赖 → 并行扇出 | 顺序阶段 | 凭感觉 | 流水线阶段 |
| **隔离性** | 每个 agent 独立 worktree | 单 worktree | 共享 worktree（冲突风险） | 独立仓库/分支 |
| **质量门禁** | 逐分支 review + 证据 + 集成测试 | 仅自审 | 手动 review | 仅自动测试 |
| **反浅切片** | 内置分类门禁 | 无 | 无 | 无 |
| **共享契约** | 先串行再扇出 | 无 | 手动协调 | 无 |
| **连续模式** | Roadmap 自动选择 | 无 | 无 | 无 |
| **适用场景** | 宽度（多任务并行） | 深度（单个复杂任务） | 简单改动 | 合并后验证 |

## 🔢 并发规则

| Agent 数 | 适用场景 |
|----------|---------|
| **1**（串行） | 共享契约正在编辑、涉及硬件/支付、下一步依赖当前结果、写入集有重叠 |
| **2**（默认） | 标准并行，模块互不相交 |
| **3**（上限） | 全部安全：无活跃共享契约、main 干净、写入集完全不相交、无硬件/支付任务 |

目标是减少日历时间，不增加合并风险。能开更多 agent 不代表应该开。

## 🔗 相关项目

- **[codex-orchestrator](https://github.com/indiekitai/codex-orchestrator)** — 面向 Codex App 用户的姐妹项目。相同的编排哲学（有界契约、证据纪律、反浅切片），但适配了 Codex App 的异步 session 模型，附带 Go CLI helper 提供持久化 ledger、heartbeat、routine 和 policy/eval 能力。

## 📄 许可证

MIT — 见 [LICENSE](LICENSE)

---

由 [IndieKit.ai](https://indiekit.ai) 构建 — 面向 AI 原生工作流的开源开发者工具。
