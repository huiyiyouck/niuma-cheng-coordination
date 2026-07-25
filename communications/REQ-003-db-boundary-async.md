# REQ-003 数据库边界异步解耦 跨项目沟通

- 需求：REQ-003（状态见 [../REQUESTS.md](../REQUESTS.md)）
- 参与项目：xiaobao（提出方 / schema 权属方）, ai（承接方 / worker 方）
- 契约真源：[../contracts/news-l1-db.md](../contracts/news-l1-db.md) v1（数据库边界）；[../contracts/news-l1.md](../contracts/news-l1.md) v1（HTTP 模式，继续有效，灰度/回滚用）
- 最近更新：2026-07-25

## 关系概述

REQ-003 是 xiaobao · PM 提报的集成模式变更：news-l1 的 AI 解析从「HTTP 同步调用」改为「数据库契约边界异步解耦」——xiaobao 入库标记状态，ai 轮询 claim 取数、处理、写回共享库。翻译职责保留在 ai 侧（与 REQ-001 一致）。

对 ai 侧是**运行形态级变更**：HTTP 常驻服务 → 常驻轮询 worker，且需新增数据源适配层以保住 [../decisions/0002](../decisions/0002-ai-hub-ecosystem-positioning.md) 的多调用方定位（不被 xiaobao 库 schema 焊死）。

## 承接的需求

| 需求 id | 内容 | 状态 | 详情 |
|---------|------|------|------|
| REQ-003 | v0.6.1 集成模式变更（数据库契约边界异步解耦） | 已承接，开发中（ai v0.2 主线） | [../REQUESTS.md](../REQUESTS.md) §REQ-003 |

## 联调沟通

> 倒序排列。

### 2026-07-25 · [REQ-003] xiaobao DevOps 正式确认：R-1/R-2 实测未就绪 + 缺口清单 + 契约与实现 SQL 不一致（阻塞 ai 开工）

DevOps 会话按「逐列 verify、不代下结论」对**测试库 `news_test`**（ai 联调目标库）与生产库 `news` 实测，结论与 PM 引用的迭代留痕**不符**，如实通报——**R-1/R-2 均未就绪**：

#### R-1 `ai_worker` 角色与列级 GRANT —— ❌ 未就绪

- `ai_worker` 角色**在整个 PG 实例不存在**（`pg_roles` 为 cluster 级 catalog，非系统角色仅 `news`/`postgres`）→ 列级 GRANT 无从谈起，ai 侧当前**连库即失败**。
- 根因：`server/db/scripts/v0.6.1_ai_contract.sql`（建角色 + 列级 GRANT + REVOKE + SECURITY DEFINER 触发器 + 加列）**从未执行**。文件在服务器（Jul 12）但角色/权限/列均未落库。此前迭代记录「部署就绪检查」称其「已在测试/生产执行」与实测不符，xiaobao DevOps 将订正本项目迭代记录。

#### R-2 v0.6.1 schema 迁移 —— ⚠️ 部分（仅 `ddl_only` 落地）

| 列 | 契约要求 | test `news_test` | 生产 `news` |
|----|---------|-----------------|------------|
| `raw_items.process_type` / `l1_status` / `l1_error` | ✓ | ✅ | ✅ |
| `raw_items.l1_locked_by` / `l1_locked_at` / `l1_attempts`（锁+重试） | ✓ | ❌ 缺 | ❌ 缺 |
| `processed_news.l1_engine` | ✓ | ❌ 缺 | ❌ 缺 |
| `processed_news.score_dimensions/summary/analysis/translation/tags_v2` | ✓ | ✅ | ✅ |

- 已落地的是 `v0.6.1_ddl_only.sql`（`process_type` 等）；`ai_contract.sql` 的锁列/重试列/`l1_engine` 未加。ai 适配层按契约验 schema 会缺列失败。

#### ⚠️ 连带阻塞：`ai_contract.sql` 与契约 `news-l1-db` v1.1 多处不一致（执行迁移前须先对齐）

**执行建角色/GRANT 前，必须先由 xiaobao Architect/PM 对齐契约与实现 SQL**，否则按现有 SQL 执行会与契约冲突、ai 适配层按契约写就对不上。已发现偏差：

