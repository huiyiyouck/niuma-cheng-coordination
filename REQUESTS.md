# 跨项目需求提报中心

> 这是 niuma-cheng 生态的**需求池**。任一项目的任一角色都可以把跨项目需求提报到这里。
> **提报时不指定由哪个项目承接**——各项目的 PM/Architect 按本项目定位主动评估认领，Owner 也可直接指派。
> 承接之后才在提出方↔承接方之间建立 `communications/{a}__{b}.md` 做协作与联调。
> 生命周期：已提报 → 评估中 → 已承接（/已拒绝）→ 开发中（转入迭代）→ 联调中 → 已关闭。
> 框架基线修正提案见下方「基线修正提案池（BCR）」；BCR 使用独立状态机，不走普通需求的联调生命周期。
> 规则见各项目 `docs/baseline/cross-project-collaboration.md` §跨项目需求流转。

## 需求池

| 需求 id | 提出方 | 内容 | 承接方 | 转入迭代 | 状态 | 沟通文档 |
|---------|--------|------|--------|----------|------|----------|
| REQ-001 | xiaobao · Developer | 新闻 L1 处理：四维原始评分 + 五类标签 + 摘要 + 翻译 + 按需工具调用 | ai · PM（ck） | ai v0.1（待启动） | 联调中 | [communications/xiaobao__ai.md](communications/xiaobao__ai.md) |

---

## REQ-001 · 新闻 L1 处理

- 提出方：xiaobao · Developer
- 内容：把 L1 推理从新闻平台主进程解耦，由独立服务承载 —— 产出四维**原始**评分（`score`+`reason`）、五类标签、摘要、翻译、按需工具调用（KB 检索 / 链接读取 / Web 搜索）
- 边界：承接方只产原始评分，加权 `score_total` 留 xiaobao
- 承接方：`ai`（AI 处理中枢）· PM（ck）
- 承接说明：本需求先于跨项目需求流转机制既成事实推进并打通管道；2026-06-22 `ai` 项目完成 Bootstrap 并立项后，由其 PM（ck）评估**正式承接**，补齐「正规提报（xiaobao · Developer）→ 承接（ai · PM）」留痕闭环
- 转入迭代：ai v0.1 标准迭代（待启动）—— 真实 L1 处理（评分/标签/摘要/翻译由 stub 转真实）规划进 v0.1，迭代门禁下一步启动
- 当前状态：联调中 —— 契约 [news-l1 v1](contracts/news-l1.md) 定稿，端到端单条已通过，3–5 条小批量观察进行中
- 联调记录：见 [communications/xiaobao__ai.md](communications/xiaobao__ai.md) B 区

---

## 基线修正提案池

> 本区登记 target = `agent-workflow` 的基线修正提案（BCR, Baseline Change Request）。
> BCR 由 Owner + agent-workflow 真源维护会话（General）评估、采纳和落地；下游项目只能提报，不直接修改本项目 `docs/baseline/`。
> 状态机：已提报 → 评估中 → 已采纳 / 部分采纳 / 已拒绝 / 转 v2 候选 → 已落地真源 → 回流中 → 已回流下游。

| BCR id | 提出方 | 摘要 | 影响范围 | 状态 | 真源评估记录 | 真源落地 commit | 回流清单 | 备注 |
|--------|--------|------|----------|------|--------------|----------------|----------|------|
| BCR-001 | agent-workflow · General（P8 自举） | 基线修正提案从 Owner 人肉带回改为 coordination BCR 池登记与追踪 | agent-workflow baseline；已接入工作流的下游项目同步规则 | 回流中（已落地真源） | Owner 已评审通过；agent-workflow PR #4 合并 P8 方案（`fe99ac3`） | `fc22e75`（merge 663f59b） | ai: 待 sync；xiaobao: 已回流（`c8c66ce`, 2026-06-22）；workboard: 不适用（未接入） | 自举样例：先登记 BCR，再改真源 baseline；自举例外仅限本次 P8 |
| BCR-002 | xiaobao（对齐真源时工作区留痕，2026-06-22） | communications 沟通文档命名轴：按项目对（真源现状）vs 按需求一份 | agent-workflow `cross-project-collaboration.md` §communications；回流后影响各下游同名章节 | 已提报 | 待评估 | — | 待评估后定 | 原以 ad-hoc 形式落 agent-workflow progress，按 P8 标准迁入本池，原文件已删 |

### BCR-001 · 基线修正提案走 coordination 管理

- 目标：把下游 `[基线修正提案]` 从口头带回真源，改为写入 coordination 的 BCR 池，形成有状态、可追溯、可回流的治理流程。
- 采纳依据：agent-workflow P8 方案已通过 PR #4 合并（`fe99ac3`）。
- 下一步：真源已落地（`fc22e75`，merge 663f59b，PR #5），xiaobao 已 sync 回流（`c8c66ce`，2026-06-22）；待 ai sync 回流后，全部完成再置「已回流下游」终态。

---

## BCR-002 · communications 命名轴（按项目对 vs 按需求）

- 提出方：xiaobao —— 对齐真源（P7 第 3 步）时，工作区有一处未提交改动把 `communications/` 命名轴从「按项目对」翻成「按需求一份」，覆盖前抢救留痕。
- 目标：评议是否调整 `cross-project-collaboration.md` §communications 的命名轴。
- 背景：真源现状为**按项目对** `communications/{a}__{b}.md`（一对项目共用一份，承载该对所有需求）；xiaobao 提案为**按需求** `communications/{REQ-id}-{短名}.md`（一个需求一份）。该改动带的「创建时机 / 职责分工」措辞与真源新版一致，说明是看过真源新思路后有意翻转命名轴，非旧版残留。

| 维度 | 按项目对（真源现状） | 按需求（xiaobao 提案） |
|------|---------------------|----------------------|
| 命名 | `{a}__{b}.md` | `{REQ-id}-{短名}.md` |
| 粒度 | 一对项目共用一份 | 一个需求一份 |
| 文件数 | 有界（最多 C(N,2) 对） | 随需求量线性增长 |
| 与 REQUESTS.md | 一份多 REQ，靠条目标注区分 | 与需求 id 一一对应，反查直达 |
| 长期可读性 | 单份越积越长、跨需求混杂 | 需求关闭即归档，边界清晰 |

- 倾向（供参考，非结论）：需求量小时按项目对更省心；需求量大、跨项目需求频繁时按需求更可维护。可折中为真源保留按项目对默认，文档点明「需求量大时可改按需求」，把轴的选择留给生态规模。
- 待评估 / 采纳 / 落地：Owner + agent-workflow 真源会话。
