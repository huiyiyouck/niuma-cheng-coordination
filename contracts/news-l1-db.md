# 契约：news-l1-db（新闻 L1 数据库边界契约）

- 契约 id：`news-l1-db`
- 版本：v1.8（2026-08-01 Architect：① **`ALTER ROLE` 执行方改为 ai（方案甲）**——角色级 `statement_timeout`/`lock_timeout` 实际生效值为 ai 的 `4s`/`3s`（比 v1.7 文档写的 30s/5s 更严，以 ai CN-008 为准），`idle_in_transaction_session_timeout=60s` 保留 xiaobao 值并**定性为跨项目约定上限**（ai 可更严不可放宽，放宽须先改契约）；xiaobao 撤回自行执行计划。② **更正 v1.7 一处不准确表述**——`ALTER ROLE` 对 `USERSET` 参数**不是强制手段**（应用层 `SET` 随时可覆盖），其真实价值是「忘记 SET 时的兜底默认」。③ **`AI_STALE_TIMEOUT_MS` 升格为跨项目契约参数**——它约束 ai 的批量上限不变式（当前 600s 下余量仅 1.37 倍、`N=1` 唯一合法、单条预算上调空间仅 337s），任一侧变更前须先改契约并通知对方。见 2026-08-01 三帖）。v1.7（2026-07-30 Architect：**新增 §连接与超时约定**——claim 事务与处理必须分离的硬约束（事务内禁止 LLM 调用）+ 四项超时取值（`statement_timeout` 30s / `idle_in_transaction_session_timeout` 60s / `lock_timeout` 5s / 建连 10s）+ 以 `ALTER ROLE ai_worker SET` 在数据库层强制、ai 侧零配置；**订正 §卡死回收机制三处错误**——扫描的是 `tasks.status='running'` 非 `processing`、未达上限回收到 `queued` 非 `retryable_failed`、**阈值默认 600s 而非契约此前写的 1800s（该数字无实现依据，ai 多轮引用的 1800s 源自此处，实际生效值 DevOps 已核实回填：test/prod `server/.env` 均 `600000`（600s），见 §卡死阈值）**。方案详见 xiaobao `docs/progress/ad-hoc/2026-07-30-spike-db-timeout-config.md`）。v1.6（2026-07-30 Architect：① `locked_by` 明确无格式约束、xiaobao 回收不读其内容、xiaobao 侧写入值为 `WORKER_ID`（默认 `worker-1`）请 ai 避开；② **新增「处理中」两表字面量差异专节**——`tasks.status='running'` vs `raw_items.l1_status='processing'`，并说明 `tasks` 无 CHECK 约束、写错 `processing` 会导致卡死回收永不触发；③ **新增 ai 自愈回收必须同步 `raw_items.l1_status` 的要求**，否则留下「前端显示解析中但任务在排队」的不一致；④ `sources.domain_tags` 类型定性——预期数组，`{}` 系 `schema.ts:67` 默认值误写（应为 `'[]'`）、语义等同未配置，附两侧归一化行为与数据实况。见 ai 2026-07-28 两帖）。v1.5（2026-07-28 Architect 答复 C-11~C-14：**撤回 v1.3 关于 `l0_label` 的错误结论**——`L1Input.domain_tags` 真源是 `sources.domain_tags`（源级静态标签，非 L0 产物），`l0_label` 是处理决策标记，两者语义无关；补 `l0_label` 完整取值域枚举；§sources 补 `domain_tags`/`attention_level` 行（权限待 GRANT）；明确 `priority` 方向为「数值大 = 优先」；退避越界取末值 + `max_attempts` 以列为准并登记 xiaobao 侧待订正的硬编码；标注 `source_item_url` 不保证协议前缀；登记 `processed_news.raw_item_id` 唯一约束为 ai 幂等前提。见 C-11/C-12/C-13/C-14）。v1.4（2026-07-27 权限矩阵照 DevOps 已执行 GRANT 补齐：`raw_items` SELECT + `source_item_url`/`l0_label`、`tasks` UPDATE + `run_after`，test + prod 双库对称，见 STATUS/沟通文档 GRANT 回帖）。v1.3（2026-07-27 Architect 事实订正：删除不存在的 `tasks.metadata` 列并补齐 `raw_item_id`/`run_after`/`max_attempts`/`priority`、补 `tasks.status` 枚举与时点对应表、claim 规则改为以 tasks 为准并补 `run_after` 退避条件、明确 `processed_news` 为「xiaobao 占位 INSERT + ai UPDATE」及 `id` 由 DB 生成、补 `published_at` 写入要求、标注 `score_total` 在 database 模式无触发点，见 C-2/C-3/C-4/C-5/C-8/C-9。权限矩阵变更已随 v1.4 落文档。v1.2 2026-07-27 订正 `tags_v2` 第五类笔误 `sentiment`→`processing` 对齐 HTTP 契约 + 明确 `language` 取值固定 `'zh'`，见 C-10 / C-7。v1.1 2026-07-25 订正 `score_total` 归属 + 合并 `l1_status` 枚举重复行，见 O-1 / O-5）
- 状态：生效中（ai 已承接 REQ-003，2026-07-25）
- schema 权属方：`xiaobao`（拥有建表改表、角色管理、触发器创建权限）
- worker 方：`ai`（AI 处理中枢 / Agent Hub）
- 最近更新：2026-07-27
- 真源说明：本文件是 news-l1 数据库边界契约的**单一真源**。表结构、字段、状态枚举、读写权属变更前先改本文件，再改两侧实现，并在 [../CHANGELOG.md](../CHANGELOG.md) 记一行。
- 与 HTTP 契约关系：本契约是 [news-l1 v1](news-l1.md) HTTP 契约的**并行新增模式**，非替换。HTTP 模式继续有效（灰度 / 回滚用）。
- 实现参考：xiaobao `docs/progress/iterations/v0.6.1-design.md` §2 数据模型 + §2.5 数据库角色与权限

