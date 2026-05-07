---
name: bun-runtime
description: Bun as runtime, package manager, bundler, and test runner. When to choose Bun vs Node, migration notes, and Vercel support.
origin: ECC
---

# Bun Runtime

Bun 是一个快速的一体化 JavaScript 运行时和工具包：运行时、包管理器、打包器和测试运行器。

## 何时使用

- **优先使用 Bun**：新 JS/TS 项目、安装/运行速度重要的脚本、使用 Bun 运行时的 Vercel 部署，以及当你想要单一工具链（运行 + 安装 + 测试 + 构建）时。
- **优先使用 Node**：最大生态系统兼容性、假设 Node 的遗留工具，或依赖项有已知 Bun 问题时。

当以下场景时使用：采用 Bun、从 Node 迁移、编写或调试 Bun 脚本/测试，或在 Vercel 或其他平台上配置 Bun。

## 工作原理

- **运行时**：即插即用的 Node 兼容运行时（基于 JavaScriptCore，用 Zig 实现）。
- **包管理器**：`bun install` 比 npm/yarn 显著更快。当前 Bun 版本默认 lockfile 为 `bun.lock`（文本）；旧版本使用 `bun.lockb`（二进制）。
- **打包器**：内置打包器和转译器，用于应用和库。
- **测试运行器**：内置 `bun test`，API 类似 Jest。

**从 Node 迁移**：用 `bun run script.js` 或 `bun script.js` 替换 `node script.js`。运行 `bun install` 代替 `npm install`；大多数包都能工作。使用 `bun run` 运行 npm 脚本；`bun x` 用于 npx 风格的一次性运行。Node 内置模块受支持；在存在 Bun API 的地方优先使用以获得更好性能。

**Vercel**：在项目设置中将运行时设为 Bun。构建：`bun run build` 或 `bun build ./src/index.ts --outdir=dist`。安装：`bun install --frozen-lockfile` 用于可复现的部署。

## 示例

### 运行和安装

```bash
# 安装依赖（创建/更新 bun.lock 或 bun.lockb）
bun install

# 运行脚本或文件
bun run dev
bun run src/index.ts
bun src/index.ts
```

### 脚本和环境

```bash
bun run --env-file=.env dev
FOO=bar bun run script.ts
```

### 测试

```bash
bun test
bun test --watch
```

```typescript
// test/example.test.ts
import { expect, test } from "bun:test";

test("add", () => {
  expect(1 + 2).toBe(3);
});
```

### 运行时 API

```typescript
const file = Bun.file("package.json");
const json = await file.json();

Bun.serve({
  port: 3000,
  fetch(req) {
    return new Response("Hello");
  },
});
```

## 最佳实践

- 提交 lockfile（`bun.lock` 或 `bun.lockb`）以确保可复现的安装。
- 优先使用 `bun run` 运行脚本。对于 TypeScript，Bun 原生运行 `.ts`。
- 保持依赖最新；Bun 和生态系统发展很快。
