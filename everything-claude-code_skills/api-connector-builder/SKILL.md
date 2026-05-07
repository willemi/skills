---
name: api-connector-builder
description: Build a new API connector or provider by matching the target repo's existing integration pattern exactly. Use when adding one more integration without inventing a second architecture.
origin: ECC direct-port adaptation
version: "1.0.0"
---

# API Connector Builder

当任务是为项目添加原生集成界面，而不仅仅是通用 HTTP 客户端时，使用本技能。

要点是匹配主机仓库的模式：

- 连接器布局
- 配置模式
- 认证模型
- 错误处理
- 测试风格
- 注册/发现连接

## 何时使用

- "为这个项目构建 Jira 连接器"
- "按照现有模式添加 Slack provider"
- "为此 API 创建新的集成"
- "构建匹配仓库连接器样式的插件"

## 防护栏

- 当仓库已有集成架构时不要发明新的
- 不要仅从供应商文档开始；首先从现有仓库内连接器开始
- 如果仓库期望注册连接、测试和文档，不要停止在传输代码
- 如果仓库有更新的当前模式，不要盲目照搬旧连接器

## 工作流

### 1. 了解内部风格

检查至少 2 个现有连接器/provider 并映射：

- 文件布局
- 抽象边界
- 配置模型
- 重试/分页约定
- 注册钩子
- 测试固件和命名

### 2. 缩小目标集成范围

仅定义仓库实际需要的界面：

- 认证流程
- 关键实体
- 核心读/写操作
- 分页和速率限制
- webhook 或轮询模型

### 3. 以仓库原生层构建

典型切片：

- config/schema
- client/transport
- mapping layer
- connector/provider entrypoint
- registration
- tests

### 4. 对照源模式验证

新连接器在代码库中应该看起来显而易见，而非从不同生态系统导入。

## 参考形状

### Provider-style

```
providers/
  existing_provider/
    __init__.py
    provider.py
    config.py
```

### Connector-style

```
integrations/
  existing/
    client.py
    models.py
    connector.py
```

### TypeScript plugin-style

```
src/integrations/
  existing/
    index.ts
    client.ts
    types.ts
    test.ts
```

## 质量检查清单

- [ ] 匹配仓库中现有的集成模式
- [ ] 存在配置验证
- [ ] 认证和错误处理是显式的
- [ ] 分页/重试行为遵循仓库规范
- [ ] 注册/发现连接是完整的
- [ ] 测试镜像主机仓库的风格
- [ ] 文档/示例已更新（如果仓库期望）

## 相关技能

- `backend-patterns`
- `mcp-server-patterns`
- `github-ops`