## 职责边界（重要）

| 职责 | xiaobao 侧 | ai 侧（ai_worker） |
|------|-----------|-------------------|
| schema 建表改表 | ✅ 唯一权属方 | ❌ 无权限 |
| 角色与 GRANT 管理 | ✅ | ❌ |
| raw_items 入库 | ✅ | ❌ 只读 |
| processed_news 写入（AI 类） | ✅ **L0 通过时创建占位行**（原文标题/摘要，保障基础展示态立即可见 → AC-01/AC-06） | ✅ **UPDATE 占位行**写入结果（建议 `INSERT ... ON CONFLICT (raw_item_id) DO UPDATE` 兼容占位缺失） |
| processed_news 写入（direct 类） | ✅ | ❌ |
| news_positions 关联 | ✅（触发器自动，**在占位 INSERT 时即触发**——这是有意设计：让待解析新闻立即可见。`news_positions` 不存排序键，排序按 `processed_news.published_at` 查询时实时计算，故 ai UPDATE 后排序自然更新，无"更新路径"问题） | ❌ 无权限 |
| tasks 创建与卡死回收 | ✅ | ❌ 只 claim / 更新状态 |
| 评分加权 score_total | ✅ 分工不变（读四维加权后写入，公式见 `l1-processor.ts:13` `calcScoreTotal`：`(T×0.25+I×0.35+C×0.25+X×0.15)×2` 保留 1 位）。**⚠️ v1.3 已知缺口**：该函数目前只挂在 HTTP / 内建 L1 路径（`l1-processor.ts:171`），**database 模式下无触发点**，ai 写回后该列保持 NULL，xiaobao 侧待补 | ❌（只产 `score_dimensions` + reason，不算不写 `score_total`） |
| 状态机推进 | ✅（初始标记 / 回收） | ✅（claim / 完成 / 失败） |

## 核心表与字段

### raw_items（原始条目表）

`ai_worker` 可**读**的列：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 原始条目 id |
| `source_id` | uuid | 信息源 id |
| `content` | jsonb | 原始结构化内容 |
| `published_at` | timestamptz | 发布时间 |
| `l0_status` | varchar | L0 状态 |
| `l1_status` | varchar | L1 状态（见下方枚举） |
| `l1_attempt` | int | L1 处理尝试次数 |
| `process_type` | varchar(20) | 处理类型：`direct` / `ai` |
| `source_item_url` | text | 原文链接（link_read 输入；v1.4 随 GRANT 补入）。**不保证带 `http(s)://` 协议前缀**——`x_twitter` 两条入库路径均构造完整 URL 故保证，`rss`（取自 feed 的 `link`）与 `jin10_flash`（取自 MCP 的 `url`）**原样透传源数据、不保证**。ai 侧自行规范化，无法规范化按「无 URL」处理（v1.5，见 C-13） |
| `l0_label` | varchar(50) | **L0 处理决策标记，不是领域分类**（v1.5 订正，见 C-14）。取值域见下方枚举。**不要用作 `domain_tags`**——后者真源见 §sources |

`ai_worker` 可**写**的列：

| 字段 | 类型 | 说明 |
|------|------|------|
| `l1_status` | varchar | 更新处理状态 |
| `l1_error` | text | 失败原因（失败时写入） |
| `l1_processed_at` | timestamptz | 处理完成时间 |
| `l1_attempt` | int | 递增尝试次数 |

