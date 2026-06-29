# 0002 · ai 定位升级为生态内部通用 AI 处理中枢（supersede 0001-D5）

- 日期：2026-06-29
- 影响项目：`xiaobao`、`ai`
- 状态：已拍板（Owner 2026-06-29）
- 取代：[0001-ai-hub-split.md](0001-ai-hub-split.md) 的 **D5**（仅 D5；D1–D4 继续有效）
- 详细真源：ai 项目 `docs/progress/ad-hoc/2026-06-29-product-brief-positioning.md`
- 关联需求：REQ-002（架构调研，前置）、REQ-001（news-l1 真实化，落地）

## 背景

0001-D5 原定「中枢承载 xiaobao 平台的多种 AI 能力，不做泛化的多项目通用平台」。
Owner 2026-06-29 重新拍板，将 ai 定位升级为生态内部通用 AI 处理中枢。

## 决策

| 编号 | 决策 |
|------|------|
| D5'（取代 D5） | ai 定位升级为 **niuma-cheng 生态内部的通用 AI 处理中枢（Agent Hub）**：生态内多个项目未来都可调用同一 AI 处理服务。 |

边界澄清：
- 「生态**内部**通用」≠ 对外泛化平台 / 多租户 / 开放 SaaS；表达「未来多个内部项目可调用同一服务」，非现在建泛化平台。
- 「Agent Hub」为产品代号，**不预设实现形态**（真 agent vs 确定性编排由 REQ-002 定）。
- 仅取代 D5；**D1–D4 继续有效**（独立服务 / xiaobao 用 Node·ai 用 Python / 异步调用 / `tasks` 为 xiaobao 业务真源）。

## 对各项目影响

- `ai`：定位升级；骨架按「多调用方 + 多任务类型」预留扩展点（接口分发 / 任务注册 / 调用方标识 / 运行记录模型），但 v0.1 **仍只实现 news-l1 一个 task-type**，不为不存在的第二个调用方写实现；具体架构待 REQ-002。
- `xiaobao`：**无破坏性变更** —— 调用关系（xiaobao→ai）、`score_total` 加权留 xiaobao、`news-l1` 契约职责边界均不变；无需立即改动。

## 附注

- 本决策不改 `news-l1` 契约；若 REQ-002 架构结论需改 API/schema，另走 `contracts/` 契约变更流程。