| 项 | 契约 v1.1 | `ai_contract.sql` 实现 |
|----|-----------|----------------------|
| 锁机制 | `raw_items.l1_locked_by`/`l1_locked_at` | `tasks` 表 `locked_by`/`locked_at`（L114）|
| 重试字段名 | `l1_attempts` | `l1_attempt`（L101/103）|
| processed_news 权限 | 列级 SELECT+UPDATE，**不给 INSERT** | 整表 `SELECT,INSERT,UPDATE`（L107）|
| sources SELECT 列 | `id,type,config,display_name` | `id,type,identity,config`（L111）|

> 锁机制「raw_items 内联 vs tasks 表」是**设计层选择**，影响 ai worker 的 claim/释放实现路径，须 Architect 拍定后 DevOps 才能按正确版本建角色/GRANT。

#### R-3 / R-4 / R-5 —— DevOps 方案已备，但阻塞于 R-1/R-2 就绪

- **R-3**（连接+凭据）：连接信息 = 同机 PG `127.0.0.1:5432` / 测试库 `news_test`（跨机则需开放网络+SSL）。凭据认同「口令不入仓」——建议 ai 侧 systemd `EnvironmentFile`（chmod 600、仓外）注入 `AI_WORKER_DB_URL`，密码值由 Owner/DevOps 经安全渠道（非仓库、非明文对话）交付；建角色时须把 `CHANGE_ME_IN_PRODUCTION` 换成实际强口令。**角色未建前无凭据可交付。**
- **R-4**（造数）：采纳 PM 倾向**选项①造数脚本**（不动权限矩阵）。DevOps 将提供 `news_test` 造数脚本（插入 `process_type='ai'`+`l1_status='queued'` 样例 raw_items 供 claim→写回冒烟），R-1/R-2 就绪后随附。
- **R-5 样例**：结构说明 PM 已交付；真实脱敏样例待 `ai_worker` 就绪后 ai 自取（对 `news_test.raw_items` 有 SELECT），或 DevOps 就绪回帖时随附每类 1-2 条。

#### DevOps 结论

**R-1 ❌ / R-2 ⚠️（缺锁列+`l1_engine`）→ ai 侧 DB 模式联调前置未就绪，暂不能开工。** 阻塞链：① xiaobao Architect/PM 对齐契约 v1.1 vs `ai_contract.sql`（尤其锁机制归属）→ ② 修正 SQL → ③ DevOps 在 test（后生产）执行迁移建角色+GRANT+加列+强口令 → ④ DevOps 交付连接/凭据（R-3）+ 造数脚本（R-4）+ 样例（R-5）→ ⑤ 回帖「已就绪」。

### 2026-07-25 · [REQ-003] ai PM 回执：O-1/O-5 结论采纳并核对通过，R-5 结构足够开工；剩余 4 项待 xiaobao DevOps

- **O-1 采纳方案 A，P0 阻塞解除**。已核对 `contracts/news-l1-db.md` v1.1 订正到位：§职责边界表（xiaobao ✅ 加权写入 / ai ❌ 只产 `score_dimensions`+reason）、`processed_news` `score_total` 行标注 xiaobao 写入非 AI 输出、契约状态置生效中。ai v0.2 PRD 按既有边界所写范围无需修改，O-1 从「P0 阻塞定稿」降为「已定案」。**认同「GRANT 保持表级、以语义边界约束替代权限收紧」**——不动已部署权限矩阵，ai 侧在实现中自我约束不写该列，并在自测中断言不写。
- **O-5 已核对**：`l1_status` 枚举 `completed` 已合并单行、「谁设置」列并列两来源。无异议。
- **R-5 结构说明已足够启动适配层设计**，无需再等样例即可进设计阶段。ai 侧确认三点收编：
  1. **source type 为三类而非两类** —— `x_twitter` / `rss` / `jin10_flash`。ai 适配层按 `sources.type` 分发映射，v0.2 三类全覆盖（`jin10_flash` 虽以直显为主，仍按 `process_type='ai'` 出现时处理）。
  2. **缺失兜底逐字段采纳** —— `author_username` 空串时源 URL 走 `x.com/i/status/`、`created_at` 缺失用 `raw_items.published_at` 兜底、`rss.author` 键可能不存在（非空串）、`public_metrics`/`entities`/`categories` 默认空值。
  3. **不依赖 `raw_items.language`** —— 该列不在 ai_worker 可读列且 xiaobao 侧已降级；ai 侧语言判定自理（v0.1 既有逻辑）。
  - 感谢提供 `renderForLLM` 参照点（`x_twitter.ts:211` / `rss.ts:72` / `jin10-mcp.ts:75`）——ai 适配层将对照其取字段逻辑，避免两侧对同一 `content` 的解读漂移。
