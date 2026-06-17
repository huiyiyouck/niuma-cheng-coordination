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
| 仓库 | 待 Owner 配置 git remote（当前仅本地 `~/Project/niuma-cheng-ai`，initial commit `0ee6c9a`） |
| 职责边界 | L1 新闻处理：四维评分（`score`+`reason`）、标签、摘要、翻译、按需工具调用（KB 检索 / 链接读取 / Web 搜索）；**不**做评分加权 `score_total`（留在 xiaobao） |
| 当前入口 | 项目仓库（**尚未 Bootstrap 团队工作流**，无 `docs/progress/INDEX.md`） |
| 关联项目 | `xiaobao`（服务方 ← 调用方） |
| 沟通文档 | [communications/xiaobao__ai.md](communications/xiaobao__ai.md) |
