# xiaobao ↔ ai 协作沟通

- 参与项目：`xiaobao`, `ai`
- 定位：`xiaobao` 与 `ai` **承接需求之后**的协作与联调沟通（需求源头见 [../REQUESTS.md](../REQUESTS.md)）
- 契约真源：[../contracts/news-l1.md](../contracts/news-l1.md)（涉及接口/字段时以此为准）
- 当前状态入口：[../STATUS.md#news-l1-contract](../STATUS.md)
- 最近更新：2026-06-17

## 关系概述

`ai` 承接了 `xiaobao` 提报的 L1 处理需求。技术上：`xiaobao` 通过 `POST /v1/runs/news-l1` 调用 `ai`，传入原始证据（`L1Input`），取回处理结果（`RunResponse.output: L1Output`）。

> 调用方向是 xiaobao→ai，但需求方向双向 —— `ai` 同样可以向 `xiaobao` 提需求（如调整入参、补字段、改投递节奏），这类需求同样先进 [../REQUESTS.md](../REQUESTS.md)。

职责切分：

- `xiaobao`：信息源、抓取、L0 分类、展示、评分**加权**（`score_total`）。
- `ai`：L1 推理 —— 四维**原始**评分、标签、摘要、翻译、按需工具调用。

## 承接的需求

| 需求 id | 内容 | 状态 | 详情 |
|---------|------|------|------|
| REQ-001 | 新闻 L1 处理 | 联调中 | [../REQUESTS.md#req-001](../REQUESTS.md) |

## 联调沟通

> 承接后的接口对接 / 字段对齐 / 调试 / 版本跟进，倒序排列，条目标注所属需求 id。

### 2026-06-17 · [REQ-001] 契约真源迁移

- xiaobao WM 在 coordination 建立 `news-l1` v1 契约真源，取代此前散落在 xiaobao spike 提案中的临时真源。
- 两侧当前实现与 v1 一致，无需立即改代码。
- 约定：后续任一侧改契约，先改 `contracts/news-l1.md`，再改代码，CHANGELOG 记一行。

### 2026-06-16 · [REQ-001] 端到端打通验证

- `ai` 端 `news-l1` 流水线本机冒烟通过；`xiaobao` `news_test` 库单条真实 raw_item 经 `ai` 处理端到端成功（task `6de4139b-...` succeeded，`processed_news` 写入，`tags_v2.processing` 含 `engine:agent`）。
- 待观察：3–5 条小批量并发、耗时、失败回退与成本（归属 xiaobao Developer）。

## 待跟进

| 事项 | 归属 | 状态 |
|------|------|------|
| ai 配置 git remote 并 Bootstrap 团队工作流 | Owner / ai 会话 | 待启动（见 [../STATUS.md#ai-bootstrap](../STATUS.md)） |
| REQ-001 承接留痕补登 | ai PM/Architect | 待 ai Bootstrap |
| REQ-001 news-l1 小批量观察 | xiaobao Developer | 进行中 |
