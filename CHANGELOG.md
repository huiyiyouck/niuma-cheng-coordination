# 跨项目变更日志

> 记录跨项目重大事件、契约 breaking change、迁移提醒。
> 单项目内部迭代不在此记录。倒序排列（最新在上）。

## 2026-07-27

- **`news-l1-db` 契约订正 v1.2 → v1.3（C-2/C-3/C-4/C-5/C-8/C-9 回应）** → [contracts/news-l1-db.md](contracts/news-l1-db.md)。xiaobao · Architect 逐处代码核查后的事实订正：① **删除 `tasks.metadata` 列**——该列不存在、系起草错误，致 ai 侧 C-9 推出「关联只能走 jsonb 表达式、可能全表扫」的错误结论；补齐 `raw_item_id`(uuid,FK) / `run_after` / `max_attempts` / `priority`；② 补 `tasks.status` 4 值枚举与 ai 四时点对应表（C-2 闭合）；③ **claim 规则改为以 `tasks` 为准并补 `AND run_after <= now()`**——原 SQL 无时间条件会使退避完全失效（C-4；根因是 GRANT 漏 `run_after` 列，权限侧待 DevOps 执行）；④ `processed_news` 明确为「xiaobao 占位 INSERT + ai UPDATE」并说明触发器早触发是有意设计（**推翻 ai 侧「ai INSERT」推断**，理由是 AC-01/AC-06 要求 L0 通过后立即可见），`id` 由 DB 生成、`raw_item_id` 唯一约束已有（C-3 闭合）；⑤ 补 `published_at` 写回要求（占位行当前为 NULL 致排序沉底）；⑥ 标注 `score_total` 补算函数只挂在 HTTP 路径、**database 模式无触发点**；⑦ 订正 `l1_status` 枚举中 `queued` / `not_started` 的设置时点描述。非 breaking（均为文档与实现对齐）。**ai 侧 4 条阻塞项至此全部闭合**（C-10 由 PM 本日闭合）。权限矩阵变更（`tasks.run_after` UPDATE + `raw_items.source_item_url` / `l0_label` SELECT）待 xiaobao DevOps 在 `news_test` 执行 GRANT 后另行升版。影响项目：`ai`、`xiaobao`。
- **`news-l1-db` 契约订正 v1.1 → v1.2（C-10 / C-7 回应）** → [contracts/news-l1-db.md](contracts/news-l1-db.md)。xiaobao · PM 回应 ai 侧契约缺项分派中 PM 名下 4 项：① C-10 定案——`sentiment` 本阶段**不引入**（前后端零消费 + 须先扩 HTTP 契约，登记后续迭代候选），`tags_v2` 第五类回归 `processing` 对齐 `news-l1` HTTP v1（起草笔误订正，变更纪律第 5 条）；② C-7——`processed_news.language` 取值定为**产出内容语种固定 `'zh'`**；③ Q-2 context 恒空知悉接受；④ Q-6 rss/jin10_flash 近期不接入 AI 链路，同意验收分层。非 breaking。**C-10 整条闭合，ai 侧阻塞余 C-2/C-3/C-5 三条待 xiaobao Architect**。影响项目：`ai`、`xiaobao`。

## 2026-07-25

- **`news-l1-db` 契约订正 v1 → v1.1（O-1 / O-5 回应）** → [contracts/news-l1-db.md](contracts/news-l1-db.md)。xiaobao · PM 回应 ai 侧承接时提出的 P0 契约冲突：`score_total` 归属两处笔误订正为 **归 xiaobao 写入**（ai 只产 `score_dimensions`+reason，与 `news-l1` HTTP v1、ai 业务边界、契约变更纪律第 5 条三处真源对齐；代码事实：xiaobao `l1-processor.ts` `calcScoreTotal` 现即如此实现）——采纳 ai 侧倾向的方案 A。同时合并 `l1_status` 枚举表 `completed` 重复两行（O-5）。非 breaking（订正笔误回归既有边界，无实现变更）。影响项目：`ai`（PRD 可解除 P0 阻塞）、`xiaobao`。R-5（content/config jsonb 结构）已在沟通文档回应；R-1~R-4 转 xiaobao DevOps / Owner。

## 2026-07-12

- **REQ-003 R2 更新 + 新增数据库边界契约 `news-l1-db` v1** → [contracts/news-l1-db.md](contracts/news-l1-db.md)。xiaobao v0.6.1 集成模式变更：news-l1 AI 解析从 HTTP 同步调用改为数据库契约边界异步解耦（ai 改轮询 worker + 适配层封装）。翻译保留在 ai 侧（2026-07-05 初版曾计划剥离，后经 Owner 决策调整为保留）。新增数据库角色 `ai_worker` + 列级 GRANT + SECURITY DEFINER 触发器 + tasks claim 卡死回收机制。非 breaking（HTTP 模式契约继续有效，作为灰度/回滚路径）。影响项目：`xiaobao`、`ai`。
- **REQ-003 状态更新**：xiaobao 侧 PRD R2 + 设计 R2 PM 复审均已通过，数据库边界契约 v1 已出稿，待 ai · PM 评估承接。

## 2026-07-01

- **REQ-001 联调证据回填**：xiaobao 测试环境 `/debug/ai` 已部署，xiaobao→ai news-l1 主链路成功；ai→xiaobao `kb-search` 命中用例成功。补充 `kb-search` v1 空结果语义：`results: []` 是 200 正常响应，不等同工具失败。非 breaking。影响项目：`xiaobao`、`ai`。
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
