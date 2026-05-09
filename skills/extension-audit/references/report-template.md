# 审核报告模板

审核完成后，必须按以下格式输出结构化报告。

---

## 模板

```markdown
# 扩展安全审核报告

## 基本信息
- **扩展名称**：
- **扩展类型**：（Chrome 扩展 / VS Code 扩展 / npm 包 / 其他）
- **版本号**：
- **开发者**：
- **审核日期**：

## 权限清单

| 权限 | 风险等级 | 是否必要 | 说明 |
|------|---------|---------|------|
| ...  | ...     | ...     | ... |

## 发现项汇总

| 编号 | 类别 | 严重程度 | 描述 | 文件:行号 |
|------|------|---------|------|----------|
| ...  | ...  | ...     | ... | ...      |

## 严重程度统计
- 严重：N 项
- 高：N 项
- 中：N 项
- 低：N 项

## 详细发现

### [编号] [标题]
- **严重程度**：
- **类别**：
- **文件**：`path/to/file:line`
- **描述**：
- **代码片段**：
  ```
  相关代码
  ```
- **风险说明**：
- **修复建议**：
  ```
  修复代码示例
  ```

## 审核结论

- [ ] **通过** — 无严重/高危项，可上架
- [ ] **有条件通过** — 存在中/低危项，需开发者限期修复，可先上架
- [ ] **驳回** — 存在严重/高危项，必须修复后重新提交
- [ ] **否决** — 存在恶意行为，永久拒绝上架

## 审核意见
[具体意见与建议]
```

---

## 结论判定标准

| 严重程度 | 对结论的影响 |
|---------|------------|
| 严重 | 直接驳回或否决 |
| 高 | 驳回，修复后重新审核 |
| 中 | 有条件通过，需限期修复 |
| 低 | 通过，建议优化 |

---

## 否决条件（发现即永久拒绝）

- 包含后门或恶意代码
- 故意混淆代码逃避审核
- 窃取用户敏感数据并外传
- 包含挖矿代码
- 包含 DDoS 攻击工具
- 包含键盘记录器
- 伪装为其他扩展（仿冒）

---

## 常见修复建议模板

### XSS — v-html / dangerouslySetInnerHTML

```
问题：使用 v-html/dangerouslySetInnerHTML 渲染不可信内容，未进行 HTML 消毒

修复方案：引入 DOMPurify 库，对所有不可信 HTML 进行清洗

// Vue
import DOMPurify from 'dompurify'
<div v-html="DOMPurify.sanitize(content)"></div>

// React
import DOMPurify from 'dompurify'
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(content) }} />
```

### 权限过度 — <all_urls>

```
问题：host_permissions/content_scripts 声明 <all_urls>，访问所有网站

修复方案：限制到实际需要的域名

"host_permissions": [
  "https://api.your-domain.com/*"
]
"content_scripts": [{
  "matches": ["https://specific-site.com/*"],
  "js": ["content.js"]
}]
```

### Token 存储 — localStorage

```
问题：用户 Token 明文存储在 localStorage，同源脚本可访问

修复方案：使用 chrome.storage.session（MV3），会话结束自动清除

// 替代
await chrome.storage.session.set({ userToken: token })

// 如果需要跨会话持久化，使用 chrome.storage.local + 加密
await chrome.storage.local.set({ encryptedToken: encrypt(token) })
```

### 消息通信 — 缺少发送方验证

```
问题：background 消息监听器未验证 sender，恶意页面可伪造消息

修复方案：检查 sender.id 是否为扩展自身

chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (sender.id !== chrome.runtime.id) {
    return; // 拒绝来自未知来源的消息
  }
  // 处理消息...
});
```

### 全局对象暴露

```
问题：内容脚本在全局对象上暴露操控接口，恶意网站可调用

修复方案：使用闭包或 Symbol 隐藏接口，不暴露到全局

// 不安全
globalThis.__MY_EXTENSION__ = { activate, deactivate };

// 安全 — 使用闭包，不暴露到全局
const extensionBridge = (() => {
  // 所有逻辑在闭包内
  function activate() { ... }
  function deactivate() { ... }
  // 仅通过 chrome.runtime.sendMessage 与扩展通信
  return { activate, deactivate };
})();
```

### 远程脚本加载

```
问题：动态从远程 URL 加载脚本，可在审核后修改扩展行为

修复方案：将脚本打包到扩展内部，使用本地路径

// 不安全
script.src = 'https://cdn.example.com/sdk.js'

// 安全
script.src = chrome.runtime.getURL('lib/sdk.js')
```

### CSP 未声明

```
问题：manifest.json 未声明 Content Security Policy

修复方案：显式声明严格 CSP

"content_security_policy": {
  "extension_pages": "script-src 'self'; object-src 'none'"
}
```