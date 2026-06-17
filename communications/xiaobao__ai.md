# xiaobao ↔ ai 沟通记录

- 参与项目：`xiaobao`, `ai`
- 关系类型：调用方（xiaobao）/ 服务方（ai），共享 HTTP 契约
- 契约真源：[../contracts/news-l1.md](../contracts/news-l1.md)
- 当前状态入口：[../STATUS.md#news-l1-contract](../STATUS.md)
- 最近更新：2026-06-17

## 关系概述

`xiaobao` 新闻平台在 L1 处理阶段，通过 `POST /v1/runs/news-l1` 调用 `ai` Agent Hub，传入原始证据（`L1Input`），取回结构化处理结果（`RunResponse.output: L1Output`）。

职责切分：

- `xiaobao`：信息源、抓取、L0 分类、展示、评分**加权**（`score_total`）。
- `ai`：L1 推理 —— 四维**原始**评分、标签、摘要、翻译、按需工具调用。

本文件有两个作用，分两区维护：**A 需求沟通**（xiaobao 提、ai 承接）+ **B 联调沟通**（需求确立后的对接与调试）。

---

## A. 需求沟通（xiaobao 提 → ai 承接）

> xiaobao 向 ai 提出的能力需求台账。每条需求落「提出 → 承接 → 关闭」全过程。
> 需求一旦涉及接口/字段，对应契约以 [../contracts/](../contracts/) 为单一真源。

| 需求 id | 提出方 | 需求 | 承接方 | 对应契约 | 状态 |
|---------|--------|------|--------|----------|------|
| REQ-001 | xiaobao | 新闻 L1 处理：四维原始评分 + 五类标签 + 摘要 + 翻译 + 按需工具调用 | ai | [news-l1 v1](../contracts/news-l1.md) | ✅ 已承接并联调通过 |

### REQ-001 · 新闻 L1 处理

- 提出：xiaobao（v0.6 实现阶段，L1 推理从平台主进程解耦）
- 承接：ai 以 `POST /v1/runs/news-l1` 提供，LangGraph 固定流水线
- 边界：ai 只产**原始**四维评分（`score`+`reason`），加权 `score_total` 留 xiaobao
- 状态：契约 v1 定稿，端到端联调通过（详见下方 B 区 2026-06-16 记录）

---

## B. 联调沟通（需求确立后的对接 / 调试）

> 接口对接、字段对齐、调试问题、版本跟进等持续往来，倒序排列。

### 2026-06-17 · 契约真源迁移

- xiaobao WM 在 coordination 建立 `news-l1` v1 契约真源，取代此前散落在 xiaobao spike 提案中的临时真源。
- 两侧当前实现与 v1 一致，无需立即改代码。
- 约定：后续任一侧改契约，先改 `contracts/news-l1.md`，再改代码，CHANGELOG 记一行。

### 2026-06-16 · 端到端打通验证

- `ai` 端 `news-l1` 流水线本机冒烟通过；`xiaobao` `news_test` 库单条真实 raw_item 经 `ai` 处理端到端成功（task `6de4139b-...` succeeded，`processed_news` 写入，`tags_v2.processing` 含 `engine:agent`）。
- 待观察：3–5 条小批量并发、耗时、失败回退与成本（归属 xiaobao Developer）。

---

## 待跟进

| 事项 | 归属 | 状态 |
|------|------|------|
| ai 配置 git remote 并 Bootstrap 团队工作流 | Owner / ai 会话 | 待启动（见 [../STATUS.md#ai-bootstrap](../STATUS.md)） |
| REQ-001 news-l1 小批量观察 | xiaobao Developer | 进行中 |