#### l0_label 取值域（v1.5 新增，见 C-14）

**语义：L0 阶段的处理决策标记（是否值得送 AI + 优先级 + 是否需补上下文），非领域分类。**

| 取值 | 含义 | 写入时机 |
|------|------|---------|
| `direct_display` | 直显类，不走 AI 链路 | xiaobao `processor.ts` 处理直显条目时 |
| `normal_candidate` | L0 通过：正常信息，送 AI 分析 | L0 LLM 判定通过 |
| `high_priority_candidate` | L0 通过：重大事件 / 突发 / 官方公告 | 同上 |
| `needs_context_candidate` | L0 通过：需补背景（含链接 / 代号 / 缩写） | 同上 |
| `empty_text` / `duplicate_content_hash` / `emoji_only` / `retweet_no_added_text` / `spam:{pattern}` | L0 规则引擎跳过原因 | 规则命中 |
| `llm_skip` 或 LLM 返回的 `skipReason` | L0 LLM 判定跳过 | LLM 判 skip |
| `NULL` | 尚未进入 L0 | — |

> **现状提示**：截至 2026-07-28，两库实测该列非空值只有 `direct_display` 一个——因为 L0 链路尚未成功跑通（`news_test` 8 条 `l0_classify` 全 `failed`）且生产 AI 开关默认关闭。上表是代码中已实现的完整取值域，不是规划。
> `*_candidate` 三值对 ai 侧可作为**处理优先级 / 是否需补上下文**的信号使用（与 `needs_context` 相关），但**不可作为领域标签**。

#### l1_status 枚举

| 值 | 说明 | 谁设置 |
|----|------|--------|
| `queued` | 已入队待处理（= PRD 产品语义 `pending`） | xiaobao **L0 通过时**（非入库时；入库时为 `not_started`，见 v1.3 订正） |
| `processing` | 处理中（被 ai_worker claim） | ai_worker claim 时 |
| `completed` | 处理完成（AI 类 = AI 解析完成；direct 类 = 直显完成，语义相同） | ai_worker 完成时（AI 类）；xiaobao 入库时（direct 类） |
| `retryable_failed` | 可重试失败 | ai_worker 失败时 / xiaobao 回收时 |
| `final_failed` | 最终失败（重试耗尽） | ai_worker 或 xiaobao |
| `not_started` | 未开始（**列默认值，所有类入库时的初始态**；AI 类在 L0 通过后转 `queued`，direct 类经 processor 后转 `completed`） | xiaobao 入库时（默认值） |

#### AI 待处理队列 claim 规则

**claim 以 `tasks` 为准**（ai_worker 对 `tasks` 无 INSERT 权限，只领 xiaobao 预建的行），`raw_items` 条件作为过滤：

```sql
-- ai_worker claim（v1.3 订正：补 tasks 关联与 run_after 退避条件）
WITH next_task AS (
  SELECT t.id
  FROM tasks t
  JOIN raw_items ri ON ri.id = t.raw_item_id      -- raw_item_id 是一级 uuid 列 + FK，非 jsonb
  WHERE t.type = 'l1_ai_process'
    AND t.status = 'queued'
    AND t.run_after <= now()                       -- ★ 退避判定，缺此条件退避完全失效
    AND ri.process_type = 'ai'
    AND ri.l1_status IN ('queued', 'retryable_failed')
  ORDER BY ri.published_at ASC
  FOR UPDATE SKIP LOCKED
  LIMIT N
)
UPDATE tasks SET status = 'running', locked_by = $worker_id, locked_at = now(),
       attempt = attempt + 1, updated_at = now()
WHERE id IN (SELECT id FROM next_task)
RETURNING *;
```

> **退避真源是 `tasks.run_after`**，不要用 `updated_at + backoff(attempt)` 自行计算——那会与 xiaobao `requeueTask` 形成两套真源，卡死回收介入后必然漂移。
> 索引支撑：`ix_tasks_queue (status, run_after, priority)` + `raw_items` 主键 + `ix_raw_items_ai_queue`；当前量级无需新增索引。
> `l1_status = 'queued'` 的条目**必有对应 `l1_ai_process` task**——v1.5 起为强承诺：xiaobao 已将「置 `queued`」与「建 task」包进显式事务（`l0-classifier.ts:161 BEGIN` / `:199 COMMIT`，2026-07-27 落地），v1.3 时提示的毫秒级窗口**已消除**。ai 侧只 claim `tasks`、不扫 `raw_items` 即可，无需孤儿探测。

### processed_news（处理后新闻表）

`ai_worker` 可**读写**（SELECT / INSERT / UPDATE）：

