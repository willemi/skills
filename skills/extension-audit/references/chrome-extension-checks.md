# Chrome 扩展专项安全检查

本文档包含 Chrome/Browser 扩展的专项安全检查项。这些检查是通用漏洞模式之外的扩展特有风险。

---

## 1. 权限与 Manifest 检查

### 1.1 host_permissions 过度声明

**检查方式**: 读取 `manifest.json`，搜索 `host_permissions`

**风险模式**:
```json
// 严重 - 访问所有网站
"host_permissions": ["<all_urls>"]
"host_permissions": ["http://*/*", "https://*/*"]

// 中 - 访问特定域名
"host_permissions": ["https://api.example.com/*"]
```

**判定**:
- `<all_urls>` / `*://*/*` → **严重** — 必须限制到实际需要的域名
- 多个不相关域名 → **高** — 需要逐一验证必要性
- 仅 API 域名 → **低** — 合理

### 1.2 content_scripts 注入范围

**检查方式**: 读取 `manifest.json`，搜索 `content_scripts`

**风险模式**:
```json
// 严重 - 注入到所有页面
"content_scripts": [{
  "matches": ["<all_urls>"],
  "js": ["content.js"]
}]

// 中 - 注入到特定页面
"content_scripts": [{
  "matches": ["https://example.com/*"],
  "js": ["content.js"]
}]
```

**判定**:
- `<all_urls>` → **严重** — 扩展在每个页面执行，攻击面巨大
- 宽泛匹配（`https://*/*`）→ **高**
- 特定域名 → **低**

### 1.3 web_accessible_resources 暴露

**检查方式**: 读取 `manifest.json`，搜索 `web_accessible_resources`

**风险模式**:
```json
// 严重 - 任意网站可访问所有资源
"web_accessible_resources": [{
  "resources": ["*"],
  "matches": ["<all_urls>"]
}]

// 高 - 暴露 JS/JSON 文件
"web_accessible_resources": [{
  "resources": ["config.json", "sdk.js"],
  "matches": ["<all_urls>"]
}]

// 低 - 仅暴露图标等静态资源
"web_accessible_resources": [{
  "resources": ["icons/*.png"],
  "matches": ["https://specific-site.com/*"]
}]
```

**判定标准**:
- 暴露 `.js`、`.json`、`.html` 文件给 `<all_urls>` → **严重/高**
- 暴露 `.png`、`.svg` 给 `<all_urls>` → **中**（可能被用于指纹识别）
- 仅暴露给特定域名 → **低**

### 1.4 externally_connectable

**检查方式**: 读取 `manifest.json`，搜索 `externally_connectable`

**风险模式**:
```json
// 严重 - 允许任意网站发送消息
"externally_connectable": {
  "matches": ["*://*/*"]
}

// 中 - 允许特定网站
"externally_connectable": {
  "matches": ["https://example.com/*"]
}
```

**判定**: 允许任意网站 → **严重**；特定域名 → 需验证必要性

### 1.5 Content Security Policy

**检查方式**: 读取 `manifest.json`，搜索 `content_security_policy`

**风险模式**:
```json
// 严重 - 允许不安全脚本
"content_security_policy": {
  "extension_pages": "script-src 'self' 'unsafe-eval' https:;"
}

// 正常
"content_security_policy": {
  "extension_pages": "script-src 'self'; object-src 'none'"
}
```

**判定**:
- 包含 `unsafe-eval` → **高**
- 包含 `unsafe-inline` → **高**
- 包含 `https:` 宽泛源 → **中**
- 仅有 `script-src 'self'; object-src 'none'` → **正常**
- 未声明 → **中**（MV3 默认 `script-src 'self'`，但仍建议显式声明）

---

## 2. 消息通信安全

### 2.1 Background 消息监听器 — 发送方验证

**Grep 搜索**: `chrome\.runtime\.onMessage\.addListener|chrome\.runtime\.onMessageExternal\.addListener`

**安全模式**:
```javascript
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  // 验证消息来源
  if (sender.id !== chrome.runtime.id) {
    console.warn('Rejected message from unknown sender:', sender.id);
    return;
  }
  // 处理消息...
});
```

**不安全模式**:
```javascript
chrome.runtime.onMessage.addListener((message, _sender, sendResponse) => {
  // 直接处理，不验证 sender
  if (message.action === 'doSomething') {
    // 危险：任何页面的 content script 都可以发送此消息
  }
});
```

**判定**:
- 不检查 `sender.id` 或 `sender.origin` → **高**
- 检查 `sender.id === chrome.runtime.id` → **正常**
- `onMessageExternal` 不检查 origin → **严重**

### 2.2 Content Script 消息发送

**Grep 搜索**: `chrome\.runtime\.sendMessage|chrome\.tabs\.sendMessage`

