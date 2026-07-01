# 跨项目变更日志

> 记录跨项目重大事件、契约 breaking change、迁移提醒。
> 单项目内部迭代不在此记录。倒序排列（最新在上）。

## 2026-07-01

- **新增契约 `kb-search` v1** → [contracts/kb-search.md](contracts/kb-search.md)。用于 ai 在 news-l1 处理中按需反向调用 xiaobao 做库内新闻实时检索。非 breaking（首次登记）。影响项目：`xiaobao`、`ai`。
- **xiaobao 响应 REQ-001 联调诉求**：实现前端 `/debug/ai` 验收页、后端 `POST /v1/ai-debug/news-l1-runs`、AI Hub HTTP 客户端、`POST /v1/kb-search`。`news-l1` v1 契约不变。影响项目：`xiaobao`、`ai`。

## 2026-06-29

- **`ai` 定位升级为生态内部通用 AI 处理中枢**（Owner 拍板）→ [decisions/0002](decisions/0002-ai-hub-ecosystem-positioning.md)，supersede 0001-D5（仅 D5，D1–D4 仍有效）。生态内多项目未来均可调用同一 AI 服务，多调用方预留；v0.1 仍只实现 news-l1 一个 task-type。非 breaking：xiaobao 调用关系 / `score_total` 加权 / `news-l1` 契约边界均不变。已同步 `PROJECTS.md` ai 节，元信息台账待根索引订正。影响项目：`ai`、`xiaobao`。

## 2026-06-17

- **coordination 仓库初始化骨架**（xiaobao WM）。建立 `README.md` / `PROJECTS.md` / `STATUS.md` / `CHANGELOG.md` / `contracts/news-l1.md` / `communications/xiaobao__ai.md` / `decisions/`。
- **契约 `news-l1` v1 收编为单一真源** → [contracts/news-l1.md](contracts/news-l1.md)。从 `ai` 项目 `src/agent_hub/schemas.py` 迁移，与之前散落在 xiaobao spike 提案 §6/§7 的临时真源对齐。非 breaking（首次登记）。影响项目：`xiaobao`、`ai`，两侧当前实现与 v1 一致，无需立即跟进。
- **确立跨项目需求流转机制 + 需求提报中心**（xiaobao WM，Owner 确认）。新增 [REQUESTS.md](REQUESTS.md) 作为需求池：提报时**不指定承接方**，由项目 PM/Architect 主动认领或 Owner 直接指派；`communications/` 重新定位为**承接之后**的协作与联调（不再承载提报台账）。规则入各项目 baseline §跨项目需求流转。REQ-001 登记为既有需求（ai 承接、联调中）。

## 2026-06-16

- **niuma-cheng 生态从单项目扩为多项目**（来源：xiaobao Developer 提案 + WM 基线化）。新增 `ai` 项目（Python + FastAPI + LangGraph，initial commit `0ee6c9a`）作为独立 AI 处理中枢；与 `xiaobao` 通过 HTTP 契约 `news-l1` 耦合。详见 [decisions/0001-ai-hub-split.md](decisions/0001-ai-hub-split.md)。
