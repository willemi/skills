---
name: ai-regression-testing
description: Regression testing strategies for AI-assisted development. Sandbox-mode API testing without database dependencies, automated bug-check workflows, and patterns to catch AI blind spots where the same model writes and reviews code.
origin: ECC
---

# AI 回归测试

专为 AI 辅助开发设计的测试模式，其中同一模型编写代码并审查它——创建只有自动化测试能捕获的系统性盲点。

## 何时激活

- AI Agent（Claude Code、Cursor、Codex）修改了 API 路由或后端逻辑
- 发现并修复了 bug——需要防止重新引入
- 项目有可用于无 DB 测试的沙盒/模拟模式
- 代码更改后运行 `/bug-check` 或类似审查命令
- 存在多个代码路径（沙盒 vs 生产、功能标志等）

## 核心问题

当 AI 编写代码然后审查自己的工作时，它将相同的假设带入两个步骤。这产生可预测的失败模式：

```
AI 编写修复 → AI 审查修复 → AI 说"看起来正确" → Bug 仍然存在
```

真实世界示例（在生产环境中观察到）：

```
修复 1：向 API 响应添加了 notification_settings
  → 忘记将其添加到 SELECT 查询
  → AI 审查并遗漏了（相同盲点）

修复 2：将其添加到 SELECT 查询
  → TypeScript 构建错误（列不在生成的类型中）
  → AI 审查了修复 1 但没有捕获 SELECT 问题

修复 3：改为 SELECT *
  → 修复了生产路径，忘记了沙盒路径
  → AI 审查并再次遗漏（第 4 次发生）

修复 4：测试在第一次运行时捕获 PASS：
```

模式：**沙盒/生产路径不一致**是 #1 AI 引入的回归。

## 沙盒模式 API 测试

具有 AI 友好架构的大多数项目有沙盒/模拟模式。这是快速、无 DB 的 API 测试的关键。

### 设置（Vitest + Next.js App Router）

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  test: {
    environment: "node",
    globals: true,
    include: ["__tests__/**/*.test.ts"],
    setupFiles: ["__tests__/setup.ts"],
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "."),
    },
  },
});
```

```typescript
// __tests__/setup.ts
// 强制沙盒模式——不需要数据库
process.env.SANDBOX_MODE = "true";
process.env.NEXT_PUBLIC_SUPABASE_URL = "";
process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY = "";
```

### Next.js API 路由测试助手

```typescript
// __tests__/helpers.ts
import { NextRequest } from "next/server";

export function createTestRequest(
  url: string,
  options?: {
    method?: string;
    body?: Record<string, unknown>;
    headers?: Record<string, string>;
    sandboxUserId?: string;
  },
): NextRequest {
  const { method = "GET", body, headers = {}, sandboxUserId } = options || {};
  const fullUrl = url.startsWith("http") ? url : `http://localhost:3000${url}`;
  const reqHeaders: Record<string, string> = { ...headers };

  if (sandboxUserId) {
    reqHeaders["x-sandbox-user-id"] = sandboxUserId;
  }

  const init: { method: string; headers: Record<string, string>; body?: string } = {
    method,
    headers: reqHeaders,
  };

  if (body) {
    init.body = JSON.stringify(body);
    reqHeaders["content-type"] = "application/json";
  }

  return new NextRequest(fullUrl, init);
}

export async function parseResponse(response: Response) {
  const json = await response.json();
  return { status: response.status, json };
}
```

### 编写回归测试

关键原则：**为发现的 bug 写测试，而非为能工作的代码**。

```typescript
// __tests__/api/user/profile.test.ts
import { describe, it, expect } from "vitest";
import { createTestRequest, parseResponse } from "../../helpers";
import { GET, PATCH } from "@/app/api/user/profile/route";

// 定义契约——响应中必须有哪些字段
const REQUIRED_FIELDS = [
  "id",
  "email",
  "full_name",
  "phone",
  "role",
  "created_at",
  "avatar_url",
  "notification_settings",  // ← bug 发现后添加
];

describe("GET /api/user/profile", () => {
  it("returns all required fields", async () => {
    const req = createTestRequest("/api/user/profile");
    const res = await GET(req);
    const { status, json } = await parseResponse(res);

    expect(status).toBe(200);
    for (const field of REQUIRED_FIELDS) {
      expect(json.data).toHaveProperty(field);
    }
  });

  // 回归测试——这个精确 bug 被 AI 引入了 4 次
  it("notification_settings is not undefined (BUG-R1 regression)", async () => {
    const req = createTestRequest("/api/user/profile");
    const res = await GET(req);
    const { json } = await parseResponse(res);

    expect("notification_settings" in json.data).toBe(true);
    const ns = json.data.notification_settings;
    expect(ns === null || typeof ns === "object").toBe(true);
  });
});
```

### 测试沙盒/生产一致性

最常见的 AI 回归：修复生产路径但忘记沙盒路径（或相反）。

```typescript
// 测试沙盒模式响应是否符合预期契约
describe("GET /api/user/messages (conversation list)", () => {
  it("includes partner_name in sandbox mode", async () => {
    const req = createTestRequest("/api/user/messages", {
      sandboxUserId: "user-001",
    });
    const res = await GET(req);
    const { json } = await parseResponse(res);

    // 这捕获了一个 bug，其中 partner_name 被添加
    // 到生产路径但未添加到沙盒路径
    if (json.data.length > 0) {
      for (const conv of json.data) {
        expect("partner_name" in conv).toBe(true);
      }
    }
  });
});
```

## 将测试集成到 Bug-Check 工作流

### 自定义命令定义

```markdown
<!-- .claude/commands/bug-check.md -->
# Bug Check

## 步骤 1：自动化测试（强制，不能跳过）

