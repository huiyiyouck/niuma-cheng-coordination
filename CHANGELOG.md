# 跨项目变更日志

> 记录跨项目重大事件、契约 breaking change、迁移提醒。
> 单项目内部迭代不在此记录。倒序排列（最新在上）。

## 2026-06-17

- **coordination 仓库初始化骨架**（xiaobao WM）。建立 `README.md` / `PROJECTS.md` / `STATUS.md` / `CHANGELOG.md` / `contracts/news-l1.md` / `communications/xiaobao__ai.md` / `decisions/`。
- **契约 `news-l1` v1 收编为单一真源** → [contracts/news-l1.md](contracts/news-l1.md)。从 `ai` 项目 `src/agent_hub/schemas.py` 迁移，与之前散落在 xiaobao spike 提案 §6/§7 的临时真源对齐。非 breaking（首次登记）。影响项目：`xiaobao`、`ai`，两侧当前实现与 v1 一致，无需立即跟进。

## 2026-06-16

- **niuma-cheng 生态从单项目扩为多项目**（来源：xiaobao Developer 提案 + WM 基线化）。新增 `ai` 项目（Python + FastAPI + LangGraph，initial commit `0ee6c9a`）作为独立 AI 处理中枢；与 `xiaobao` 通过 HTTP 契约 `news-l1` 耦合。详见 [decisions/0001-ai-hub-split.md](decisions/0001-ai-hub-split.md)。
