# 框架特定安全检查

不同前端框架有各自的安全风险点。本参考文档覆盖常见框架的 XSS 和安全相关模式。

---

## Vue.js

### v-html（最常见 XSS 向量）

**Grep 搜索**: `v-html`

**风险模式**:
```vue
<!-- 严重 - 直接渲染未经消毒的数据 -->
<div v-html="message.content"></div>
<div v-html="apiResponse"></div>
<div v-html="userInput"></div>
```

**安全替代**:
```vue
<!-- 安全 - 文本插值自动转义 -->
<div>{{ message.content }}</div>

<!-- 安全 - 使用 DOMPurify 消毒 -->
<div v-html="sanitize(message.content)"></div>
```

**判定**:
- `v-html` 渲染 API 响应/AI 输出 → **严重**（不可信来源）
- `v-html` 渲染用户输入 → **严重**
- `v-html` 渲染硬编码/受控内容 → **中**
- `v-html` + DOMPurify 消毒 → **低**

### 其他 Vue 风险

| 模式 | Grep | 风险 | 说明 |
|------|------|------|------|
| v-text 误用 | `v-text` | 低 | v-text 是安全的（自动转义），但需确认非 v-html |
| 动态组件 | `:is=` | 中 | 动态组件名可被注入 |
| $refs 操作 DOM | `\$refs\..*\.innerHTML` | 高 | 绕过 Vue 的转义 |
| render 函数 | `h\s*\(\s*["']div["'].*domProps` | 高 | render 函数中 innerHTML |

---

## React

### dangerouslySetInnerHTML

**Grep 搜索**: `dangerouslySetInnerHTML`

**风险模式**:
```jsx
// 严重 - 直接渲染未经消毒的数据
<div dangerouslySetInnerHTML={{ __html: message.content }} />
<div dangerouslySetInnerHTML={{ __html: apiResponse }} />
```

**安全替代**:
```jsx
// 安全 - JSX 自动转义
<div>{message.content}</div>

// 安全 - 使用 DOMPurify
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(content) }} />
```

### 其他 React 风险

| 模式 | Grep | 风险 | 说明 |
|------|------|------|------|
| href 注入 | `href\s*=\s*\{.*(?:location\|window\|url)` | 高 | javascript: 协议注入 |
| useRef + innerHTML | `useRef.*\.innerHTML` | 高 | 绕过 React 转义 |
| findDOMNode | `findDOMNode.*innerHTML` | 高 | 绕过 React 转义 |

---

## Angular

### innerHTML 绑定

**Grep 搜索**: `innerHTML|outerHTML`

**风险模式**:
```html
<!-- 高 - 直接绑定未经消毒的 HTML -->
<div [innerHTML]="userContent"></div>
```

**安全替代**:
```html
<!-- Angular 默认消毒 innerHTML，但仍建议使用 DomSanitizer -->
<div [innerHTML]="sanitizedContent"></div>
```

### bypassSecurityTrust

**Grep 搜索**: `bypassSecurityTrust(?:Html\|Url\|ResourceUrl\|Script\|Style)`

**风险模式**:
```typescript
// 严重 - 绕过 Angular 安全检查
this.sanitizer.bypassSecurityTrustHtml(userInput)
this.sanitizer.bypassSecurityTrustScript(dynamicScript)
this.sanitizer.bypassSecurityTrustUrl(userUrl)
```

**判定**: 任何 `bypassSecurityTrust*` 调用 → **高**，需逐一验证输入来源

---

## jQuery

### .html() 方法

**Grep 搜索**: `\.html\s*\(|\.append\s*\(|\.prepend\s*\(|\.after\s*\(|\.before\s*\(`

**风险模式**:
```javascript
// 严重 - 使用用户输入
$('.container').html(userInput)
$('.list').append('<li>' + userData + '</li>')
```

**安全替代**:
```javascript
// 安全 - 使用 text()
$('.container').text(userInput)
$('.list').append($('<li>').text(userData))
```

---

## Svelte

### {@html} 指令

**Grep 搜索**: `\{@html`

**风险模式**:
```svelte
<!-- 严重 - 直接渲染未经消毒的数据 -->
<div>{@html message.content}</div>
```

**安全替代**:
```svelte
<!-- 安全 - 文本插值自动转义 -->
<div>{message.content}</div>

<!-- 安全 - 使用 DOMPurify -->
<div>{@html sanitize(message.content)}</div>
```

---

## Vanilla JavaScript

### DOM 操作

| 模式 | Grep | 风险 | 安全替代 |
|------|------|------|---------|
| innerHTML | `\.innerHTML\s*=` | 高 | `textContent` |
| outerHTML | `\.outerHTML\s*=` | 高 | DOM API |
| insertAdjacentHTML | `\.insertAdjacentHTML\s*\(` | 高 | `insertAdjacentElement` |
| document.write | `document\.write(?:ln)?\s*\(` | 严重 | DOM API |
| eval | `eval\s*\(` | 严重 | `JSON.parse` 或重构 |
| Function | `new\s+Function\s*\(` | 严重 | 重构代码 |
| setTimeout 字符串 | `setTimeout\s*\(\s*["']` | 高 | `setTimeout(fn, ms)` |

---

## 通用推荐

### HTML 消毒库

对所有不可信 HTML 内容，推荐使用以下消毒库之一：

| 库 | 安装 | 用法 |
|---|------|------|
| DOMPurify | `npm install dompurify` | `DOMPurify.sanitize(html)` |
| sanitize-html | `npm install sanitize-html` | `sanitizeHtml(html, options)` |
| xss | `npm install xss` | `filterXSS(html)` |

### 消毒配置建议

```javascript
import DOMPurify from 'dompurify'

// 最严格配置 — 仅允许基础格式标签
const CLEAN = DOMPurify.sanitize(html, {
  ALLOWED_TAGS: ['p', 'b', 'i', 'em', 'strong', 'br', 'ul', 'ol', 'li', 'h3', 'h4'],
  ALLOWED_ATTR: [],
  FORBID_TAGS: ['style', 'script', 'iframe', 'form', 'input', 'button', 'object', 'embed'],
  FORBID_ATTR: ['onerror', 'onload', 'onclick', 'onmouseover', 'style', 'srcdoc'],
})
```

---

## 审核时的检查流程

1. **Grep 搜索框架特定模式**（并行执行）:
   - `v-html` (Vue)
   - `dangerouslySetInnerHTML` (React)
   - `innerHTML` (Angular / Vanilla)
   - `{@html` (Svelte)
   - `\.html\s*\(` (jQuery)
   - `bypassSecurityTrust` (Angular)

2. **对每个匹配项**:
   - 使用 `Read` 查看上下文（前后 10 行）
   - 判断数据来源：API 响应 / 用户输入 / 硬编码 / 受控数据
   - 检查是否有消毒处理（DOMPurify / 自定义过滤）

3. **记录发现**:
   - 文件路径:行号
   - 数据来源
   - 是否有消毒
   - 风险等级