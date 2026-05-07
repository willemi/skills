---
name: architecture-decision-records
description: Capture architectural decisions made during Claude Code sessions as structured ADRs. Auto-detects decision moments, records context, alternatives considered, and rationale. Maintains an ADR log so future developers understand why the codebase is shaped the way it is.
origin: ECC
---

# 架构决策记录

在编码会话期间捕获架构决策，使其成为结构化 ADR。决策不再只存在于 Slack 线程、PR 评论或某人的记忆中，而是产生结构化的 ADR 文档，与代码一起存放。

## 何时激活

- 用户明确说"让我们记录这个决策"或"ADR 这个"
- 用户在重要备选方案之间选择（框架、库、模式、数据库、API 设计）
- 用户说"我们决定..."或"我们做 X 而不是 Y 的原因是..."
- 用户问"我们为什么选择 X？"（读取现有 ADR）
- 在讨论架构权衡的设计阶段

## ADR 格式

使用 Michael Nygard 提出的轻量级 ADR 格式，专为 AI 辅助开发改编：

```markdown
# ADR-NNNN：[决策标题]

**日期**：YYYY-MM-DD
**状态**：proposed | accepted | deprecated | superseded by ADR-NNNN
**决策者**：[参与者]

## 上下文

是什么问题促使我们做出此决策或更改？

[2-5 句描述情况、约束和影响因素]

## 决策

我们提议和/或做什么更改？

[1-3 句清晰陈述决策]

## 考虑的替代方案

### 替代方案 1：[名称]
- **优点**：[收益]
- **缺点**：[缺点]
- **为什么不选**：[拒绝此方案的具体原因]

### 替代方案 2：[名称]
- **优点**：[收益]
- **缺点**：[缺点]
- **为什么不选**：[拒绝此方案的具体原因]

## 后果

此更改使得什么变得更容易或更困难？

### 正面
- [收益 1]
- [收益 2]

### 负面
- [权衡 1]
- [权衡 2]

### 风险
- [风险与缓解措施]
```

## 工作流程

### 捕获新 ADR

当检测到决策时刻：

1. **初始化（仅首次）** — 如果 `docs/adr/` 不存在，在创建前询问用户确认目录、包含索引表标题的 `README.md`（见下方 ADR 索引格式）和用于手动使用的空白 `template.md`。未经明确同意不要创建文件。
2. **识别决策** — 提取正在做的核心架构选择
3. **收集上下文** — 是什么问题引起的？存在什么约束？
4. **记录替代方案** — 考虑了哪些其他选项？为什么被拒绝？
5. **陈述后果** — 权衡是什么？什么变得更容易/更困难？
6. **分配编号** — 扫描 `docs/adr/` 中现有 ADR 并递增
7. **确认并写入** — 将 ADR 草案呈现给用户审查。仅在明确批准后写入 `docs/adr/NNNN-decision-title.md`。如果用户拒绝，放弃草案而不写入任何文件。
8. **更新索引** — 追加到 `docs/adr/README.md`

### 读取现有 ADR

当用户问"我们为什么选择 X？"：

1. 检查 `docs/adr/` 是否存在——如果不存在，回复："此项目中未找到 ADR。要开始记录架构决策吗？"
2. 如果存在，扫描 `docs/adr/README.md` 索引寻找相关条目
3. 读取匹配的 ADR 文件并呈现上下文和决策部分
4. 如果未找到匹配，回复："未找到该决策的 ADR。要现在记录一个吗？"

### ADR 目录结构

```
docs/
└── adr/
    ├── README.md              ← 所有 ADR 的索引
    ├── 0001-use-nextjs.md
    ├── 0002-postgres-over-mongo.md
    ├── 0003-rest-over-graphql.md
    └── template.md            ← 手动使用的空白模板
```

### ADR 索引格式

```markdown
# Architecture Decision Records

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0001](0001-use-nextjs.md) | Use Next.js as frontend framework | accepted | 2026-01-15 |
| [0002](0002-postgres-over-mongo.md) | PostgreSQL over MongoDB for primary datastore | accepted | 2026-01-20 |
| [0003](0003-rest-over-graphql.md) | REST API over GraphQL | accepted | 2026-02-01 |
```

## 决策检测信号

留意对话中表明架构决策的以下模式：

**明确信号**
- "我们就用 X 吧"
- "我们应该用 X 而不是 Y"
- "这个权衡是值得的因为..."
- "将这个记录为 ADR"

**隐含信号**（建议记录 ADR——未经用户确认不要自动创建）
- 比较两个框架或库并得出结论
- 做出数据库模式设计选择并有理由说明
- 在架构模式之间选择（单体 vs 微服务、REST vs GraphQL）
- 在评估备选方案后决定部署基础设施
- 决定认证/授权策略

## 什么是好的 ADR

### 要做

- **具体** — "使用 Prisma ORM" 而非"使用 ORM"
- **记录原因** — 理由比内容更重要
- **包含被拒绝的替代方案** — 未来的开发者需要知道考虑了什么
- **诚实陈述后果** — 每个决策都有权衡
- **保持简短** — ADR 应在 2 分钟内可读完
- **使用现在时** — "我们使用 X" 而非"我们将使用 X"

### 不要做

- 记录琐碎决策——变量命名或格式化选择不需要 ADR
- 写论文——如果上下文部分超过 10 行，就太长了
- 省略替代方案——"我们只是选了它"不是有效理由
- 在不标记的情况下回溯——如果记录过去决策，注明原始日期
- 让 ADR 过时——被取代的决策应引用其替代品

## ADR 生命周期

```
proposed → accepted → [deprecated | superseded by ADR-NNNN]
```

- **proposed**：决策正在讨论中，尚未提交
- **accepted**：决策生效并被遵循
- **deprecated**：决策不再相关（例如功能被移除）
- **superseded**：较新的 ADR 取代此 ADR（始终链接替换者）

## 值得记录的决策类别

| 类别 | 示例 |
|------|------|
| **技术选择** | 框架、语言、数据库、云提供商 |
| **架构模式** | 单体 vs 微服务、事件驱动、CQRS |
| **API 设计** | REST vs GraphQL、版本策略、认证机制 |
| **数据建模** | 模式设计、规范化决策、缓存策略 |
| **基础设施** | 部署模型、CI/CD 管道、监控堆栈 |
| **安全** | 认证策略、加密方法、密钥管理 |
| **测试** | 测试框架、覆盖率目标、E2E vs 集成平衡 |
| **流程** | 分支策略、审查流程、发布节奏 |

## 与其他技能集成

- **Planner agent**：当 planner 提出架构变更时，建议创建 ADR
- **Code reviewer agent**：标记引入架构变更但没有对应 ADR 的 PR
