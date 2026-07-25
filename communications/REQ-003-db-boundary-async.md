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
| 1 | O-1 `score_total` 归属冲突择一回应（方案 A / B） | xiaobao · PM（必要时 Owner 拍板） | 待回应（阻塞 ai PRD 定稿） |
| 2 | O-5 `l1_status` 枚举 `completed` 重复订正 | xiaobao（schema/契约权属方） | 待回应 |
| 3 | R-1 `ai_worker` 角色与列级 GRANT 就绪确认 | xiaobao · DevOps | 待回应 |
| 4 | R-2 v0.6.1 schema 迁移测试库落地确认 | xiaobao · DevOps | 待回应 |
| 5 | R-3 共享库连接信息与凭据注入渠道 | Owner / DevOps 双侧 | 待回应 |
| 5b | **R-4 测试库造数方式**（ai 只有 `raw_items` SELECT 权限，无造数则 DB 模式冒烟无法执行） | xiaobao · DevOps | 待回应（2026-07-25 由 ai DevOps 在 PRD R1 Review 发现补提） |
| 6 | ai v0.2 PRD R1 三方 Review（Architect/Developer/DevOps） | ai 项目组 | 进行中（DevOps 已交：未通过，4 高 2 中 1 低；Architect / Developer 待做） |
| 7 | 端到端联调（正常解析 / 失败重试 / 卡死回收 / ai 不可用时 xiaobao 不阻塞 / 双模式切换） | 双侧 | 待 ai 实现阶段完成后启动 |
