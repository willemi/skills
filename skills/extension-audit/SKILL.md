---
name: extension-audit
description: 审核三方开发者提交的扩展，检测安全漏洞、远程代码注入、权限滥用，输出结构化审核报告。适用于 Chrome 扩展、VS Code 扩展、浏览器插件、应用插件、npm 包等各类扩展。
triggers:
  - 审核扩展
  - extension audit
  - 扩展安全审核
  - 插件审核
  - 审核插件
  - 代码审核扩展
  - 上架审核
  - extension review
  - 审查扩展权限
  - 检测远程注入
---

# 扩展安全审核 Skill

你是一名扩展平台安全审核员。严格按照以下流程执行审核，**每一步都必须使用工具实际扫描代码**，不能仅凭经验判断。

## 核心原则
1. **仅做静态分析** — 不执行任何代码或构建脚本
2. **并行搜索** — 同一步骤内的多个 Grep 调用必须并行发出
3. **看上下文** — 匹配到模式后必须用 Read 查看前后代码，确认是否真正危险
4. **标注来源** — 每个发现必须记录 文件:行号
5. **混淆即风险** — 无法审核的混淆代码直接标记为高风险

---

## Phase 1: 识别扩展类型

使用 Glob 扫描目标目录根文件，同时 Read 关键配置文件：

**并行执行**:
- `Glob { path: 目标目录, pattern: "manifest.json" }`
- `Glob { path: 目标目录, pattern: "package.json" }`
- `Glob { path: 目标目录, pattern: "AndroidManifest.xml" }`
- `Glob { path: 目标目录, pattern: "*.pem" }`  ← 发现私钥文件即标记严重

根据标志文件判断类型：
- `manifest.json` 含 `manifest_version` → Chrome/Browser 扩展
- `package.json` 含 `engines.vsix` → VS Code 扩展
- `AndroidManifest.xml` → Android 插件
- 仅 `package.json` → npm 包

**Chrome 扩展额外步骤**: 立即 Read `manifest.json`（或生成它的 TS 文件如 `manifest/chrome.ts`），提取：
- `permissions`
- `host_permissions`
- `content_scripts`
- `web_accessible_resources`
- `externally_connectable`
- `content_security_policy`
- `background`

---

## Phase 2: 全量源码扫描（并行 Grep）

对目标目录下所有源代码文件**并行**执行以下 Grep 搜索。每个搜索返回匹配行号，用于后续 Read 验证。

### 2.1 远程代码注入（最关键）

**并行发出以下 Grep 调用**:
```
Grep { pattern: "\\beval\\s*\\(", path: 目标目录 }
Grep { pattern: "(?:new\\s+)?Function\\s*\\(", path: 目标目录 }
Grep { pattern: "setTimeout\\s*\\(\\s*[\"']", path: 目标目录 }
Grep { pattern: "child_process", path: 目标目录 }
Grep { pattern: "\\b(?:exec|execSync|spawn|spawnSync|fork)\\s*\\(", path: 目标目录 }
Grep { pattern: "import\\s*\\(\\s*[\"']https?://", path: 目标目录 }
Grep { pattern: "createElement\\s*\\(\\s*[\"']script", path: 目标目录 }
Grep { pattern: "createElement\\s*\\(\\s*[\"']iframe", path: 目标目录 }
Grep { pattern: "\\.src\\s*=\\s*[\"']https?://", path: 目标目录 }
Grep { pattern: "WebAssembly\\.instantiate", path: 目标目录 }
```

### 2.2 XSS 风险（含框架特定）

**并行发出以下 Grep 调用**:
```
Grep { pattern: "v-html", path: 目标目录 }
Grep { pattern: "dangerouslySetInnerHTML", path: 目标目录 }
Grep { pattern: "\\.innerHTML\\s*=", path: 目标目录 }
Grep { pattern: "\\.outerHTML\\s*=", path: 目标目录 }
Grep { pattern: "insertAdjacentHTML", path: 目标目录 }
Grep { pattern: "document\\.write(?:ln)?\\s*\\(", path: 目标目录 }
Grep { pattern: "\\{@html", path: 目标目录 }
Grep { pattern: "bypassSecurityTrust", path: 目标目录 }
Grep { pattern: "\\.html\\s*\\(", path: 目标目录 }  ← jQuery
```

### 2.3 数据泄露与不安全存储

**并行发出以下 Grep 调用**:
```
Grep { pattern: "localStorage\\.setItem", path: 目标目录 }
Grep { pattern: "sessionStorage\\.setItem", path: 目标目录 }
Grep { pattern: "chrome\\.storage\\.(?:local|sync|session)\\.set", path: 目标目录 }
Grep { pattern: "(?:password|secret|api_key|token|private_key)\\s*[:=]\\s*[\"']", path: 目标目录 }
Grep { pattern: "http://.*(?:password|token|key|secret|credential|login|auth)", path: 目标目录 }
Grep { pattern: "rejectUnauthorized\\s*:\\s*false", path: 目标目录 }
```

### 2.4 Chrome 扩展消息通信

