---
name: agent-eval
description: Head-to-head comparison of coding agents (Claude Code, Aider, Codex, etc.) on custom tasks with pass rate, cost, time, and consistency metrics
origin: ECC
tools: Read, Write, Edit, Bash, Grep, Glob
---

# Agent 评估技能

一个轻量级 CLI 工具，用于在可复现的任务上对编码 Agent 进行头对头比较。每个"哪个编码 Agent 最好？"的比较都靠感觉——本工具将其系统化。

## 何时激活

- 在自己的代码库上比较编码 Agent（Claude Code、Aider、Codex 等）
- 在采用新工具或模型之前衡量 Agent 性能
- 在 Agent 更新模型或工具时运行回归检查
- 为团队提供数据驱动的 Agent 选择决策

## 安装

> **注意：** 审查源代码后，从其仓库安装 agent-eval。

## 核心概念

### YAML 任务定义

以声明式定义任务。每个任务指定要做什么、涉及哪些文件以及如何评判成功：

```yaml
name: add-retry-logic
description: Add exponential backoff retry to the HTTP client
repo: ./my-project
files:
  - src/http_client.py
prompt: |
  Add retry logic with exponential backoff to all HTTP requests.
  Max 3 retries. Initial delay 1s, max delay 30s.
judge:
  - type: pytest
    command: pytest tests/test_http_client.py -v
  - type: grep
    pattern: "exponential_backoff|retry"
    files: src/http_client.py
commit: "abc1234"  # 固定到特定提交以确保可复现性
```

### Git Worktree 隔离

每次 Agent 运行都有自己独立的 git worktree——无需 Docker。这提供了可复现的隔离，使 Agent 之间不会互相干扰或损坏基础仓库。

### 收集的指标

| 指标 | 衡量内容 |
|------|----------|
| 通过率 | Agent 是否生成了通过评判的代码？ |
| 成本 | 每个任务的 API 花费（可用时） |
| 时间 | 完成所需的实际时间（秒） |
| 一致性 | 多次运行的通过率（例如 3/3 = 100%） |

## 工作流程

### 1. 定义任务

创建一个 `tasks/` 目录，每个任务一个 YAML 文件：

```bash
mkdir tasks
# 编写任务定义（参见上方模板）
```

### 2. 运行 Agent

针对你的任务执行 Agent：

```bash
agent-eval run --task tasks/add-retry-logic.yaml --agent claude-code --agent aider --runs 3
```

每次运行：
1. 从指定提交创建一个新的 git worktree
2. 将提示词交给 Agent
3. 运行评判标准
4. 记录通过/失败、成本和时间

### 3. 比较结果

生成比较报告：

```bash
agent-eval report --format table
```

```
Task: add-retry-logic (3 runs each)
┌──────────────┬───────────┬────────┬────────┬─────────────┐
│ Agent        │ Pass Rate │ Cost   │ Time   │ Consistency │
├──────────────┼───────────┼────────┼────────┼─────────────┤
│ claude-code  │ 3/3       │ $0.12  │ 45s    │ 100%        │
│ aider        │ 2/3       │ $0.08  │ 38s    │  67%        │
└──────────────┴───────────┴────────┴────────┴─────────────┘
```

## 评判类型

### 基于代码（确定性）

```yaml
judge:
  - type: pytest
    command: pytest tests/ -v
  - type: command
    command: npm run build
```

### 基于模式

```yaml
judge:
  - type: grep
    pattern: "class.*Retry"
    files: src/**/*.py
```

### 基于模型（LLM 作为评判者）

```yaml
judge:
  - type: llm
    prompt: |
      Does this implementation correctly handle exponential backoff?
      Check for: max retries, increasing delays, jitter.
```

## 最佳实践

- **从 3-5 个任务开始**，代表你的真实工作负载，而非玩具示例
- **每个 Agent 至少运行 3 次试验**以捕获方差——Agent 是非确定性的
- **在任务 YAML 中固定提交**，使结果在数天/数周内可复现
- **每个任务至少包含一个确定性评判**（测试、构建）——LLM 评判会增加噪声
- **同时追踪成本和通过率**——成本高出 10 倍但通过率 95% 的 Agent 可能不是正确选择
- **对任务定义进行版本控制**——它们是测试固件，应像代码一样对待

## 链接

- 仓库：[github.com/joaquinhuigomez/agent-eval](https://github.com/joaquinhuigomez/agent-eval)
