# 0001 · AI 处理中枢拆为独立项目（D1–D5）

- 日期：2026-06-16
- 影响项目：`xiaobao`、`ai`
- 状态：已拍板（Owner 二轮讨论收敛）
- 详细真源：xiaobao 项目 `docs/progress/ad-hoc/2026-06-16-spike-langgraph-agent-hub-proposal.md` §12
- 契约产物：[../contracts/news-l1.md](../contracts/news-l1.md)

## 背景

niuma-cheng 新闻平台需要把 L1（LLM 推理）能力从平台主进程中解耦。经 Owner 二轮讨论，拍定以下方向，并据此创建独立的 `ai` 项目。

## 决策（D1–D5）

| 编号 | 决策 |
|------|------|
| D1 | 评分方案：保留既有评分设计，旧实现代码废弃，由中枢重做 |
| D2 | AI 能力解耦为**独立服务**（独立项目 `niuma-cheng-ai`），而非内嵌平台进程 |
| D3 | 技术栈分工：平台 `xiaobao` 用 Node；中枢 `ai` 用 Python（FastAPI + LangGraph） |
| D4 | 调用模型：**异步**（平台投递任务 / 中枢处理，`tasks` 为平台业务真源，run 仅处理单条证据） |
| D5 | 中枢定位：承载**平台的多种 AI 能力**，不做泛化的多项目通用平台 |

## 对各项目的影响

- `xiaobao`：v0.6 可先收尾、暂不启用 LLM/Agent 链路；评分**加权** `score_total` 留在平台侧；通过 `news-l1` 契约调用中枢。
- `ai`：作为独立仓库承载 L1 处理（四维原始评分 + 标签 + 摘要 + 翻译 + 按需工具）；需在自身仓库 Bootstrap 团队工作流（见 [../STATUS.md#ai-bootstrap](../STATUS.md)）。

## 附注

- 4 核 8G 服务器实测可承载（前提：外部 API 推理）。
- 两类新闻的产品更正属 xiaobao 产品范围，由 PM 在 v0.6.1 PRD 收口，不在本跨项目决策内。