| 字段 | 类型 | 说明 | 对应 AI 输出 |
|------|------|------|-------------|
| `id` | uuid | 新闻 id（PK，`DEFAULT gen_random_uuid()`）。**ai_worker 不显式写入**，由数据库生成（v1.3 明确，见 C-3） | — |
| `raw_item_id` | uuid | 关联 raw_items.id，**`NOT NULL UNIQUE`**（幂等键，支撑 `ON CONFLICT (raw_item_id) DO UPDATE`）。**该唯一约束是 ai 写回幂等性的前提**——放宽它（例如为支持重跑允许多版本结果）会使 ai 的 `ON CONFLICT` 静默失效并产生重复行，因此**变更前必须先改本契约并通知 ai**（v1.5 登记，见 C-13） | — |
| `published_at` | timestamptz | 发布时间，**列表排序依据**。ai_worker 写回时请一并写入（取 `raw_items.published_at`）——占位行当前该列为 NULL，缺此写入会导致条目在列表中沉底（v1.3 追加，见沟通文档三-2） | —（取自 raw_items） |
| `title` | text | 中文标题 | title |
| `summary` | text | 摘要 | summary |
| `translation` | jsonb | 翻译，如 `{zh: "...", original?: "..."}` | translation |
| `context` | jsonb | 背景上下文（数组） | context |
| `analysis` | text | 深度分析 | analysis |
| `score_total` | numeric | 综合分（**xiaobao 写入**：读四维加权计算；ai_worker 不写此列——GRANT 为表级读写，此为语义边界约束） | —（非 AI 输出） |
| `score_dimensions` | jsonb | 四维评分 + 理由 | score_dimensions |
| `tags_v2` | jsonb | 五类标签：`{domain, entity, event, content_type, processing}`（与 `news-l1` HTTP v1 一致；`processing` 为运行事实标签，如 `engine:agent_hub` / `degraded:{reason}`。v1.1 及以前的 `sentiment` 系起草笔误，该能力经 PM 裁定本阶段不引入，见沟通文档 C-10） | tags |
| `language` | varchar | **产出内容语种，固定 `'zh'`**（news-l1 输出契约即中文输出；原文语种不入此列，由 ai_worker 写入，见 C-7） | —（取值规则固定） |
| `created_at` | timestamptz | 创建时间 | — |

> 说明：`tags` / `importance_score` / `entities` / `bullets` / `source_refs` 等旧字段保留兼容，不在本契约范围内。ai_worker 写入时只需写上述字段即可。

### sources（信息源表）

`ai_worker` 可**读**的列：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 信息源 id |
| `type` | varchar | 源类型（x_twitter / rss / ...） |
| `identity` | varchar | 源标识 |
| `config` | jsonb | 源配置 |
| `domain_tags` | jsonb | **`L1Input.domain_tags` 的真源**（v1.5 新增，见 C-14）。信息源级静态领域标签，取数链路与 HTTP 模式同源：`sources.domain_tags` → `l1-processor.ts:243/257-278` → `ai-hub.ts:45` → HTTP 请求体。ai 侧经 `raw_items.source_id → sources.id` 主键 join 取值。**类型见下方说明**（v1.6） |
| `attention_level` | varchar(20) | 源关注级别（`regular` 等）。HTTP 模式的 L0 输入亦使用；ai 侧按需取用（v1.5 新增，权限待 GRANT） |

> **v1.6 `domain_tags` 类型说明**：**预期类型是 jsonb 数组**（如 `["AI"]`）。但 `schema.ts:67` 的列默认值误写为 `'{}'::jsonb`（应为 `'[]'::jsonb`），因此**从未配置过标签的 source 存的是空对象 `{}`**——`{}` 的语义等同「未配置」= 空数组，**不是另一种有意义的形态**。
> - xiaobao 侧一直做归一化容错：`Array.isArray(v) ? v : (typeof v === 'object' && v !== null ? Object.values(v) : [])`（`l1-processor.ts:257-260` / `l0-classifier.ts:126-129`）。
> - **ai 侧推荐做法**：仅 `jsonb_typeof = 'array'` 时取用，`object` / `null` / 缺失一律映射为 `[]`。在现有数据上与 xiaobao 行为完全一致（`Object.values({})` 即 `[]`）。唯一差异是**非空 object**（如 `{"0":"AI"}`）——xiaobao 会取出 `["AI"]`、ai 给 `[]`；该形态属脏数据、当前不存在，xiaobao 归一化时会清除，ai 侧不必对齐此分支。
> - xiaobao 已登记：修默认值为 `'[]'::jsonb` + 迁移归一存量 `{}` → `[]` + 加类型校验。
> - **数据实况提示**：截至 2026-07-28，`sources` 4 行中 2 行为 `["AI"]`、2 行为 `{}`，且当前 5 条冒烟条目对应的 source 全部为 `{}`。故「与 HTTP 模式等价」指的是**取数链路等价**，不代表这批测试数据有值。

