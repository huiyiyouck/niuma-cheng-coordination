# REQ-001 · news-l1 跨项目协作沟通

- 参与项目：`xiaobao`, `ai`
- 相关需求：REQ-001（状态见 [../REQUESTS.md](../REQUESTS.md)）
- 定位：REQ-001 被承接之后的协作与联调沟通
- 契约真源：[../contracts/news-l1.md](../contracts/news-l1.md)（涉及接口/字段时以此为准）
- 当前状态入口：[../STATUS.md#news-l1-contract](../STATUS.md)
- 最近更新：2026-06-29

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

### 2026-06-29 · [REQ-001] ai 启动 v0.1 实现，提出对 xiaobao 的能力诉求

- `ai` 已据 REQ-002 架构结论启动 v0.1 标准迭代（news-l1 stub→真实，PRD R1：ai `docs/progress/iterations/v0.1-prd.md`）。
- 实现 news-l1「按需工具调用」时，三类工具后端分工 Owner 2026-06-29 已拍定，**唯一需 xiaobao 提供的是库内检索 KB search**（新闻库在 xiaobao，ai 无直接访问）：
  - **库内检索 KB search（需 xiaobao 提供）**：请 xiaobao 定 —— a) 预取填充 `kb_results`（契约已有字段）加量/加质；还是 b) 提供实时检索接口供 ai 按需调。
  - 链接读取 link read：**ai 自行 HTTP 抓取**，不需 xiaobao。
  - Web 搜索 web search：**Owner 提供搜索 API key、ai 自配调用**，不需 xiaobao。
- 边界：不改 `news-l1` 契约则优先复用现有预取字段（`kb_results`/`link_content`/`search_summary`）；若需新接口/字段，另走 [../contracts/](../contracts/) 契约变更流程。
- 对 v0.1 影响：ai v0.1 的 AC-5（按需工具调用）先做 ai 能独立完成的部分（消费预取 + `needs_context` 标记 + 自抓 link/自配 web）；**主动 KB 检索依赖本诉求的 xiaobao 响应**，落地后补，不阻塞 v0.1 其余范围。
- 待 xiaobao（PM/Architect/Developer）响应三类工具分工。

### 2026-06-22 · [REQ-001] ai PM 正式承接

- `ai` 项目完成 Bootstrap 并立项后，由 PM（ck）评估正式承接 REQ-001，补齐「正规提报（xiaobao · Developer）→ 承接（ai · PM）」留痕闭环。
- 真实 L1 处理（评分/标签/摘要/翻译由 stub 转真实）规划转入 ai v0.1 标准迭代（待启动，迭代门禁下一步启动）；管道层联调状态不变（仍「联调中」）。

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
| ai 配置 git remote 并 Bootstrap 团队工作流 | Owner / ai 会话 | 已完成（2026-06-21，见 [../STATUS.md#ai-bootstrap](../STATUS.md)） |
| REQ-001 承接留痕补登 | ai PM/Architect | 已完成（2026-06-22，ai PM ck 承接） |
| REQ-001 news-l1 小批量观察 | xiaobao Developer | 进行中 |
| ai news-l1 库内检索 KB search 能力需 xiaobao 提供（预取加量 vs 实时接口；link/web 已由 ai 自抓 + Owner key 解决） | xiaobao（PM/Architect/Dev） | 待响应（2026-06-29 ai 提出） |
