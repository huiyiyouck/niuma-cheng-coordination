# xiaobao ↔ ai 协作主纲

- 参与项目：`xiaobao`, `ai`
- 定位：本文件是 `xiaobao` 与 `ai` 之间**所有跨项目协作的总账与管理入口**，双向对等。任一方都可向另一方提需求；每条需求落「需求沟通 → 需求下的联调沟通」全过程。
- 契约真源：[../contracts/news-l1.md](../contracts/news-l1.md)（涉及接口/字段时以此为准，本文件只引用不复制）
- 当前状态入口：[../STATUS.md#news-l1-contract](../STATUS.md)
- 最近更新：2026-06-17

## 两项目关系（事实，非需求方向）

技术上：`xiaobao` 通过 `POST /v1/runs/news-l1` 调用 `ai` 的 L1 处理能力。

> 注意：HTTP 调用方向是 xiaobao→ai，但**需求方向是双向的**——`ai` 同样可以向 `xiaobao` 提需求（如调整入参、补充字段、改变投递节奏）。本文件不把任何一方固定为「需求方」或「承接方」。

职责切分：

- `xiaobao`：信息源、抓取、L0 分类、展示、评分**加权**（`score_total`）。
- `ai`：L1 推理 —— 四维**原始**评分、标签、摘要、翻译、按需工具调用。

---

## A. 需求沟通

> 两项目之间提出的协作需求台账。任一方可提，落「提出 → 确认 → 关闭」。涉及接口/字段的需求，契约以 [../contracts/](../contracts/) 为单一真源。

| 需求 id | 提出方 → 面向方 | 需求 | 对应契约 | 状态 |
|---------|----------------|------|----------|------|
| REQ-001 | xiaobao → ai | 新闻 L1 处理：四维原始评分 + 五类标签 + 摘要 + 翻译 + 按需工具调用 | [news-l1 v1](../contracts/news-l1.md) | ✅ 已确认并联调通过 |

### REQ-001 · 新闻 L1 处理

- 提出方 → 面向方：xiaobao → ai
- 内容：L1 推理从平台主进程解耦，由 ai 以 `POST /v1/runs/news-l1` 承载
- 边界：ai 只产**原始**四维评分（`score`+`reason`），加权 `score_total` 留 xiaobao
- 状态：契约 v1 定稿，端到端联调通过（详见 B 区 2026-06-16）

---

## B. 联调沟通

> 每条需求确立后的对接 / 字段对齐 / 调试 / 版本跟进，倒序排列。条目可标注归属的需求 id。

### 2026-06-17 · [REQ-001] 契约真源迁移

- xiaobao WM 在 coordination 建立 `news-l1` v1 契约真源，取代此前散落在 xiaobao spike 提案中的临时真源。
- 两侧当前实现与 v1 一致，无需立即改代码。
- 约定：后续任一侧改契约，先改 `contracts/news-l1.md`，再改代码，CHANGELOG 记一行。

### 2026-06-16 · [REQ-001] 端到端打通验证

- `ai` 端 `news-l1` 流水线本机冒烟通过；`xiaobao` `news_test` 库单条真实 raw_item 经 `ai` 处理端到端成功（task `6de4139b-...` succeeded，`processed_news` 写入，`tags_v2.processing` 含 `engine:agent`）。
- 待观察：3–5 条小批量并发、耗时、失败回退与成本（归属 xiaobao Developer）。

---

## 待跟进

| 事项 | 归属 | 状态 |
|------|------|------|
| ai 配置 git remote 并 Bootstrap 团队工作流 | Owner / ai 会话 | 待启动（见 [../STATUS.md#ai-bootstrap](../STATUS.md)） |
| REQ-001 news-l1 小批量观察 | xiaobao Developer | 进行中 |
