---
name: ai-first-engineering
description: Engineering operating model for teams where AI agents generate a large share of implementation output.
origin: ECC
---

# AI 优先工程

为 AI Agent 生成大量实施输出的团队设计流程、审查和架构时，使用本技能。

## 流程转变

1. 规划质量比打字速度更重要。
2. 评估覆盖比轶事信心更重要。
3. 审查重点从语法转移到系统行为。

## 架构要求

优先考虑对 Agent 友好的架构：
- 显式边界
- 稳定契约
- 类型化接口
- 确定性测试

避免隐式行为分散在隐藏约定中。

## AI 优先团队的代码审查

审查：
- 行为回归
- 安全假设
- 数据完整性
- 失败处理
- 推出安全

自动化格式/lint 已经强制执行样式时，最小化花费在样式问题上的时间。

## 招聘和评估信号

强大的 AI 优先工程师：
- 干净地分解模糊的工作
- 定义可测量的验收标准
- 产生高信号提示词和评估
- 在交付压力下执行风险控制

## 测试标准

提高生成代码的测试门槛：
- 需要受影响的域有回归覆盖
- 显式的边缘情况断言
- 接口边界的集成检查
