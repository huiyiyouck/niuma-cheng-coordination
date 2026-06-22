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
| 当前入口 | 项目内 `docs/progress/INDEX.md`（当前 v0.6 标准迭代，实现阶段联调精修） |
| 关联项目 | `ai`（调用方 → 服务方） |
| 沟通文档 | [communications/xiaobao__ai.md](communications/xiaobao__ai.md) |

## ai — AI 处理中枢（Agent Hub）

| 字段 | 内容 |
|------|------|
| 项目 id | `ai` |
| 名称 | AI 处理中枢 / Agent Hub |
| 技术栈 | Python + FastAPI + LangGraph |
| 仓库 | `git@github.com:huiyiyouck/niuma-cheng-ai.git`（本地 `~/Project/niuma-cheng-ai`） |
| 职责边界 | L1 新闻处理：四维评分（`score`+`reason`）、标签、摘要、翻译、按需工具调用（KB 检索 / 链接读取 / Web 搜索）；**不**做评分加权 `score_total`（留在 xiaobao） |
| 当前入口 | 项目内 `docs/progress/INDEX.md`（**已接入团队工作流**，工作台已初始化，未进入角色工作；当前迭代：无） |
| 关联项目 | `xiaobao`（服务方 ← 调用方） |
| 沟通文档 | [communications/xiaobao__ai.md](communications/xiaobao__ai.md) |

## agent-workflow — 团队工作流真源

| 字段 | 内容 |
|------|------|
| 项目 id | `agent-workflow` |
| 名称 | AI 助手团队工作流真源 |
| 技术栈 | Markdown + shell scripts |
| 仓库 | `git@github.com:huiyiyouck/agent-workflow.git`（本地 `~/Project/agent-workflow`） |
| 职责边界 | 工作流入口、baseline、templates、安装/同步脚本的单一真源；**只承接基线修正提案（BCR）**，不承接业务功能 / 接口契约 |
| 当前入口 | 真源仓库 `docs/ROADMAP.md`（P8 基线修正提案走 coordination 管理，自举中） |
| 关联项目 | `xiaobao`、`ai` 等已接入 agent-workflow 的业务项目 |
| 沟通文档 | 无固定 communications；基线修正提案见 [REQUESTS.md#基线修正提案池](REQUESTS.md#基线修正提案池) |
