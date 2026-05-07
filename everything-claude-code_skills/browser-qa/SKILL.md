---
name: browser-qa
description: Use this skill to automate visual testing and UI interaction verification using browser automation after deploying features.
origin: ECC
---

# Browser QA — 自动化视觉测试与交互

## 何时使用

- 部署功能到 staging/preview 之后
- 需要验证跨页面 UI 行为时
- 发布之前——确认布局、表单、交互确实工作
- 审查触及前端代码的 PR 时
- 无障碍审计和响应式测试

## 工作原理

使用浏览器自动化 MCP（claude-in-chrome、Playwright 或 Puppeteer）像真实用户一样与线上页面交互。

### 阶段 1：冒烟测试
```
1. 导航到目标 URL
2. 检查控制台错误（过滤噪声：分析、第三方）
3. 验证网络请求中没有 4xx/5xx
4. 在桌面 + 移动视口截图首屏
5. 检查 Core Web Vitals：LCP < 2.5s, CLS < 0.1, INP < 200ms
```

### 阶段 2：交互测试
```
1. 点击每个导航链接——验证无死链接
2. 用有效数据提交表单——验证成功状态
3. 用无效数据提交表单——验证错误状态
4. 测试认证流程：登录 → 受保护页面 → 登出
5. 测试关键用户旅程（结账、注册、搜索）
```

### 阶段 3：视觉回归
```
1. 在 3 个断点（375px、768px、1440px）截图关键页面
2. 与基线截图比较（如果已存储）
3. 标记 > 5px 的布局偏移、缺失元素、溢出
4. 如适用检查暗色模式
```

### 阶段 4：无障碍
```
1. 在每个页面上运行 axe-core 或等效工具
2. 标记 WCAG AA 违规（对比度、标签、焦点顺序）
3. 验证键盘导航端到端工作
4. 检查屏幕阅读器地标
```

## 输出格式

```markdown
## QA 报告 — [URL] — [时间戳]

### 冒烟测试
- 控制台错误：0 个关键，2 个警告（分析噪声）
- 网络：全部 200/304，无失败
- Core Web Vitals：LCP 1.2s ✓, CLS 0.02 ✓, INP 89ms ✓

### 交互
- [✓] 导航链接：12/12 工作
- [✗] 联系表单：无效邮箱缺少错误状态
- [✓] 认证流程：登录/登出工作

### 视觉
- [✗] Hero 区域在 375px 视口溢出
- [✓] 暗色模式：所有页面一致

### 无障碍
- 2 个 AA 违规：hero 图片缺少 alt 文本，页脚链接对比度低

### 结论：修复后发布（2 个问题，0 个阻塞）
```

## 集成

与任何 browser MCP 配合使用：
- `mChild__claude-in-chrome__*` 工具（首选——使用你的实际 Chrome）
- 通过 `mcp__browserbase__*` 的 Playwright
- 直接 Puppeteer 脚本

与 `/canary-watch` 搭配进行部署后监控。
