---
name: autonomous-loops
description: "Patterns and architectures for autonomous Claude Code loops — from simple sequential pipelines to RFC-driven multi-agent DAG systems."
origin: ECC
---

# Autonomous Loops Skill

> 兼容性说明（v1.8.0）：`autonomous-loops` 保留一个版本。
> 当前的主要技能名称是 `continuous-agent-loop`。新循环指导应在此处编写，
> 而本技能保持可用以避免破坏现有工作流。

模式、架构和参考实现，用于自主运行 Claude Code。涵盖从简单的 `claude -p` 管道到完整的 RFC 驱动多智能体 DAG 编排。

## 何时使用

- 设置无需人工干预的自主开发工作流
- 为你的问题选择合适的循环架构（简单 vs 复杂）
- 构建 CI/CD 风格的持续开发管道
- 运行具有合并协调的并行 Agent
- 在循环迭代间实现上下文持久性
- 向自主工作流添加质量门和清理通道

## 循环模式谱系

从最简单到最复杂：

| 模式 | 复杂度 | 适合场景 |
|------|--------|----------|
| [顺序管道](#1-顺序管道-claude--p) | 低 | 每日开发步骤、脚本化工作流 |
| [NanoClaw REPL](#2-nanoclaw-repl) | 低 | 交互式持久会话 |
| [无限智能体循环](#3-无限智能体循环) | 中 | 并行内容生成、规范驱动的工作 |
| [Continuous Claude PR 循环](#4-continuous-claude-pr-loop) | 中 | 具有 CI 门的多日迭代项目 |
| [De-Sloppify 模式](#5-the-de-sloppify-pattern) | 附加项 | 任何 Implementer 步骤后的质量清理 |
| [Ralphinho / RFC 驱动 DAG 编排](#6-ralphinho--rfc-driven-dag-orchestration) | 高 | 大型功能、带合并队列的多单元并行工作 |

---

## 1. 顺序管道（`claude -p`）

**最简单的循环。** 将每日开发分解为一系列非交互式 `claude -p` 调用。每个调用是一个带有明确提示的重点步骤。

### 核心洞察

> 如果你连这样的循环都想不出来，说明你甚至无法在交互模式下让 LLM 修复你的代码。

`claude -p` 标志以非交互方式运行 Claude Code，并附带一个提示词，完成后退出。链式调用构建管道：

```bash
#!/bin/bash
# daily-dev.sh — 功能分支的顺序管道

set -e

# 步骤 1：实现功能
claude -p "Read the spec in docs/auth-spec.md. Implement OAuth2 login in src/auth/. Write tests first (TDD). Do NOT create any new documentation files."

# 步骤 2：De-sloppify（清理通道）
claude -p "Review all files changed by the previous commit. Remove any unnecessary type tests, overly defensive checks, or testing of language features (e.g., testing that TypeScript generics work). Keep real business logic tests. Run the test suite after cleanup."

# 步骤 3：验证
claude -p "Run the full build, lint, type check, and test suite. Fix any failures. Do not add new features."

# 步骤 4：提交
claude -p "Create a conventional commit for all staged changes. Use 'feat: add OAuth2 login flow' as the message."
```

### 关键设计原则

1. **每个步骤是隔离的** — 每次 `claude -p` 调用的新上下文窗口意味着步骤之间没有上下文渗透。
2. **顺序重要** — 步骤顺序执行。每一个都基于前一个留下的文件系统状态进行构建。
3. **负面指令危险** — 不要说"不要测试类型系统"。而是添加单独的清理步骤（参见 [De-Sloppify 模式](#5-the-de-sloppify-pattern)）。
4. **退出码传播** — `set -e` 在失败时停止管道。

### 变体

**使用模型路由：**
```bash
# 使用 Opus 进行研究（深度推理）
claude -p --model opus "Analyze the codebase architecture and write a plan for adding caching..."

# 使用 Sonnet 实现（快速、能力强）
claude -p "Implement the caching layer according to the plan in docs/caching-plan.md..."

# 使用 Opus 审查（彻底）
claude -p --model opus "Review all changes for security issues, race conditions, and edge cases..."
```

**使用环境上下文：**
```bash
# 通过文件传递上下文，而不是提示词长度
echo "Focus areas: auth module, API rate limiting" > .claude-context.md
claude -p "Read .claude-context.md for priorities. Work through them in order."
rm .claude-context.md
```

**使用 `--allowedTools` 限制：**
```bash
# 只读分析通道
claude -p --allowedTools "Read,Grep,Glob" "Audit this codebase for security vulnerabilities..."

# 仅写入实现通道
claude -p --allowedTools "Read,Write,Edit,Bash" "Implement the fixes from security-audit.md..."
```

---

## 2. NanoClaw REPL

**ECC 内置的持久化循环。** 一个会话感知的 REPL，使用完整对话历史同步调用 `claude -p`。

```bash
# 启动默认会话
node scripts/claw.js

# 带技能上下文的命名会话
CLAW_SESSION=my-project CLAW_SKILLS=tdd-workflow,security-review node scripts/claw.js
```

### 工作原理

1. 从 `~/.claude/claw/{session}.md` 加载对话历史
2. 每条用户消息与完整历史一起发送到 `claude -p`
3. 响应追加到会话文件（Markdown 作为数据库）
4. 会话在重启后保持

### NanoClaw vs 顺序管道

| 使用场景 | NanoClaw | 顺序管道 |
|----------|----------|----------|
| 交互式探索 | 是 | 否 |
| 脚本自动化 | 否 | 是 |
| 会话持久性 | 内置 | 手动 |
| 上下文累积 | 每轮增长 | 每步重新开始 |
| CI/CD 集成 | 差 | 优秀 |

有关完整详细信息，请参阅 `/claw` 命令文档。

---

## 3. 无限智能体循环

**双提示系统。** 协调并行子智能体以规范驱动生成。由 disler 开发（归功于：@disler）。

### 架构：双提示系统

```
PROMPT 1（编排器）              PROMPT 2（子智能体）
┌─────────────────────┐             ┌──────────────────────┐
│ 解析规范文件         │             │ 接收完整上下文        │
│ 扫描输出目录        │  部署      │ 读取分配的数字        │
│ 规划迭代            │────────────│ 严格按照规范          │
│ 分配创意目录        │  N 个智能体 │ 生成独特输出          │
│ 管理波次            │             │ 保存到输出目录       │
└─────────────────────┘             └──────────────────────┘
```

### 模式

1. **规范分析** — 编排器读取规范文件（Markdown）定义生成内容
2. **目录侦察** — 扫描现有输出以找到最高迭代编号
3. **并行部署** — 启动 N 个子智能体，每个获得：
   - 完整规范
   - 唯一的创意方向
   - 特定的迭代编号（无冲突）
   - 现有迭代的快照（用于保持唯一性）
4. **波次管理** — 对于无限模式，以 3-5 个为一组部署，直到上下文耗尽

### 通过 Claude Code 命令实现

创建 `.claude/commands/infinite.md`：

```markdown
从 $ARGUMENTS 解析以下参数：
1. spec_file — 规范 markdown 文件路径
2. output_dir — 迭代保存位置
3. count — 整数 1-N 或 "infinite"

阶段 1：深入阅读并理解规范。
阶段 2：列出 output_dir，找到最高迭代编号。从 N+1 开始。
阶段 3：规划创意方向——每个智能体获得不同的主题/方法。
阶段 4：并行部署子智能体（Task 工具）。每个接收：
  - 完整规范文本
  - 当前目录快照
  - 分配的迭代编号
  - 独特的创意方向
阶段 5（无限模式）：以 3-5 个为一组循环，直到上下文不足。
```

**调用：**
```bash
/project:infinite specs/component-spec.md src/ 5
/project:infinite specs/component-spec.md src/ infinite
```

### 批处理策略

| 数量 | 策略 |
|------|------|
| 1-5 | 所有智能体同时 |
| 6-20 | 每批 5 个 |
| infinite | 每波 3-5 个，逐步复杂化 |

### 关键洞察：通过分配实现唯一性

不要依赖智能体自行区分。编排器**分配**每个智能体特定的创意方向和迭代编号。这防止并行智能体之间出现重复概念。

---

## 4. Continuous Claude PR 循环

**生产级 shell 脚本。** 在连续循环中运行 Claude Code，创建 PR、等待 CI、自动合并。由 AnandChowdhary 创建（归功于：@AnandChowdhary）。

### 核心循环

```
┌─────────────────────────────────────────────────────┐
│  CONTINUOUS CLAUDE ITERATION                         │
│                                                     │
│  1. Create branch (continuous-claude/iteration-N)   │
│  2. Run claude -p with enhanced prompt              │
│  3. (Optional) Reviewer pass — separate claude -p   │
│  4. Commit changes (claude generates message)       │
│  5. Push + create PR (gh pr create)                 │
│  6. Wait for CI checks (poll gh pr checks)          │
│  7. CI failure? → Auto-fix pass (claude -p)         │
│  8. Merge PR (squash/merge/rebases)                 │
│  9. Return to main → repeat                         │
│                                                     │
│  Limit by: --max-runs N | --max-cost $X             │
│            --max-duration 2h | completion signal     │
└─────────────────────────────────────────────────────┘
```

### 安装

> **警告：** 审查代码后从其仓库安装 continuous-claude。不要直接将外部脚本管道到 bash。

### 用法

```bash
# 基础：10 次迭代
continuous-claude --prompt "Add unit tests for all untested functions" --max-runs 10

# 成本限制
continuous-claude --prompt "Fix all linter errors" --max-cost 5.00

# 时间限制
continuous-claude --prompt "Improve test coverage" --max-duration 8h

# 带代码审查通道
continuous-claude \
  --prompt "Add authentication feature" \
  --max-runs 10 \
  --review-prompt "Run npm test && npm run lint, fix any failures"

# 通过 worktrees 并行
continuous-claude --prompt "Add tests" --max-runs 5 --worktree tests-worker &
continuous-claude --prompt "Refactor code" --max-runs 5 --worktree refactor-worker &
wait
```

### 跨迭代上下文：SHARED_TASK_NOTES.md

关键创新：`SHARED_TASK_NOTES.md` 文件跨迭代持久化：

```markdown
## Progress
- [x] Added tests for auth module (iteration 1)
- [x] Fixed edge case in token refresh (iteration 2)
- [ ] Still need: rate limiting tests, error boundary tests

## Next Steps
- Focus on rate limiting module next
- The mock setup in tests/helpers.ts can be reused
```

Claude 在迭代开始时读取此文件，在迭代结束时更新它。这桥接了独立 `claude -p` 调用之间的上下文差距。

### CI 失败恢复

当 PR 检查失败时，Continuous Claude 自动：
1. 通过 `gh run list` 获取失败运行 ID
2. 生成新的 `claude -p` 并附带 CI 修复上下文
3. Claude 通过 `gh run view` 检查日志、修复代码、提交、推送
4. 重新等待检查（最多 `--ci-retry-max` 次尝试）

### 完成信号

Claude 可以通过输出特定短语来发出"我完成了"信号：

```bash
continuous-claude \
  --prompt "Fix all bugs in the issue tracker" \
  --completion-signal "CONTINUOUS_CLAUDE_PROJECT_COMPLETE" \
  --completion-threshold 3  # 连续 3 次信号后停止
```

连续三次迭代发出完成信号会停止循环，防止已完成工作上的浪费。

### 关键配置

| 标志 | 用途 |
|------|------|
| `--max-runs N` | N 次成功迭代后停止 |
| `--max-cost $X` | 花费 $X 后停止 |
| `--max-duration 2h` | 经过时间后停止 |
| `--merge-strategy squash` | squash、merge 或 rebase |
| `--worktree <name>` | 通过 git worktrees 实现并行 |
| `--disable-commits` | 试运行模式（无 git 操作） |
| `--review-prompt "..."` | 每次迭代添加审查通道 |
| `--ci-retry-max N` | 自动修复 CI 失败（默认：1） |

---

## 5. De-Sloppify 模式

**任意循环的附加模式。** 在每个 Implementer 步骤后添加专门的清理/重构通道。

### 问题

当要求 LLM 使用 TDD 实现时，它把"写测试"理解得太字面：
- 测试 TypeScript 类型系统工作的测试（测试 `typeof x === 'string'`）
- 类型系统已保证的过度防御性运行时检查
- 测试框架行为而非业务逻辑的测试
- 混淆实际代码的过度错误处理

### 为什么不用负面指令？

在 Implementer 提示中添加"不要测试类型系统"或"不要添加不必要的检查"有下游影响：
- 模型对所有测试变得犹豫
- 跳过合法的边缘情况测试
- 质量不可预测地下降

### 解决方案：单独通道

与其限制 Implementer，让其保持 Thorough。然后添加专门的清理 Agent：

```bash
# 步骤 1：实现（让其彻底）
claude -p "Implement the feature with full TDD. Be thorough with tests."

# 步骤 2：De-sloppify（单独上下文，重点清理）
claude -p "Review all changes in the working tree. Remove:
- Tests that verify language/framework behavior rather than business logic
- Redundant type checks that the type system already enforces
- Over-defensive error handling for impossible states
- Console.log statements
- Commented-out code

Keep all business logic tests. Run the test suite after cleanup to ensure nothing breaks."
```

### 在循环上下文中

```bash
for feature in "${features[@]}"; do
  # 实现
  claude -p "Implement $feature with TDD."

  # De-sloppify
  claude -p "Cleanup pass: review changes, remove test/code slop, run tests."

  # 验证
  claude -p "Run build + lint + tests. Fix any failures."

  # 提交
  claude -p "Commit with message: feat: add $feature"
done
```

### 关键洞察

> 不要添加具有下游质量影响的负面指令，而是添加单独的 de-sloppify 通道。两个专注的 Agent 比一个受限的 Agent 更好。

---

## 6. Ralphinho / RFC 驱动 DAG 编排

**最复杂的模式。** RFC 驱动的多智能体管道，将规范分解为依赖 DAG，每个单元通过分级质量管道运行，并通过 Agent 驱动的合并队列落地。由 enitrat 创建（归功于：@enitrat）。

### 架构概览

```
RFC/PRD 文档
       │
       ▼
  分解（AI）
   将 RFC 分解为具有依赖 DAG 的工作单元
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  RALPH 循环（最多 3 轮）                               │
│                                                      │
│  对于每个 DAG 层（顺序，按依赖）：                      │
│                                                      │
│  ┌── 质量管道（每个单元并行） ───────────┐            │
│  │  每个单元在自己的 worktree 中：       │            │
│  │  研究 → 计划 → 实现 → 测试 → 审查     │            │
│  │  （深度随复杂度层级变化）              │            │
│  └─────────────────────────────────────┘            │
│                                                      │
│  ┌── 合并队列 ─────────────────────────────┐         │
│  │  变基到 main → 运行测试 → 落地或驱逐    │         │
│  │  驱逐的单元以冲突上下文重新进入          │         │
│  └─────────────────────────────────────────┘         │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### RFC 分解

AI 读取 RFC 并生成工作单元：

```typescript
interface WorkUnit {
  id: string;              // kebab-case 标识符
  name: string;            // 人类可读名称
  rfcSections: string[];   // 此单元处理的 RFC 部分
  description: string;     // 详细描述
  deps: string[];          // 依赖（其他单元 ID）
  acceptance: string[];    // 具体的验收标准
  tier: "trivial" | "small" | "medium" | "large";
}
```

**分解规则：**
- 优先选择更少的、内聚的单元（最小化合并风险）
- 最小化跨单元文件重叠（避免冲突）
- 测试与实现一起（绝不分离"实现 X"+"测试 X"）
- 仅在实际代码依赖存在时有依赖

依赖 DAG 决定执行顺序：
```
Layer 0: [unit-a, unit-b]     ← 无依赖，并行运行
Layer 1: [unit-c]             ← 依赖于 unit-a
Layer 2: [unit-d, unit-e]     ← 依赖于 unit-c
```

### 复杂度层级

不同层级获得不同的管道深度：

| 层级 | 管道阶段 |
|------|----------|
| **trivial** | 实现 → 测试 |
| **small** | 实现 → 测试 → 代码审查 |
| **medium** | 研究 → 计划 → 实现 → 测试 → PRD 审查 + 代码审查 → 审查修复 |
| **large** | 研究 → 计划 → 实现 → 测试 → PRD 审查 + 代码审查 → 审查修复 → 最终审查 |

这防止对简单更改进行昂贵操作，同时确保架构变更获得彻底审查。

### 单独上下文窗口（消除作者偏见）

每个阶段在独立的 Agent 进程中运行，带有自己的上下文窗口：

| 阶段 | 模型 | 用途 |
|------|------|------|
| 研究 | Sonnet | 读取代码库 + RFC，生成上下文文档 |
| 计划 | Opus | 设计实现步骤 |
| 实现 | Codex | 根据计划编写代码 |
| 测试 | Sonnet | 运行构建 + 测试套件 |
| PRD 审查 | Sonnet | 规范符合性检查 |
| 代码审查 | Opus | 质量 + 安全检查 |
| 审查修复 | Codex | 解决审查问题 |
| 最终审查 | Opus | 质量门（仅大型层级） |

**关键设计：** 审查者从未编写其审查的代码。这消除了作者偏见——自审中常见问题的最大来源。

### 带驱逐的合并队列

质量管道完成后，单元进入合并队列：

```
单元分支
    │
    ├─ 变基到 main
    │   └─ 冲突？ → 驱逐（捕获冲突上下文）
    │
    ├─ 运行构建 + 测试
    │   └─ 失败？ → 驱逐（捕获测试输出）
    │
    └─ 通过 → 快速前进 main，推送，删除分支
```

**文件重叠智能：**
- 非重叠单元可并行投机落地
- 重叠单元依次落地，每次重新变基

**驱逐恢复：**
当被驱逐时，捕获完整上下文（冲突文件、差异、测试输出）并在下次 Ralph 轮次反馈给实现者：

```markdown
## 合并冲突 — 下次落地前解决

你之前的实现与先落地的另一个单元冲突。
重构你的更改以避免以下冲突的文件/行。

{包含差异的完整驱逐上下文}
```

### 阶段之间的数据流

```
research.contextFilePath ──────────────────→ plan
plan.implementationSteps ──────────────────→ implement
implement.{filesCreated, whatWasDone} ─────→ test, reviews
test.failingSummary ───────────────────────→ reviews, implement（下一轮）
reviews.{feedback, issues} ────────────────→ review-fix → implement（下一轮）
final-review.reasoning ────────────────────→ implement（下一轮）
evictionContext ───────────────────────────→ implement（合并冲突后）
```

### Worktree 隔离

每个单元在隔离的 worktree 中运行（使用 jj/Jujutsu，而非 git）：
```
/tmp/workflow-wt-{unit-id}/
```

同一单元的管道阶段**共享**一个 worktree，跨 研究 → 计划 → 实现 → 测试 → 审查 保持状态（上下文文件、计划文件、代码更改）。

### 关键设计原则

1. **确定性执行** — 预先分解锁定并行性和排序
2. **杠杆点的人为审查** — 工作计划是单一最高杠杆干预点
3. **关注点分离** — 每个阶段在单独的上下文窗口中，由单独的 Agent 执行
4. **带上下文的冲突恢复** — 完整驱逐上下文支持智能重运行，而非盲目重试
5. **层级驱动深度** — 琐碎更改跳过研究/审查；大型更改获得最大审查
6. **可恢复工作流** — 完整状态持久化到 SQLite；可从任何点恢复

### 何时使用 Ralphinho 对比更简单模式

| 信号 | 使用 Ralphinho | 使用更简单模式 |
|------|----------------|----------------|
| 多个相互依赖的工作单元 | 是 | 否 |
| 需要并行实现 | 是 | 否 |
| 可能发生合并冲突 | 是 | 否（顺序即可） |
| 单文件更改 | 否 | 是（顺序管道） |
| 多日项目 | 是 | 可能（continuous-claude） |
| 已有规范/RFC | 是 | 可能 |
| 快速迭代一件事 | 否 | 是（NanoClaw 或管道） |

---

## 选择正确的模式

### 决策矩阵

```
任务是单一重点更改吗？
├─ 是 → 顺序管道或 NanoClaw
└─ 否 → 是否有书面规范/RFC？
         ├─ 是 → 需要并行实现吗？
         │        ├─ 是 → Ralphinho（DAG 编排）
         │        └─ 否 → Continuous Claude（迭代 PR 循环）
         └─ 否 → 需要同一事物的许多变体吗？
                  ├─ 是 → 无限智能体循环（规范驱动生成）
                  └─ 否 → 带 de-sloppify 的顺序管道
```

### 组合模式

这些模式可以很好地组合：

1. **顺序管道 + De-Sloppify** — 最常见组合。每个实现步骤都获得清理通道。

2. **Continuous Claude + De-Sloppify** — 为每次迭代添加带有 de-sloppify 指令的 `--review-prompt`。

3. **任意循环 + 验证** — 在提交门之前使用 ECC 的 `/verify` 命令或 `verification-loop` 技能。

4. **Ralphinho 的分层方法用于更简单循环** — 即使在顺序管道中，你也可以将简单任务路由到 Haiku，复杂任务到 Opus：
   ```bash
   # 简单格式化修复
   claude -p --model haiku "Fix the import ordering in src/utils.ts"

   # 复杂架构更改
   claude -p --model opus "Refactor the auth module to use the strategy pattern"
   ```

---

## 反模式

### 常见错误

1. **无退出条件的无限循环** — 始终有最大运行次数、最大成本、最长时间或完成信号。

2. **迭代间无上下文桥接** — 每个 `claude -p` 调用从新开始。使用 `SHARED_TASK_NOTES.md` 或文件系统状态桥接上下文。

3. **重试相同的失败** — 如果迭代失败，不要只是重试。捕获错误上下文并反馈给下一次尝试。

4. **负面指令而非清理通道** — 不要说"不要做 X"。添加单独的通道移除 X。

5. **一个上下文窗口中的所有 Agent** — 对于复杂工作流，将关注点分离到不同的 Agent 进程。审查者绝不应该是作者。

6. **并行工作中忽略文件重叠** — 如果两个并行 Agent 可能编辑同一文件，你需要合并策略（顺序落地、重新变基或冲突解决）。

---

## 参考资料

| 项目 | 作者 | 链接 |
|------|------|------|
| Ralphinho | enitrat | credit: @enitrat |
| Infinite Agentic Loop | disler | credit: @disler |
| Continuous Claude | AnandChowdhary | credit: @AnandChowdhary |
| NanoClaw | ECC | 本仓库中的 `/claw` 命令 |
| Verification Loop | ECC | 本仓库中的 `skills/verification-loop/` |