**检查要点**:
- content script 发送的消息是否包含敏感数据（token、用户信息）
- background 是否将消息转发给不安全的接收者

### 2.3 postMessage 跨域通信

**Grep 搜索**: `postMessage|window\.addEventListener\s*\(\s*["']message["']`

**检查要点**:
- 是否验证 `event.origin`
- 是否处理来自不可信源的消息

---

## 3. 内容脚本安全

### 3.1 全局对象暴露

**Grep 搜索**: `globalThis\.\w+\s*=|window\.\w+\s*=(?!\s*undefined)`

**风险**: 内容脚本在页面上下文中暴露全局对象，恶意网站可以调用这些接口操控扩展行为。

**不安全示例**:
```javascript
// 任何网页都可以调用
globalThis.__MY_EXTENSION__ = {
  activate: activateExtension,
  deactivate: deactivateExtension,
  sendData: sendUserData
};
```

**判定**: 暴露操控接口到全局对象 → **高**

### 3.2 DOM 注入到宿主页面

**Grep 搜索**: `innerHTML\s*=|\.appendChild\s*\(|\.insertBefore\s*\(`

**检查要点**:
- 注入的 HTML 是否包含用户输入
- 注入的元素是否可能被宿主页面篡改
- 是否使用了 `Shadow DOM` 隔离

### 3.3 事件监听器注册

**Grep 搜索**: `document\.addEventListener|window\.addEventListener`

**检查要点**:
- 是否在不需要时清理监听器
- 是否可能干扰宿主页面的功能
- 是否监听了敏感事件（键盘输入、鼠标轨迹等）

---

## 4. 数据存储安全

### 4.1 Token/敏感数据存储位置

**Grep 搜索**:
- `chrome\.storage\.local\.set|chrome\.storage\.sync\.set`
- `localStorage\.setItem|sessionStorage\.setItem`

**风险等级**:

| 存储方式 | 风险 | 说明 |
|---------|------|------|
| `localStorage` | **严重** | 同源脚本可访问，持久化存储 |
| `sessionStorage` | **高** | 同源脚本可访问，会话结束清除 |
| `chrome.storage.local` | **中** | 仅扩展可访问，但无加密 |
| `chrome.storage.session` (MV3) | **低** | 仅扩展可访问，会话结束清除 |
| `chrome.storage.sync` | **中** | 跨设备同步，需注意数据泄露范围 |

**判定**:
- Token/密码存 localStorage → **严重**
- Token 存 chrome.storage.local → **中**（可接受但建议加密）
- Token 存 chrome.storage.session → **低**（推荐）

### 4.2 存储数据加密

**Grep 搜索**: `crypto\.subtle\.encrypt|CryptoJS|AES|RSA`

**检查要点**:
- 敏感数据是否加密存储
- 加密密钥是否硬编码
- 是否使用安全的加密算法

---

## 5. 后台脚本 (Service Worker) 安全

### 5.1 动态脚本注入

**Grep 搜索**: `chrome\.scripting\.executeScript|chrome\.tabs\.executeScript`

**检查要点**:
- 注入的脚本来源（扩展内部 vs 远程 URL）
- `file` 参数是否硬编码（安全）vs 动态拼接（危险）

**安全模式**:
```javascript
await chrome.scripting.executeScript({
  target: { tabId },
  files: ['content.js']  // 硬编码文件名
});
```

**不安全模式**:
```javascript
// 从消息中获取文件名 - 可能注入任意脚本
await chrome.scripting.executeScript({
  target: { tabId },
  files: [message.scriptFile]
});
```

### 5.2 fetch/XMLHttpRequest 从 Service Worker

**Grep 搜索**: `fetch\s*\(|XMLHttpRequest|new Request\s*\(`

**检查要点**:
- 请求 URL 是否硬编码
- 是否发送敏感数据到外部服务器
- 是否验证响应来源

### 5.3 Side Panel / Popup 通信

**Grep 搜索**: `chrome\.runtime\.connect|chrome\.runtime\.sendMessage`

**检查要点**:
- Port 连接是否验证身份
- 消息格式是否严格校验
- 是否存在消息伪造风险

---

## 6. 扩展页面安全

### 6.1 外部链接打开

**Grep 搜索**: `target\s*=\s*["_']blank["']|window\.open\s*\(`

**检查要点**:
- 外部链接是否添加 `rel="noopener noreferrer"`
- 是否打开不可信 URL

### 6.2 扩展内部页面

**Grep 搜索**: `chrome-extension://|browser\.runtime\.getURL`

**检查要点**:
- 扩展内部页面是否接受外部参数
- URL 参数是否被安全处理
- 是否存在内部页面间的 XSS