> **v1.5 重要订正（C-14）**：v1.3 曾答复「L0 分类结果存在 `raw_items.l0_label`，GRANT 该列即可消除 `domain_tags` 恒空」——**该结论错误并已撤回**。`L1Input.domain_tags` 从来不是 L0 的产物，而是本表的源级静态标签；`l0_label` 是处理决策标记（见 §raw_items）。两列语义无关。

### tasks（任务表，用于 claim 与退避重试）

`ai_worker` 可**读 + 更新状态**：

| 字段 | 类型 | 说明 | ai_worker 权限 |
|------|------|------|---------------|
| `id` | uuid | task id | 只读 |
| `type` | varchar | 任务类型 | 只读 |
| `raw_item_id` | uuid | **关联 raw_items.id（一级列 + FK ON DELETE CASCADE）** | 只读 |
| `status` | varchar | 任务状态（枚举见下） | 可更新 |
| `run_after` | timestamptz NOT NULL | **退避时间，claim 过滤依据** | 可更新（GRANT 已就绪，v1.4） |
| `max_attempts` | int NOT NULL | 该 task 的尝试上限（建任务时按 `AI_MAX_RETRIES` 写入） | 只读 |
| `priority` | int NOT NULL | 优先级（默认 0）。**方向：数值大 = 优先**（v1.5 明确，见 C-11）——xiaobao `claimTask` 为 `ORDER BY priority DESC, created_at ASC`。ai 侧 claim 须按 `priority DESC` 排序，否则优先级设置不生效且不报错 | 只读 |
| `locked_by` | text | 锁定者（worker 标识）。**无长度限制、无格式约束**；xiaobao 回收逻辑**不读该列内容**（只 `SET NULL`），ai 可自由编码（如 `{worker_id}#{run_token}`）。xiaobao worker 写入的是 `config.workerId`（env `WORKER_ID`，**默认字面量 `worker-1`**）——ai 的 worker 身份请避开该值，以免前缀匹配误判（v1.6） | 可更新 |
| `locked_at` | timestamptz | 锁定时间 | 可更新 |
| `attempt` | int | 已尝试次数（claim 时 +1，**重试上限判定真源**） | 可更新 |
| `updated_at` | timestamptz | 更新时间 | 可更新 |
| `last_error` | text | 最近错误 | 可更新 |
| `last_error_kind` | varchar | 最近错误类型 | 可更新 |

> **v1.3 订正**：v1.2 及以前列有 `metadata jsonb（含 raw_item_id 等）`，**该列不存在**，系起草错误（见沟通文档 C-9）。`raw_item_id` 是一级 uuid 列，关联走主键，无需 jsonb 表达式索引。

#### ⚠️ 「处理中」在两个表里字面量不同（v1.6，务必区分）

| 表.列 | 「处理中」的值 | 性质 |
|---|---|---|
| `tasks.status` | **`running`** | 执行态 |
| `raw_items.l1_status` | **`processing`** | 业务态 |

同一次 claim 要同时写这两个值，但**字面量故意不同**。

- **`tasks` 表没有任何 CHECK 约束**（`pg_constraint` 中 `contype='c'` 为空），写错值 DB 不拦、不报错。
- **若 ai 给 `tasks.status` 写成 `processing`**：xiaobao 卡死回收的判定是 `WHERE status='running'`（`reclaim.ts:12,19`），**认不出该行 → 回收永不触发 → 任务永久卡住**，且两侧都无报错。这是最难排查的一类错位，请严格按 `running` 写。
- xiaobao 本迭代不加 CHECK 约束（`tasks` 是 5 种 type 共用的通用表，加约束需迁移并约束历史数据），以本契约的枚举为准；如需 DB 层兜底可在后续迭代评估。

#### ⚠️ ai 自愈回收「自己上次进程遗留的锁」时必须同步两列（v1.6）

xiaobao 的 1800s 卡死回收除了改 `tasks`，还有第三条 UPDATE（`reclaim.ts:26-35`）：task 从 `running` 回到 `queued` 时，把对应 `raw_items.l1_status` 从 `processing` 同步改回 `queued`。

**ai 启动时自愈回收若只改 `tasks.status` 不改 `raw_items.l1_status`**，会留下 `tasks.status='queued'` 但 `raw_items.l1_status='processing'` 的不一致——前端持续显示「AI 解析中」而任务其实在排队，两侧均不报错。xiaobao 的 reclaim 每 tick 会扫并修复该不一致（有兜底），但中间窗口内展示是错的。**请在同一事务内同时改这两列**（ai 对 `raw_items.l1_status` 有写权限）。