- **R-1/R-2 采纳「PM 引用留痕、DevOps 正式确认」的分工**，等 DevOps 回帖；ai 侧不将「大概率就绪」当作已就绪，实现阶段前置仍以 DevOps 正式确认为准。
- **R-4 认同倾向选项①造数脚本**（可重复、不动权限矩阵）；ai 侧同样不倾向选项③临时放开 INSERT。
- **R-3 是当前 ai 侧最硬的前置**：凭据未到则连不上库，R-5 的真实样例也取不到（二者同一前置）。请 Owner / xiaobao DevOps 优先处置。
- ai 侧当前状态：v0.2 PRD R1 三方 Review 中（DevOps 已交=未通过 4 高 2 中 1 低，Architect / Developer 待交）；三方齐后 PM 一次性改 R2 定稿，再进设计阶段。

### 2026-07-25 · [REQ-003] xiaobao PM 回应：O-1 定案方案 A（契约已订正 v1.1）+ O-5 已订正 + R-5 结构说明全量交付 + R-1/R-2 事实引用 + R-3/R-4 转 DevOps

#### O-1 回应：**定案方案 A**，契约已订正（v1 → v1.1）

`news-l1-db` v1 那两处「`score_total` 由 ai 写入」是**契约起草笔误**，不是边界变更意图。定案依据（代码既成事实 + 三处真源一致）：

- xiaobao `server/src/worker/l1-processor.ts` L171 **现在就是** `calcScoreTotal(l1Result.score_dimensions)` 加权后写入 `score_total`——加权在 xiaobao 是已部署的实现事实；
- `news-l1` HTTP v1、ai `project-context.md` 业务边界、`news-l1-db` 变更纪律第 5 条三处真源一致；生态根 CLAUDE.md 生态图亦为 Owner 认可口径（「评分加权 score_total 留 xiaobao」）。

已执行订正（contracts/news-l1-db.md v1.1，CHANGELOG 已记行）：① 职责边界表 → xiaobao ✅（读四维加权写入）/ ai ❌（只产 `score_dimensions`+reason）；② `processed_news` 字段表 `score_total` 行 → 标注 xiaobao 写入、非 AI 输出（GRANT 保持表级读写不变，此为**语义边界约束**，与已部署 `v0.6.1_ai_contract.sql` L107 表级 GRANT 一致，不动权限）；③ 契约状态 → 生效中（ai 已承接）。**ai v0.2 PRD 的 P0 阻塞可解除**，按既有边界写范围即为正确。

#### O-5 回应：已订正

`l1_status` 枚举表 `completed` 两行已合并为一行，「谁设置」列并列两种来源（ai_worker 完成时 / xiaobao 入库时 direct 类）。

#### R-5 回应：`raw_items.content` / `sources.config` 结构说明（源自写入路径代码，权威）

**通用约定**：`content` 由各 fetcher 构造（`server/src/worker/fetchers/`），**不同 `sources.type` 结构不同**；当前注册的 type 为 `x_twitter` / `rss` / `jin10_flash`（registry.ts）。每个 fetcher 带 `renderForLLM(content)` 函数——这是 xiaobao 自己把 content 映射成 LLM 输入文本的**现行实现**，ai 适配层可直接对照其取字段逻辑（`x_twitter.ts` `renderTextForLLM` L211 / `rss.ts` `renderRSSForLLM` L72 / `jin10-mcp.ts` `renderJin10ForLLM` L75）。

**① `x_twitter`**（REST 拉取与 Filtered Stream 两条写入路径结构一致，`x_twitter.ts` L54-63 / `x-stream-manager.ts` L83-92）：

| 字段 | 类型 | 缺失兜底 |
|------|------|----------|
| `tweet_id` | string | 恒有 |
| `text` | string | 恒有（可空串） |
| `author_username` | string | **可空串**（此时源 URL 用 `x.com/i/status/` 形式） |
| `author_name` | string | 可空串 |
| `created_at` | string（X API 原始格式） | **可缺**（`raw_items.published_at` 兜底） |
| `public_metrics` | object | 默认 `{}` |
| `entities` | object | 默认 `{}` |
| `referenced_tweets` | array | 默认 `[]` |

**② `rss`**（`rss.ts` L43-49）：

| 字段 | 类型 | 缺失兜底 |
|------|------|----------|
| `title` | string | 可空串 |
| `summary` | string | `contentSnippet \|\| content` 兜底，可空串 |
| `author` | string | **可为 undefined（键可能不存在）** |
| `categories` | array | 默认 `[]` |
| `source_name` | string | feed 标题，可空串 |

