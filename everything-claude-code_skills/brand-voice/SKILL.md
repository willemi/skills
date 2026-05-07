---
name: brand-voice
description: Build a source-derived writing style profile from real posts, essays, launch notes, docs, or site copy, then reuse that profile across content, outreach, and social workflows. Use when the user wants voice consistency without generic AI writing tropes.
origin: ECC
---

# 品牌声音

从真实来源材料构建持久的声音档案，然后在各处使用该档案，而不是从头重新推导风格或默认为通用 AI 文案。

## 何时激活

- 用户想要特定声音的内容或外联
- 为 X、LinkedIn、邮件、发布帖子、主题串或产品更新写作
- 跨渠道调整已知作者的语气
- 现有内容通道需要可重用的风格系统而非一次性模仿

## 来源优先级

使用可用的最强真实来源集，按此顺序：

1. 近期的原创 X 帖子和主题串
2. 文章、散文、备忘录、发布笔记或通讯
3. 有效的真实外发邮件或 DM
4. 产品文档、更新日志、README 框架和网站文案

不要使用通用平台范例作为来源材料。

## 收集工作流

1. 尽可能收集 5 到 20 个代表性样本。
2. 优先使用近期材料而非旧材料，除非用户说较旧的写作更典范。
3. 如果来源集明显分开，分离"公开发布声音"与"私人工作声音"。
4. 如果有可用的实时 X 访问，在起草前使用 `x-api` 拉取近期原创帖子。
5. 如果网站文案重要，包含当前 ECC 登录页面和 repo/plugin 框架。

## 提取内容

- 节奏和句子长度
- 压缩 vs 解释
- 大写规范
- 括号使用
- 问题频率和目的
- 主张的尖锐程度
- 数字、机制或收据的频繁程度
- 过渡如何工作
- 作者从不做什么

## 输出契约

产生一个可重用的 `VOICE PROFILE` 块，下游技能可直接消费。使用 [references/voice-profile-schema.md](references/voice-profile-schema.md) 中的模式。

保持配置文件结构化和足够短，可在会话上下文中重用。重点不是文学批评。重点是操作重用。

## Affaan / ECC 默认值

如果用户想要 Affaan / ECC 声音且实时来源有限，从此开始除非较新的来源材料覆盖它：

- 直接、压缩、具体
- 具体细节、机制、收据和数字胜过形容词
- 括号用于限定、缩小或过度澄清
- 大写是常规的，除非有真正理由打破
- 问题罕见，不应作为诱饵使用
- 语气可以是尖锐、生硬、怀疑或平淡的
- 过渡应感觉是有代价的，而非 smoothed over

## 硬性禁止

删除并重写以下任何内容：

- 虚假的好奇心钩子
- "不是 X，只是 Y"
- "没有废话"
- 强制小写
- LinkedIn 思想领袖节奏
- 诱饵问题
- "兴奋分享"
- 通用创始人旅程填充
- 老套的括号

## 持久化规则

- 在同一会话的 Related tasks 中重用最新确认的 `VOICE PROFILE`。
- 如果用户要求持久化制品，在请求的工作区位置或内存表面保存档案。
- 不要创建存储个人声音指纹的仓库跟踪文件，除非用户明确要求。

## 下游使用

在以下使用前或内部使用本技能：

- `content-engine`
- `crosspost`
- `lead-intelligence`
- 文章或发布写作
- 跨 X、LinkedIn 和邮件的冷热外联

如果另一个技能已有部分声音捕获部分，本技能是真实来源。
