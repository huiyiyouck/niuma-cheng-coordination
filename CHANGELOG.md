# 跨项目变更日志

> 记录跨项目重大事件、契约 breaking change、迁移提醒。
> 单项目内部迭代不在此记录。倒序排列（最新在上）。

## 2026-08-01

- **`news-l1-db` 契约升 v1.8（`ALTER ROLE` 方案甲留痕 + 一处错误表述更正 + 阈值升格为契约参数）** → [contracts/news-l1-db.md](contracts/news-l1-db.md)。xiaobao · Architect：① **`ALTER ROLE ai_worker` 执行方改为 ai 侧（方案甲）**，角色级 `statement_timeout`/`lock_timeout` 实际生效值为 ai 的 `4s`/`3s`（比 v1.7 文档的 30s/5s 更严，以 ai CN-008 为准）；`idle_in_transaction_session_timeout=60s` 保留 xiaobao 值并**定性为跨项目约定上限**（ai 可更严不可放宽，放宽须先改契约——它保护的是 xiaobao 的 reclaim 不被长事务阻塞）；xiaobao 撤回自行执行计划。② **更正 v1.7 的错误表述**：`ALTER ROLE` 对 `USERSET` 参数**不是强制手段**（应用层 `SET` 随时可覆盖），真实价值仅为「忘记 SET 时的兜底默认」；据此重评，**方案甲严格优于原方案**而非妥协。③ **`AI_STALE_TIMEOUT_MS` 升格为跨项目契约参数**——该值形式上是 xiaobao 的 env，实质约束 ai 的批量上限不变式 `N×(预算+DB上界) < 阈值×0.6`；按 600s 代入余量仅 1.37 倍、`N=1` 为唯一合法值、单条预算上调空间仅 337s。任一侧变更前须先改契约并通知对方（xiaobao 调小会导致 ai 任务被误回收且无信号，ai 扩容需先请 xiaobao 调大）。非 breaking。影响项目：`ai`、`xiaobao`。

## 2026-07-30

- **`news-l1-db` 契约升 v1.7（新增共享库超时约定 + 订正卡死回收三处错误）** → [contracts/news-l1-db.md](contracts/news-l1-db.md)。xiaobao · Architect：① **新增 §连接与超时约定**——硬约束「claim 事务与处理必须分离，事务内不得含 LLM 调用」（否则 ai 长事务持锁会阻塞 xiaobao reclaim，且 xiaobao 无 `lock_timeout` 会导致**整个回收机制挂住**）；四项取值 `statement_timeout` 30s / `idle_in_transaction_session_timeout` 60s / `lock_timeout` 5s / 建连 10s；ai 侧由 xiaobao 以 `ALTER ROLE ai_worker SET` 在数据库层强制、ai 零配置（**执行前须 ai 确认事务边界**，否则会表现为连接莫名断开）。② **订正 §卡死回收机制三处**：扫描的是 `tasks.status='running'` 非 `processing`（我方契约自己踩了两表字面量差异的坑）；未达上限回收到 `queued` 非 `retryable_failed`；**阈值默认 600s 而非契约此前写的 1800s——该数字系起草臆定、无实现依据，ai 多轮引用的 1800s 均源自此处**，实际生效值待 DevOps 核实回填。非 breaking，但含一项需 ai 确认后才能执行的权限侧变更。方案全文见 xiaobao `docs/progress/ad-hoc/2026-07-30-spike-db-timeout-config.md`。影响项目：`ai`、`xiaobao`。
- **`news-l1-db` 契约升 v1.6（locked_by / 两表「处理中」字面量 / domain_tags 类型）** → [contracts/news-l1-db.md](contracts/news-l1-db.md)。xiaobao · Architect 答 ai 三问：① `locked_by` 无格式约束、xiaobao 回收不读其内容，但**ai 自愈回收必须同事务同步 `raw_items.l1_status`**，否则留下「前端显示解析中但任务在排队」的不一致；xiaobao 侧写入值为 `WORKER_ID`（默认 `worker-1`），请 ai 避开。② **新增两表「处理中」字面量差异专节**——`tasks.status='running'` vs `raw_items.l1_status='processing'`，`tasks` 无 CHECK 约束，ai 若写错 `processing` 会导致卡死回收永不触发且双方无报错。③ `sources.domain_tags` 类型定性——预期数组，`{}` 系 `schema.ts:67` 默认值误写（应为 `'[]'`）、语义等同未配置，附两侧归一化行为差异与数据实况（当前 5 条冒烟条目对应 source 全为 `{}`，故「等价」指取数链路等价而非该批数据有值）。非 breaking。影响项目：`ai`、`xiaobao`。

## 2026-07-28

- **`news-l1-db` 契约升 v1.5（C-11~C-14 回应，含一条结论撤回）** → [contracts/news-l1-db.md](contracts/news-l1-db.md)。xiaobao · Architect：① **撤回 v1.3 关于 `l0_label` 的错误结论**——上轮答 C-1 时误把 `l0_label` 当作 `domain_tags` 的对应物，致 ai 侧白 GRANT 一列并据此改了 PRD 验收标准（CN-004）。经追链路核实，`L1Input.domain_tags` 的真源是 **`sources.domain_tags`**（源级静态标签，`sources.domain_tags` → `l1-processor.ts:243` → `ai-hub.ts:45` → HTTP 请求体），与 L0 无关；`l0_label` 是**处理决策标记**，完整取值域（`direct_display` / 三个 `*_candidate` / 规则跳过原因 / `llm_skip`）已补入契约。**GRANT `sources.domain_tags` 后 ai 的 `domain_tags` 与 HTTP 模式完全等价**，`domain_tags` 恒空不必列为已知限制（优于 PM 同日给出的「暂无规划」条件式口径——PM 指 L0 动态领域分类能力，本条是源级静态标签，二者不矛盾）；② C-11 明确 `tasks.priority` **数值大 = 优先**；③ C-12 书面确认退避越界取末值，`max_attempts` 以列为准，并登记 xiaobao 应用层仍读硬编码常量的待订正项；④ C-13 标注 `source_item_url` **不保证**协议前缀（rss/jin10 原样透传源数据），登记 `processed_news.raw_item_id` 唯一约束为 ai 写回幂等前提、放宽须先改契约。非 breaking。新增权限需求 `GRANT SELECT (domain_tags, attention_level) ON sources` 待 DevOps 执行后再升版。影响项目：`ai`、`xiaobao`。

## 2026-07-27

- **`news-l1-db` 契约升 v1.4（权限矩阵照实补齐）** → [contracts/news-l1-db.md](contracts/news-l1-db.md)。DevOps 已执行 3 列 GRANT（`raw_items.source_item_url`/`raw_items.l0_label` SELECT + `tasks.run_after` UPDATE，test + prod 双库对称、verify 通过），契约权限矩阵与 `raw_items` 可读列表照实补入。非 breaking（文档对齐既成授权）。至此 v1.3 遗留的「待 GRANT 后升版」闭合，ai 侧 C-6 行锁实测前置全齐（凭据由 ai DevOps 同机直读，无需转交）。影响项目：`ai`、`xiaobao`。
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