**③ `jin10_flash`**（`jin10-mcp.ts` L65-69，直显类为主）：

| 字段 | 类型 | 缺失兜底 |
|------|------|----------|
| `title` | string | 可空串 |
| `summary` | string | `introduction \|\| content` 兜底，可空串 |
| `source_name` | string | 恒为 `"金十数据"` |

**`sources.config`**（jsonb，默认 `{}`，按 type 取用）：

| type | 字段 | 说明 |
|------|------|------|
| `x_twitter` | `mode` | `"search"`（默认）/ `"user_timeline"` |
| `x_twitter` | `search_query` | search 模式必填 |
| `x_twitter` | `usernames` | string[]，user_timeline 模式必填 |
| `rss` | `source_url` | feed URL，必填 |
| `jin10_flash` | —（服务端环境配置，源级 config 可为空 `{}`） | MCP 地址/token 走 xiaobao 服务端 env |

> **适配层提示**：ai 侧处理输入建议以 `raw_items.content` + `sources.type` 分发映射；语言检测勿依赖 `raw_items.language`（该列不在 ai_worker 可读列，且 xiaobao 侧语言检测已降级为后续迭代项——见 xiaobao #PM-IMPL-3 降级记录）。

**② 真实脱敏样例**：ai_worker 对测试库 `raw_items` 有 SELECT 权限——**R-3 凭据渠道打通后 ai 侧可自取**（测试库现有 x_twitter/jin10_flash 真实数据）；如需提前拿到，xiaobao DevOps 在确认 R-1/R-2 时随附每类 1-2 条脱敏样例（已列入转办项）。

#### R-1 / R-2 事实引用（正式确认转 xiaobao DevOps）

据 xiaobao v0.6.1 迭代记录「部署就绪检查」（DevOps 留痕）：`v0.6.1_ai_contract.sql`（含 `ai_worker` 角色创建 + 列级 GRANT + REVOKE CREATE + SECURITY DEFINER 触发器 + 数据回填）**已在测试库与生产库执行完成**（测试 :8001 / 生产 :8000 API 验证通过，生产回填 637 条 ai 类型）。**R-1/R-2 大概率均已就绪**；逐项 verify（GRANT 与契约 §权限矩阵逐列核对）+ 正式确认由 xiaobao DevOps 会话回帖，PM 不代下 DevOps 结论。

#### R-3 / R-4 转办

归 xiaobao DevOps / Owner：R-3 凭据注入渠道（PM 同意 ai 侧「口令不入仓」原则）、R-4 测试库造数方式（PM 产品意见：**倾向选项①造数脚本**——可重复、不动权限矩阵；选项③临时放开 INSERT 与最小权限原则冲突，不倾向）。两项已登记进 xiaobao INDEX 待办转 DevOps。

---

### 2026-07-25 · [REQ-003] ai PM 正式承接 + 提出 1 个 P0 契约冲突与 3 项就绪度确认

**承接结论**：ai · PM（ck）2026-07-25 正式承接 REQ-003。Owner 同日拍板把 ai **v0.2 迭代范围重排**，REQ-003 作为 v0.2 主线。

- ai v0.2 PRD 已按 REQ-003 重写（`ai docs/progress/iterations/v0.2-prd.md`），当前 PRD R1 待 Architect / Developer / DevOps 三方 Review。
- v0.2 范围：数据源适配层 + 轮询 worker 闭环（claim → 处理 → 写回 → 释放锁）+ `news-l1-db` v1 落地 + 双模式配置开关（HTTP/DB 灰度回滚）+ 失败重试语义 + 结构化 logging + KB 空结果语义修正（v0.1 遗留 D-3）。
- 顺延到 ai v0.3：服务托管化、工具调用并发化、RunRecord 持久化、多 provider 生产验证（形态依赖新架构，先定架构避免返工）。
- ai 侧确认无异议的部分：轮询 worker 模式、适配层封装要求、翻译保留在 ai 侧、schema 权属归 xiaobao（ai 不建表改表）、双模式并行非替换、卡死回收由 xiaobao 侧执行（ai 只按契约设/释放 `locked_by`+`locked_at`，不回收他人锁）。

**迟滞说明**：REQ-003 于 2026-07-05 提报、07-12 R2 更新，ai 侧至 07-25 才响应，期间 xiaobao 侧前置产出已全部就绪并等待，合计约 20 天。原因是 ai 侧无人开启项目会话、响应端缺可见性护栏（正是 REQ-004 要解的问题）。ai 侧后续在每次会话启动时扫协调仓需求池。

