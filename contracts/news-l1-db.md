# 契约：news-l1-db（新闻 L1 数据库边界契约）

- 契约 id：`news-l1-db`
- 版本：v1.4（2026-07-27 权限矩阵照 DevOps 已执行 GRANT 补齐：`raw_items` SELECT + `source_item_url`/`l0_label`、`tasks` UPDATE + `run_after`，test + prod 双库对称，见 STATUS/沟通文档 GRANT 回帖）。v1.3（2026-07-27 Architect 事实订正：删除不存在的 `tasks.metadata` 列并补齐 `raw_item_id`/`run_after`/`max_attempts`/`priority`、补 `tasks.status` 枚举与时点对应表、claim 规则改为以 tasks 为准并补 `run_after` 退避条件、明确 `processed_news` 为「xiaobao 占位 INSERT + ai UPDATE」及 `id` 由 DB 生成、补 `published_at` 写入要求、标注 `score_total` 在 database 模式无触发点，见 C-2/C-3/C-4/C-5/C-8/C-9。权限矩阵变更已随 v1.4 落文档。v1.2 2026-07-27 订正 `tags_v2` 第五类笔误 `sentiment`→`processing` 对齐 HTTP 契约 + 明确 `language` 取值固定 `'zh'`，见 C-10 / C-7。v1.1 2026-07-25 订正 `score_total` 归属 + 合并 `l1_status` 枚举重复行，见 O-1 / O-5）
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
| `source_item_url` | text | 原文链接（rss/x 类 link_read 输入；v1.4 随 GRANT 补入） |
| `l0_label` | varchar | L0 分类结果（进 prompt 与 KB 检索，消除 domain_tags 恒空；v1.4 随 GRANT 补入） |

`ai_worker` 可**写**的列：

| 字段 | 类型 | 说明 |
|------|------|------|
| `l1_status` | varchar | 更新处理状态 |
| `l1_error` | text | 失败原因（失败时写入） |
| `l1_processed_at` | timestamptz | 处理完成时间 |
| `l1_attempt` | int | 递增尝试次数 |

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
> `l1_status = 'queued'` 的条目在正常路径下必有对应 `l1_ai_process` task（同函数内连续写入，但**非同一事务**，存在毫秒级窗口，xiaobao 已登记加事务的待办）。ai 侧只 claim `tasks`、不扫 `raw_items` 即可，无需孤儿探测。

### processed_news（处理后新闻表）

`ai_worker` 可**读写**（SELECT / INSERT / UPDATE）：

| 字段 | 类型 | 说明 | 对应 AI 输出 |
|------|------|------|-------------|
| `id` | uuid | 新闻 id（PK，`DEFAULT gen_random_uuid()`）。**ai_worker 不显式写入**，由数据库生成（v1.3 明确，见 C-3） | — |
| `raw_item_id` | uuid | 关联 raw_items.id，**`NOT NULL UNIQUE`**（幂等键，支撑 `ON CONFLICT (raw_item_id) DO UPDATE`） | — |
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
| `priority` | int NOT NULL | 优先级（默认 0） | 只读 |
| `locked_by` | text | 锁定者（worker 标识） | 可更新 |
| `locked_at` | timestamptz | 锁定时间 | 可更新 |
| `attempt` | int | 已尝试次数（claim 时 +1，**重试上限判定真源**） | 可更新 |
| `updated_at` | timestamptz | 更新时间 | 可更新 |
| `last_error` | text | 最近错误 | 可更新 |
| `last_error_kind` | varchar | 最近错误类型 | 可更新 |

> **v1.3 订正**：v1.2 及以前列有 `metadata jsonb（含 raw_item_id 等）`，**该列不存在**，系起草错误（见沟通文档 C-9）。`raw_item_id` 是一级 uuid 列，关联走主键，无需 jsonb 表达式索引。

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
| `l1_ai_process` | AI 处理任务 | `tasks.max_attempts`（默认 3，由 `AI_MAX_RETRIES` 配置） | [60s, 300s, 900s]，写入 `run_after` |

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
| xiaobao | 定时扫描 `processing` 状态且 `locked_at` 超阈值的任务，强制回收为 `retryable_failed` |

卡死阈值：30 分钟（1800 秒）。超过阈值未完成的任务由 xiaobao 侧回收并重试。

## 变更纪律

1. 任一字段增删改、类型变化、语义变化前，先改本文件并更新版本/最近更新日期。
2. breaking change（删字段、改类型、改必填性、权限收紧/放宽）必须在 [../CHANGELOG.md](../CHANGELOG.md) 标记 **BREAKING** 并点名需要跟进的项目。
3. 两侧实现改完后，各自在本项目 `docs/progress/INDEX.md` 或相关记录中写明已跟进的本契约版本。
4. schema 权属归 xiaobao，ai 侧不得建表改表。schema 变更由 xiaobao 侧发起，提前知会 ai 侧。
5. 本契约与 `news-l1` HTTP 契约并行存在，输出字段语义保持一致。若输出字段语义有差异，以 `news-l1` HTTP 契约为准，本契约同步更新。