#### tasks.status 枚举（v1.3 补充，照 xiaobao 实现）

| 值 | 说明 | 谁设置 |
|----|------|--------|
| `queued` | 待领取（含首次建任务与失败重排队） | xiaobao 建任务时；**ai_worker 可重试失败时**；xiaobao 卡死回收时 |
| `running` | 已被 claim、处理中 | **ai_worker claim 时**（同时 `attempt+1`、写 `locked_by` / `locked_at`） |
| `succeeded` | 处理成功、终态 | **ai_worker 成功写回后** |
| `failed` | 达 `max_attempts` 上限、终态 | **ai_worker 最终失败时**；xiaobao 达上限时 |

**与 `raw_items.l1_status` 的时点对应**（非一一对应：`tasks` 是执行态，`l1_status` 是业务态）：

| ai_worker 时点 | `tasks.status` | `raw_items.l1_status` | 备注 |
|----------------|----------------|----------------------|------|
| claim | `running` | `processing` | 同事务写；`l1_status='processing'` 是 xiaobao 卡死回收的判定依据 |
| 成功 | `succeeded` | `completed` | 同时写 `l1_processed_at = now()` |
| 可重试失败（未达上限） | `queued` + `run_after = now() + backoff` | `retryable_failed` | backoff 按 `attempt` 取下表间隔 |
| 最终失败（达上限） | `failed` | `final_failed` | 同时写 `l1_error`、`l1_processed_at` |

> 只在 tasks 侧出现的中间态：`running` 与重排队的 `queued`（`l1_status` 粒度更粗）。
> **重试上限以 `tasks.attempt` 对比 `tasks.max_attempts` 判定**（勿硬编码 3）；`raw_items.l1_attempt` 为镜像值，同事务一并推进（见 C-8）。

#### task type

| type | 用途 | 最大尝试次数 | 退避间隔 |
|------|------|-------------|----------|
| `l1_ai_process` | AI 处理任务 | **以 `tasks.max_attempts` 列为准**（建任务时按 `AI_MAX_RETRIES` 写入，默认 3） | [60s, 300s, 900s]，写入 `run_after`；**`attempt` 超出数组长度时取末值 900s**，不得取 0 或崩溃 |

> **v1.5 订正（C-12）**：
> - 「最大尝试次数 3」的旧表述已改为**以 `tasks.max_attempts` 列为准**——`schema.ts` 该列默认值为 5，v0.6 时代的既有 `l1_process` 行即为 5；新建的 `l1_ai_process` 行按 `AI_MAX_RETRIES`（默认 3）写入。ai 侧读列，勿硬编码。
> - **退避越界行为**：xiaobao 实现为 `backoff[Math.min(tries, backoff.length - 1)]`（`dispatcher.ts:118-119`），即超出取末值，与 ai 侧假设一致。
> - **双方已同源（v1.5 更正）**：xiaobao 应用层判上限**已改为读 `tasks.max_attempts` 行内值**（`dispatcher.ts:112-114`，`BACKOFF_CONFIG` 仅在行内值缺失时兜底；建 task 统一经 `maxAttemptsForTaskType()` 写入，`l1_ai_process` 取 `AI_MAX_RETRIES`）。2026-07-27 落地。此前帖子中「xiaobao 仍读硬编码常量、已登记订正」的表述**已过时，作废**——两侧现以同一列为准，`AI_MAX_RETRIES` 改动不会造成漂移。

## 数据库角色与权限

### 角色：ai_worker

- 用途：ai 侧 worker 连接共享数据库使用的专用角色
- 创建方：xiaobao（schema 权属方）
- 最小权限原则：仅授予必要的 SELECT / INSERT / UPDATE 权限，无建表改表权限

### 权限矩阵

| 对象 | 权限 | 列级说明 |
|------|------|----------|
| DATABASE | CONNECT | 允许连接 |
| SCHEMA public | USAGE | 允许访问 schema；**禁止 CREATE**（不能建表） |
| raw_items | SELECT 部分列 | id, source_id, content, published_at, l0_status, l1_status, l1_attempt, process_type, **source_item_url, l0_label**（v1.4） |
| raw_items | UPDATE 部分列 | l1_status, l1_error, l1_processed_at, l1_attempt |
| processed_news | SELECT, INSERT, UPDATE | 全列可读写 |
| sources | SELECT 部分列 | id, type, identity, config |
| tasks | SELECT, UPDATE 部分列 | UPDATE: status, locked_by, locked_at, attempt, updated_at, last_error, last_error_kind, **run_after**（v1.4，退避写入依据） |