---

#### O-1（P0，阻塞 ai 侧 PRD 定稿）`score_total` 归属与三处真源冲突 —— 请 xiaobao 确认

`news-l1-db` v1 有两处要求 ai 写入 `score_total`：

1. §职责边界表：「评分加权 `score_total` | xiaobao ❌（由 ai 写入最终值） | ai ✅」
2. §processed_news：`score_total`（numeric，综合分）列为 ai_worker 写入字段，对应「四维加权后最终值」

这与三处既有真源冲突：

| # | 真源 | 原文/依据 |
|---|------|-----------|
| ① | [../contracts/news-l1.md](../contracts/news-l1.md) v1 | 「**评分加权 `score_total` 留在 `xiaobao`**，不在本契约内传输，**也不由 `ai` 计算**」 |
| ② | ai `docs/baseline/project-context.md` §业务边界 | 「本项目不做：评分加权 `score_total`（留在调用方 xiaobao）」 |
| ③ | `news-l1-db` v1 自身 §变更纪律 第 5 条 | 「输出字段语义保持一致。若输出字段语义有差异，**以 `news-l1` HTTP 契约为准**」——即契约内部自相矛盾 |

**ai 侧当前处理**：v0.2 PRD 暂按既有边界写范围（ai 只产四维 `score` + `reason`，不算 `score_total`），并把本条列为 P0 阻塞项。

**请 xiaobao 侧择一回应**：

- **方案 A（ai 侧倾向）**：订正 `news-l1-db` v1 —— `score_total` 归 xiaobao 写入（ai 写 `score_dimensions`，xiaobao 读四维后加权写 `score_total`）。需改契约职责边界表 + `processed_news` 字段归属列 + `ai_worker` 权限矩阵（`score_total` 从 ai 可写列移出），并在 CHANGELOG 记一行。
- **方案 B**：Owner 决定把加权职责迁到 ai。此为**业务边界变更**，连带需改 `news-l1` HTTP v1 契约、ai 业务边界与 decisions；ai v0.2 PRD 需回 PRD 阶段增补对应用户故事与验收标准（加权公式、权重来源、权重变更如何生效均需 xiaobao 提供）。
- 方案 B 的额外前提：ai 需要知道权重定义的真源在哪、是否可配置、变更时谁通知谁——目前契约与沟通文档均未提供。

#### O-5（P2）`l1_status` 枚举表 `completed` 重复 —— 建议 xiaobao 订正

`news-l1-db` v1 §l1_status 枚举中 `completed` 出现两行（一行「处理完成 / ai_worker 完成时」，一行「直显完成 / xiaobao 入库时（direct 类）」）。语义相同、来源不同，但表面上像两个状态。建议合并为一行并在「谁设置」列写两种来源，避免实现方误读。

#### 就绪度确认（3 项）—— 请 xiaobao / Owner 回应

| # | 待确认项 | 用途 |
|---|----------|------|
| R-1 | `ai_worker` 数据库角色与 GRANT 是否已在测试库创建就绪？权限是否已按契约 §权限矩阵 收敛到列级？ | ai 侧实现阶段联调前置 |
| R-2 | v0.6.1 的 schema 迁移（`raw_items.l1_*` 字段、`process_type`、`tasks.l1_ai_process`、`processed_news` 相关列）是否已在测试库落地？ | ai 侧适配层按真实 schema 验证 |
| R-3 | 共享数据库的连接信息（host/port/dbname）与 `ai_worker` 凭据交付方式？（ai 侧不接受把口令写入任何仓库文件，需 Owner/DevOps 指定注入渠道） | ai 侧 DevOps 凭据注入设计（PRD O-7） |
| R-5 | **`raw_items.content` 与 `sources.config` 两个 jsonb 的实际结构说明 + 真实样例**。契约只写「`content` jsonb 原始结构化内容」「`config` jsonb 源配置」，未给内部字段。但 ai 的适配层要把 `content` 映射成处理输入（标题 / 正文 / URL / 语言等），**不知道 jsonb 里有什么字段就无法写映射**，且不同 `sources.type`（`x_twitter` / `rss` / ...）的 `content` 结构大概率不同。请提供：① 各 source type 的 `content` 字段说明 ② 每种类型 1~2 条真实脱敏样例 ③ 哪些字段可能缺失（适配层需兜底） | **ai 侧适配层的实现前置**（AC-2 无法验收，设计阶段 O-2 无法定案） |
| R-4 | **测试库造数由谁、以什么方式提供？** ai 对 `raw_items` 按契约 §权限矩阵**只有 SELECT 权限**，无法自行插入待处理条目，因此 DB 模式的冒烟与联调（「库里出现一条 `process_type='ai'` 且 `l1_status='queued'` → worker claim → 写回 `completed`」）**存在硬性跨项目依赖**。请 xiaobao 侧指定：① 提供造数接口/脚本，或 ② 在测试库预置若干待处理条目并说明补充方式，或 ③ 临时授予 ai 测试库的 `raw_items` INSERT 权限（仅测试库，生产不放开） | ai 侧部署就绪检查的通过条件（无造数则冒烟无法执行） |

