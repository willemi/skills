---
name: autonomous-agent-harness
description: Transform Claude Code into a fully autonomous agent system with persistent memory, scheduled operations, computer use, and task queuing. Replaces standalone agent frameworks (Hermes, AutoGPT) by leveraging Claude Code's native crons, dispatch, MCP tools, and memory. Use when the user wants continuous autonomous operation, scheduled tasks, or a self-directing agent loop.
origin: ECC
---

# Autonomous Agent Harness（自主 Agent Harness）

仅使用原生功能和 MCP 服务器，将 Claude Code 转变为持久化、自我引导的 Agent 系统。

## 许可与安全边界

自主操作必须由用户明确请求并限定范围。除非用户已批准该能力且目标工作区为当前设置，否则不要创建计划、分派远程 Agent、写入持久内存、使用计算机控制、向外发布、修改第三方资源，或对私人通信采取行动。

在启用重复性或事件驱动操作之前，优先进行试运行计划和本地队列文件。将凭据、私有工作区导出、个人数据集和特定账户的自动化排除在可重用 ECC 工件之外。

## 何时激活

- 用户希望 Agent 持续运行或按计划运行
- 设置定期触发的自动化工作流
- 构建能记住跨会话上下文的个人 AI 助手
- 用户说"每天运行这个"、"定期检查这个"、"继续监控"
- 想要复制 Hermes、AutoGPT 或类似自主 Agent 框架的功能
- 需要结合计算机使用和计划执行

## 架构

```
┌──────────────────────────────────────────────────────────────┐
│                    Claude Code Runtime                        │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐ │
│  │  Crons   │  │ Dispatch │  │ Memory   │  │ Computer    │ │
│  │ Schedule │  │ Remote   │  │ Store    │  │ Use         │ │
│  │ Tasks    │  │ Agents   │  │          │  │             │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬──────┘ │
│       │              │             │                │        │
│       ▼              ▼             ▼                ▼        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              ECC Skill + Agent Layer                  │    │
│  │                                                      │    │
│  │  skills/     agents/     commands/     hooks/        │    │
│  └──────────────────────────────────────────────────────┘    │
│       │              │             │                │        │
│       ▼              ▼             ▼                ▼        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              MCP Server Layer                        │    │
│  │                                                      │    │
│  │  memory    github    exa    supabase    browser-use  │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

## 核心组件

### 1. 持久内存

使用 Claude Code 内置内存系统，并用 MCP 内存服务器增强结构化数据。

**内置内存**（`~/.claude/projects/*/memory/`）：
- 用户偏好、反馈、项目上下文
- 存储为带有 frontmatter 的 markdown 文件
- 在会话启动时自动加载

**MCP 内存服务器**（结构化知识图）：
- 实体、关系、观察
- 可查询的图结构
- 跨会话持久性

**内存模式：**

```
# 短期：当前会话上下文
使用 TodoWrite 进行会话内任务跟踪

# 中期：项目内存文件
写入 ~/.claude/projects/*/memory/ 以供跨会话回忆

# 长期：MCP 知识图
使用 mcp__memory__create_entities 创建永久结构化数据
使用 mcp__memory__create_relations 创建关系映射
使用 mcp__memory__add_observations 为已知实体添加新事实
```

### 2. 计划任务（Crons）

使用 Claude Code 的计划任务创建重复性 Agent 操作。

**设置 cron：**

```
# 通过 MCP 工具
mcp__scheduled-tasks__create_scheduled_task({
  name: "daily-pr-review",
  schedule: "0 9 * * 1-5",  // 工作日早上 9 点
  prompt: "Review all open PRs in affaan-m/everything-claude-code. For each: check CI status, review changes, flag issues. Post summary to memory.",
  project_dir: "/path/to/repo"
})

# 通过 claude -p（程序模式）
echo "Review open PRs and summarize" | claude -p --project /path/to/repo
```

**有用的 cron 模式：**

| 模式 | 时间表 | 用途 |
|------|--------|------|
| 每日站立会 | `0 9 * * 1-5` | 审查 PR、问题、部署状态 |
| 每周审查 | `0 10 * * 1` | 代码质量指标、测试覆盖率 |
| 每小时监控 | `0 * * * *` | 生产环境健康、错误率检查 |
| 夜间构建 | `0 2 * * *` | 运行完整测试套件、安全扫描 |
| 会议前准备 | `*/30 * * * *` | 为即将到来的会议准备上下文 |

### 3. 分派 / 远程 Agent

为事件驱动工作流远程触发 Claude Code Agent。

**分派模式：**

```bash
# 从 CI/CD 触发
curl -X POST "https://api.anthropic.com/dispatch" \
  -H "Authorization: Bearer $ANTHROPIC_API_KEY" \
  -d '{"prompt": "Build failed on main. Diagnose and fix.", "project": "/repo"}'

