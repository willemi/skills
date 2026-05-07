---
name: agent-payment-x402
description: Add x402 payment execution to AI agents — per-task budgets, spending controls, and non-custodial wallets via MCP tools. Use when agents need to pay for APIs, services, or other agents.
origin: community
---

# Agent 支付执行（x402）

使 AI Agent 能够在内置支出控制下进行自主支付。使用 x402 HTTP 支付协议和 MCP 工具，使 Agent 可以在无需托管风险的情况下支付外部服务、API 或其他 Agent。

## 何时使用

当以下场景时使用：你的 Agent 需要支付 API 调用、购买服务、与其他 Agent 结算、强制执行每任务支出限制或管理非托管钱包。与 cost-aware-llm-pipeline 和 security-review 技能天然搭配。

## 工作原理

### x402 协议
x402 将 HTTP 402（需要付款）扩展为机器可协商的流程。当服务器返回 `402` 时，Agent 的支付工具会自动协商价格、检查预算、签署交易并重试——无需人工介入。

### 支出控制
每次支付工具调用都强制执行 `SpendingPolicy`：
- **每任务预算** — 单个 Agent 操作的最大支出
- **每会话预算** — 整个会话的累计限制
- **白名单接收者** — 限制 Agent 可以支付的地址/服务
- **速率限制** — 每分钟/小时的最大交易数

### 非托管钱包
Agent 通过 ERC-4337 智能账户持有自己的密钥。编排器在委派之前设置策略；Agent 只能在范围内支出。没有资金池，没有托管风险。

## MCP 集成

支付层暴露标准的 MCP 工具，可以插入任何 Claude Code 或 Agent harness 设置。

> **安全提示**：始终固定包版本。此工具管理私钥——未固定版本的 `npx` 安装会引入供应链风险。

```json
{
  "mcpServers": {
    "agentpay": {
      "command": "npx",
      "args": ["agentwallet-sdk@6.0.0"]
    }
  }
}
```

### 可用工具（Agent 可调用）

| 工具 | 用途 |
|------|------|
| `get_balance` | 检查 Agent 钱包余额 |
| `send_payment` | 向地址或 ENS 发送支付 |
| `check_spending` | 查询剩余预算 |
| `list_transactions` | 所有支付的审计追踪 |

> **注意**：支出策略由**编排器**在委派给 Agent 之前设置——而非由 Agent 自身设置。这防止 Agent 自行提升支出限制。通过编排层或预任务钩子中的 `set_policy` 配置策略，绝不要将其作为 Agent 可调用工具。

## 示例

### MCP 客户端中的预算强制执行

在构建调用 agentpay MCP 服务器的编排器时，在分派付费工具调用之前强制执行预算。

> **前置条件**：在添加 MCP 配置之前安装包——不带 `-y` 的 `npx` 将在非交互式环境中提示确认，导致服务器挂起：`npm install -g agentwallet-sdk@6.0.0`

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

async function main() {
  // 1. 在构建传输之前验证凭据。
  //    缺少密钥必须立即失败——绝不让子进程在没有认证的情况下启动。
  const walletKey = process.env.WALLET_PRIVATE_KEY;
  if (!walletKey) {
    throw new Error("WALLET_PRIVATE_KEY is not set — refusing to start payment server");
  }

  // 通过 stdio 传输连接到 agentpay MCP 服务器。
  // 仅白名单服务器需要的环境变量——绝不将所有 process.env 转发
  // 给管理私密的第三方子进程。
  const transport = new StdioClientTransport({
    command: "npx",
    args: ["agentwallet-sdk@6.0.0"],
    env: {
      PATH: process.env.PATH ?? "",
      NODE_ENV: process.env.NODE_ENV ?? "production",
      WALLET_PRIVATE_KEY: walletKey,
    },
  });
  const agentpay = new Client({ name: "orchestrator", version: "1.0.0" });
  await agentpay.connect(transport);

  // 2. 在委派给 Agent 之前设置支出策略。
  //    始终验证成功——静默失败意味着没有控制处于活跃状态。
  const policyResult = await agentpay.callTool({
    name: "set_policy",
    arguments: {
      per_task_budget: 0.50,
      per_session_budget: 5.00,
      allowlisted_recipients: ["api.example.com"],
    },
  });
  if (policyResult.isError) {
    throw new Error(
      `Failed to set spending policy — do not delegate: ${JSON.stringify(policyResult.content)}`
    );
  }

  // 3. 在任何付费操作之前使用 preToolCheck
  await preToolCheck(agentpay, 0.01);
}

// 预工具钩子：具有四条不同错误路径的故障封闭预算强制执行。
async function preToolCheck(agentpay: Client, apiCost: number): Promise<void> {
  // 路径 1：拒绝无效输入（NaN/Infinity 会绕过 < 比较）
  if (!Number.isFinite(apiCost) || apiCost < 0) {
    throw new Error(`Invalid apiCost: ${apiCost} — action blocked`);
  }

  // 路径 2：传输/连接失败
  let result;
  try {
    result = await agentpay.callTool({ name: "check_spending" });
  } catch (err) {
    throw new Error(`Payment service unreachable — action blocked: ${err}`);
  }

  // 路径 3：工具返回错误（例如认证失败，钱包未初始化）
  if (result.isError) {
    throw new Error(
      `check_spending failed — action blocked: ${JSON.stringify(result.content)}`
    );
  }

  // 路径 4：解析并验证响应结构
  let remaining: number;
  try {
    const parsed = JSON.parse(
      (result.content as Array<{ text: string }>)[0].text
    );
    if (!Number.isFinite(parsed?.remaining)) {
      throw new TypeError("missing or non-finite 'remaining' field");
    }
    remaining = parsed.remaining;
  } catch (err) {
    throw new Error(
      `check_spending returned unexpected format — action blocked: ${err}`
    );
  }

  // 路径 5：预算超支
  if (remaining < apiCost) {
    throw new Error(
      `Budget exceeded: need $${apiCost} but only $${remaining} remaining`
    );
  }
}

main().catch((err) => {
  console.error(err);
  process.exitCode = 1;
});
```

## 最佳实践

- **在委派之前设置预算**：在生成子 Agent 时，通过编排层附加 SpendingPolicy。绝不要给 Agent 无限制的支出。
- **固定依赖版本**：始终在 MCP 配置中指定确切版本（例如 `agentwallet-sdk@6.0.0`）。部署到生产环境之前验证包完整性。
- **审计追踪**：在任务后钩子中使用 `list_transactions` 记录支出内容和原因。
- **故障封闭**：如果支付工具不可用，阻止付费操作——不要回退到未计量的访问。
- **与 security-review 搭配使用**：支付工具是高权限的。应像对待 shell 访问一样严格审查。
- **先用测试网测试**：开发时使用 Base Sepolia；生产环境切换到 Base 主网。

## 生产参考

- **npm**：[`agentwallet-sdk`](https://www.npmjs.com/package/agentwallet-sdk)
- **已合并到 NVIDIA NeMo Agent 工具包**：[PR #17](https://github.com/NVIDIA/NeMo-Agent-Toolkit-Examples/pull/17) — x402 支付工具用于 NVIDIA 的 Agent 示例
- **协议规范**：[x402.org](https://x402.org)
