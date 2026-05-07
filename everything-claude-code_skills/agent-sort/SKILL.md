---
name: agent-sort
description: Build an evidence-backed ECC install plan for a specific repo by sorting skills, commands, rules, hooks, and extras into DAILY vs LIBRARY buckets using parallel repo-aware review passes. Use when ECC should be trimmed to what a project actually needs instead of loading the full bundle.
origin: ECC
---

# Agent Sort

当仓库需要项目特定的 ECC 界面而非默认完整安装时，使用本技能。

目标不是猜测什么"感觉有用"。目标是使用实际代码库的证据对 ECC 组件进行分类。

## 何时使用

- 项目只需要 ECC 子集，完整安装太嘈杂
- 仓库栈清晰，但没有人想手动逐个整理技能
- 团队想要基于 grep 证据的可重复安装决策而非观点
- 需要将始终加载的日常工作界面与可搜索的库/参考界面分离
- 仓库漂移到错误的语言、规则或钩子集，需要清理

## 不可协商的规则

- 使用当前仓库作为真实来源，而非通用偏好
- 每个 DAILY 决策必须引用具体的仓库证据
- LIBRARY 不意味着"删除"；意味着"保留可访问但不默认加载"
- 不要安装当前仓库不能使用的钩子、规则或脚本
- 优先使用 ECC 原生界面；不要引入第二个安装系统

## 输出

按顺序产生这些工件：

1. DAILY 清单
2. LIBRARY 清单
3. 安装计划
4. 验证报告
5. 如果项目需要，可选的 `skill-library` 路由

## 分类模型

仅使用两个桶：

- `DAILY`
  - 应为此仓库的每个会话加载
  - 与仓库的语言、框架、工作流或操作界面强烈匹配
- `LIBRARY`
  - 值得保留，但不值得默认加载
  - 应保持可通过搜索、路由技能或选择性手动使用访问

## 证据来源

在做任何分类之前先使用仓库本地证据：

- 文件扩展名
- 包管理器和 lockfiles
- 框架配置
- CI 和钩子配置
- 构建/测试脚本
- 导入和依赖清单
- 明确描述栈的仓库文档

有用的命令包括：

```bash
rg --files
rg -n "typescript|react|next|supabase|django|spring|flutter|swift"
cat package.json
cat pyproject.toml
cat Cargo.toml
cat pubspec.yaml
cat go.mod
```

## 并行审查通道

如果并行子代理可用，将审查分为这些通道：

1. Agents
   - 分类 `agents/*`
2. Skills
   - 分类 `skills/*`
3. Commands
   - 分类 `commands/*`
4. Rules
   - 分类 `rules/*`
5. Hooks and scripts
   - 分类钩子界面、MCP 健康检查、帮助脚本和操作系统兼容性
6. Extras
   - 分类上下文、示例、MCP 配置、模板和指导文档

如果子代理不可用，按顺序运行相同的通道。

## 核心工作流

### 1. 读取仓库

在分类任何内容之前确立真实栈：

- 使用的语言
- 使用的框架
- 主要包管理器
- 测试栈
- lint/format 栈
- 部署/运行时界面
- 已存在的操作员集成

### 2. 构建证据表

对于每个候选界面，记录：

- 组件路径
- 组件类型
- 建议的桶
- 仓库证据
- 简要理由

使用此格式：

```text
skills/frontend-patterns | skill | DAILY | 84 .tsx 文件，存在 next.config.ts | 核心前端栈
skills/django-patterns   | skill | LIBRARY | 无 .py 文件，无 pyproject.toml       | 此仓库未激活
rules/typescript/*       | rules | DAILY | package.json + tsconfig.json            | 活跃的 TS 仓库
rules/python/*           | rules | LIBRARY | 零 Python 源文件                     | 仅保持可访问
```

### 3. 决定 DAILY vs LIBRARY

当符合以下条件时提升到 `DAILY`：

- 仓库明确使用匹配的栈
- 组件足够通用，每次会话都有帮助
- 仓库已经依赖相应的运行时或工作流

当符合以下条件时降级为 `LIBRARY`：

- 组件是栈外
- 仓库以后可能需要，但不是每天
- 它添加上下文开销而没有即时相关性

### 4. 构建安装计划

将分类转化为行动：

- DAILY skills -> 安装或保留在 `.claude/skills/`
- DAILY commands -> 仅当仍然有用时保留为显式 shim
- DAILY rules -> 仅安装匹配的语言集
- DAILY hooks/scripts -> 仅保留兼容的
- LIBRARY surfaces -> 通过搜索或 `skill-library` 保持可访问

如果仓库已经使用选择性安装，更新该计划而不是创建另一个系统。

### 5. 创建可选的库路由

如果项目想要可搜索的库界面，创建：

- `.claude/skills/skill-library/SKILL.md`

该路由应包含：

- DAILY vs LIBRARY 的简短说明
- 分组的触发关键词
- 库引用存放位置

不要将每个技能正文复制到路由中。

### 6. 验证结果

应用计划后，验证：

- 每个 DAILY 文件在预期位置存在
- 过时的语言规则没有被保留激活
- 不兼容的钩子没有被安装
- 最终的安装实际匹配仓库栈

返回包含以下内容的紧凑报告：

- DAILY 计数
- LIBRARY 计数
- 移除的陈腐界面
- 开放问题

## 交接

如果下一步是交互式安装或修复，交接给：

- `configure-ecc`

如果下一步是重叠清理或目录审查，交接给：

- `skill-stocktake`

如果下一步是更广泛的上下文修剪，交接给：

- `strategic-compact`

## 输出格式

按此顺序返回结果：

```text
STACK
- 语言/框架/运行时摘要

DAILY
- 带证据的始终加载项

LIBRARY
- 带证据的可搜索/参考项

INSTALL PLAN
- 应安装、移除或路由的内容

VERIFICATION
- 运行的检查和剩余差距
```