# 从 webhook 触发
# GitHub webhook → dispatch → Claude agent → fix → PR

# 从另一个 Agent 触发
claude -p "Analyze the output of the security scan and create issues for findings"
```

### 4. 计算机使用

利用 Claude 的计算机使用 MCP 进行物理世界交互。

**能力：**
- 浏览器自动化（导航、点击、填写表单、截图）
- 桌面控制（打开应用、输入、鼠标控制）
- 超出 CLI 的文件系统操作

**在 harness 中的用例：**
- Web UI 自动化测试
- 表单填写和数据录入
- 基于截图的监控
- 多应用工作流

### 5. 任务队列

管理跨会话边界持久化的任务队列。

**实现：**

```
# 通过内存实现任务持久化
将任务队列写入 ~/.claude/projects/*/memory/task-queue.md

# 任务格式
---
name: task-queue
type: project
description: 自主操作的持久化任务队列
---

## Active Tasks
- [ ] PR #123: 审查并批准，如果 CI 通过
- [ ] 监控部署：每 30 分钟检查 /health，持续 2 小时
- [ ] 研究：在 AI 工具领域找到 5 个潜在客户

## Completed
- [x] 每日站立会：审查了 3 个 PR，2 个问题
```

## 替换 Hermes

| Hermes 组件 | ECC 等价物 | 如何做 |
|--------------|------------|--------|
| Gateway/Router | Claude Code dispatch + crons | 计划任务触发 Agent 会话 |
| 内存系统 | Claude memory + MCP memory server | 内置持久化 + 知识图 |
| 工具注册表 | MCP 服务器 | 动态加载的工具提供者 |
| 编排 | ECC skills + agents | 技能定义指导 Agent 行为 |
| 计算机使用 | computer-use MCP | 原生的浏览器和桌面控制 |
| 上下文管理器 | 会话管理 + 内存 | ECC 2.0 会话生命周期 |
| 任务队列 | 内存持久化任务列表 | TodoWrite + 内存文件 |

## 设置指南

### 步骤 1：配置 MCP 服务器

确保以下内容在 `~/.claude.json` 中：

```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@anthropic/memory-mcp-server"]
    },
    "scheduled-tasks": {
      "command": "npx",
      "args": ["-y", "@anthropic/scheduled-tasks-mcp-server"]
    },
    "computer-use": {
      "command": "npx",
      "args": ["-y", "@anthropic/computer-use-mcp-server"]
    }
  }
}
```

### 步骤 2：创建基础 Crons

```bash
# 每日早晨简报
claude -p "Create a scheduled task: every weekday at 9am, review my GitHub notifications, open PRs, and calendar. Write a morning briefing to memory."

# 持续学习
claude -p "Create a scheduled task: every Sunday at 8pm, extract patterns from this week's sessions and update the learned skills."
```

### 步骤 3：初始化内存图

```bash
# 引导你的身份和上下文
claude -p "Create memory entities for: me (user profile), my projects, my key contacts. Add observations about current priorities."
```

### 步骤 4：启用计算机使用（可选）

授予计算机使用 MCP 必要的浏览器和桌面控制权限。

## 示例工作流

### 自主 PR 审查者
```
Cron: 工作日每 30 分钟
1. 检查监控仓库中的新 PR
2. 对于每个新 PR：
   - 拉取分支到本地
   - 运行测试
   - 使用 code-reviewer agent 审查更改
   - 通过 GitHub MCP 发布审查评论
3. 用审查状态更新内存
```

### 个人研究 Agent
```
Cron: 每天早上 6 点
1. 检查内存中保存的搜索查询
2. 为每个查询运行 Exa 搜索
3. 总结新发现
4. 与昨天结果比较
5. 写入摘要到内存
6. 标记高优先级项目供早晨审查
```

### 会议准备 Agent
```
触发：每个日历事件前 30 分钟
1. 读取日历事件详情
2. 搜索与会者相关内存上下文
3. 拉取与会者的近期邮件/Slack 线程
4. 准备 talking points 和议程建议
5. 将准备文档写入内存
```

## 约束

- Cron 任务在隔离会话中运行——它们不与非交互式会话共享上下文，除非通过内存。
- 计算机使用需要明确许可授予。不要假设访问权限。
- 远程分派可能有速率限制。设计 crons 时使用适当的间隔。
- 内存文件应保持简洁。归档旧数据而不是让文件无限制增长。
- 始终验证计划任务是否成功完成。为 cron 提示添加错误处理。