首先运行这些命令，然后再进行任何代码审查：

    npm run test       # Vitest 测试套件
    npm run build      # TypeScript 类型检查 + 构建

- 如果测试失败 → 报告为最高优先级 bug
- 如果构建失败 → 报告类型错误为最高优先级
- 仅在两者都通过后才继续到步骤 2

## 步骤 2：代码审查（AI 审查）

1. 沙盒/生产路径一致性
2. API 响应形状符合前端预期
3. SELECT 子句完整性
4. 带回滚的错误处理
5. 乐观更新竞态条件

## 步骤 3：对于每个修复的 bug，提议一个回归测试
```

### 工作流

```
用户："バグチェックして"（或 "/bug-check"）
  │
  ├─ 步骤 1：npm run test
  │   ├─ 失败 → 发现机械 bug（不需要 AI 判断）
  │   └─ 通过 → 继续
  │
  ├─ 步骤 2：npm run build
  │   ├─ 失败 → 发现类型错误（机械检测）
  │   └─ 通过 → 继续
  │
  ├─ 步骤 3：AI 代码审查（牢记已知盲点）
  │   └─ 报告发现
  │
  └─ 步骤 4：对于每个修复，编写回归测试
      └─ 下次 bug-check 捕获是否修复破坏内容
```

## 常见 AI 回归模式

### 模式 1：沙盒/生产路径不匹配

**频率**：最常见（4 分之 3 的回归中观察到）

```typescript
// 失败：AI 仅添加到生产路径
if (isSandboxMode()) {
  return { data: { id, email, name } };  // 缺少新字段
}
// 生产路径
return { data: { id, email, name, notification_settings } };

// 通过：两条路径必须返回相同的形状
if (isSandboxMode()) {
  return { data: { id, email, name, notification_settings: null } };
}
return { data: { id, email, name, notification_settings } };
```

**捕获它的测试**：

```typescript
it("sandbox and production return same fields", async () => {
  // 在测试环境中，沙盒模式强制开启
  const res = await GET(createTestRequest("/api/user/profile"));
  const { json } = await parseResponse(res);

  for (const field of REQUIRED_FIELDS) {
    expect(json.data).toHaveProperty(field);
  }
});
```

### 模式 2：SELECT 子句遗漏

**频率**：添加新列时 Supabase/Prisma 常见

```typescript
// 失败：新列添加到响应但未添加到 SELECT
const { data } = await supabase
  .from("users")
  .select("id, email, name")  // notification_settings 不在这里
  .single();

return { data: { ...data, notification_settings: data.notification_settings } };
// → notification_settings 总是 undefined

// 通过：使用 SELECT * 或显式包含新列
const { data } = await supabase
  .from("users")
  .select("*")
  .single();
```

### 模式 3：错误状态泄露

**频率**：中等——当向现有组件添加错误处理时

```typescript
// 失败：设置错误状态但旧数据未清除
catch (err) {
  setError("Failed to load");
  // 预订仍显示来自上一个标签页的数据！
}

// 通过：错误时清除相关状态
catch (err) {
  setReservations([]);  // 清除陈旧数据
  setError("Failed to load");
}
```

### 模式 4：无正确回滚的乐观更新

```typescript
// 失败：失败时无回滚
const handleRemove = async (id: string) => {
  setItems(prev => prev.filter(i => i.id !== id));
  await fetch(`/api/items/${id}`, { method: "DELETE" });
  // 如果 API 失败，项目从 UI 消失但仍在 DB 中
};

// 通过：捕获之前状态并在失败时回滚
const handleRemove = async (id: string) => {
  const prevItems = [...items];
  setItems(prev => prev.filter(i => i.id !== id));
  try {
    const res = await fetch(`/api/items/${id}`, { method: "DELETE" });
    if (!res.ok) throw new Error("API error");
  } catch {
    setItems(prevItems);  // 回滚
    alert("Failed to delete");
  }
};
```

## 策略：在发现 bug 的地方测试

不要 aim for 100% 覆盖率。而是：

```
在 /api/user/profile 发现 bug     → 为 profile API 写测试
在 /api/user/messages 发现 bug    → 为 messages API 写测试
在 /api/user/favorites 发现 bug   → 为 favorites API 写测试
在 /api/user/notifications 未发现 bug → 不要写测试（目前）
```

**为什么这对 AI 开发有效：**

1. AI 倾向于犯**相同类别的错误**反复
2. bug 聚集在复杂区域（认证、多路径逻辑、状态管理）
3. 一旦测试，那个精确的回归**不可能再次发生**
4. 测试数量随 bug 修复有机增长——无浪费工作

## 快速参考

| AI 回归模式 | 测试策略 | 优先级 |
|---|---|---|
| 沙盒/生产不匹配 | 断言沙盒模式下的相同响应形状 | 高 |
| SELECT 子句遗漏 | 断言响应中的所有必填字段 | 高 |
| 错误状态泄露 | 断言错误时状态清理 | 中 |
| 缺少回滚 | 断言 API 失败时状态恢复 | 中 |
| 类型转换掩盖 null | 断言字段不是 undefined | 中 |

## 做 / 不要做

**要做：**
- 发现 bug 后立即写测试（如果可能的话在修复之前）
- 测试 API 响应形状，而非实现
- 作为每次 bug-check 的第一步运行测试
- 保持测试快速（沙盒模式下总共 < 1 秒）
- 以阻止的 bug 命名测试（例如"BUG-R1 regression"）

**不要做：**
- 为从未有 bug 的代码写测试
- 信任 AI 自审作为自动化测试的替代
- 跳过沙盒路径测试，因为"只是模拟数据"
- 单元测试足够时写集成测试
- aim for 覆盖率百分比——aim for 回归预防