**并行发出以下 Grep 调用**:
```
Grep { pattern: "chrome\\.runtime\\.onMessage\\.addListener", path: 目标目录 }
Grep { pattern: "chrome\\.runtime\\.onMessageExternal\\.addListener", path: 目标目录 }
Grep { pattern: "chrome\\.runtime\\.sendMessage", path: 目标目录 }
Grep { pattern: "chrome\\.tabs\\.sendMessage", path: 目标目录 }
Grep { pattern: "postMessage", path: 目标目录 }
Grep { pattern: "chrome\\.scripting\\.executeScript", path: 目标目录 }
```

### 2.5 内容脚本暴露

**并行发出以下 Grep 调用**:
```
Grep { pattern: "globalThis\\.", path: 目标目录 }
Grep { pattern: "window\\.__", path: 目标目录 }
Grep { pattern: "window\\.[A-Z_]+\\s*=", path: 目标目录 }  ← 全局大写变量
```

### 2.6 供应链风险

**并行发出以下 Grep 调用**:
```
Grep { pattern: "(?:pre|post)?install.*?(?:curl|wget|bash)", path: 目标目录, glob: "package.json" }
Grep { pattern: "__proto__", path: 目标目录 }
```

---

## Phase 3: 上下文验证

对 Phase 2 中每个 Grep 匹配结果，使用 Read 查看匹配行及其前后 10-20 行，确认：

1. **数据来源** — 匹配的变量/参数来自哪里？（用户输入 / API 响应 / 硬编码 / 受控数据）
2. **是否有消毒** — 不可信数据在被使用前是否经过 DOMPurify 等消毒处理？
3. **是否真正危险** — 例如 `eval("2+2")` 硬编码字符串风险远低于 `eval(userInput)`

**优先级**: 先验证严重类别（远程代码注入 > XSS > 数据泄露 > 消息通信 > 内容脚本暴露）

---

## Phase 4: Chrome 扩展专项深度检查

如果是 Chrome/Browser 扩展，在 Phase 3 基础上额外检查以下内容。参考 [chrome-extension-checks.md](references/chrome-extension-checks.md)。

### 4.1 Manifest 权限最小化

读取 manifest 文件，逐项评估每个权限：
- 是否**必要**（代码中是否实际使用了该 API）
- 是否**最小**（是否可以用更小的权限替代，如 `activeTab` 替代 `<all_urls>`）
- **host_permissions** 是否限制到实际需要的域名

### 4.2 消息处理器安全性

对每个 `chrome.runtime.onMessage.addListener` 匹配：
- **Read** 查看完整的消息处理逻辑
- 检查是否验证 `sender.id === chrome.runtime.id`
- 检查消息格式是否严格校验（type/action 字段）
- 检查是否有消息可被恶意页面伪造

### 4.3 内容脚本隔离

对 content script 文件：
- 检查是否在全局对象上暴露接口（`globalThis.xxx = ...`）
- 检查 DOM 注入是否使用 Shadow DOM 隔离
- 检查事件监听器是否会在扩展卸载时清理

### 4.4 web_accessible_resources 暴露

检查 manifest 中声明：
- 是否暴露 `.js`、`.json`、`.html` 文件
- `matches` 是否为 `<all_urls>`
- 是否被宿主页面用于指纹识别扩展

### 4.5 Token/Session 存储方式

搜索 Token 存储位置：
- `localStorage` → **严重**
- `chrome.storage.local` → **中**
- `chrome.storage.session` → **低**（推荐）

---

## Phase 5: 依赖项审查

**Read** `package.json`：
1. 列出所有 `dependencies`（非 devDependencies）生产依赖
2. 检查 `scripts` 中的 `preinstall`/`postinstall` 是否执行网络请求或系统命令
3. 检查是否有可疑包名（typosquatting，如 `lodsh` 仿冒 `lodash`）

---

## Phase 6: 生成审核报告

按 [report-template.md](references/report-template.md) 输出结构化报告。关键要求：

1. **每个发现项必须包含**: 文件路径:行号 + 代码片段 + 风险说明 + 修复建议
2. **修复建议必须具体**: 给出代码级修复示例，不能只说"建议修复"
3. **结论判定**:
   - 无严重/高危 → **通过**
   - 有中/低危 → **有条件通过**
   - 有严重/高危 → **驳回**
   - 有恶意行为 → **否决**

---

## 参考文档

详细模式和判定规则见：
- [漏洞模式参考](references/vulnerability-patterns.md) — 全部正则模式和风险等级
- [Chrome 扩展专项检查](references/chrome-extension-checks.md) — 消息安全、内容脚本隔离、存储安全
- [框架特定检查](references/framework-specific.md) — Vue/React/Angular/jQuery/Svelte XSS 模式
- [报告模板](references/report-template.md) — 报告格式和修复建议模板

## 常见误区
- 不要运行扩展或安装脚本
- 不要对混淆代码做假设，要求开发者提供源码映射
- 不要忽略 `package-lock.json` / `yarn.lock`
- 不要只看 Grep 匹配就判定，必须 Read 上下文验证
- 图片资源（png/jpg/gif/svg/ico/webp）不纳入代码审核范围
- 始终对比声明的权限与实际使用的 API 是否一致