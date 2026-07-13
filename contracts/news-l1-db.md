# 契约：news-l1-db（新闻 L1 数据库边界契约）

- 契约 id：`news-l1-db`
- 版本：v1
- 状态：已出稿，待 ai 侧承接确认
- schema 权属方：`xiaobao`（拥有建表改表、角色管理、触发器创建权限）
- worker 方：`ai`（AI 处理中枢 / Agent Hub）
- 最近更新：2026-07-12
- 真源说明：本文件是 news-l1 数据库边界契约的**单一真源**。表结构、字段、状态枚举、读写权属变更前先改本文件，再改两侧实现，并在 [../CHANGELOG.md](../CHANGELOG.md) 记一行。
- 与 HTTP 契约关系：本契约是 [news-l1 v1](news-l1.md) HTTP 契约的**并行新增模式**，非替换。HTTP 模式继续有效（灰度 / 回滚用）。
- 实现参考：xiaobao `docs/progress/iterations/v0.6.1-design.md` §2 数据模型 + §2.5 数据库角色与权限

## 职责边界（重要）

| 职责 | xiaobao 侧 | ai 侧（ai_worker） |
|------|-----------|-------------------|
| schema 建表改表 | ✅ 唯一权属方 | ❌ 无权限 |
| 角色与 GRANT 管理 | ✅ | ❌ |
| raw_items 入库 | ✅ | ❌ 只读 |
| processed_news 写入（AI 类） | ❌ 占位创建 | ✅ 结果写入 |
| processed_news 写入（direct 类） | ✅ | ❌ |
| news_positions 关联 | ✅（触发器自动） | ❌ 无权限 |
| tasks 创建与卡死回收 | ✅ | ❌ 只 claim / 更新状态 |
| 评分加权 score_total | ❌（由 ai 写入最终值） | ✅ |
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
| `queued` | 已入队待处理（= PRD 产品语义 `pending`） | xiaobao 入库时 |
| `processing` | 处理中（被 ai_worker claim） | ai_worker claim 时 |
| `completed` | 处理完成 | ai_worker 完成时 |
| `retryable_failed` | 可重试失败 | ai_worker 失败时 / xiaobao 回收时 |
| `final_failed` | 最终失败（重试耗尽） | ai_worker 或 xiaobao |
| `not_started` | 未开始 | xiaobao 入库时（direct 类） |
| `completed` | 直显完成 | xiaobao 入库时（direct 类） |

#### AI 待处理队列 claim 规则

```sql
-- ai_worker 取数条件
WHERE process_type = 'ai'
  AND l1_status IN ('queued', 'retryable_failed')
ORDER BY published_at ASC
LIMIT N
```

> Claim 机制通过 `tasks` 表实现（见下方），避免多 worker 竞争。

### processed_news（处理后新闻表）

`ai_worker` 可**读写**（SELECT / INSERT / UPDATE）：

| 字段 | 类型 | 说明 | 对应 AI 输出 |
|------|------|------|-------------|
| `id` | uuid | 新闻 id（PK） | — |
| `raw_item_id` | uuid | 关联 raw_items.id | — |
| `title` | text | 中文标题 | title |
| `summary` | text | 摘要 | summary |
| `translation` | jsonb | 翻译，如 `{zh: "...", original?: "..."}` | translation |
| `context` | jsonb | 背景上下文（数组） | context |
| `analysis` | text | 深度分析 | analysis |
| `score_total` | numeric | 综合分 | 四维加权后最终值 |
| `score_dimensions` | jsonb | 四维评分 + 理由 | score_dimensions |
| `tags_v2` | jsonb | 五类标签：`{domain, entity, event, content_type, sentiment}` | tags |
| `language` | varchar | 语言标识 | — |
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
| `status` | varchar | 任务状态 | 可更新 |
| `locked_by` | varchar | 锁定者（worker 标识） | 可更新 |
| `locked_at` | timestamptz | 锁定时间 | 可更新 |
| `attempt` | int | 已尝试次数 | 可更新 |
| `updated_at` | timestamptz | 更新时间 | 可更新 |
| `last_error` | text | 最近错误 | 可更新 |
| `last_error_kind` | varchar | 最近错误类型 | 可更新 |
| `metadata` | jsonb | 元数据（含 raw_item_id 等） | 只读 |

#### task type

| type | 用途 | 最大尝试次数 | 退避间隔 |
|------|------|-------------|----------|
| `l1_ai_process` | AI 处理任务 | 3 | [60s, 300s, 900s] |

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
| raw_items | SELECT 部分列 | id, source_id, content, published_at, l0_status, l1_status, l1_attempt, process_type |
| raw_items | UPDATE 部分列 | l1_status, l1_error, l1_processed_at, l1_attempt |
| processed_news | SELECT, INSERT, UPDATE | 全列可读写 |
| sources | SELECT 部分列 | id, type, identity, config |
| tasks | SELECT, UPDATE 部分列 | UPDATE: status, locked_by, locked_at, attempt, updated_at, last_error, last_error_kind |

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
