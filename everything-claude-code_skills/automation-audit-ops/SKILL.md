---
name: automation-audit-ops
description: Evidence-first automation inventory and overlap audit workflow for ECC. Use when the user wants to know which jobs, hooks, connectors, MCP servers, or wrappers are live, broken, redundant, or missing before fixing anything.
origin: ECC
---

# Automation Audit Ops

当用户问什么自动化是活跃的、哪些作业坏了、哪里存在重叠、或什么工具和连接器此刻正在做有用的工作时，使用本技能。

这是审计优先操作员技能。任务是产生证据支持的清单以及保留/合并/删除/修复-下一步建议集，然后重写任何内容。

## 技能栈

在相关时将这些 ECC 原生技能拉入工作流：

- `workspace-surface-audit` 用于连接器、MCP、钩子、app 清单
- `knowledge-ops` 当审计需要调和活动仓库真相与持久上下文时
- `github-ops` 当答案取决于 CI、计划工作流、issue 或 PR 自动化时
- `ecc-tools-cost-audit` 当真正的问题是 webhook 扇出、排队作业或兄弟应用仓库中的计费消耗时
- `research-ops` 当本地清单必须与当前平台支持或公共文档比较时
- `verification-loop` 用于证明修复后状态而非依赖假设的恢复

## 何时使用

- 用户问"我有什么自动化"、"什么是活跃的"、"什么是坏的"或"有什么重叠"
- 任务跨越 cron 作业、GitHub Actions、本地钩子、MCP 服务器、连接器、包装器或 app 集成
- 用户想知道什么从另一个 Agent 系统移植而来以及什么仍需要在 ECC 内重建
- 工作区积累了做同一事情的多种方式且用户想要一个规范通道

## 防护栏

- 除非用户明确要求修复，否则从只读开始
- 分开：
  - 已配置
  - 已认证
  - 最近已验证
  - 陈旧或损坏
  - 完全缺失
- 不要仅因为技能或配置引用了就声称工具是活跃的
- 在证据表存在之前不要合并或删除重叠界面

## 工作流

### 1. 清点真实界面

在理论化之前读取当前活跃界面：

- 仓库钩子和本地钩子脚本
- GitHub Actions 和计划工作流
- MCP 配置和启用的服务器
- 连接器-或 app 备份的集成
- 包装脚本和仓库特定自动化入口点

按界面分组：

- 本地运行时
- 仓库 CI / 自动化
- 连接的外部系统
- 消息传递 / 通知
- 计费 / 客户操作
- 研究 / 监控

### 2. 按活跃状态分类每个项目

对于每个发现的自动化，标记：

- 已配置
- 已认证
- 最近已验证
- 陈旧或损坏
- 缺失

然后分类问题类型：

- 活动故障
- 认证中断
- 陈旧状态
- 重叠或冗余
- 缺失能力

### 3. 追溯证明路径

每个重要主张都用具体来源支持：

- 文件路径
- 工作流运行
- 钩子日志
- 配置条目
- 最近的命令输出
- 精确的故障签名

如果当前状态模糊，直接说明而不是假装审计完成。

### 4. 以保留/合并/删除/修复-下一步告终

对于每个重叠或可疑界面，返回一个调用：

- keep
- merge
- cut
- fix next

价值在于将嘈杂的自动化折叠为单一规范 ECC 通道，而非保留每个历史路径。

## 输出格式

```text
CURRENT SURFACE
- automation
- source
- live state
- proof

FINDINGS
- active breakage
- overlap
- stale status
- missing capability

RECOMMENDATION
- keep
- merge
- cut
- fix next

NEXT ECC MOVE
- 要加强的确切技能/钩子/工作流/app 通道
```

## 陷阱

- 当可以读取实时清单时不要从记忆回答
- 不要将"存在于配置"视为"工作"
- 在命名破损的高信号路径之前不要修复低价值冗余
- 如果用户首先要求清单，不要将任务扩大为仓库重写

## 验证

- 重要主张引用实时证明路径
- 每个发现的自动化用清晰的活跃状态类别标记
- 最终建议区分保留/合并/删除/修复下一步
