# 跨项目需求提报中心

> 这是 niuma-cheng 生态的**需求池**。任一项目的任一角色都可以把跨项目需求提报到这里。
> **提报时不指定由哪个项目承接**——各项目的 PM/Architect 按本项目定位主动评估认领，Owner 也可直接指派。
> 承接之后才在提出方↔承接方之间建立 `communications/{a}__{b}.md` 做协作与联调。
> 生命周期：已提报 → 评估中 → 已承接（/已拒绝）→ 开发中（转入迭代）→ 联调中 → 已关闭。
> 规则见各项目 `docs/baseline/cross-project-collaboration.md` §跨项目需求流转。

## 需求池

| 需求 id | 提出方 | 内容 | 承接方 | 转入迭代 | 状态 | 沟通文档 |
|---------|--------|------|--------|----------|------|----------|
| REQ-001 | xiaobao · Developer | 新闻 L1 处理：四维原始评分 + 五类标签 + 摘要 + 翻译 + 按需工具调用 | ai（承接人待 Bootstrap 后补登 PM/Architect） | ai 待立项 | 联调中 | [communications/xiaobao__ai.md](communications/xiaobao__ai.md) |

---

## REQ-001 · 新闻 L1 处理

- 提出方：xiaobao · Developer
- 内容：把 L1 推理从新闻平台主进程解耦，由独立服务承载 —— 产出四维**原始**评分（`score`+`reason`）、五类标签、摘要、翻译、按需工具调用（KB 检索 / 链接读取 / Web 搜索）
- 边界：承接方只产原始评分，加权 `score_total` 留 xiaobao
- 承接方：`ai`（AI 处理中枢）
- 承接说明：本需求在跨项目需求流转机制建立**之前**已既成事实推进并跑通；ai 项目 Bootstrap 团队工作流后，由其 PM/Architect 补登正式承接留痕
- 转入迭代：ai 待立项（ai 尚未 Bootstrap）
- 当前状态：联调中 —— 契约 [news-l1 v1](contracts/news-l1.md) 定稿，端到端单条已通过，3–5 条小批量观察进行中
- 联调记录：见 [communications/xiaobao__ai.md](communications/xiaobao__ai.md) B 区