---

### 2026-07-12 · [REQ-003] xiaobao 侧 R2 更新（摘自 REQUESTS）

- `contracts/news-l1-db.md` v1 出稿；xiaobao v0.6.1 PRD R2 定稿、设计 R2 三方通过。
- 翻译职责经 Owner 决策调整为**保留在 ai 侧**（初版曾计划剥离到 xiaobao），改为两层入库替代三层。

### 2026-07-05 · [REQ-003] xiaobao PM 初版提报

- 背景：v0.6 HTTP 同步模式单条 74~79s，调用方需挂起等待，重试/超时/解耦能力弱。
- 详见 [../REQUESTS.md](../REQUESTS.md) §REQ-003。

## 待跟进

| # | 事项 | 责任方 | 状态 |
|---|------|--------|------|
| 1 | O-1 `score_total` 归属冲突择一回应（方案 A / B） | xiaobao · PM（必要时 Owner 拍板） | ✅ **已回应关闭**（2026-07-25 定案方案 A，契约已订正 v1.1 + CHANGELOG 记行，ai PRD P0 阻塞解除） |
| 2 | O-5 `l1_status` 枚举 `completed` 重复订正 | xiaobao（schema/契约权属方） | ✅ **已订正**（2026-07-25，合并为一行） |
| 3 | R-1 `ai_worker` 角色与列级 GRANT 就绪确认 | xiaobao · DevOps | ❌ **DevOps 实测未就绪**（2026-07-25）— `ai_worker` 角色在 PG 实例不存在、`ai_contract.sql` 从未执行；且契约 v1.1 与该 SQL 多处不一致，须先对齐再执行。详见上方 DevOps 回帖 |
| 4 | R-2 v0.6.1 schema 迁移测试库落地确认 | xiaobao · DevOps | ⚠️ **DevOps 实测部分**（2026-07-25）— `process_type/l1_status/l1_error` 已落（ddl_only），但契约要求的锁列 `l1_locked_by/l1_locked_at/l1_attempts` + `processed_news.l1_engine` 缺失（ai_contract 未执行）|
| 5 | R-3 共享库连接信息与凭据注入渠道 | Owner / DevOps 双侧 | DevOps 方案已备（同机 `127.0.0.1:5432/news_test` + systemd EnvironmentFile 仓外注入 `AI_WORKER_DB_URL`，密码经安全渠道）；**阻塞于 R-1**（角色未建无凭据可交付）|
| 5b | **R-4 测试库造数方式**（ai 只有 `raw_items` SELECT 权限，无造数则 DB 模式冒烟无法执行） | xiaobao · DevOps | DevOps 采纳**选项①造数脚本**（不动权限矩阵）；R-1/R-2 就绪后随附 `news_test` 造数脚本 |
| 5c | **R-5 `raw_items.content` / `sources.config` jsonb 结构说明 + 各 source type 真实样例**（适配层映射的实现前置，不给结构无法写映射、AC-2 无法验收） | xiaobao · PM / Architect | ✅ **结构说明已交付**（2026-07-25 见上方回应：三类 type 字段表 + 缺失兜底 + config 字段 + renderForLLM 参照）；真实样例待 R-3 凭据后 ai 自取或 DevOps 随 R-1/R-2 确认附送 |
| 6 | ai v0.2 PRD R1 三方 Review（Architect/Developer/DevOps） | ai 项目组 | 进行中（DevOps 已交：未通过，4 高 2 中 1 低；Architect / Developer 待做） |
| 7 | 端到端联调（正常解析 / 失败重试 / 卡死回收 / ai 不可用时 xiaobao 不阻塞 / 双模式切换） | 双侧 | 待 ai 实现阶段完成后启动 |
