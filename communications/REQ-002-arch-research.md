# REQ-002 AI 处理架构调研 跨项目沟通

- 需求：REQ-002（状态见 [../REQUESTS.md](../REQUESTS.md)）
- 参与项目：xiaobao, ai
- 契约真源：[../contracts/news-l1.md](../contracts/news-l1.md)（本调研为预研性质，结论若要改契约另走契约流程）
- 最近更新：2026-06-29

## 关系概述

REQ-002 是 xiaobao · Architect 提报、Owner 指派给 `ai` 的**架构调研 / 预研**需求，是 REQ-001「L1 真实化」的**前置**。要求 `ai` 项目组：

- 调研两个参考项目：`Horizon`（`/root/Horizon`，Python + asyncio，重度 LLM，与 Agent Hub 几乎同构）、`ai-news-aggregator`（`/root/ai-news-aggregator`，TS/Node，轻度 LLM 反向参照）；
- 回答 4 个架构岔路口（见 [../REQUESTS.md](../REQUESTS.md) §REQ-002）；
- 形成 ai 自己的架构方案。

> REQ-002 是预研性质，非契约变更；`news-l1` 契约真源仍以 [../contracts/news-l1.md](../contracts/news-l1.md) 为准。

## 承接的需求

| 需求 id | 内容 | 状态 | 详情 |
|---------|------|------|------|
| REQ-002 | AI 处理架构调研（参考项目借鉴） | 已承接 | [../REQUESTS.md](../REQUESTS.md) §REQ-002 |

## 联调沟通

> 倒序排列。

### 2026-06-29 · [REQ-002] ai PM 正式承接

- Owner 2026-06-29 拍板把 ai 定位升级为**生态内部通用 AI 处理中枢**（见 [../decisions/0002-ai-hub-ecosystem-positioning.md](../decisions/0002-ai-hub-ecosystem-positioning.md)，supersede 0001-D5）。
- ai PM（ck）据此评估**正式承接 REQ-002**：承接登记由 PM 做，**架构方案实质产出归 Architect**（PM 不替架构拍板）。
- 调研范围因定位升级而扩大：在原「news-l1 真实化架构」基础上，增加「生态内部通用骨架」维度（多调用方 / 多任务类型的接口分发、任务注册、调用方标识、运行记录模型等扩展点）。
- 衔接：REQ-002 作为 ai v0.1 的前置架构调研；v0.1 PRD 待 REQ-002 给出架构结论后由 ai PM 创建。
- 下一步：ai 切 Architect 角色启动调研（读 Horizon / aggregator、答 4 岔路口、出数据架构方案）。

## 待跟进

| 事项 | 归属 | 状态 |
|------|------|------|
| REQ-002 架构调研与方案产出 | ai · Architect | 待启动（切角色后开始） |
| v0.1 PRD（REQ-001 真实化，前置本调研结论） | ai · PM | 待 REQ-002 架构结论 |
