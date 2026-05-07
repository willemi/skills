---
name: benchmark
description: Use this skill to measure performance baselines, detect regressions before/after PRs, and compare stack alternatives.
origin: ECC
---

# Benchmark — 性能基线与回归检测

## 何时使用

- PR 之前和之后测量性能影响
- 为项目设置性能基线
- 用户报告"感觉慢"时
- 发布之前——确保达到性能目标
- 将你的栈与备选方案比较

## 工作原理

### 模式 1：页面性能

通过 browser MCP 测量真实浏览器指标：

```
1. 导航到每个目标 URL
2. 测量 Core Web Vitals：
   - LCP（Largest Contentful Paint）——目标 < 2.5s
   - CLS（Cumulative Layout Shift）——目标 < 0.1
   - INP（Interaction to Next Paint）——目标 < 200ms
   - FCP（First Contentful Paint）——目标 < 1.8s
   - TTFB（Time to First Byte）——目标 < 800ms
3. 测量资源大小：
   - 总页面重量（目标 < 1MB）
   - JS 包大小（目标 < 200KB gzipped）
   - CSS 大小
   - 图像权重
   - 第三方脚本重量
4. 统计网络请求数
5. 检查渲染阻塞资源
```

### 模式 2：API 性能

对 API 端点进行基准测试：

```
1. 每个端点命中 100 次
2. 测量：p50、p95、p99 延迟
3. 追踪：响应大小、状态码
4. 负载测试：10 个并发请求
5. 对比 SLA 目标
```

### 模式 3：构建性能

测量开发反馈循环：

```
1. 冷构建时间
2. 热重载时间（HMR）
3. 测试套件时长
4. TypeScript 检查时间
5. Lint 时间
6. Docker 构建时间
```

### 模式 4：之前/之后对比

在更改之前和之后运行以测量影响：

```
/benchmark baseline    # 保存当前指标
# ... 进行更改 ...
/benchmark compare     # 与基线对比
```

输出：
```
| 指标 | 之前 | 之后 | 增量 | 结果 |
|------|------|------|------|------|
| LCP | 1.2s | 1.4s | +200ms | 警告：WARN |
| Bundle | 180KB | 175KB | -5KB | ✓ 更好 |
| Build | 12s | 14s | +2s | 警告：WARN |
```

## 输出

将基线存储在 `.ecc/benchmarks/` 中为 JSON。Git 跟踪以便团队共享基线。

## 集成

- CI：每个 PR 上运行 `/benchmark compare`
- 与 `/canary-watch` 搭配进行部署后监控
- 与 `/browser-qa` 搭配进行完整发布前检查清单