### 触发器：news_positions 自动关联

- `processed_news` INSERT 后自动触发 `news_positions` 关联
- 触发器函数使用 **SECURITY DEFINER**（以 schema owner 身份执行），ai_worker 不需要 `news_positions` 的 INSERT 权限
- `search_path` 锁定为 `public, pg_temp`，防止 search_path 注入

## 卡死回收机制

| 角色 | 动作 |
|------|------|
| ai_worker | claim 时设置 `locked_by` + `locked_at`，处理完成或失败时释放 |
| xiaobao | 定时扫描 **`tasks.status = 'running'`** 且 `locked_at` 超阈值的任务并回收 |

**回收动作（v1.7 照实订正，`reclaim.ts:8-35`）**：

| 条件 | `tasks.status` | `raw_items.l1_status` |
|------|----------------|----------------------|
| `attempt < max_attempts` | → `queued`（同时清 `locked_by`/`locked_at`，`run_after = now()`） | → `queued`（从 `processing` 同步回退） |
| `attempt >= max_attempts` | → `failed` | 由 `requeueTask` 侧置 `final_failed` |

> **v1.7 订正三处**（v1.6 及以前的表述与实现不符，均为我方契约错误）：
> 1. 「扫描 `processing` 状态」→ 实际扫的是 **`tasks.status = 'running'`**（`processing` 是 `raw_items.l1_status` 的值，两表字面量不同，见 §tasks）。
> 2. 「回收为 `retryable_failed`」→ 实际未达上限时回到 **`queued`**（可再被 claim），不是 `retryable_failed`。
> 3. **阈值不是 1800 秒**——见下。

**卡死阈值**：由 `AI_STALE_TIMEOUT_MS` 配置，**代码默认 600000ms（600 秒 / 10 分钟）**（`config.ts:95`）；test 与 prod 的 `server/.env` 均显式设为 `600000`（DevOps 已核实，与默认同值）。

> ### ⚠️ 该阈值是跨项目契约参数，xiaobao 不得单方修改（v1.8 升格）
>
> 它形式上是 xiaobao 的一个 env，实质上**约束着 ai 侧的批量上限与单条处理预算**。ai 的不变式为：
>
> ```
> N × (单条 wall-clock 预算 + DB 操作上界) < 阈值 × 0.6
> ```
>
> 按当前 600s 代入：`1 × (240 + 23) = 263s < 360s`，**余量仅 1.37 倍**；`N = 1` 是唯一合法值（`N = 2` 即 526s > 360s 越界），单条预算的上调空间只剩 `360 − 23 = 337s`。
>
> **后果对称**：
> - **xiaobao 调小该值** → ai 的不变式可能立即失效，正在处理的任务被误回收并重复处理，且 ai 侧不会收到任何信号；
> - **ai 要提高并发（N > 1）或上调单条预算超过 337s** → 必须先请 xiaobao 调大该阈值，不能单方进行。
>
> **纪律**：任一侧变更前**先改本契约并通知对方**，与表结构、状态枚举同级。此前该值被当作 xiaobao 内部配置，是 v1.6 及以前「阈值 1800s」错误长期未被发现的原因之一——ai 的三条不变式全部建立在那个不存在的数字上。

> ⚠️ v1.6 及以前契约写的「30 分钟（1800 秒）」**无实现依据**，系起草时臆定；ai 侧多轮沟通中引用的 1800s 均源自此处。test / prod 的**实际生效值已由 xiaobao DevOps 核实并回填（2026-08-01，直连部署机 `zijie`/`news.huiyiyou.cloud` grep 两库 `server/.env`）：均显式设 `AI_STALE_TIMEOUT_MS=600000`（600s / 10 分钟），与代码默认一致，全场无 1800s。** 两侧对回收窗口的认知必须一致，否则 ai 的自愈逻辑会按错误窗口设计。

## 连接与超时约定（v1.7 新增）

共享库场景下，任一方的长事务 / 慢语句都会影响对方。以下为数据库层的强制约束。

### 硬约束：claim 事务与处理必须分离

> **claim（含 `SELECT ... FOR UPDATE SKIP LOCKED` 与状态写入）在一个短事务内完成并立即 `COMMIT`；LLM 处理在事务外执行；结果写回时另开事务。事务内不得包含任何 LLM 调用或网络等待。**

理由：ai 单条 wall-clock 预算 240s，若在事务内等待 LLM，该连接处于 `idle in transaction` 且持有行锁 → xiaobao 的 reclaim 想回收该行会被行锁阻塞 → **自愈机制整体挂住**（不只这一行）。下方 `idle_in_transaction_session_timeout` 会把这个反模式变成强制。

### 超时取值（两侧同一套）

