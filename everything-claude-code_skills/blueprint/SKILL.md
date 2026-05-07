---
name: blueprint
description: >-
  Turn a one-line objective into a step-by-step construction plan for
  multi-session, multi-agent engineering projects. Each step has a
  self-contained context brief so a fresh agent can execute it cold.
  Includes adversarial review gate, dependency graph, parallel step
  detection, anti-pattern catalog, and plan mutation protocol.
  TRIGGER when: user requests a plan, blueprint, or roadmap for a
  complex multi-PR task, or describes work that needs multiple sessions.
  DO NOT TRIGGER when: task is completable in a single PR or fewer
  than 3 tool calls, or user says "just do it".
origin: community
---

# Blueprint — 构建计划生成器

将一句话目标转化为任何编码 Agent 都能冷启动执行的逐步构建计划。

## 何时使用

- 将大型功能分解为多个具有清晰依赖顺序的 PR
- 规划跨越多个会话的重构或迁移
- 跨子智能体协调并行工作流
- 会话间上下文丢失会导致返工的任何任务

**不要用于**可以在单个 PR 中完成、少于 3 个工具调用或用户说"直接做"的任务。

## 工作原理

Blueprint 运行 5 阶段管道：

1. **Research** — 预检（git、gh 认证、远程、默认分支），然后读取项目结构、现有计划和内存文件以收集上下文。
2. **Design** — 将目标分解为每个一个 PR 大小的步骤（通常 3-12 步）。分配依赖边、并行/串行顺序、模型层（最强 vs 默认）以及每步的回滚策略。
3. **Draft** — 将自包含的 Markdown 计划文件写入 `plans/`。每个步骤包括上下文简介、任务列表、验证命令和退出标准——因此新 Agent 可以在不读取先前步骤的情况下执行任何步骤。
4. **Review** — 将对抗性审查委托给最强模型子 Agent（例如 Opus），对照清单和反模式目录。在最终确定前修复所有关键发现。
5. **Register** — 保存计划，更新内存索引，并向用户呈现步骤数量和并行性摘要。

Blueprint 自动检测 git/gh 可用性。使用 git + GitHub CLI，它生成完整的分支/PR/CI 工作流计划。没有它们时，它切换到直接模式（就地编辑，无分支）。

## 示例

### 基本用法

```
/blueprint myapp "migrate database to PostgreSQL"
```

产生 `plans/myapp-migrate-database-to-postgresql.md`，包含如步骤：
- 步骤 1：添加 PostgreSQL 驱动和连接配置
- 步骤 2：为每个表创建迁移脚本
- 步骤 3：更新存储库层使用新驱动
- 步骤 4：添加针对 PostgreSQL 的集成测试
- 步骤 5：删除旧数据库代码和配置

### 多智能体项目

```
/blueprint chatbot "extract LLM providers into a plugin system"
```

产生具有并行步骤的计划（例如"实现 Anthropic 插件"和"实现 OpenAI 插件"在插件接口步骤完成后并行运行）、模型层分配（接口设计步骤用最强，实现用默认）以及每次步骤后验证的不变量（例如"所有现有测试通过"、"核心中无 provider 导入"）。

## 关键特性

- **冷启动执行** — 每个步骤包含自包含的上下文简介。无需先前上下文。
- **对抗性审查门** — 每个计划由最强模型子 Agent 对照涵盖完整性、依赖正确性和反模式检测的清单进行审查。
- **分支/PR/CI 工作流** — 内置于每个步骤。当 git/gh 缺失时优雅降级到直接模式。
- **并行步骤检测** — 依赖图识别没有共享文件或输出依赖的步骤。
- **计划变更协议** — 步骤可以分割、插入、跳过、重新排序或放弃，带有正式协议和审计追踪。
- **零运行时风险** — 纯 Markdown 技能。整个仓库仅包含 `.md` 文件——无钩子、无 shell 脚本、无可执行代码、无 `package.json`、无构建步骤。在安装或调用时除了 Claude Code 的原生 Markdown 技能加载器外什么也不运行。

## 安装

本技能包含在 Everything Claude Code 中。安装 ECC 时不需要单独安装。

### 完整 ECC 安装

如果你从 ECC 仓库签出工作，验证技能是否存在：

```bash
test -f skills/blueprint/SKILL.md
```

稍后更新时，在更新前审查 ECC diff：

```bash
cd /path/to/everything-claude-code
git fetch origin main
git log --oneline HEAD..origin/main       # 在更新前审查新提交
git checkout <reviewed-full-sha>          # 固定到特定审查的提交
```

### 独立 vendored 安装

如果你仅在完整 ECC 安装之外 vendoring 此技能，将审查过的文件从 ECC 仓库复制到 `~/.claude/skills/blueprint/SKILL.md`。Vendored 副本没有 git 远程，因此通过从审查过的 ECC 提交重新复制文件而非运行 `git pull` 来更新它们。

## 要求

- Claude Code（用于 `/blueprint` 斜杠命令）
- Git + GitHub CLI（可选——启用完整分支/PR/CI 工作流；Blueprint 检测缺失并自动切换到直接模式）

## 来源

灵感来自 antbotlab/blueprint——上游项目和参考设计。
