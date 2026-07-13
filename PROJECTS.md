# 项目目录

> 本文件是跨项目目录真源。每个项目的仓库、职责、当前入口和关联沟通文档以此为准。
> 沟通文档必须双向链接，避免孤儿记录。

## xiaobao — 牛马成新闻平台

| 字段 | 内容 |
|------|------|
| 项目 id | `xiaobao` |
| 名称 | 牛马成新闻平台 |
| 技术栈 | Node.js + Fastify + Vue |
| 仓库 | `git@github.com:huiyiyouck/niuma-cheng-xiaobao.git` |
| 职责边界 | 信息源管理、抓取调度、L0 分类、新闻展示、评分加权（`score_total`）；**不**做 L1 LLM 推理 |
| 当前入口 | 项目内 `docs/progress/INDEX.md`（当前 v0.6.1 迭代，实现 R1，设计文档已定稿） |
| 关联项目 | `ai`（调用方 → 服务方） |
| 沟通文档 | 见 [REQUESTS.md](REQUESTS.md)（REQ-001 已关闭；REQ-003 待 ai · PM 评估承接） |

## ai — AI 处理中枢（Agent Hub）

| 字段 | 内容 |
|------|------|
| 项目 id | `ai` |
| 名称 | AI 处理中枢 / Agent Hub |
| 技术栈 | Python + FastAPI + LangGraph |
| 仓库 | `git@github.com:huiyiyouck/niuma-cheng-ai.git`（本地 `~/Project/niuma-cheng-ai`） |
| 定位 | niuma-cheng **生态内部通用** AI 处理中枢（Agent Hub）：生态内多项目未来均可调用同一 AI 处理服务，多调用方预留；非对外泛化平台。见 [decisions/0002](decisions/0002-ai-hub-ecosystem-positioning.md)（supersede 0001-D5） |
| 职责边界 | 首落地 xiaobao news-l1：四维评分（`score`+`reason`）、标签、摘要、翻译、按需工具调用（KB 检索 / 链接读取 / Web 搜索）；**不**做评分加权 `score_total`（留在 xiaobao）。v0.1 仅实现 news-l1 一个 task-type，多调用方/多任务为预留扩展点 |
| 当前入口 | 项目内 `docs/progress/INDEX.md`（**已接入团队工作流**，v0.1 已交付并关闭（2026-07-04，`v0.1-summary.md`）；当前迭代：无） |
| 关联项目 | `xiaobao`（服务方 ← 调用方） |
| 沟通文档 | 见 [REQUESTS.md](REQUESTS.md)（REQ-001 已关闭；REQ-003 待 ai · PM 评估承接） |

## agent-workflow — AI 助手团队工作流真源

| 字段 | 内容 |
|------|------|
| 项目 id | `agent-workflow` |
| 名称 | AI 助手团队工作流真源（内部定位：一人公司 AI 组织操作架构·SOP 真源） |
| 技术栈 | Markdown + shell scripts |
| 仓库 | `git@github.com:huiyiyouck/agent-workflow.git`（本地 `~/Project/agent-workflow`） |
| 职责边界 | 一人公司 AI 组织操作架构 SOP 的单一真源（对外品牌：AI 开发团队工作流）；工作流入口、baseline、templates、安装/同步脚本；**只承接基线修正提案（BCR）**，不承接业务功能 / 接口契约 |
| 当前入口 | 真源仓库 `docs/ROADMAP.md`（P8 基线修正提案走 coordination 管理，自举中） |
| 关联项目 | `xiaobao`、`ai`、`workboard` 等已接入 agent-workflow 的项目 |
| 沟通文档 | 无固定 communications；基线修正提案见 [REQUESTS.md#基线修正提案池](REQUESTS.md#基线修正提案池) |

## workboard — 跨项目 Agent 工作看板

| 字段 | 内容 |
|------|------|
| 项目 id | `workboard` |
| 名称 | 跨项目 Agent 工作看板 |
| 技术栈 | React 18 + Vite + Tailwind v4 + shadcn（前端，对齐小报）+ 本地 Node 聚合后端（只读 Markdown/config） |
| 仓库 | `git@github.com:huiyiyouck/niuma-cheng-workboard.git`（本地 `~/Project/niuma-cheng-workboard`） |
| 职责边界 | 只读聚合各项目团队工作流状态、接入诊断、跨项目需求与阻塞；**不回写**被监控项目 |
| 当前入口 | 项目内 `docs/progress/INDEX.md`（**已接入 agent-workflow @66c1e1a**；**v0.1 已上线** 生产 `workboard.huiyiyou.cloud`；定位仍为只读看板，"管理中枢"为 v0.2 未来方向、未落地） |
| 关联项目 | `xiaobao` / `ai` / `coordination` 等（只读聚合对象） |
| 沟通文档 | 无（按需求承接后建 `communications/{REQ-id}-{短名}.md`） |