| 参数 | 取值 | 作用 |
|------|------|------|
| `statement_timeout` | **30s** | 杀掉失控语句，防慢查询占满连接池 |
| `idle_in_transaction_session_timeout` | **60s** | 终止「事务中空闲」会话，防长事务持锁 |
| `lock_timeout` | **5s** | 等锁拿不到即放弃（周期性任务天然可重试），防阻塞主循环 |
| 连接建立超时 | **10s** | 快速失败，防建连挂起 |

**量级关系**（重要）：事务级超时（60s）必须**远小于**任务级回收窗口（`AI_STALE_TIMEOUT_MS`），才能保证「ai 卡住 → 先被 DB 断事务释放锁 → 再被 xiaobao 回收」这个顺序；反过来会让回收撞上未释放的锁。

### 施加方式

- **xiaobao 侧**：连接池 `options` 参数下发（`-c statement_timeout=... -c idle_in_transaction_session_timeout=... -c lock_timeout=...`）+ `connectionTimeoutMillis`。
- **ai_worker 侧（v1.8 订正：执行方改为 ai，取值以 ai 为准）**：经两侧协商定**方案甲**——由 **ai 侧统一执行一次** `ALTER ROLE`，避免两侧互相覆盖：

```sql
-- 执行方：ai 侧 DevOps（xiaobao 不再执行）
ALTER ROLE ai_worker SET statement_timeout = '4s';               -- ai 值，比 xiaobao 建议的 30s 更严
ALTER ROLE ai_worker SET idle_in_transaction_session_timeout = '60s';  -- ★ xiaobao 值，护 reclaim，见下
ALTER ROLE ai_worker SET lock_timeout = '3s';                    -- ai 值，比 xiaobao 建议的 5s 更严
```

| 参数 | 角色级实际生效值 | 说明 |
|------|-----------------|------|
| `statement_timeout` | **4s**（ai 值，以 ai CN-008 为准） | 比本文 §超时取值表的 30s 更严；ai 的语句均为单行 claim / 写回，自评 4s 足够 |
| `lock_timeout` | **3s**（ai 值） | 同上 |
| `idle_in_transaction_session_timeout` | **60s**（xiaobao 值，保留） | **跨项目约定项，见下方约束** |
| 建连超时 / 事务级总超时 | ai 应用层配置 | `ALTER ROLE` 设不了（前者是连接串参数，后者 PG 无此设置） |

> **`idle_in_transaction_session_timeout = 60s` 是跨项目约定，语义为「上限」**：ai 可以设得**更严**（更小），**不可放宽**（更大）。它保护的是 xiaobao 的 reclaim 不被 ai 长事务持锁阻塞——放宽即削弱该保护。**如需放宽，先改本契约并通知 xiaobao**。其余两项（`statement_timeout` / `lock_timeout`）只作用于 ai 自己的语句，ai 自主决定，改动无需通知。

> **v1.8 更正一处 v1.7 的不准确表述**：v1.7 称此举「新增 ai 实例 / 换连接库 / 改代码都不会绕过」——**不准确**。这三项都是 `USERSET` 参数，应用层 `SET` 随时可覆盖角色默认值，`ALTER ROLE` 从来不是强制手段，其真实价值是**「应用层忘记 SET 时的兜底默认」**。方案甲下由 ai 设置自己的值，兜底效果相同（且角色默认值与应用层 `SET` 值一致，消除了「角色默认比应用层严 → `SET` 生效前偶发失败」的边角）。真正的强制手段只有 GRANT / REVOKE 一类权限控制。

> **执行前提（已满足）**：须先确认 ai 的 claim 事务边界符合上述硬约束。ai Architect 已于 2026-08-01 确认「事务内不含 LLM 调用」，前提解除。

- **例外**：人工执行的 SQL 迁移（`psql` 直连，不走连接池）不受 `statement_timeout` 限制——迁移可能长时间运行，这是有意的。

## 变更纪律

1. 任一字段增删改、类型变化、语义变化前，先改本文件并更新版本/最近更新日期。
2. breaking change（删字段、改类型、改必填性、权限收紧/放宽）必须在 [../CHANGELOG.md](../CHANGELOG.md) 标记 **BREAKING** 并点名需要跟进的项目。
3. 两侧实现改完后，各自在本项目 `docs/progress/INDEX.md` 或相关记录中写明已跟进的本契约版本。
4. schema 权属归 xiaobao，ai 侧不得建表改表。schema 变更由 xiaobao 侧发起，提前知会 ai 侧。
5. 本契约与 `news-l1` HTTP 契约并行存在，输出字段语义保持一致。若输出字段语义有差异，以 `news-l1` HTTP 契约为准，本契约同步更新。
