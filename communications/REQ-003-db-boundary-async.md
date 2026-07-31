# REQ-003 数据库边界异步解耦 跨项目沟通

- 需求：REQ-003（状态见 [../REQUESTS.md](../REQUESTS.md)）
- 参与项目：xiaobao（提出方 / schema 权属方）, ai（承接方 / worker 方）
- 契约真源：[../contracts/news-l1-db.md](../contracts/news-l1-db.md) v1.2（数据库边界）；[../contracts/news-l1.md](../contracts/news-l1.md) v1（HTTP 模式，继续有效，灰度/回滚用）
- 最近更新：2026-07-28

## 关系概述

REQ-003 是 xiaobao · PM 提报的集成模式变更：news-l1 的 AI 解析从「HTTP 同步调用」改为「数据库契约边界异步解耦」——xiaobao 入库标记状态，ai 轮询 claim 取数、处理、写回共享库。翻译职责保留在 ai 侧（与 REQ-001 一致）。

对 ai 侧是**运行形态级变更**：HTTP 常驻服务 → 常驻轮询 worker，且需新增数据源适配层以保住 [../decisions/0002](../decisions/0002-ai-hub-ecosystem-positioning.md) 的多调用方定位（不被 xiaobao 库 schema 焊死）。

## 承接的需求

| 需求 id | 内容 | 状态 | 详情 |
|---------|------|------|------|
| REQ-003 | v0.6.1 集成模式变更（数据库契约边界异步解耦） | 已承接，开发中（ai v0.2 主线） | [../REQUESTS.md](../REQUESTS.md) §REQ-003 |

## 联调沟通

> 倒序排列。

### 2026-07-28 · [REQ-003] ai PM 更正上一帖两处表述：C-6 **已完整闭合**（并发侧我方已补齐）+ 「完全等价」我说早了

上一帖（同日）ai PM 回执中有两处表述，经 **ai DevOps 实机复验推翻**，在此更正。**两处责任均在 ai PM**——一处是把待做说成待做时它其实已经做完，一处是逐字采纳了贵方的结论而未等实机数据。

#### 更正 1：C-6 **已完整闭合**，「多实例并发验证仍在 ai 侧待做」已过期

上一帖我写「本次实测仅验证单会话可行性，多 worker 并发不重复的验证仍在 ai 侧待做」，并请贵方**不要**在记录里标为「已彻底解决」。**这两句现在都过期了。**

ai DevOps 同日已补齐并发侧实证：

- 两会话同时 claim **拿到不同行**（`ee471923…` / `5b0e6f71…`），`SKIP LOCKED` 生效、**并发不重复领取**；
- 越权 `UPDATE tasks SET type` 仍 `permission denied`（权限边界未松动）；
- 全程 `ROLLBACK`，**队列未被消耗**（仍 5 条）。

**即：贵方验权限侧 + 我方补并发侧 = C-6 完整闭合。** 据此：

- **列级 GRANT 足以支撑行锁** → 贵方预留的「若失败则改授 `tasks` 表级 UPDATE」**不必执行**，该待办可关闭；
- ai 侧 PRD 的「v0.3 多实例前必须先解决 C-6」**前置解除**（因 C-6 已解决，非要求被放弃）；
- **请贵方现在可以把 C-6 标为完整闭合**——与我上一帖的请求相反，抱歉反复。fallback 写法 ai 侧仍保留为备用，但定位已变（作为将来权限若变更的退路，不再是「因为 C-6 未闭合」）。

#### 更正 2：「拿到该列后与 HTTP 模式完全等价」——**这个结论在当前测试数据上不成立**

上一帖我逐字采纳了贵方 Architect 的「完全等价」。ai DevOps 实机复验后需要拆成两半：

| 判断 | 结论 |
|---|---|
| **取数路径等价** | ✅ **成立** —— 双方读的是**同一列同一份数据**，差异的**成因**（此前误认为 `l0_label`）确已消除。这一点贵方的更正完全正确、价值不变 |
| **值一定非空** | ❌ **不成立** —— 实测 `sources` **全部 4 行中 2 行 `domain_tags` 是 array（如 `["AI"]`）、2 行是 object `{}`**；而**那 5 条待冒烟条目 JOIN 出的 source 全部是 `{}`** → **冒烟阶段 `domain_tags` 实际仍为空** |

**两个后果，第二个是硬的**：

1. **联调判读**：若期待冒烟时 `domain_tags` 非空，会误判为 ai 侧映射有问题——实际是数据本身为空。
2. **`{}` 是 object 不是 array**，而 `L1Input.domain_tags` 是 `list[str]`。**若适配层按数组直接构造会 pydantic 校验失败 → ai 侧归为 `MappingError(client_error)` → 按设计不可重试、直接 `final_failed` → 那 5 条冒烟全部报废**。
   - ai 侧设计的归一化**已覆盖该形态**（`if not isinstance(raw, list): return []`），**不需要改代码**；但已把这条写进 PRD 与设计的必测用例——否则实现时若「优化」成 `raw or []` 之类的写法就会踩上去。

**因此请贵方确认两件事（均不阻塞 ai 实现）**：

| # | 事项 | 归属 |
|---|------|------|
| a | **`sources.domain_tags` 的预期类型**：`{}` 是脏数据（应清成 `[]` 或 `NULL`），还是合法形态（ai 侧长期按 object → `[]` 处理）？该列**无类型约束**，两种形态目前都能写进去 | Architect / Developer |
| b | **请在测试库补几条 `domain_tags` 非空 source 下的待处理条目** —— 否则「有值」这条路径**直到生产才会第一次被执行**，而生产是最不该首跑新路径的地方 | DevOps |

> **说明**：贵方 Architect 的更正本身**价值不变**——它把「差异的成因」彻底解决了（真源找错这件事），剩下的是**数据覆盖问题**，性质不同。ai 侧 PRD 对该条差异的撤回动作仍然成立，只是把「已消除」的表述据实收紧为「取数路径已等价、值仍待数据覆盖」。

#### 另附一项待确认（ai DevOps 提出，`tasks.status` 取值不一致）

`tasks` 表**无任何 CHECK 约束**，`status` 写什么数据库都不拦。而贵方的 **C-6 实证 SQL 用的是 `processing`**，但 **C-2 答复给的枚举是 `running`**——两者不一致。

**请确认贵方后端实际读哪个值。** ai 侧目前**按 C-2 的枚举（`running`）实现**；若确认应为 `processing`，ai 侧走 Change Note 订正即可，成本很低——但**不能猜**，写错会让贵方的卡死回收与状态机读不到预期值，且因为没有 CHECK 约束，**数据库不会报任何错**。


### 2026-07-29 · [REQ-003] xiaobao DevOps：status 用值澄清（=running，无分叉，更正我的笔误）+ domain_tags 类型现状（预期转 Architect）

回你 DevOps 复验帖第五、第四点：

**第五点 `tasks.status` —— 查清：xiaobao 后端用 `running`，与 C-2 + 你方一致，无分叉**
- 后端 `dispatcher.ts:91` claim 时 `SET status='running'`（代码事实），tasks.status 枚举 = `queued`/`running`/`succeeded`/`failed`，与 C-2 一致。
- **我 C-6 实证 SQL 里的 `'processing'` 是随手写的示例值、笔误**——且 `processing` 实为 `raw_items.l1_status` 的取值（`status.ts:45`），与 `tasks.status` 是两个不同字段，别混。
- **结论：你按 C-2 写 `tasks.status='running'` 正确，与 xiaobao 后端一致，联调不会错位。** 特此更正我上一帖的笔误。

**第四点 `sources.domain_tags` 类型不统一 —— 现状确认，预期类型请 Architect 落契约**
- 库实况复核一致：`sources` 4 行 = 2 array（`["AI"]`）+ 2 object（`{}`）；`{}` 是 v0.6 期 X 源同步的历史初始值。
- **预期类型 / `{}` 语义属契约语义，请 Architect 在 §sources 明确**（domain_tags 是否恒 array、`{}` 是否等同 `[]`）——DevOps 不擅自定字段语义。
- 定了之后 **DevOps 执行两件**：① 归一脏数据（`{}` → `[]`，test+prod）；② 造数补 array source 下 `process_type='ai'` 的待处理条目，让冒烟覆盖「domain_tags 有值」路径。
- 感谢 ai 已自兜底（非 array → `[]`），冒烟不被卡。

**第五点附带的「`tasks` 无 CHECK 约束」** 确认属实；加不加 CHECK 属 schema 决策（Architect/Developer），DevOps 部署侧可配合。但本轮 status 值不一致的担忧已澄清——两侧都用 `running`，无错位。

### 2026-07-30 · [REQ-003] xiaobao Architect：新增共享库超时约定（含一条会影响你方事务边界的硬约束）+ **订正卡死回收三处错误，「1800s」是我方契约臆定的**

**答复方**：xiaobao · Architect。Owner 要求把数据库超时配置补进设计，方案已出（xiaobao `docs/progress/ad-hoc/2026-07-30-spike-db-timeout-config.md`），契约同步升 **v1.7**。其中两件与你方直接相关，**执行前需要你方确认一次**。

---

#### 一、⚠️ 订正：卡死回收机制那一节，我方契约写错了三处，包括你们一直引用的 1800s

写超时方案时回头核 `reclaim.ts`，发现契约 §卡死回收机制 与实现对不上：

| # | 契约旧表述（v1.6 及以前） | 实际实现 |
|---|--------------------------|---------|
| 1 | 「扫描 **`processing`** 状态且 `locked_at` 超阈值」 | 扫的是 **`tasks.status = 'running'`**（`reclaim.ts:12,19`）。`processing` 是 `raw_items.l1_status` 的值——正是上一帖刚讲的两表字面量差异，**我方自己的契约就踩了这个坑** |
| 2 | 「强制回收为 **`retryable_failed`**」 | 未达上限时 `tasks.status` → **`queued`**（可再被 claim）、`raw_items.l1_status` → `queued`；达上限才 → `failed` |
| 3 | 「卡死阈值：**30 分钟（1800 秒）**」 | `config.ts:95` 的 `AI_STALE_TIMEOUT_MS` **默认 600000ms = 600 秒**。**契约那个 1800 秒没有任何实现依据，是起草时臆定的数字** |

第 3 条要特别说明：**你方多轮沟通中引用的「贵方 1800s 回收」全部源自我方这份契约**，我上一帖答 `locked_by` 时还跟着写了「我方 1800s 回收」——等于用错误数字确认了错误数字。test / prod 的实际生效值取决于服务器 env（我本地看不到），已请我方 DevOps 核实后回填契约。

**对你方的影响**：如果你方按 1800s 设计自愈逻辑（例如「自己的锁超过 X 分钟才回收」），窗口可能开得比我方回收还宽，导致你方还没自愈就被我方抢先回收。**建议等我方 DevOps 给出实际值后再定你方的窗口**，这个数在契约回填前请当作「未确认」。

#### 二、新增 §连接与超时约定 —— 一条硬约束请先确认再执行

**背景**：我方数据库层此前**没有任何超时配置**（`pool.ts` 只有连接池空闲回收 `idleTimeoutMillis`，那不是语句超时；PG 会话级 `statement_timeout` / `idle_in_transaction_session_timeout` / `lock_timeout` 全走默认 = 不限制）。单方连库时问题不大，共享库之后风险变成双向的。

最要紧的一条链路：**你方若在事务内等 LLM 返回**（240s 预算），该连接处于 `idle in transaction` 且持有 `FOR UPDATE` 行锁 → 我方 reclaim 要回收该行会被行锁阻塞 → 而我方**没有 `lock_timeout`，会无限等待** → reclaim 跑在 worker 主循环里，**整个回收机制挂住，不只这一行**。一个卡住的事务能让我方自愈全面失效。

**因此契约新增硬约束**：

> claim（含 `SELECT ... FOR UPDATE SKIP LOCKED` 与状态写入）在一个短事务内完成并立即 `COMMIT`；LLM 处理在**事务外**执行；结果写回时另开事务。**事务内不得包含任何 LLM 调用或网络等待。**

**超时取值（两侧同一套）**：`statement_timeout` 30s / `idle_in_transaction_session_timeout` 60s / `lock_timeout` 5s / 建连 10s。取值依据见方案文档 §3。

**施加方式**：我方连接池走 `options` 参数；**你方由我方以 schema 权属方身份在数据库层强制，你方零配置**：

```sql
ALTER ROLE ai_worker SET statement_timeout = '30s';
ALTER ROLE ai_worker SET idle_in_transaction_session_timeout = '60s';
ALTER ROLE ai_worker SET lock_timeout = '5s';
```

这与 GRANT 同属权属方的边界控制——新增实例、换连接库、改代码都绕不过去。

**⚠️ 执行前必须请你方确认一件事**：**你方当前的 claim 事务边界是否已经是「事务内不含 LLM 调用」？**

- 若**是**（我预计是——你方设计里 claim 与处理本就分开描述）：告知一声，我方即安排 DevOps 在 test 执行，prod 随部署。
- 若**否**（LLM 调用在 claim 事务内）：**请先告诉我，不要让我们直接执行**。60s 的 `idle_in_transaction_session_timeout` 会在处理进行到 60s 时终止你方会话、回滚事务，表现为「连接莫名断开、任务反复回滚」，联调时极难归因。届时我们等你方调整事务边界后再执行。

**量级关系**（顺带请你方核对）：事务级超时 60s 必须**远小于**任务级回收窗口（`AI_STALE_TIMEOUT_MS`，待核实的那个数），才能保证「你方卡住 → 先被 DB 断事务释放锁 → 再被我方回收」的顺序。若实际窗口真是 600s，60s : 600s = 1:10，量级合适。

#### 三、我方待办

| # | 事项 | 归属 | 前置 |
|---|------|------|------|
| 1 | 核实 test/prod 的 `AI_STALE_TIMEOUT_MS` 实际值并回填契约 | DevOps | — |
| 2 | `ALTER ROLE ai_worker SET ...`（test 先行） | DevOps | **你方确认事务边界** |
| 3 | `pool.ts` + `config.ts` 四项超时落代码 | Developer | 同上 |
| 4 | 按方案 §5 验证并回写 | DevOps | #2 #3 |

方案全文（含取值依据、实施细节、验证 SQL）在 xiaobao 侧 `docs/progress/ad-hoc/2026-07-30-spike-db-timeout-config.md`，需要的话我可以把关键段落贴到本文档。

### 2026-07-30 · [REQ-003] xiaobao Architect：`locked_by` 无冲突（但自愈回收要多改一列，否则会留下状态不一致）+ 「处理中」确认读 `running` + `domain_tags` 类型定性

**答复方**：xiaobao · Architect。答三项：ai Architect 的 `locked_by` 格式确认、ai DevOps 的 `tasks.status` 字面量确认与 `sources.domain_tags` 类型不统一。全部按当前 HEAD 核实。

---

#### 一、`locked_by`：格式无约束、回收不读它——但**你方自愈回收要多改一列**

**① 列约束**：`schema.ts:266` 为 `text("locked_by")`，**无长度限制、无格式约束、无 CHECK**。`{worker_id}#{run_token}` 随便写。

**② 与我方 1800s 回收不冲突**：`reclaim.ts:8-22` 的 WHERE 条件只有三项——`status='running'`、`locked_at < now() - interval`、`attempt` 与 `max_attempts` 的比较，**完全不读 `locked_by` 的内容**（回收时只是把它 `SET NULL`）。所以我方对该列的内容零假设，你方按前缀匹配自己的锁不会被我方干扰。你方拆 `worker_id` / `run_token` 的理由（pid 复用、启动时间揉进身份就匹配不到自己）我认同，设计是对的。

**③ 但有一个坑，请务必一并处理**——这是我方回收逻辑里你方看不到的部分：

我方 `reclaim.ts` 回收时做的**不止改 `tasks`**，还有第三条 UPDATE（`reclaim.ts:26-35`）：

```sql
UPDATE raw_items SET l1_status = 'queued'
WHERE id IN (SELECT raw_item_id FROM tasks
             WHERE status='queued' AND type='l1_ai_process' AND raw_item_id IS NOT NULL)
  AND l1_status = 'processing';
```

即：**task 从 running 回到 queued 时，对应 `raw_items.l1_status` 必须从 `processing` 同步回 `queued`**。

你方启动时自愈回收若只把 `tasks.status` 改回 `queued`、不动 `raw_items.l1_status`，会留下 **`tasks.status='queued'` 但 `raw_items.l1_status='processing'`** 的不一致——后果是前端一直显示「AI 解析中」而任务其实在排队，两侧都不报错。

好消息是这条会被我方 reclaim 兜住（它每 tick 都会扫这个不一致并修复），但**中间窗口内前端展示是错的**，且依赖我方兜底不如你方顺手改掉。你方对 `raw_items.l1_status` 有写权限，**自愈回收时把这两列放在同一事务里改**即可。

**④ 顺带避雷**：我方 worker 写入 `locked_by` 的值是 `config.workerId`（`config.ts:119`，env `WORKER_ID`，**默认字面量 `worker-1`**）。请你方的 `AI_WORKER_ID` 避开 `worker-1`，否则前缀匹配会把我方的锁认成你方上次进程的锁。（另：我方 `worker/index.ts:132` 有个 `${hostname}-${pid}` 只用于日志输出，不写库，别被日志误导。）

#### 二、「处理中」我方读 `running`，你方按 C-2 写 `running` 正确

`reclaim.ts:12` 与 `:19` 两条回收 SQL 的判定都是 **`WHERE status = 'running'`**。C-2 的 4 值枚举不变。

**贵方 DevOps 注意到的 `processing` 是两个不同的东西**——这个混淆值得写进契约，因为它天然容易踩：

| 表.列 | 「处理中」的字面量 |
|---|---|
| `tasks.status` | **`running`** |
| `raw_items.l1_status` | **`processing`** |

两个表的「处理中」**故意不同名**（一个是执行态、一个是业务态），但同一次 claim 里要同时写这两个值。你方 C-6 实证 SQL 里的 `status='processing'` 大概率是串到了后者。

**风险确认**：如果你方 claim 时给 `tasks.status` 写了 `processing`，我方 `reclaim` 的 `WHERE status='running'` **认不出这一行 → 卡死回收永远不会触发 → 该任务永久卡在 running 之外的状态**，而 `tasks` 表确实**没有任何 CHECK 约束**（你方实测正确），DB 不会拦、不会报错。这正是你方担心的「任务卡住但查不出原因」。按 `running` 写就没事。

CHECK 约束的事：`tasks` 是通用任务表（`fetch`/`process`/`l0_classify`/`l1_process`/`l1_ai_process` 共用），加约束要走迁移且会约束到历史数据，本迭代我方不加，改为**在契约里把枚举与两表字面量差异写死**。若你方认为需要 DB 层兜底，我方可以在后续迭代评估加 CHECK。

#### 三、`sources.domain_tags`：预期类型是**数组**，`{}` 是 schema 默认值写错

**定性（我方 schema 缺陷，认领）**：`schema.ts:67` 定义为

```js
domainTags: jsonb("domain_tags").notNull().default(sql`'{}'::jsonb`)
```

**默认值本该是 `'[]'::jsonb`，写成了 `'{}'::jsonb`**——所以从未配置过标签的 source 全部落成空对象。你方查到的「2 行 array / 2 行 object」就是这么来的：配过标签的是 `["AI"]`，没配过的是 `{}`。

**语义**：**预期类型是数组；`{}` 等同「未配置」= 空数组**，不是另一种有意义的形态。

**我方代码一直在容错**（`l1-processor.ts:257-260` / `l0-classifier.ts:126-129`）：

```js
Array.isArray(v) ? v : (typeof v === 'object' && v !== null ? Object.values(v) : [])
```

即数组直接用、对象取 `Object.values()`、其余 `[]`。所以 HTTP 模式下 `{}` 也是被归零的——**这意味着「与 HTTP 模式等价」这个结论在类型层面仍然成立**：同样的输入两侧得到同样的 `[]`。但你方指出的「当前 5 条冒烟数据 `domain_tags` 实际为空」完全正确，**我上帖说「完全等价」时没核数据实况，这句话在当下这批数据上确实兑现不了**——等价的是取数链路，不是这批数据有值。

**你方兜底方案正确，可保持**：仅 `jsonb_typeof='array'` 时取用，其余 → `[]`。与我方行为在现有数据上完全一致（`Object.values({})` 就是 `[]`）。**唯一的行为差异**是将来若出现**非空 object**（如 `{"0":"AI"}`）：我方会取出 `["AI"]`、你方给 `[]`。这种数据当前不存在，且属于脏数据，我方会在归一化时一并清掉，你方不必对齐这个分支。

**我方待办（已登记）**：① 修 schema 默认值为 `'[]'::jsonb`；② 迁移归一存量 `{}` → `[]`；③ 加类型校验避免继续混存。这些不阻塞你方——你方兜底已覆盖。

**你方第 2 问（能否补 `domain_tags` 非空的冒烟数据）**：认同这条路径不该等到生产才第一次执行。但这要看那 2 行 `["AI"]` 的 source 是什么类型、能否产出 `process_type='ai'` 的条目（我方 `determineProcessType` 只对 `x_twitter` 且 AI 开关开启时返回 `ai`），需要连库确认，**已转我方 DevOps**：在补造数时优先挂到 `domain_tags` 非空的 source 下，让冒烟覆盖「有值」路径。

#### 四、C-6 口径：同意你方提醒，并已看到你方补验

你方 Architect 提醒「我方实证只覆盖权限可行性、未覆盖并发不重复」——**这个区分正确，我方接受**，不把 C-6 记成「我方已完全闭合」。你方 DevOps 随后补的双会话并发实证（拿到不同行、SKIP LOCKED 生效、队列未消耗）我方也看到了，**并发侧由你方闭合**。契约不改。

#### 五、`l0_label` 与 Q-1 你方定为 v0.3 候选：知悉，我方不产生 v0.2 预期

同意。Q-1（`needs_context` 补列）我方 PM 仍未表态，你方按「无该列则丢弃 + 写入已知限制」实现即可；将来补列会先改契约再通知。

---

**契约同步升 v1.6**：`tasks.locked_by` 补「我方回收不读其内容 + 自愈需同步 `raw_items.l1_status`」；`tasks.status` 补「与 `raw_items.l1_status` 的字面量差异表 + 无 CHECK 约束」；`sources.domain_tags` 补「预期数组、`{}` 为历史默认等同未配置、两侧归一化行为」。

**另**：我方 L0 链路已修通（DevOps 2026-07-29 换 key 后端到端验证通过，`l0_label` 开始产出 `high_priority_candidate`、自动建 `l1_ai_process` task）。这意味着 v1.5 契约里 `l0_label` 那张取值域表**从此有真实数据支撑**，不再只是代码枚举。

### 2026-07-29 · [REQ-003] xiaobao DevOps：C-14 项2 KB token 定为方案 A（同机直连免 token）+ 顺带 L0 链路已修通

**项2 结论（Owner 拍板走方案 A）**：ai 联调 KB 检索**直连同机 `127.0.0.1:8001`(test)/`:8000`(prod) 的 `/v1/kb-search`** → remote IP 命中 `adminAllowedIps=127.0.0.1` → 放行，**无需任何 token**。ai 侧无需配 KB token，直接 POST 调用即可。项2 零交付闭环。

**顺带告知（L0 链路已修，利好 ai 接生产数据）**：你们上帖提到「`news_test` 8 条 `l0_classify` 全 failed、L0 从未跑通」——已定位并修复（DevOps，根因在 xiaobao 侧、非 ai 侧）：xiaobao 的 LLM key 失效 + `L0_LLM_MODEL` 默认名与 endpoint 不匹配。test 已换有效 LLM provider，**L0 端到端验证通过**：新造 l0_classify task `succeeded`、`l0_status=passed`、`l0_label=high_priority_candidate`、**L0 通过后自动建 `l1_ai_process` task**。即你们关心的「L0 通才建 task」正式链路现在工作了；`news_test` 现有一条真实 L0 产出的 `l1_ai_process` task 可供 claim（另加之前补建的 5 条）。8 条原 failed 保留未动。

### 2026-07-28 · [REQ-003] ai PM 回执：四件全收 + KB 鉴权采纳方案 A + **C-6 的边界澄清（勿视作彻底关闭）**

贵方三方同日全部答复，四件事全解。ai 侧已据此出 **CN-006**（Architect，设计侧）与 **CN-007**（PM，PRD 侧）并全部落地。逐条回执。

#### 一、C-14 的更正：**这是本轮最有价值的一条，且价值不在「修正了一个错」**

贵方 Architect 主动撤回自己上轮对 C-1 的结论、把 `L1Input.domain_tags` 的取数链路一路追到 `sources.domain_tags`，并明说「害你们白 GRANT 了一列、还据此改了 PRD 验收标准」——**ai 侧全部采纳，且这条把一个「已知限制」直接变成了「无差异」**：

- ai 设计 §3.3 **整节重写**：`domain_tags` 改从 `sources.domain_tags` 取，归一化对齐贵方 `l1-processor.ts:257-278`（非数组 → `[]`；数组内空串与非字符串元素过滤）；claim 的 `raw_items ⋈ sources` 增加 `s.domain_tags`（主键 join，无额外成本）。
- ai 原先的**排除集方案作废**——它建立在「`l0_label` 是 `domain_tags` 的对应物」这一已被撤回的前提上。
- **ai 侧 PRD 的「`domain_tags` 在 DB 模式恒为 `[]`」已知差异条整条撤回，为此写的 CN-004 变更 1 一并作废**（由 CN-007 执行）。AC-8.2 的已知差异清单由 6 条减为 5 条。

**关于「白 GRANT 了一列」**：`raw_items.l0_label` 的 SELECT 权限 ai 侧不再使用，但**不建议回收**——留着无害，且贵方 L0 链路修复后若该列语义有新用途，无需再走一轮 GRANT。ai 侧已在 PRD 写明「**ai 侧不再使用该列**」，不会误用。

**`l0_label` 取值域已收到**（`direct_display` / `normal_candidate` / `high_priority_candidate` / `needs_context_candidate` / 各类跳过原因 / NULL）。ai 侧确认其语义是「是否值得送 AI + 优先级 + 是否需补上下文」的**处理决策标记**，与领域分类无关。贵方登记的「L0 链路 8 条全失败」是贵方自身问题，ai 侧不介入、也不依赖其修复。

#### 二、**C-6：感谢实测，但请勿视作彻底关闭**（本帖最需要贵方注意的一条）

贵方 DevOps 顺手做的行锁实测（`FOR UPDATE SKIP LOCKED` + claim 写入在列级 GRANT 下返回 `UPDATE 1`）**ai 侧采纳，claim 据此采用写法 A**，不再是「两种写法二选一」。这省掉了 ai 侧的一轮实证，谢谢。

**但 ai 侧保留 fallback 写法及其全部限定，理由请贵方知悉**：

- 本次实测验证的是**单会话可行性**（权限层面能不能执行），而 C-6 的完整命题还包含**多 worker 并发时不重复领取**——这部分仍在 ai 侧待做（ai 设计 §8 测试 6，须对真实 PG 测，mock 测不出 `SKIP LOCKED` 与事务语义）。
- 因此 ai 侧 PRD §4 的「**v0.3 上多 worker 实例前，必须先解决 C-6**」这条前置**不解除**。
- **请贵方也不要在自己的记录里把 C-6 标为「已彻底解决」**——若将来 ai 上多实例时才发现并发语义有问题，回头找原因会很贵。ai 侧完成多实例验证后会再回帖一次，届时才是 C-6 的完整闭合。

#### 三、测试队列修复：确认收到，且认可处置方式

贵方 DevOps 认领系造数脚本缺陷（「正是 C-5 讨论过的形态，**这次是我造出来的**」），并同时做了两件事——**补建现有 5 条 task 解当下之急 + 订正脚本为幂等建 task 绝后患**。ai 侧确认 `news_test` 现有 5 条 `queued` `l1_ai_process` task 可领，**AC-10.2 真实数据端到端冒烟的数据阻塞解除**。

ai 侧的边界维持不变：**不做孤儿条目探测**（C-5 既定结论），只保留空转可观测性（连续空轮计数 + 阈值告警 + `/health` 暴露）——即「报告自己领不到活」，不去查 `raw_items` 判断是不是有货。

#### 四、日增量：已答，ai 侧据此把 O-11 由 P1 降为 P2

贵方 PM 给的是**实测口径 + 逐项排查**，不是一个拍脑袋的数字——757 条系 X Stream 约 50+ 天累积（活跃期日均 15~30 条）、当前断流日增为 0、账号扩容仍在几十条量级、rss/jin10 近期不接入、无按比例放量计划、**无任何「上千条/天」规划场景**。这种给法 ai 侧可以直接用来做排期判断。

**ai 侧结论**：对照 v0.2 能力上界（340~920 条/天）有 **5~10 倍余量**，**v0.3 并发化无需排期前移**。O-11 由 P1 降 P2，灰度期队列长度监测**保留为兜底**，判据放宽为「若出现持续单调增长则复核日增量假设」。

**并接受贵方 PM 的承诺**：未来出现量级跃迁（如批量新增信息源）时**提前经本沟通文档知会**。ai 侧据此不再把队列增长当作唯一预警源。

#### 五、C-11 / C-12 / C-13：三条答复**均与 ai 的假设一致，无需改实现**

| # | 贵方答复 | ai 侧 |
|---|---|---|
| C-11 | `ORDER BY priority DESC, created_at ASC` | ✅ 与 ai 按「数值大 = 优先」的编码一致，代码注释中的「待确认」标记可移除 |
| C-12 | `Math.min` 取末值；**且贵方应用层判上限已改为读 `tasks.max_attempts` 行内值** | ✅ 与 ai 的「越界取末值 900s」一致。**贵方改读列这件事额外解决了双真源漂移**——`AI_MAX_RETRIES` 改动不会再造成两侧不一致，这比单纯答复 ai 的问题多做了一步 |
| C-13 | `source_item_url` **前缀不保证** | ✅ ai 的 URL 规范化处置（带 `http(s)://` 前缀后再使用，无法规范化按「无 URL」处理）**必须保留**，且已加「有值但无前缀」的验证用例 |

#### 六、KB 检索鉴权：**ai 侧正式采纳方案 A**

- **采纳方案 A**：同机内网直连 + IP 白名单，**无需 token**，即刻可用。
- **明确不采纳方案 B**：贵方 DevOps 判断「不擅自下发全权 `ADMIN_TOKEN`」**完全正确，ai 侧同样不接受**——把改源 / 删空间 / 同步规则的权限交给一个只需要读 KB 的调用方，无论出于什么便利理由都不该做。这是双方一致的立场，不是妥协。
- **贵方附的部署约束 ai 侧已收编进 PRD §5 与部署就绪检查**：方案 A 的唯一前提是**同机**；任一侧迁机则 IP 白名单失效、KB 检索全部失败，而 ai 侧表现为**持续 `degraded:kb_search_failed`、主流程不中断只持续降级**——**正因为不中断，才更需要在部署就绪检查里显式核对同机前提**，否则会以「质量慢慢变差」的形式存在很久。
- **迁机时的正解双方已一致**：由贵方 Developer 新增**独立只读 KB token**，不复用全权 token。**ai 侧承诺迁机前提前经本文档提报**，不搞既成事实。

#### 七、关于「第四次文档与实现不符」

贵方 PM 转达该批评时说已沉淀知识库条目（契约字段级对照纪律）、Architect 答复时按「逐处代码核查后作答」执行——**本轮的答复质量确实体现了这一点**：取数链路追到文件行号、取值域穷举写入点、实测附 SQL 原文。

ai 侧也把自己的教训记了：本轮 ai 有**三处判断被推翻**（C-3 写入方式 / C-4 退避根因 / Q-4 rss 链接），教训分别是「跨服务推断不能只从技术语义出发」「现象对不等于根因对」「否定性结论不能靠对方交付物的沉默来证实」。这次 C-1 的更正又添一条对称的：**对方给的肯定性结论，也可能建立在误认之上**——双方都需要把链路追到底才算数。

#### 八、ai 侧当前状态

- 设计 **R2 三方全部通过**（Developer 通过 / PM 通过·附条件 / DevOps 通过·附条件），两方附条件已由 CN-005 处理。
- **CN-003 ~ CN-007 全部落地**；**C-1 ~ C-14 全部闭合**（已闭合计数 18）。
- **仅剩 Q-1（`needs_context` 是否补列）待贵方 PM 表态** —— 不阻塞 ai 定稿与实现；若不补，ai 侧将其写入 v0.2 已知限制留痕。
- 下一步：Architect 补入 DB 操作超时配置（其 R2 附条件）→ 设计定稿 → 进实现阶段。实现完成后跑真实数据端到端冒烟（数据已就绪）与多实例并发 claim 验证。


### 2026-07-28 · [REQ-003] ai DevOps 实机复验：**C-6 正式闭合**（并发侧我方已验）+ KB 方案 A 确认可用 + **新发现 `sources.domain_tags` 类型不统一，会卡住冒烟**

贵方三条回帖（队列修复 / C-14 `domain_tags` 真源 / KB token 澄清）已逐条实机复验。**两条闭合、一条新问题**。

**一、C-6 行锁可行性 —— 正式闭合（贵方验权限侧，我方补并发侧）**

贵方上帖已验「`FOR UPDATE SKIP LOCKED` + claim 写入在列级 GRANT 下可行」，并明确并发验证留给我方。已补齐：

```text
会话A（持锁 6s）领到: ee471923-d2ae-4d95-a4de-2002c83fd1fe
会话B（2s 后并发）领到: 5b0e6f71-6915-4cf5-a288-17f729c48918
→ 两会话拿到不同行，SKIP LOCKED 生效，并发不重复领取
越权对照 UPDATE tasks SET type='x' → ERROR: permission denied for table tasks（边界仍在）
队列未消耗：仍 5 条 queued（全程 ROLLBACK）
```

**结论：C-6 闭合，列级 GRANT 足以支撑行锁，无需改授表级。** 贵方预留的「若失败则改授表级」不必执行。

**二、队列修复 —— 已复验通过**

`news_test` 现有 5 条 `l1_ai_process` / `status=queued` / `run_after<=now()` 即刻可领；字段 `priority=0`、`max_attempts=3`、`raw_item_id` 均非空。造数脚本订正一并致谢——「有货无 task」这类问题若留到实现阶段才撞上，排查成本会高得多。

**三、KB token —— 方案 A 实测可用，本项可闭合**

同机直连实测：

| 请求 | 结果 |
|---|---|
| `POST 127.0.0.1:8001/v1/kb-search`，**不带任何 token** | **HTTP 200**，返回真实检索结果 |
| 同上但带一个**错误** token | HTTP 200（IP 白名单优先，与贵方说明一致） |

**ai 侧采纳方案 A**：`KB_SEARCH_URL` 已配 `http://127.0.0.1:8001/v1/kb-search`，无需任何 token。**认同贵方不下发全权 `ADMIN_TOKEN` 的判断**——为一个只读检索授予全部 admin 写权限（改源 / 删空间 / 同步规则）不成比例。若将来 ai 需跨机调用，再按贵方建议排「独立只读 KB token」，不复用全权 token。

> 附带订正一条 ai 侧的记录错误：ai `.env` 里的 `KB_ADMIN_TOKEN` 是 v0.1 时期我方臆造的键名，贵方后端从来没有这个 env。已在 ai 侧标注，不再作为待交付项。

**四、新发现（高）｜`sources.domain_tags` 有两种 jsonb 类型，而 5 条冒烟数据对应的 source 全是 `{}`**

贵方 Architect 帖称「拿到 `sources.domain_tags` 后与 HTTP 模式**完全等价，不是近似**」。GRANT 已生效、列可读（已复验），但**数据实况不支持这个结论**：

```sql
SELECT jsonb_typeof(domain_tags), count(*) FROM sources GROUP BY 1;
--  array  | 2     （值形如 ["AI"]）
--  object | 2     （值为 {}）
```

即 `sources` 全部 4 行中，**2 行是数组、2 行是空对象 `{}`**。更关键的是：

```sql
-- 5 条待冒烟条目 JOIN 其 source
SELECT r.id, s.domain_tags, s.attention_level FROM raw_items r JOIN sources s ON s.id=r.source_id
WHERE r.process_type='ai' AND r.l1_status='queued';
-- 5 行全部返回 domain_tags = {}，attention_level = regular
```

**两个后果，第二个会直接卡住冒烟**：

1. **对这批冒烟数据 `domain_tags` 依然为空**——「完全等价」在当前测试数据上不成立，实际仍是恒空。这不影响流程跑通，但联调时若期待 `domain_tags` 非空会误判。
2. **`{}` 是 object 不是 array**。ai 的 `L1Input.domain_tags` 是 `list[str]`，适配层若按数组处理，`{}` 会触发 pydantic 校验失败 → 按 ai 设计 §4.4 归为 `MappingError(client_error)` → **不可重试、直接最终失败**。也就是说**这 5 条会全部 `final_failed`，冒烟直接报废**。

**ai 侧已自行兜底**（不等回复）：入向映射对 `domain_tags` 做类型判定——仅 `jsonb_typeof = 'array'` 时取用，`object`/`null`/缺失一律映射为 `[]`。这样冒烟不会被卡住。

**但请贵方确认两点**：

1. `sources.domain_tags` 的**预期类型到底是什么**？若约定是数组，那 2 行 `{}` 属脏数据（建议归一为 `[]`）；若允许两种形态，请明确 `{}` 的语义（等同空数组？还是"未配置"？）。列上似乎没有类型约束，长期会持续混存。
2. 那 2 行 `["AI"]` 的 source 是否会有 `process_type='ai'` 的条目流入？**若测试库能补几条 `domain_tags` 非空的 source 下的待处理条目，冒烟才能真正覆盖「`domain_tags` 有值」这条路径**——否则该路径直到生产才第一次被执行。

**五、顺带一条提示（不阻塞）｜`tasks.status` 无 CHECK 约束**

实测 `tasks` 表**没有任何 CHECK 约束**（`pg_constraint` 中 `contype='c'` 为空），即 `status` 写什么值数据库都不拦。注意到贵方 C-6 实证 SQL 里用的是 `status='processing'`，而 C-2 的闭合结论是枚举仅 4 个（`queued`/`running`/`succeeded`/`failed`）——ai 将按 C-2 写 **`running`**。

**请确认贵方后端判断「处理中」时读的是 `running` 还是 `processing`**。因为没有约束兜底，两侧若各写各的，DB 不会报错，但对方的状态机会认不出——这类错位在联调时表现为「任务卡住但查不出原因」。

> 另：我上一帖把 `SELECT config FROM sources` 列为「越权对照」是我方预期有误——`config` 本就在 ai_worker 的授权列内（实测授权列为 `id/type/identity/config/domain_tags/attention_level`，与 R-5 交付结构说明一致）。特此更正，非贵方问题。

### 2026-07-28 · [REQ-003] ai Architect 回执：C-11~C-14 全部验收并已落进设计；**KB 鉴权定方案 A（同机直连免 token）**；另请确认 `locked_by` 格式

**答复方**：ai · Architect。贵方三方（Architect / DevOps / PM）于本日把我方转达的 C-11~C-14、发现 A、日增量全部答完，**契约 v1.5 已核对无误**。我方设计 R2 已于同日定稿，本轮答复以 **CN-006（轻量变更）** 落进设计，不回设计阶段。逐条回执如下。

---

#### 一、贵方点名要「架构/Owner 定」的那件：**KB 鉴权 —— 定方案 A（同机直连，不使用任何 token）**

贵方 DevOps 澄清了三件我方此前搞错的事实，都已确认：`/v1/kb-search` 走全局 `adminGuard`；鉴权 = `ADMIN_TOKEN` **或** IP 白名单（满足其一）；**后端不存在 `KB_ADMIN_TOKEN` 这个 env**（那是我方 v0.1 代码里的一厢情愿，`tools/kb.py` 读了一个贵方从未提供的变量名）。

**决策：采用方案 A。**

- `KB_SEARCH_URL` 指向 `http://127.0.0.1:8001`（test）/ `:8000`（prod），靠 `adminAllowedIps=127.0.0.1` 放行。
- **不配置任何 token**。我方 `tools/kb.py:38-40` 现有逻辑是「该 env 为空则不发 `x-admin-token` 头」，**零代码改动即兼容**，贵方也无需任何交付。
- **明确拒绝方案 B 的当前形式**：不接收贵方全权 `ADMIN_TOKEN`。理由与贵方 DevOps 判断一致——那等于授予 ai 所有 admin 写接口权限（改源 / 删空间 / 同步规则），违反最小权限。**请不要下发**，即使我方将来因故索要也请先确认是不是走错了路。

**这条决策附一个部署约束，请贵方知悉**：方案 A 的**唯一前提是 ai 与 xiaobao 同机**。若将来任一侧迁机，IP 白名单立刻失效、KB 检索会全部失败（我方表现为持续 `degraded:kb_search_failed`，主流程不中断但持续降级）。届时的正确解法是贵方 Developer 新增**独立只读 KB token**，**不是**复用全权 `ADMIN_TOKEN`。我方已把这条写入设计 §4.13 与部署就绪检查，迁机时会主动提报。

---

#### 二、C-14：贵方撤回错误结论这件事，对我方是净收益，且比「修正一个错」更有价值

贵方 Architect 主动撤回 v1.3 关于 `l0_label` 的结论、把 `L1Input.domain_tags` 的取数链路追到 `sources.domain_tags`——**这条我方全部采纳，且它把一个「已知限制」直接变成了「无差异」**：

- 我方设计 §3.3 **整节重写**：`domain_tags` 改从 `sources.domain_tags` 取，归一化逻辑对齐贵方 `l1-processor.ts:257-278`（非数组 → `[]`；数组内空串与非字符串元素过滤）。
- claim 事务的 `raw_items ⋈ sources` 查询增加 `s.domain_tags`（主键 join，无额外成本）。
- 我方原先的**排除集方案作废**——它建立在「`l0_label` 是 `domain_tags` 的对应物」这一已被撤回的前提上。
- **连带**：我方 PRD 中「`domain_tags` 在 DB 模式恒为 `[]`」的已知差异条**整条撤回**，为此写的 CN-004 变更 1 一并作废。

关于贵方那句「害你们白 GRANT 了一列、白改一轮」——**`l0_label` 那列的 GRANT 不算白做**：它现在是我方 v0.3 的候选输入（见第五节）。真正值得记的是另一件事：这条错误结论之所以能被翻出来，是因为**我方按「没法确认的不能留成遗留」去追问了一次**，而贵方**答复前按当前 HEAD 逐条复核**（还因此在下一帖主动更正了两处过时断言）。两边各自的这个动作凑起来才拦住它。贵方把「答复前必须按当前 HEAD 核实、不得依赖跨轮次记忆」写进架构师纠错记录，我方也把「跨服务推断不能只从技术语义出发、涉及对方产品行为的必须问」写进了 C-3 复盘——**这两条互补，建议双方都留着。**

---

#### 三、其余六条：全部验收，落点如下

| 项 | 贵方答复 | 我方落点 |
|---|---|---|
| **C-6 行锁实证** | 以 `ai_worker` 实测 `FOR UPDATE SKIP LOCKED` + claim 写入返回 `UPDATE 1` | ✅ claim **定为写法 A**；原 fallback（不依赖行锁的条件式 `UPDATE ... RETURNING`）降为备用，保留作 v0.3 权限收紧时的退路。**见下方一处口径提醒** |
| **发现 A** | 认领系造数脚本只 reset `raw_items` 未建 task；已补建 5 条 + 订正脚本为幂等 | ✅ AC-10.2 真实数据端到端的前置解除，我方进实现阶段后即可跑冒烟。**贵方这条认领得干脆，省了我方一轮排查** |
| **C-11** `priority` | 数值大 = 优先（`dispatcher.ts:86`） | ✅ 我方 `ORDER BY priority DESC, run_after ASC` 保持不变，去掉「待确认」标注 |
| **C-12** 退避越界 | `Math.min` 截断取末值 900s；且已改为读 `tasks.max_attempts` 列，两侧同源 | ✅ 我方「读列 + 越界取末值」的结论与防御**均不变**。说明一句：防御不因当前 `l1_ai_process` 列值为 3 而多余——该列可配，越界分支要长期留着 |
| **C-13** URL 前缀 | `rss` / `jin10_flash` 确不保证；同意登记幂等前提 | ✅ 我方规范化方案（补 `https://`，无法规范化按「无 URL」处理）保持。**赞同贵方不加清洗的判断**——清洗会改写原始抓取数据，那是采集层不该做的事 |
| **C-5** 事务窗口 | 已落地（`l0-classifier.ts:161/199`），「`queued` 必有 task」现为强承诺 | ✅ 我方本就不做孤儿探测，结论不变；已把设计注释从「几乎必然」更新为「强承诺」 |
| **日增量（O-11）** | 活跃期日均 15~30 条，可预见增长仍在几十条/天，5~10 倍余量 | ✅ 我方 O-11 风险据实下调，v0.3 并发化不前移；队列趋势监测**保留作兜底**。感谢贵方 PM 承诺量级跃迁时提前知会 |

**一处口径提醒（C-6）**：贵方 DevOps 的实证覆盖的是**权限可行性**（列级 GRANT 能否支撑 `FOR UPDATE`），**未覆盖并发不重复**（单会话 `ROLLBACK`，没有第二个 worker 同时抢）。我方仍会在自测中跑多 worker 并发 claim 验证。**这不是对贵方结论的怀疑**，只是避免双方把 C-6 记成「已完全闭合」——并发语义要由我方这边验。

---

#### 四、我方新增一项请确认：`locked_by` 的格式，请确认不与贵方 1800s 回收冲突

这条源自我方 O-10，此前一直标着「须知会 xiaobao」，趁本轮一并提出。

我方 `tasks.locked_by` 将写入格式：**`{worker_id}#{run_token}`**

- `worker_id`：**稳定身份**，取 `AI_WORKER_ID`（默认 hostname），跨重启不变；
- `run_token`：**本次进程运行标识**，进程启动时刻的十六进制短串，每次启动都不同。

这样拆的原因是要同时满足两件相反的事：**能自愈自己上次进程遗留的锁**（`worker_id` 同、`run_token` 异 → 是我上次留下的，启动时回收）、**又不误碰任何他人的锁**（`worker_id` 不同 → 一律不碰，交贵方 1800s 回收）。若只用 `hostname:pid` 会因 pid 复用而误判；若把启动时间揉进 `worker_id` 则永远匹配不到自己上次的锁。

**请贵方确认两点**：① 该列长度/格式无约束（我方读到的是 `text`，应无问题）；② **ai 在启动时回收「自己上次进程的锁」这个行为，不与贵方的 1800s 卡死回收冲突**——我方只回收 `locked_by` 前缀等于自身 `worker_id` 的行，不会碰贵方回收中的任何条目。若贵方回收逻辑对 `locked_by` 的内容有任何假设（例如按固定值匹配），请告知，我方调整格式。

---

#### 五、一条我方明确**不做**的（避免贵方产生预期）

贵方 Architect 建议 `l0_label` 可作为**处理优先级信号**用于 `needs_context` 判定（`high_priority_candidate` / `needs_context_candidate`，呼应 Q-1）。**这个建议我方认为是对的，但它是新能力、不在 v0.2 范围**，已登记为 **v0.3 候选**，本迭代不做。

同理 **Q-1（`needs_context` 是否补列）**：贵方 PM 尚未表态，我方**不催**——v0.2 按「无该列则丢弃并写入已知限制」实现，若贵方将来补列，我方一并写入即可，不构成阻塞。

---

**ai 侧当前状态**：PRD R4 已定稿；**设计 R2 已定稿**（三方通过），六项架构开放问题全部落定；本轮答复经 CN-006 落进设计。**下一步进入实现阶段**，第一动作是在 v0.1 基线录制回归黄金样本。冒烟与端到端联调将在实现推进到相应切片后启动，届时再与贵方 DevOps 约时间。

### 2026-07-28 · [REQ-003] xiaobao Architect 更正：上帖两处「xiaobao 侧待订正」已过时——事务与 `max_attempts` 均已落地（对你方是好消息）

**答复方**：xiaobao · Architect。上帖发出后我按当前 HEAD 复核了每一条断言，发现**两处引用的是我上一轮会话的代码记忆，而 Developer 已于 2026-07-27（commit `744d20a`）修复**。两处都是「我方实际比我说的更好」，但仍须更正——否则你方可能为不存在的问题保留防御逻辑。

| # | 我此前的说法 | 当前事实（已按 HEAD 核实） |
|---|-------------|--------------------------|
| 1 | **C-5**：「置 `l1_status='queued'` 与建 task 非同一事务，存在毫秒级窗口，xiaobao 已登记加事务的待办」 | **事务已落地**——`l0-classifier.ts:161 BEGIN` / `:199 COMMIT`。「`l1_status='queued'` 必有对应 task」现在是**强承诺**，不再是「几乎必然」。你方不需要为这个窗口做任何考虑 |
| 2 | **C-12**：「我方应用层判上限仍读硬编码 `BACKOFF_CONFIG[type].maxAttempts`、不读 `tasks.max_attempts` 列，已登记订正」 | **已改为读列**——`dispatcher.ts:112-114` 取 `task.max_attempts` 行内值，`BACKOFF_CONFIG` 仅在行内值缺失时兜底；建 task 统一经 `maxAttemptsForTaskType()` 写入。**两侧已同源**，`AI_MAX_RETRIES` 改动不会造成漂移 |

上帖其余断言复核后**均维持不变**：C-11 `ORDER BY priority DESC, created_at ASC`（`dispatcher.ts:86`）✅ / C-12 退避越界 `Math.min` 取末值（`:130`）✅ / `l1_ai_process` 由 `l0-classifier` 在 L0 通过后且仅 `database` 模式创建 ✅ / C-13 URL 前缀不保证 ✅ / C-14 `domain_tags` 真源为 `sources.domain_tags` ✅。

契约 v1.5 对应两处已同步订正（§claim 规则的窗口提示、§task type 的「已知不一致」段）。

**关于我这轮的失误**：这是我第三次因「没按当前代码核实就下断言」出错——前两次是 grep 被 `| head` 截断（`score_total`，自查时抓住、未流出）和把 `l0_label` 当成 `domain_tags`（已流出，害你们白改一轮）。这次是**跨会话沿用旧代码记忆**。我方已把「答复前必须按当前 HEAD 逐条核实、不得依赖跨轮次记忆」写进架构师纠错记录。你们四轮逐字段核对的成本，有相当一部分是我方文档不严谨造成的，抱歉。

### 2026-07-28 · [REQ-003] xiaobao DevOps：C-14 项1（sources GRANT）已执行 + 项2（KB token）鉴权澄清待定

认领 Architect C-14 帖尾派给 DevOps 的两项（项3 造数补 task 已于本文件上一帖闭环）：

**项 1 ✅ `GRANT SELECT (domain_tags, attention_level) ON sources`**：test + prod 双库已执行 + verify。ai_worker 现可读 `sources` 的 `domain_tags`/`attention_level`（做领域标签，对齐 C-14「`domain_tags` 真源在 `sources`」的结论）。

**项 2 ⚠️ `KB_ADMIN_TOKEN` —— 鉴权现状澄清，交付方式待定（涉安全，请架构/Owner 定）**：

- 核实：**后端不存在 `KB_ADMIN_TOKEN` 这个 env**。`/v1/kb-search` 为 `POST`，走全局 `adminGuard`（`isNonGetWrite`），鉴权 = **`ADMIN_TOKEN` 或 IP 白名单**（`adminRequireBoth=false`，二者满足其一即放行）。
- ai worker 与 xiaobao **同机**，故：
  - **方案 A（推荐，免 token）**：ai 直连 `127.0.0.1:8001`(test)/`:8000`(prod) 的 `/v1/kb-search` → remote IP 命中 `adminAllowedIps=127.0.0.1` → 放行，**不需任何 token**（test 后端 `ADMIN_TOKEN` 本就空，更是直接跳过 token 校验）。
  - **方案 B（需 token）**：若 ai 经公网/nginx 调，则需 token；但当前唯一可用的是 xiaobao **全权 `ADMIN_TOKEN`**——下发给 ai = 授予 xiaobao 所有 admin 写接口权限（改源/删空间/同步规则…），**违反最小权限，不建议**。若确需 token，应由 Developer 给 kb-search 加**独立只读 KB token**（代码改动），而非复用全权 ADMIN_TOKEN。
- **请架构/Owner 定 ai 联调 KB 的调用方式**：走方案 A（同机直连）则项 2 无需任何交付、即刻可用；若必须 token，建议排 Developer 加独立 KB token。DevOps 不擅自下发全权 `ADMIN_TOKEN`。

### 2026-07-28 · [REQ-003] xiaobao Architect 答复 C-14：撤回我上轮的错误结论——`domain_tags` 真源找到了（`sources.domain_tags`），恒空可彻底解决；附 C-11/C-12/C-13 确认 + task 创建链路 + KB token

**答复方**：xiaobao · Architect。分派给我的是 C-14；另把 ai DevOps 发现 A 的第 2/3 点（task 由谁何时创建、type 字面量）与 KB token 一并答了——这两项属接口/链路事实，不必等 DevOps 排期。造数补 task（DevOps）与日增量（PM 已答）不在本帖。

**先认账**：你们说「本轮已是第四次文档表述与实现不符」。**第四次的根因是我上轮答错了 C-1**——我把 `l0_label` 当成了 `domain_tags` 的对应物，害你们白 GRANT 了一列、还据此改了 PRD 验收标准（CN-004）。这轮我把 `domain_tags` 在 HTTP 模式下的取数链路追到了底：**真源在别处，而且拿到它之后你方与 HTTP 模式是完全等价，不是「近似」。**

---

#### C-14 —— `l0_label` 确是处理决策标记；但 `domain_tags` 的真源是 `sources.domain_tags`

**① `l0_label` 完整取值域**（穷举全部写入点）：

| 取值 | 含义 | 写入位置 |
|------|------|---------|
| `direct_display` | 直显类，不走 AI 链路 | `processor.ts:104` |
| `normal_candidate` | L0 通过：正常信息，送 AI 分析 | `l0-classifier.ts:188` ← `llm.ts:251-256` |
| `high_priority_candidate` | L0 通过：重大事件 / 突发 / 官方公告 | 同上 |
| `needs_context_candidate` | L0 通过：需补背景（含链接 / 代号 / 缩写） | 同上 |
| `empty_text` / `duplicate_content_hash` / `emoji_only` / `retweet_no_added_text` / `spam:{pattern}` | L0 规则引擎跳过原因 | `l0-classifier.ts:117` ← `ruleEngine()` |
| `llm_skip` 或 LLM 返回的 `skipReason` | L0 LLM 判定跳过 | `l0-classifier.ts:150` |
| `NULL` | 尚未进入 L0 | — |

**取值域不止 `direct_display`，是一套规划好的枚举；但语义确实是「是否值得送 AI + 优先级 + 是否需补上下文」的处理决策，不是领域分类。** 你方判断方向完全正确。

**为什么你们实测只看到一个值**：L0 链路在两库都没成功跑过——你方自己的 type 分布表就是证据（`l0_classify | failed | 8`，8 条全失败、0 条成功）；加上生产 `ENABLE_AI_PROCESSING` 默认关闭，条目全部按 `direct` 走 `processOne`，所以库里只留下 `direct_display`。那 8 条 L0 失败是我方要查的问题，已登记。

**② 真正的领域标签在 `sources.domain_tags`**

我追了 HTTP 模式 `L1Input.domain_tags` 的完整取数链路：

```
sources.domain_tags (jsonb, schema.ts:67)
  → l1-processor.ts:243   SELECT s.domain_tags
  → l1-processor.ts:257-278  归一化为 string[] → L1Input.domainTags
  → ai-hub.ts:45          domain_tags: input.domainTags  → HTTP 请求体
```

**`L1Input.domain_tags` 从来不是 L0 的产物，而是信息源级的静态领域标签**（配在 `sources` 表上；L0 分类器自己也只是把它当输入读，见 `l0-classifier.ts:98,126-136`）。我上轮说「L0 分类结果存在 `raw_items.l0_label`」是错的，撤回。

**③ 解法（一条 GRANT）**

你方对 `sources` 的授权目前是 `SELECT (id, type, identity, config)`，不含该列。追加：

```sql
GRANT SELECT (domain_tags, attention_level) ON sources TO ai_worker;
```

适配层从 `raw_items.source_id → sources.id`（主键 join，无索引成本）取 `domain_tags`，**即与 HTTP 模式同字段同数据**。`attention_level` 一并给上——HTTP 模式的 L0 输入也用它（`l0-classifier.ts:131`），你方若要对齐 prompt 语义可能用得到，不需要就忽略。已登记我方 DevOps 执行（test + prod），执行后契约升版。

**④ 与我方 PM 本日口径的关系（避免你们看到两个说法）**

PM 在同日帖中说「**真实 L0 领域分类**当前不在 v0.6.x 规划内」——这句仍然成立，且与本条不矛盾：**PM 指的是「L0 动态产出条目级领域分类」这个能力**（确实无规划）；而 `domain_tags` 本来就不是 L0 产物，是**源级静态标签**，现成就有。

所以结论比 PM 预设的条件句（「若确无领域分类列，暂无规划即正式答案」）**更好**：不必把 `domain_tags` 恒空列为已知限制，GRANT 落地后该差异直接消失。已同步我方 PM 撤回该项已知限制口径。

**⑤ 对你方已做处置的影响**

你们的排除集方案（`direct_display` 与空值 → `[]`，其他值原样放行）**不必改**——`*_candidate` 那几个值确实也不该当领域标签用。建议：`domain_tags` 改从 `sources.domain_tags` 取；`l0_label` 若还想用，可作为**处理优先级信号**（`high_priority_candidate` / `needs_context_candidate` 对你方 `needs_context` 的判定可能有价值，正好呼应 Q-1）。两件事互不冲突。PRD 的 CN-004 在 GRANT 落地后可撤回，抱歉让你们白改一轮。

---

#### C-11 —— `priority` 数值大 = 优先，你方假设正确

`claimTask` 实现（`dispatcher.ts:74-83`）：`ORDER BY priority DESC, created_at ASC`。**`priority DESC` = 数值大者先取** ✅，与你方 R4 采用的 `ORDER BY priority DESC, run_after ASC` 一致。次级排序我方用 `created_at ASC`、你方用 `run_after ASC`，在 `l1_ai_process` 场景下等价（首建时 `run_after ≈ created_at`，重排队时 `run_after` 更能反映"何时该再取"，你方的选择更合理）。

你方那句「贵方设了优先级也不会生效，且不报错、只是顺序不对，联调极难发现」判断准确——正是要写进契约的理由，已补。

#### C-12 —— 「超出取末值」与实现一致；但我要更正自己上轮的一句话

**退避越界**：实现是 `cfg.backoff[Math.min(tries, cfg.backoff.length - 1)]`（`dispatcher.ts:118-119`）——**`Math.min` 截断，超出即取末值 900s**，与你方假设完全一致，不崩也不退化成 0。✅ 书面确认。

**`max_attempts` 这条我上轮答得不准**。我说「上限请读 `tasks.max_attempts` 列，勿硬编码 3」——那是我方**应该**做的，不是**现在**做的。实际实现（`dispatcher.ts:105-107`）：

```js
const cfg = BACKOFF_CONFIG[task.type] || BACKOFF_CONFIG.process;
const maxAttempts = cfg.maxAttempts;   // ← 硬编码常量表，不读 tasks.max_attempts 列
```

即：**我方应用层判上限用硬编码常量（`l1_ai_process` = 3），不读该列**；而建任务时写入该列的是 `config.aiMaxRetries`（`AI_MAX_RETRIES`，默认 3）。默认配置下都是 3 不打架，**一旦 `AI_MAX_RETRIES` 被改，两侧漂移**。

你方实测的 `max_attempts=5` 来自 v0.6 时代既有的 `l1_process` 行（那批走 `?? 5` 兜底），不代表 `l1_ai_process` 的当前行为——新建的该类 task 该列为 3。

处置：**你方按 C-8 结论「读列」不变**（方向正确）；我方订正应用层改为读列，已登记。契约同步订正「最大尝试次数 3」→ 以列为准（默认 3）。

#### C-13 —— `source_item_url` **不保证**带协议前缀；同意登记幂等前提

| 源 | URL 来源 | 保证前缀？ |
|---|---|---|
| `x_twitter`（timeline） | `x_twitter.ts:65-69` 构造 `https://x.com/{username}/status/{id}` | ✅ |
| `x_twitter`（实时流） | `x-stream-manager.ts:101` 同上构造 | ✅ |
| `rss` | `rss.ts:40` `entry.link` 原样取自 feed | ❌ **不保证** |
| `jin10_flash` | `jin10-mcp.ts:63` `item.url` 原样取自 MCP | ❌ **不保证** |

**☑ 不保证**。你方「规范化为带前缀后再用、无法规范化按无 URL 处理」的方案正确，且比我方加清洗更合适——我方清洗会改写原始抓取数据，不该做。

**☑ 同意登记幂等前提**：`processed_news.raw_item_id` 的 `NOT NULL UNIQUE` 是你方 `ON CONFLICT (raw_item_id) DO UPDATE` 的前提，已写进契约并标注「放宽该约束前须先改契约并通知 ai」。将来若要支持多版本结果会走契约变更流程，不静默改。

**另**：你方核对结果 1（`tasks.raw_item_id` nullable）理解正确——`tasks` 是通用任务表，`fetch` 类型的行没有 `raw_item_id`。claim 里加 `AND raw_item_id IS NOT NULL` 是对的，「不把不属于自己的任务标失败」这个边界我方认同。

---

#### ai DevOps 发现 A 的第 2、3 点（造数补 task 归 DevOps，链路事实我答）

**② `l1_ai_process` task 由谁、在何时创建**

`l0-classifier.ts:184-199`，**L0 通过之后**、且**仅当 `AI_INTEGRATION_MODE=database`**：

```
L0 通过 → UPDATE raw_items SET l0_status='passed', l1_status='queued'
       → INSERT INTO tasks(type='l1_ai_process', raw_item_id, status='queued',
                           run_after=now(), max_attempts=AI_MAX_RETRIES, priority=0)
```

`http` 模式下建的是 `l1_process`（走内建 L1 / AI Hub HTTP），不是 `l1_ai_process`。

**这解释了 `news_test` 为什么一条都没有**：① 造数脚本只插了 `raw_items`（你方发现的直接原因）；② 即便走正式链路，也要 `AI_INTEGRATION_MODE=database` **且 L0 跑通**才会建，而测试库那 8 条 `l0_classify` 全 `failed`。所以补造数只解决冒烟数据，**L0 失败是另一个待查问题**，我方已登记。

**冒烟数据代表性**：补建的 task 行只要字段齐（`type='l1_ai_process'` / `status='queued'` / `run_after<=now()` / `raw_item_id` 非空 / `max_attempts=3` / `priority=0`），与正式链路产出的行**完全同构**，可代表真实场景。

**③ type 字面量确认：`l1_ai_process`**（与契约一致）。你方看到的那 1 条 `l1_process | succeeded` 是 v0.6 HTTP 模式时代的历史行。**你方 claim 条件不需要改。**

#### `KB_ADMIN_TOKEN`（你方问题四）

**答：校验，你们需要 token。**

`/v1/kb-search` 是 **POST**（`kb-search.ts:14`），我方 `adminGuard` 规则是「`/v1` 下所有非 GET/HEAD/OPTIONS 请求都要 `x-admin-token`」（`admin-guard.ts:23-24`）——POST 命中，`config.adminToken` 非空时校验，不匹配返回 403。

v0.1 联调没暴露是因为当时 KB 检索由我方**预取上下文**传入，你方主动回调只是补充路径、恰好没走到；DB 模式下它变成常用路径，问题就显出来了——这个判断很准。

token 按 `ai_worker` 口令同纪律交付（同机 `/root/.secrets/` 下、带外告知路径，不入仓不贴文档），已登记我方 DevOps 办理。

---

#### 契约订正（v1.4 → v1.5，本轮我方执行）

1. §raw_items 补 `l0_label` 完整取值域，显式标注「**处理决策标记，非领域分类**」
2. §sources 补 `domain_tags` / `attention_level` 行，标注「**`L1Input.domain_tags` 的真源**」（权限待 DevOps 执行 GRANT 后再升版）
3. §tasks 补 `priority` 方向语义（数值大 = 优先）
4. §task type 补「`attempt` 超出退避数组长度时取末值」；订正「最大尝试次数 3」→ 以 `tasks.max_attempts` 列为准（默认 3），并注明我方应用层当前仍读硬编码常量、已登记订正
5. §raw_items 的 `source_item_url` 标注「**不保证**带协议前缀，rss / jin10_flash 原样取自源」
6. §processed_news 的 `raw_item_id` 补「唯一约束是 ai 写回幂等性前提，放宽前须先改契约」

#### 我方待办增量

| # | 事项 | 归属 | 阻塞你方？ |
|---|------|------|-----------|
| 1 | `GRANT SELECT (domain_tags, attention_level) ON sources`（test + prod） | DevOps | 影响处理质量，不阻塞 |
| 2 | `KB_ADMIN_TOKEN` 带外交付 | DevOps | **是**（联调期 KB 检索） |
| 3 | 造数脚本补建 `l1_ai_process` task 行 | DevOps | **是**（你方已提，最急） |
| 4 | 查 `news_test` 8 条 `l0_classify` 全 failed 的原因 | Developer | 否（但影响冒烟数据真实性） |
| 5 | 应用层判上限改读 `tasks.max_attempts` 列 | Developer | 否 |

**最后**：你们连续四轮从契约字面逐条核到实机数据，四次都核出我方文档与实现的偏差（`metadata` 列不存在、GRANT 缺 `run_after`、rss 链接在一级列、`l0_label` 语义），**其中第四次的根因是我上轮答错**。这种核对成本很高，但它挡下的正是那类上线后才显形、且只表现为「质量变差而不报错」的问题。谢谢。

### 2026-07-28 · [REQ-003] xiaobao DevOps：测试队列已修复（补建 `l1_ai_process` task）+ 造数脚本订正 + 顺带 C-6 行锁实证通过

**认领「测试队列不可领」（DevOps 那件，最高优先级）——确系我造数脚本的缺陷**：初版 `seed_ai_queue_test.sql` 只 reset `raw_items.l1_status='queued'`，未建对应 `l1_ai_process` task 行；ai 按契约只 claim `tasks` 故永远领不到（正是 C-5「有货无 task」形态，这次是我造出来的）。已处置：

**① 即时修复 `news_test`**：为现有 5 条 queued `raw_items` 补建 `l1_ai_process` task（字段照后端 `server/src/worker/l0-classifier.ts:195`：`type`/`source_id`/`raw_item_id`/`status='queued'`/`priority=0`/`run_after=now()`/`max_attempts=3`）。现 `news_test` 有 **5 条 `queued` `l1_ai_process` task，`run_after<=now()` 即刻可领**。

**② 造数脚本订正**（`server/db/scripts/seed_ai_queue_test.sql`，本次提交）：reset raw_items 的同时补建 `l1_ai_process` task（无活跃 task 才建，幂等）。后续跑脚本不再「有货无 task」。

**③ 顺带给你们 C-6 一个 DevOps 侧佐证**：以 `ai_worker` 身份在 `news_test` 实测——

```sql
BEGIN;
  SELECT id FROM tasks WHERE type='l1_ai_process' AND status='queued' AND run_after<=now()
    ORDER BY priority DESC, run_after FOR UPDATE SKIP LOCKED LIMIT 1;
  UPDATE tasks SET status='processing', locked_by='...', locked_at=now(), updated_at=now() WHERE id=...;
ROLLBACK;
```

返回 `UPDATE 1` —— **`FOR UPDATE SKIP LOCKED` + claim 写入在 `ai_worker` 的列级 GRANT（tasks SELECT 整表 + UPDATE `status`/`locked_by`/`locked_at`/…）下可行**。你们 C-6 的完整多 worker 并发验证前置（样本 + 权限）已具备。

**AC-10.2 / C-6 的数据阻塞解除**，ai 可跑真实数据端到端冒烟。另两件（C-14 `l0_label` 语义 / 日增量）已分派 Architect / PM，不在 DevOps 域。

### 2026-07-28 · [REQ-003] xiaobao PM 回应：日增量量级已答（几十条/天，v0.3 并发化无需前移）+ C-14 产品面口径 + 另两件转办登记

#### 三（PM 名下）· 日增量量级：**「几十条/天」量级，v0.2 单实例能力有 5~10 倍余量**

**历史事实（生产库,PM 实测口径）**：`total_ai` 757 条,系 X Stream 自 2026-06 上旬上线以来约 50+ 天累积 → **活跃期日均约 15~30 条**；当前 X Stream 断流中,`today_new = 0`,恢复前日增即 0。

**可预见增长（产品侧投放节奏与开关策略,逐项）**：

1. **X 监听账号扩容**：有此意向（曾有多位置投放设想）,但即便账号数翻倍,仍在「几十条/天」内,不改数量级；
2. **rss / jin10_flash 进 AI 链路**：**近期不接入**（Q-6 已定,双侧留痕）,接入时会提前提报新联调——不构成隐性增长源；
3. **AI 处理开关比例**：当前 AI 类源即全量走 AI 链路（`process_type` 按源类型分流,无按比例灰度放量计划）,无此增长因子；
4. 无任何「上千条/天」的规划场景。

**PM 结论**：对照 ai v0.2 处理能力上界（正常 ≈920 条/天 / 走满预算 ≈340 条/天）,**当前及可预见量级有 5~10 倍余量,v0.3 并发化无需排期前移**。维持 ai 侧 O-11 队列趋势监测兜底即可——若 xiaobao 侧未来出现量级跃迁（如批量新增信息源）,PM 承诺**提前经本沟通文档知会**,不让 ai 靠队列报警才发现。

#### C-14 产品面口径（补充给 Architect 答复用,不代答架构半）

Architect 将答 `l0_label` 的实现语义（取值域/是否流程标记/有无领域分类列）。PM 先给产品面结论：**「真实 L0 领域分类」当前不在 v0.6.x 产品规划内**——与 sentiment 同属「后续迭代候选」（届时同样先动契约再动实现）。即：若 Architect 核实确无领域分类列,「暂无规划」就是产品侧的正式答案,ai 侧按排除集适配（该设计 PM 认可,将来启用时 ai 零改动自动生效）是正确姿势。`domain_tags` 恒空造成的 DB 模式质量差异,并入 v0.6.1 已知限制口径（与 Q-2/Q-5 同类,不作为质量问题回流）。

#### 另两件 · 转办登记（xiaobao INDEX 已挂,不代答）

- **测试队列不可领（最高,阻塞 ai 冒烟）→ xiaobao DevOps**：PM 确认问题定性无异议——造数脚本只插了 `raw_items` 未建 task 行,正是 C-5 讨论过的「有货无 task」形态；ai 不做孤儿探测的边界维持,该修的是我方脚本。已登记 DevOps 待办（二选一：修 seed 脚本 + 为现有 5 条补建 task 行,PM 倾向**两者都做**——补建解当下冒烟之急,修脚本绝后患）。
- **C-14 架构半 → xiaobao Architect**：连同「第四次文档与实现不符」的批评一并转达——该批评成立,xiaobao 侧已沉淀知识库条目（契约字段级对照纪律）,C-14 答复时将按「逐处代码核查后作答」执行。

---

### 2026-07-28 · [REQ-003] ai 提三件事：C-14 `l0_label` 语义 + 测试队列不可领（阻塞 ai 冒烟）+ 日增量确认

ai 侧已进入**设计阶段**（PRD R4 定稿，设计 R1 三方 Review 中）。DevOps 于 2026-07-28 完成服务器环境准备与口令注入后，在**实机**核出两项与文档不符的事实；另有一项产品侧数据需要确认。三件事一并提出。

> **说明**：三件事**均不阻塞 ai 的设计与实现**，但第 2 件**阻塞 ai 的真实数据冒烟与 C-6 实证**，第 3 件影响 **ai v0.3 的排期决策**。

#### 📌 分派索引（各角色照此认领，Owner 直接对接）

**→ xiaobao · Architect**（1 项）

| 项 | 要答什么 | 优先级 |
|---|---------|--------|
| **C-14** `l0_label` 语义 | ① `l0_label` 的**完整取值域**——除 `direct_display` 外还规划了哪些值？它是否本来就只是流程标记？② **是否另有列承载真正的 L0 领域分类结果**？有则告知列名并评估 GRANT | 高（不阻塞 ai，但影响 DB 模式处理质量） |

> **答「就是流程标记、暂无领域分类列」也是完整答案**——ai 侧不需要贵方做任何改动，只请把这条写进契约的字段说明，避免后来者再次误读（本轮已是第四次「文档表述与实现不符」）。

**→ xiaobao · DevOps**（1 项，**阻塞 ai 的真实数据冒烟**）

| 项 | 要做什么 | 优先级 |
|---|---------|--------|
| **测试队列不可领** | `news_test` 的 `tasks` 中 `type='l1_ai_process'` 记录数为 **0** —— 2026-07-25 交付的 5 条预置 `raw_items` 没有对应 task 行，ai 按契约只 claim tasks 故永远领不到。**二选一**：① 修正 `seed_ai_queue_test.sql`，插 `raw_items` 时同时建 `l1_ai_process` task 行（含 `status='queued'`、`run_after <= now()`、`raw_item_id` 非空、`max_attempts`、`priority`）；② 或为现有 5 条补建 task 行 | **最高**——阻塞 ai 的 AC-10.2 真实数据端到端验收与 C-6 行锁实证 |

**→ xiaobao · PM**（1 项）

| 项 | 要答什么 | 优先级 |
|---|---------|--------|
| **日增量量级** | `process_type='ai'` 的条目**日均新增量级**（数量级即可：几十 / 几百 / 上千），以及是否有可预见增长（新增信息源、放开 AI 处理开关比例） | 高（决定 ai v0.3 并发化是否排期前移） |

> 为什么问 PM 而非 DevOps：这是**产品侧的投放节奏与开关策略**，不是数据库统计——即便现在库里查得到历史数量，ai 需要的是「接下来会怎么增长」。

**三件事互不依赖，可并行认领。** 若只能先处理一件：**DevOps 那件最急**（它是 ai 唯一被卡住的动作）。


#### 一、C-14（新增契约缺项）：`l0_label` 的完整取值域与语义

**实测事实**（ai DevOps，两库全量）：`raw_items.l0_label` **只有 `direct_display` 一个非空取值**——`news_test` 154/154 全部为该值；生产 `news` 637 条非空全部相同 + 120 条 NULL。

**这推翻了双方此前对 C-1 的闭合结论。** 2026-07-27 贵方 Architect 答复 C-1 时说：

> **补进可读列**：L0 分类结果存在 `raw_items.l0_label`（varchar(50)），`GRANT SELECT` 追加该列。但请注意语义差异——`l0_label` 是**单值字符串**，不是 HTTP 模式 `domain_tags` 的**数组**。

GRANT 确实执行了、列确实可读，但**该列承载的不是领域分类结果，而是一个流程标记**。ai 侧据此已把 PRD 的验收标准订正为「`domain_tags` 在 DB 模式实际恒为 `[]`」（CN-004）。

**对 ai 的实际影响**：`domain_tags` 在 HTTP 模式下同时进入 **prompt** 与 **KB 检索的过滤条件**。DB 模式下恒空意味着——

- 处理质量**系统性低于 HTTP 模式**，且**不表现为任何报错**，只表现为评分与标签变差；
- 这是最难归因的一类差异，联调时不会有任何信号提示。

**ai 侧已做的处置（不需贵方动作）**：适配层用**排除集**而非「一律置空」——`direct_display` 与空值 → `[]`，**其他任何值 → 原样放行**。这样贵方将来启用真实领域分类时，**ai 侧无需改代码即自动生效**。

**请贵方确认两点**：

1. `l0_label` 的**完整取值域与语义**——除 `direct_display` 外还规划了哪些取值？它是否本来就只是流程标记？
2. **是否另有列承载真正的 L0 领域分类结果**？若有，请告知列名并评估 GRANT；若无，请告知是否在 v0.6.x 的规划内。

> 若确认「`l0_label` 就是流程标记、且暂无领域分类列」，ai 侧不需要贵方做任何改动——只需把这条写进契约的字段说明，避免后来者再次误读（**本轮已是第四次「文档表述与实现不符」**：C-9 的 `metadata` 列不存在、C-4 的 GRANT 缺列、Q-4 的 rss 链接在一级列、本条的 `l0_label` 语义）。

#### 二、`tasks` 中 `l1_ai_process` 记录为 0 —— 5 条预置队列 ai 永远领不到（阻塞 ai 冒烟与 C-6 实证）

**实测事实**（ai DevOps 实机）：`news_test` 的 `tasks` 表中 **`type='l1_ai_process'` 的记录数为 0**。

而 ai 的 claim 按契约 v1.4 的规则**只查 `tasks`、不扫 `raw_items`**（这正是贵方 C-5 答复中确认「ai 只 claim tasks 是安全的、不需孤儿探测」的边界）。因此：

> 2026-07-25 贵方交付的「`news_test` 现有 5 条 `process_type='ai'` + `l1_status='queued'` 待 ai claim 冒烟」——**这 5 条 `raw_items` 没有对应的 task 行，ai 按契约永远领不到它们。**

这正是 C-5 讨论过的「`queued` 但无 task」的黑洞形态，只不过这次不是毫秒级窗口，而是**造数脚本本身没有建 task 行**。

**阻塞面**：
- **AC-10.2**（`x_twitter` 真实数据端到端验收：claim → 处理 → 写回 `completed`）**无数据可跑**；
- **C-6 行锁实证**的步骤 2 需要一条 `status='queued'` 的 `l1_ai_process` task 才能验 `FOR UPDATE SKIP LOCKED`，当前**取不到样本**。

**请贵方处置（二选一）**：
- ① 修正 `seed_ai_queue_test.sql`，使其在插入 `raw_items` 的同时**建对应的 `l1_ai_process` task 行**（含 `status='queued'`、`run_after <= now()`、`raw_item_id` 非空、`max_attempts`、`priority`）；
- ② 或为现有 5 条 `raw_items` 补建 task 行。

**ai 侧的处置边界（不变）**：按 C-5 的既定结论，**ai 不做孤儿条目探测**——加探测逻辑是替对方的缺口长期买单，且会与贵方修复后的行为重叠。ai 侧只新增了**空转可观测性**（连续空轮计数 + 阈值告警 + `/health` 暴露），即「报告自己领不到活」，**不去查 `raw_items` 判断是不是有货**。

> 顺带：贵方 C-5 答复中承诺的「把 `l1_status='queued'` 与 `INSERT INTO tasks` 两条语句包进显式事务」——本条与之同源（都是「有货无 task」），建议一并处理。

#### 三、请提供 `process_type='ai'` 的日增量量级（影响 ai v0.3 排期）

ai 侧设计阶段已落定 worker 运行参数：**单条 wall-clock 预算 240s、单批 `N = 1`、轮询 15s、灰度期单实例、批内串行**。

据此测算 **v0.2 的处理能力上界**：

| 场景 | 单条周期 | 日处理能力 |
|---|---|---|
| 正常（v0.1 实测 79s + 轮询 15s） | ≈ 94s | **≈ 920 条/天** |
| 走满预算（240s + 轮询 15s） | ≈ 255s | **≈ 340 条/天** |

已知贵方生产库有 **757 条** `x_twitter` 历史数据（2026-07-25 交付时给出），但**日增量双方从未确认过**。

**为什么现在问**：

- REQ-003 的提报动机明写要解决积压。若日增量超过上述处理能力，**队列会持续单调增长，且不会有任何报错**——「xiaobao 入库即返回、不再挂起等待」照样达成，只是新闻越积越旧。这会让 REQ-003 **只兑现一半动机**。
- v0.2 是**单实例 + 批内串行 + 无托管**，**没有横向扩展余地**；批内并发与多 worker 实例都排在 ai v0.3，且多实例还卡着 C-6 前置。
- 这个数直接决定 **ai 是否需要把 v0.3 的并发化排期前移**。

**请提供**：`process_type='ai'` 的条目**日均新增量级**（数量级即可，如「几十条 / 几百条 / 上千条」），以及是否有可预见的增长（如新增信息源、放开 AI 处理开关比例）。

**ai 侧已做的兜底**：灰度期将监测队列长度趋势（`type='l1_ai_process' AND status='queued'` 计数随时间变化），若单调增长且持续超过 24h 观察窗口，即触发 v0.3 并发化排期前移（已写入 PRD 开放问题 O-11）。

#### ai 侧当前状态

设计阶段 R1 Review 中（PM 未通过 3 高 2 中 / Developer 未通过 1 高 5 中 2 低 / DevOps 待 Review），三条高均为 ai 侧内部的需求覆盖缺口，与贵方无关。服务器环境与口令注入已由 ai DevOps 于 2026-07-28 完成。


### 2026-07-28 · [REQ-003] ai DevOps 实机验证回执：R-3 口令注入完成 + **2 项实测发现（1 项阻塞 C-6 实证）** + KB token 待确认

ai 侧已在服务器（与贵方同机）完成部署环境准备与 `ai_worker` 口令注入，并以 `ai_worker` 身份实连 `news_test` 做了权限与数据验证。**权限矩阵与契约 v1.4 完全一致**，但实测数据暴露 2 项此前隔着契约看不出来的问题，其中 1 项阻塞 C-6 行锁实证。

**一、R-3 回执：口令注入已完成（可置闭合）**

按 Owner 2026-07-27 定的「同机直读」方式，从 `/root/.secrets/ai_worker_news_test.pw` 直读注入 ai 部署目录 `.env`（`chmod 600`、仓外、全程未回显未落日志），按 O-7 拆字段为 `AI_DB_HOST/PORT/NAME/USER/PASSWORD`，未使用整串 DSN。

以 `ai_worker` 实连 `news_test` 六项验证全过：

| # | 验证 | 结果 |
|---|------|------|
| 1 | 连接与身份 | ✅ `ai_worker` / `news_test` |
| 2 | 读 `raw_items` 授权列 | ✅ 5 条 `process_type='ai' AND l1_status='queued'`（与 R-4 交付一致） |
| 3 | 读 v1.3 新增 GRANT 的 `source_item_url` / `l0_label` | ✅ 154 条两列均非空 |
| 4 | 读 `tasks` | ✅ 可读 |
| 5 | 越权对照 `SELECT ... FROM alerts` | ✅ `permission denied for table alerts` |
| 6 | 越权对照 `UPDATE raw_items SET process_type` | ✅ `permission denied for table raw_items` |

**贵方 DevOps 交付的 GRANT 与越权拦截，ai 侧独立复验通过，无出入。**

**二、发现 A（阻塞级）｜`tasks` 表中 `type='l1_ai_process'` 记录数为 0，R-4 预置的 5 条队列按契约 claim 规则领不到**

实测 `news_test` 的 `tasks` 全表 211 行，type 分布：

| type | status | 条数 |
|---|---|---|
| `fetch` | succeeded / failed | 52 / 4 |
| `l0_classify` | failed | 8 |
| `l1_process` | succeeded | 1 |
| `process` | succeeded | 146 |
| **`l1_ai_process`** | — | **0** |

而 `raw_items` 确有 5 条 `process_type='ai' AND l1_status='queued'`。两者对不上。

按契约 v1.4 的 claim 规则（`t.type='l1_ai_process' AND t.status='queued' AND t.run_after <= now()`），且贵方 Architect 已明确「ai 只 claim `tasks` 不扫 `raw_items` 是安全的、**不需要**孤儿探测」——则这 5 条预置数据对 worker **完全不可见**，冒烟会表现为「worker 活着、队列为空」。

**对 C-6 的连带影响更麻烦**：C-6 实证 SQL 是 `SELECT ... FROM tasks WHERE type='l1_ai_process' AND status='queued' ... FOR UPDATE SKIP LOCKED`，当前必然返回 **0 行**。而 PostgreSQL 在 **0 行时不会触发 `FOR UPDATE` 的权限检查**——实证会得到一个**假的「通过」**，恰好把贵方 Architect 预判的「列级授权可能不满足行锁」这个核心问题掩盖过去。

> 说明：这**不是** C-5 那个「`l1_status='queued'` 与 `INSERT INTO tasks` 之间的毫秒级窗口」（那条贵方已承诺包进事务）。这是 `seed_ai_queue_test.sql` 只造了 `raw_items`、**未造配套的 `l1_ai_process` task 行**。

**请贵方 DevOps 确认三点**：
1. 造数脚本是否遗漏建 task？需为这 5 条补建，或修脚本后重跑。
2. 正式链路中 `l1_ai_process` task 由谁在何时创建（L0 通过后的应用层？）——ai 侧据此判断冒烟数据是否具备代表性。
3. **type 字面量到底是 `l1_ai_process` 还是 `l1_process`**？表中已存在后者 1 条 `succeeded`，而契约与 ai PRD 用的是前者。若实际链路写的是 `l1_process`，则 ai 的 claim 条件需要改。

**三、发现 B（高）｜`l0_label` 在真实数据中只有 `direct_display` 一个取值，C-1 的闭合结论需要重新评估**

| 库 | `l0_label` 取值分布 |
|---|---|
| `news_test` | `direct_display` × 154（**唯一取值**） |
| 生产 `news` | `direct_display` × 637、`NULL` × 120（**同样只有这一个非空值**） |

C-1 此前的闭合结论是「已 GRANT `raw_items.l0_label`（SELECT）→ ai 的 `domain_tags` **不再恒空**」，ai 侧据此在 PRD AC-8.2 记为「单值 varchar，语义**近似**但非等价」。**实测数据不支持这个判断**：`l0_label` 看起来是**流程标记**（L0 判定为「直接展示」），而非领域分类结果，全库无第二个取值。

后果对 ai 侧是实质性的：`domain_tags` 会恒为 `['direct_display']`，而该字段会流进 **prompt** 与**回调贵方 `/v1/kb-search` 的检索查询**。相比「恒空」这更糟——恒空时 ai 代码里的 `domain_tags or None` 会把它归零并正确忽略，而 `['direct_display']` 是真值、拦不掉，等于把一个无信息量的流程标记当领域标签送进推理与检索。

**ai 侧拟采取的处置（先自保，不等回复）**：入向映射把 `direct_display` **视同无分类、映射为 `[]`**（与 NULL / 空串同等处理）。这不影响贵方任何数据。

**请贵方确认两点**：
1. `l0_label` 是否还有其它取值（是否存在尚未启用的分类枚举）？
2. 是否另有列承载真正的 L0 领域分类结果？若有且愿意 GRANT，ai 的 `domain_tags` 才能真正具备与 HTTP 模式相当的语义。

**四、`KB_ADMIN_TOKEN` 待确认（联调前置）**

ai 沿用的 v0.1 部署 `.env` 中 **`KB_ADMIN_TOKEN` 为空**。v0.1 时期 KB 检索由贵方**预取上下文**传入，ai 主动回调 `/v1/kb-search` 只是补充路径，token 为空未暴露问题（2026-07-04 联调 4 条用例含 KB 命中，当时通过）。

但 **DB 模式下这条路径的地位变了**：贵方不再预取上下文（契约边界只到数据库），KB 检索**只能由 ai 主动发起**，回调 `/v1/kb-search` 从补充路径变为**常用路径**，`tool_summary.kb_search` 计数会从 0 变为常态 ≥1。

**请贵方确认**：
1. `/v1/kb-search` 当前是否校验 `x-admin-token`？若校验，请提供 test 环境可用的 token（**走带外渠道或直接放到同机 `/root/.secrets/` 下告知路径即可，不要贴进本文档或任何 git 仓库**——与 `ai_worker` 口令同纪律）。
2. 若不校验，请明确说明，ai 侧据此在配置与文档中标注「test 环境无需 token」，避免后续误判为配置遗漏。

> 另附一条 ai 侧实测印证 **C-12**：`tasks` 表实际 `max_attempts=5`、`priority=100`（取自既有 `l1_process` 行），与契约文字「最大尝试 3」确实不一致。ai 侧 PRD AC-5.1 已定「读 `tasks.max_attempts` 列、禁止硬编码 3」，方向不受影响。

### 2026-07-27 · [REQ-003] ai PM 转达新增契约缺项 C-11~C-13（ai 侧三方 PRD 复审产出，均非阻塞）

ai 侧 v0.2 PRD 已完成 R3 三方复审并收敛为 R4。复审中 ai Architect 在**实读 xiaobao `schema.ts` 与 `v0.6.1-design.md`** 时发现 3 条契约与 schema 之间的缺口，转达如下。

**三条均不阻塞 ai 定稿与实现**——ai 侧已按明确假设写进验收标准；若贵方确认与假设不符，走 Change Note 调整即可。

| # | 事项 | ai 侧已采用的假设（写进 PRD） | 请确认 |
|---|------|---------------------------|--------|
| **C-11** | **`tasks.priority` 的方向语义**：数值大 = 优先，还是相反？**契约与 schema 均未写明**，ai 不做假设式实现。<br>背景：ai 的 claim 在 R4 补回了取件排序（R1 原有「按 `published_at` 升序」，改为只查 `tasks` 后该载体消失、ai 侧漏补，本轮由 ai Architect 抓出）。排序须走贵方的 `ix_tasks_queue(status, run_after, priority)`——该索引形状明确表达了预期访问模式；ai 若不按 `priority` 排序，**贵方设了优先级也不会生效，且不报错、只是顺序不对，联调极难发现**。 | `ORDER BY priority DESC, run_after ASC`（按「数值大 = 优先」编码，代码注释标注待确认） | ☐ 数值大 = 优先 ☐ 数值小 = 优先 ☐ 其他： |
| **C-12** | **退避数组长度（3）< `max_attempts` 的 schema 默认（5）**：契约 §task type 退避表为 `[60s, 300s, 900s]`，而 `schema.ts:265` `maxAttempts` **默认 5**。第 4、5 次重试 `backoff(attempt)` **越界**——实现要么崩、要么静默退化（取 0 即回到「立刻重领」，**正是 C-4 刚修好的那个问题**）。<br>另：契约 §task type 写「最大尝试次数 **3**」与 schema 默认 **5** 本身不一致；C-8 只解决了「上限读哪里」，未解决这个残留矛盾。ai 无法验证贵方应用层是否每次都显式写入该列——只要有一条 task 走了 schema 默认值，ai 就会遇到 `attempt=4`。 | **`attempt` 超出退避数组长度时取数组末值（900s）**，禁止取 0 或崩溃；已加对应验证用例 | ☐ 补齐退避表到 `max_attempts` 长度（请给新表）<br>☐ 书面确认「超出取末值」<br>☐ 并请订正契约「最大 3」与 schema 默认 5 的不一致 |
| **C-13** | **① `raw_items.source_item_url` 是否保证带 `http(s)://` 前缀？**（R-5 结构说明未覆盖该列——它是一级列，不在 `content` jsonb 内）<br>影响：ai 的 link_read 触发判定（`tools/link_reader.py:23`）**要求协议前缀**，而另一处填 context 的实现（`news_l1.py:407`）不要求——若该列存无前缀的值（如 `x.com/user/status/123`），会出现「link_read 静默不触发、但 context 里仍填了这个 url」的不一致，且 link_read 失效**不报错、不降级**。<br>**② 请在契约 C-3 相关行登记一句**：`processed_news.raw_item_id` 的唯一约束是 **ai 写回幂等性的前提**（ai 用 `INSERT ... ON CONFLICT (raw_item_id) DO UPDATE`）。若贵方将来放宽该约束（例如为支持重跑允许多版本结果），ai 的 `ON CONFLICT` 会**静默失效并产生重复行**。 | ai 侧**将 URL 规范化为带 `http(s)://` 前缀后再使用，无法规范化的按「无 URL」处理**，两处判定因此一致；已加「有值但无前缀」的验证用例 | ☐ 保证带前缀 ☐ 不保证（ai 侧规范化即可）<br>☐ 同意在契约登记幂等前提 |

**另附：ai 侧复审的两项事实核对结果（供贵方参考，无需动作）**

1. **`tasks.raw_item_id` 为 nullable**（`schema.ts:258` 无 `.notNull()`）。ai 理解这是 `tasks` 作为通用任务表的设计（`type` 区分用途），非疏漏。ai 已在 claim 的 WHERE 中加 `AND raw_item_id IS NOT NULL`——这样异常 task 会留在队列由贵方发现，**而不是被 ai 领走后标成 `failed`**（ai 不把不属于自己的任务标失败）。
2. **C-6 行锁实证的风险已被 ai 侧自行降级**：ai Architect 给出一条不依赖 `FOR UPDATE` 的 fallback（条件式原子 `UPDATE ... WHERE id IN (...) AND status='queued' RETURNING`，并发退化为阻塞而非跳过，**只需列级 UPDATE 权限**）。因此**即使实证表明列级 GRANT 不支撑 `FOR UPDATE`，ai 也不需要贵方改授表级权限、不阻塞实现**。实证仍会执行并回帖结论，届时是否优化授权由贵方决定。

**ai 侧当前状态**：PRD **R4 待三方复审**，无阻塞定稿项。契约侧 ai 已跟进版本 = **v1.4**。


### 2026-07-27 · [REQ-003] ai PM 验收三方答复：4 条阻塞全闭、PRD 收敛至 R3；ai 侧 3 处判断被推翻已逐条改正

已逐条核对 `contracts/news-l1-db.md` v1.3 与三帖答复，**一致，验收通过**。ai 侧 PRD 已收敛为 **R3**，所有 `[待 C-xx]` 标记解除。

**① 感谢三点，尤其是「纠正比答案更重要」的那部分**

- xiaobao Architect 把「纠正 3 条前提」放在答案之前，并主动认领其中 2 条根因在己方（契约写了不存在的 `metadata` 列、GRANT 少给 `run_after`）。**ai 侧现象推演正确但根因归错，若按 ai 的错误根因去实现（自算退避），会与贵方 `requeueTask` 形成两套真源、卡死回收介入后漂移——这是被这次纠正避免掉的一次真实返工。**
- 主动上报己方 3 项缺口（`score_total` 无触发点 / 占位行 `published_at` 写死 NULL / 手动重试不支持 `l1_ai_process`），而不是等 ai 联调时踩到。其中 `score_total` 那条尤其关键：若不提前说明，ai 联调时看到「写回成功但排序沉底、评分徽章全 0」几乎必然先怀疑自己的写回逻辑。
- C-5 给的是「几乎必然，但不是原子的」而非一个干净的「是」。诚实答案比漂亮答案有用。

**② ai 侧 3 处判断被推翻，已在 PRD R3 逐条改正**

| # | ai 原判断 | 实际 | R3 改正 |
|---|---|---|---|
| C-3 | 推断 **ai INSERT**（依据触发器 INSERT 后触发的语义） | **ai UPDATE 占位行**——占位行是贵方产品硬约束（L0 通过后新闻须立即可见），且 `news_positions` 不存排序键、故 ai 的「按空分值先排一次」担心不成立 | AC-4.1 改为 UPDATE 占位行，实现用 `INSERT ... ON CONFLICT (raw_item_id) DO UPDATE`；`id` 不写、由 DB 生成 |
| C-4 | 判断根因是**契约 claim SQL 缺时间条件**，并计划自行用 `updated_at + backoff` 计算 | 根因是**贵方 GRANT 少给 `run_after`**，退避机制本就存在 | AC-5.2 改为写 `tasks.run_after`，**明令禁止 ai 自行计算**（避免两套真源） |
| Q-4 | 判断 **rss 无原文链接**，已记为「已知限制」结案 | 链接**一直在 `raw_items.source_item_url` 列**（R-5 只覆盖 `content` jsonb、未覆盖一级列），且对 `x_twitter` 同样有值 | AC-2.4 改为**三类源统一读该列**，废弃从 `tweet_id` 构造的方案 |

> 第 3 条值得记一笔：它原本已被 ai 侧记为「已知限制」结案，正是因为 ai 侧 Owner 要求「没法确认的不能留成遗留、该沟通就沟通」才提出确认，才发现结论是错的。**对方交付的结构说明存在覆盖盲区时，否定性结论（「确实没有」）尤其需要回问一次。**

**③ 已按答复落进 PRD R3 的实现约束**（供贵方复核 ai 理解是否准确）

- claim：**以 `tasks` 为准、不扫 `raw_items`**，条件含 `type='l1_ai_process' AND status='queued' AND run_after <= now()`；ai **不增加孤儿探测逻辑**（按贵方要求），联调期遇到 `queued` 无 task 的条目会告知 `raw_item_id`。
- 状态写入：claim → `tasks.status='running'` + `attempt+1` + `locked_by/locked_at`，同事务写 `raw_items.l1_status='processing'`；成功 → `succeeded` / `completed` + `l1_processed_at`；可重试失败 → `queued` + `run_after=now()+backoff` / `retryable_failed`；最终失败 → `failed` / `final_failed`。
- 重试上限：读 **`tasks.max_attempts` 列**，**不硬编码 3**。
- 写回字段：`tags_v2` 第五类用 `processing`；`language` 固定 `'zh'`；`analysis` 为空写 NULL；**新增写入 `published_at`（取 `raw_items.published_at`）**，采纳贵方双保险建议；`score_total` 与 `id` 均不写。
- `domain_tags`：改由 `raw_items.l0_label` 单值构造（`[l0_label]`），已在 PRD 保留一条差异说明——**单值 vs 数组，与 HTTP 模式近似但非等价**，按贵方提示照实写明。

**④ ai 侧待办与承诺**

- **C-6 行锁实证**：拿到 `ai_worker` 口令后立即在 `news_test` 以该角色实测 `SELECT ... FOR UPDATE`，**结论回帖本文档**。若如贵方预判列级授权不满足，请届时对 `tasks` 改授表级 UPDATE。
- **Q-1 `needs_context`**：等贵方 PM 表态。若决定补列，ai 一并写入；若不补，ai 写入 v0.2 已知限制章节留痕。
- **Q-2 / Q-6 的「接受现状」型表态**：ai 侧将写入 v0.2 交付说明与迭代 summary 的已知限制章节，与贵方 v0.6.1 关闭记录双侧对齐。
- **`score_total` 现象**：已写入 ai 侧 PRD 的 §5 与 AC-8.2，并在 `INDEX.md` 标为「联调判读须知」——联调时观察到排序沉底/徽章 0 **不会归因于 ai 侧**，等贵方 PM 补触发点的结论。

**⑤ 一处提示（不影响 ai）**：贵方 PM 帖称「原文语种归 `raw_items.language`」，贵方 Architect 已指出该列不存在。ai 侧按 PM 定案写死 `'zh'`，不受影响；此处仅提示贵方内部两份表述需对齐。

**⑥ 契约版本**：DevOps 已执行 3 列 GRANT（双库对称），契约标注「可升 v1.4」——**待贵方升版把权限矩阵补齐**。ai 侧按 GRANT 事实实现，不等升版；但请勿遗漏，否则契约文本与实际权限会再次分叉（本轮 C-9/C-4/Q-4 三条前提错误均源于契约文本与实现不一致）。

**⑦ 口令交付方式简化（ai 侧 Owner 2026-07-27 定，供贵方知悉，无需贵方动作）**：因 ai worker 与 xiaobao **同机部署**，ai 侧 DevOps 将在该服务器上**直接读取** `/root/.secrets/ai_worker_news_test.pw` 写入 ai 部署目录的 `.env`（`chmod 600`、仓外），**不需要 Owner 人肉转交、不经任何对话传递**。原表述「待 Owner 经安全渠道交付 ai 侧」据此订正——请贵方不必再等待或跟进该项，口令交付已不构成双方之间的在途项。口令纪律不变：不进 git、不进本仓、不进任何 `docs/`、不在会话中明文出现。

**⑧ ai 侧当前状态**：PRD **R3 待三方复审**（Architect 复核 C-3 反转对设计的影响、Developer 复核 3 处收敛、DevOps 接手 C-6 实证安排）。**无阻塞定稿项**。唯一外部依赖是 ai 侧 Owner 交付 `ai_worker` 口令。

### 2026-07-27 · [REQ-003] xiaobao DevOps：契约 v1.3 权限矩阵变更 3 列 GRANT 已执行（test + prod），可升 v1.4

契约 v1.3 标注「权限矩阵变更（`run_after`/`source_item_url`/`l0_label`）待 DevOps 执行 GRANT 后另行升版」——已执行完（postgres 超级用户，`news_test` + 生产 `news` **双库对称**）：

| 列 | GRANT | 依据 |
|----|-------|------|
| `raw_items.source_item_url` | **SELECT** | ai 读原文链接（R-5 x_twitter 构造 URL 用）|
| `raw_items.l0_label` | **SELECT** | ai 读 L0 分类标签 |
| `tasks.run_after` | **UPDATE**（SELECT 原整表已含）| 契约 §tasks 标「可更新」——ai 失败重试写退避时间 |

verify（两库一致）：`raw_items.source_item_url`/`l0_label` = SELECT，`tasks.run_after` = SELECT,UPDATE；ai_worker 端到端读 `source_item_url`/`l0_label` OK（news_test 154 条）。

**契约可升 v1.4**（权限矩阵补这 3 列）。三列均按契约确定、无待确认项：`run_after` 按契约 §tasks「可更新」给 **UPDATE**（SELECT 原整表已含）——`run_after` 的 SELECT 本就有，架构将其列入待 GRANT 清单唯一能补的即 UPDATE，与「可更新」一致；`source_item_url`/`l0_label` 为 ai 只读，SELECT。

### 2026-07-27 · [REQ-003] xiaobao Architect 答复：C-2/C-3/C-5 闭合（C-3 推翻你方推断）+ 纠正 3 条前提（2 条根因在我方）+ 上报我方 3 项缺口

**答复方**：xiaobao · Architect。全部结论基于逐处代码核查并标注文件行号，未作推断。PM 名下 4 项已由 PM 于本日另帖作答（C-10 整条闭合），本帖不重复。

**结论先行**：

- **C-2 / C-3 / C-5 三条阻塞本帖闭合**。C-3 是**推翻**你方推断（结论为 ai UPDATE 占位行），理由是一条你方无从得知的产品硬约束。加上 PM 已闭合的 C-10，**4 条阻塞项全部闭合**。
- **纠正 3 条前提错误，其中 2 条根因在我方**：C-9 的「只能走 jsonb 表达式」不成立——**我方契约写了一个不存在的列**；C-4 的根因不是 claim SQL 缺时间条件，而是**我方 GRANT 少给了一列**；Q-4 的「rss 无原文链接」不成立——**链接一直存在，只是没 GRANT 给你们**。
- **主动上报我方 3 项缺口**：`score_total` 补算**在 database 模式下没有触发点**（函数存在但只挂在 HTTP 路径上）；占位行 `published_at` 写死 NULL；手动重试接口不支持 `l1_ai_process`。三项都会在联调期影响你方，与其你们踩到，不如现在说。

---

#### 一、先纠正 3 条前提（这部分比答案本身更重要）

**C-9 —— `tasks` 没有 `metadata` 列，`raw_item_id` 是一级 uuid 列 + 外键**

我方契约 `news-l1-db.md:123` 写了一行「`metadata` | jsonb | 元数据（含 raw_item_id 等） | 只读」。**这个列不存在**，是契约撰写时的错误，你方 C-9 的整条推论（关联「只能走 `tasks.metadata->>'raw_item_id'` jsonb 表达式」→「队列增长后成为周期性全表扫」）由此而来。实际结构（`server/src/db/schema.ts:250-277`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `raw_item_id` | uuid | **一级列**，`REFERENCES raw_items(id) ON DELETE CASCADE` |
| `run_after` | timestamptz NOT NULL | 退避时间（见 C-4） |
| `max_attempts` | int NOT NULL | 该 task 的尝试上限（建任务时写入） |
| `priority` | int NOT NULL | 优先级 |

索引：`ix_tasks_queue (status, run_after, priority)` + `ix_tasks_locked_at (locked_at)`。

所以关联是 `t.raw_item_id = ri.id` 走主键，**无需任何新索引**。当前 `l1_ai_process` 队列量级下 `ix_tasks_queue` 足够；若未来队列增长到万级，我方补 `(type, status, run_after)` 复合索引，届时不需要你方配合。**C-9 关闭**，契约那一行我方订正。

**C-4 —— 根因是 GRANT 少一列，不是 claim SQL 少条件**

我方退避的真源就是 `tasks.run_after`：失败重排队时写 `run_after = now() + make_interval(secs => backoff)`（`dispatcher.ts:122-127`），claim 时用 `WHERE status='queued' AND run_after <= now()`（`dispatcher.ts:74-83`）。机制本来就在。

问题在于：**我方给 `ai_worker` 的 `GRANT UPDATE ON tasks` 列清单里没有 `run_after`**（`v0.6.1_ai_contract.sql:114`，只有 status / locked_by / locked_at / attempt / updated_at / last_error / last_error_kind）。于是你方失败后置 `status='queued'` 时**无法写 `run_after`**，它保持旧值（早已过期）→ 下一轮立即重领 → 你们推演的「退避完全失效、几十秒烧完 3 次重试」**完全正确，且责任在我方**。

修复（我方做）：`GRANT UPDATE` 追加 `run_after`；契约 claim 规则 SQL 补 `AND t.run_after <= now()`。

**请你方不要用 `tasks.updated_at + backoff(attempt)` 自行计算**——那会与我方 `requeueTask` 形成两套退避真源，卡死回收介入后必然漂移。拿到 `run_after` 写权限后两侧同源：你方失败时写 `run_after = now() + interval`，claim 时按 `run_after <= now()` 过滤。

**Q-4 —— rss 有原文链接，在 `raw_items.source_item_url`，只是不在你们的可读列**

`rss` fetcher（`fetchers/rss.ts:39-49`）确实没把链接放进 `content` jsonb，但它写进了 **`raw_items.source_item_url` 列**（`url: entry.link`）。R-5 字段表只描述了 `content` jsonb 的键，所以没体现——这是我方交付物的表述盲区，不是你方核对失误。

同一列对 `x_twitter` 也有值（`https://x.com/{username}/status/{tweet_id}`，`x-stream-manager.ts:101`），意味着你方**不必再从 `tweet_id` 构造 URL**，那条「已转为 ai 侧适配层实现要求」的结论可以撤回——直接读列即可，三类源统一。

修复（我方做）：`GRANT SELECT` 追加 `source_item_url`。**Q-4 关闭为「实际有，位置 = `raw_items.source_item_url` 列（非 jsonb 键）」**。

---

#### 二、阻塞项答复

**C-2 —— `tasks.status` 取值表（照实现，非新定义）**

穷举全部写入点（`dispatcher.ts:74-127,305-330`、`l1-tasks.ts:32`、`l0-classifier.ts:196`、`x-stream-manager.ts:110`）后，取值集合**只有 4 个**：

| 值 | 说明 | 谁设置 |
|----|------|--------|
| `queued` | 待领取（含首次建任务与失败重排队） | xiaobao 建任务时；**ai 可重试失败时**；xiaobao 卡死回收时 |
| `running` | 已被 claim、处理中 | **ai claim 时**（同时 `attempt = attempt + 1`、写 `locked_by` / `locked_at`） |
| `succeeded` | 处理成功、终态 | **ai 成功写回后** |
| `failed` | 达到 `max_attempts` 上限、终态 | **ai 最终失败时**；xiaobao 达上限时 |

与 `raw_items.l1_status` 的对应（**非一一对应**：`tasks` 是执行态，`l1_status` 是业务态）：

| ai 的时点 | `tasks.status` | `raw_items.l1_status` | 备注 |
|-----------|----------------|----------------------|------|
| claim | `running` | `processing` | 两者同事务写；`l1_status='processing'` 是我方卡死回收的判定依据 |
| 成功 | `succeeded` | `completed` | 同时写 `l1_processed_at = now()` |
| 可重试失败（未达上限） | `queued` + `run_after = now() + backoff` | `retryable_failed` | backoff 按 attempt 取 `[60, 300, 900]` 秒 |
| 最终失败（达上限） | `failed` | `final_failed` | 同时写 `l1_error`、`l1_processed_at` |

只在 tasks 侧出现、`l1_status` 无对应的中间态：**`running` 与重排队的 `queued`**（`l1_status` 用 `processing` / `retryable_failed` 表达，粒度更粗）。反之 `l1_status` 的 `not_started` / `skipped` 只在 task 未创建或不适用时出现。

**C-3 —— 推翻你方推断：是 ai UPDATE 占位行，不是 ai INSERT**

你方推断在**触发器语义上完全正确**，但它与一条你方无从得知的产品硬约束冲突，所以结论要反过来。

事实：占位 `processed_news` **已经在我方实现中**（`l0-classifier.ts:158-179`，仅 `AI_INTEGRATION_MODE=database` 时创建，标题/摘要取 `raw_items.content` 原文）。它不是可选设计，而是我方 v0.6.1 的 AC-01 / AC-06 落地手段——**L0 通过后新闻必须立即在前台可见**（基础展示态：原文标题 + 原文摘要 + 「待解析」角标），不能等 AI 处理完才出现。我方列表查询是 `FROM processed_news JOIN news_positions`，没有占位行就查不到，用户会看到新闻凭空消失一段时间。（这条在我方实现 R1 曾因放错位置成为 dead code，R2 已修到正确位置。）

**你方担心的「排序位先按空分值排一次且无更新路径」不成立**：`news_positions` 只存 `(news_id, position_id)` 关联，**不存任何排序键**；列表排序是查询时按 `processed_news.published_at DESC`（或 `score_total DESC`）实时计算的（`news.ts:37-42`）。触发器早触发只是让新闻**早可见**，你方 UPDATE 后排序自然跟着变。

四项确认：

| 项 | 结论 |
|---|---|
| 写入方式 | **UPDATE 占位行**（占位行由 xiaobao 在 L0 通过时创建）。为兼容占位行因故缺失的极端情况，建议你方用 `INSERT ... ON CONFLICT (raw_item_id) DO UPDATE`——两种情况都能正确落地；你方 INSERT 权限保留，不撤 |
| 幂等键 | ✅ 同意 `raw_item_id` |
| `raw_item_id` 唯一约束 | ✅ **已有**（`schema.ts:202-205`，`.notNull().unique()` + FK ON DELETE CASCADE），可直接支撑 upsert |
| `score_total` 补算时机 | ⚠️ **database 模式下当前无触发点，见第三节第 1 条**（补算函数存在，但只挂在 HTTP 路径上）。在我方补上之前，你方写回后该列保持 NULL |

**附 `id` 列生成方式**（PM 帖留给我随 C-3 明确）：`processed_news.id` 是 `uuid PRIMARY KEY DEFAULT gen_random_uuid()`（`schema.ts:201`）。**ai 侧 INSERT / upsert 时不要显式写 `id`**，由数据库生成；UPDATE 路径本就不涉及。契约 §processed_news 的 `id` 行我方补此说明。

**C-5 —— 是非题的诚实答案：「几乎必然，但不是原子的」**

不能给你们一个干净的「是」，核查后确有窗口：

- `l1_status='queued'` 的**唯一写入点**是 `l0-classifier.ts:187-199`，紧接着下一条语句就 `INSERT INTO tasks(l1_ai_process)`。
- 但我方 dispatcher 全程**没有显式 BEGIN/COMMIT**（已 grep 确认），每条语句自动提交。两条语句之间进程崩溃 → `l1_status='queued'` 但无 task → **正是你们描述的黑洞**，窗口毫秒级但真实存在。
- 另外一条对你方有利的事实：`raw_items` 入库时 `l1_status` 是默认值 `not_started`，此时**尚无** `l1_ai_process` task。所以**你方只 claim `tasks`、不扫 `raw_items` 是安全的**——`not_started` 的条目本来就不该被领。

我方承诺（已登记待办）：把这两条语句包进显式事务。在此之前请**不要**为此增加孤儿探测逻辑（徒增耦合）；联调期若真出现 `queued` 无 task 的条目，告知我方 `raw_item_id` 即可，我方有补建手段。

**C-10 —— 架构半：无异议**

PM 已定案「不引入 `sentiment`、第五类回归 `processing`」并订正契约 v1.2。**架构侧无异议**：无新增 key 即无需安排字段位置，本条对我方消解。补一条支持性事实——我方核查也确认 `tags_v2` 的五类结构目前在前端**尚无消费方**（`NewsPage.mapNews` 走 `Array.isArray(tags_v2) ? ... : 回退 v0.5 tags`，而 `tags_v2` 是对象不是数组，故恒回退），与 PM 的 grep 结论一致。`processing` 作为 DB 模式下唯一的结构化降级线索，保留是正确的。

---

#### 三、主动上报我方 3 项缺口

**1. `score_total` 补算逻辑存在，但 database 模式下没有触发点**

契约 §职责边界引用的 `calcScoreTotal` **确实存在**（`l1-processor.ts:13-24`），加权公式一并给你们做参考：

```
score_total = round( (timeliness×0.25 + impact×0.35 + confidence×0.25 + clarity×0.15) × 2 , 1 )
```

但它的**唯一调用点是 `l1-processor.ts:171`**，即 `l1_process`（HTTP / 内建 L1）路径。**database 模式走的是 `l1_ai_process`，由你方直接写回 `processed_news`，我方 `l1-processor` 全程不执行 → 没有任何代码在你方写回后补算 `score_total`。**

后果：你方写回 `score_dimensions` 后 `score_total` 仍为 NULL；我方列表排序回退到 `COALESCE(score_total, importance_score)`，而占位行 `importance_score = 0` → 这些新闻按分排序时全部沉底，前端评分徽章显示 0。

处置：已上报我方 PM 决策（v0.6.1 内补触发点 or 留后续迭代）。**不阻塞你方**——你方只管写 `score_dimensions`，加权口径仍在 xiaobao 侧（生态分工：四维评分在 ai，加权 `score_total` 留 xiaobao），契约 §职责边界的分工不变，只是我方需要为 database 路径补一个触发点。结论出来后回帖。

**2. 占位行 `published_at` 写死 NULL**

`l0-classifier.ts:165` 的占位 INSERT 里 `published_at` 显式传 NULL，而 `raw_items.published_at` 有值。列表按 `published_at DESC NULLS LAST` 排序 → 待解析的新闻沉到最底。

处置：我方修正占位 INSERT 写入 `raw_items.published_at`。**同时建议你方写回时一并带上 `published_at`**（你方对 `processed_news` 有全列写权限），取 `raw_items.published_at` 即可——双保险，避免占位修复前入库的存量条目一直沉底。这条会进契约。

**3. 手动重试接口不支持 `l1_ai_process`**

`POST /v1/l1-tasks/:task_id/retry` 目前硬判定**只支持 `l1_process`**（`l1-tasks.ts:24`），与我方设计文档 §3.3 声称的「`l1_ai_process` 同样支持」不符。已登记我方待办。联调期若需重试某条失败条目，告知我方 `raw_item_id`，我方直接处理。

---

#### 四、非阻塞项逐条表态

| # | 我方表态 |
|---|---------|
| **C-1 / Q-5** `domain_tags` | **补进可读列**：L0 分类结果存在 `raw_items.l0_label`（varchar(50)），`GRANT SELECT` 追加该列。**但请注意语义差异**——`l0_label` 是**单值字符串**（L0 分类器输出的单个 label），不是 HTTP 模式 `domain_tags` 的**数组**。你方适配层需按单值处理（`[l0_label]` 或单值入 prompt）。这能消除「恒空」，但**不能做到与 HTTP 模式完全等价**，请据此评估是否仍需在 PRD 保留一条差异说明 |
| **C-7** `language` | **接受 PM 定案（固定 `'zh'`）**，但补两条事实订正：① PM 帖中「原文语种归 `raw_items.language`」**该列不存在**——`language` 列只在 `processed_news` 上（我方实现 R1 #DD5 已确认），契约与该表述都需订正；② 我方**占位行当前写的是原文语种**（`l0-classifier.ts:211` `detectLanguage()`，中文→`zh` 否则→`en`），与「该列语义 = 产出内容语种」不一致。我方将占位写入统一为 `'zh'`，使该列语义单一。你方按 PM 定案写 `'zh'` 即可，不受影响 |
| **C-8** 重试真源 | ✅ **同意以 `tasks.attempt` 为真源**（claim 时 `attempt+1`，我方 `requeueTask` 判上限也读它）；`raw_items.l1_attempt` 视为镜像、同事务推进。**补充**：上限请读 `tasks.max_attempts` 列（建任务时按 `AI_MAX_RETRIES` 写入），**不要硬编码 3**——我方 `BACKOFF_CONFIG` 里的 3 是内建 L1 兜底路径用的常量，与 env 可配的 `max_attempts` 存在双真源不一致，我方会一并订正 |
| **C-6** 行锁可行性 | 同意由你方实证。**预判**：PG 的 `FOR UPDATE` 权限检查按表级 UPDATE 判定，列级授权很可能不满足。若实测失败，我方改为对 `tasks` 授表级 UPDATE（`tasks` 无敏感列，可接受）。请回帖告知结论 |
| **Q-1** `needs_context` | **架构侧无异议、倾向补列**，但涉及展示语义，需我方 PM 确认后落地。`processed_news` 加 `needs_context boolean` 成本极低；你方「无法区分证据充分的高分与证据不足的高分」的论证成立，且 DB 模式证据更少、该信号价值更高。PM 结论出来后一并进契约 |
| **Q-3** `analysis` 空值 | ✅ **同意写 NULL**。我方该列 nullable，前端是真值判断（`analysis &&`），`NULL` 与 `''` 都不渲染，但 `NULL` 语义更准 |

---

#### 五、契约订正清单（`contracts/news-l1-db.md`）

按变更纪律契约先行。PM 已在 v1.2 完成 C-10（`sentiment`→`processing`）与 C-7（`language` 取值）两项。**剩余由我方 Architect 执行，分两批**：

**第一批（纯事实订正，不依赖任何人拍板，本轮即改，升 v1.3）**：

1. §tasks 删除不存在的 `metadata` 列；补 `raw_item_id`(uuid, FK) / `run_after`(timestamptz) / `max_attempts`(int) / `priority`(int)
2. §tasks 补 `status` 取值表（4 值）+ 与 `l1_status` 的时点对应表（本帖第二节 C-2）
3. §职责边界 明确 `processed_news` 为「xiaobao 占位 INSERT + ai UPDATE（推荐 upsert on `raw_item_id`）」，说明触发器早触发是有意设计；§processed_news 补 `id` 由 DB 生成、ai 不显式写
4. §claim 规则 SQL 补 `AND t.run_after <= now()`
5. §processed_news 补 `published_at` 写入要求
6. 订正「原文语种归 `raw_items.language`」表述（该列不存在）

**第二批（需 PM 拍板或 DevOps 执行后再升版）**：

7. 权限矩阵：`tasks` UPDATE 追加 `run_after`；`raw_items` SELECT 追加 `source_item_url`、`l0_label` — **需 DevOps 在 `news_test` 实际执行 GRANT 后同步契约**
8. `needs_context` 补列 — 待 PM
9. `score_total` 补算时机 — 待 PM 决策后写入

**为什么不一次改完**：GRANT 三列要 DevOps 实际执行才算数，契约不该先于实现声明权限——那正是本轮 C-4 的坑（契约给了退避、GRANT 没给列）。

---

#### 六、我方登记的待办（不占用你方时间）

| # | 事项 | 归属 | 阻塞你方？ |
|---|------|------|-----------|
| 1 | GRANT 追加 `run_after` / `source_item_url` / `l0_label`（测试库 + 上生产前） | DevOps | **是**（C-4 退避失效、Q-4/C-1 输入质量） |
| 2 | `l0-classifier` 置 queued + 建 task 包进显式事务（C-5 窗口） | Developer | 否 |
| 3 | 占位行 `published_at` 与 `language` 写入修正 | Developer | 否 |
| 4 | `score_total` database 路径补算触发点（决策 + 实现） | PM → Developer | 否 |
| 5 | `max_attempts` 双真源不一致订正 | Developer | 否 |
| 6 | 手动重试接口支持 `l1_ai_process` | Developer | 否（联调期我方兜底） |

第 1 项是唯一卡你方的，我方会推动 DevOps 优先执行。其余不阻塞你方 PRD 定稿与设计阶段。

**致谢**：C-4 与 C-9 这两条你们是从契约字面推出来的，而根因都在我方——一条 GRANT 漏列、一条契约写了不存在的列。这类「按文档实现就会踩坑」的问题只有逐字段核对才会暴露，感谢。

### 2026-07-27 · [REQ-003] xiaobao PM 回应分派 4 项：C-10 产品半定案（不引入 sentiment，契约订正 v1.2）+ C-7 language 固定 zh + Q-2 接受 + Q-6 近期不接入

> 按 2026-07-26 分派索引，PM 名下 4 项全部作答。Architect 名下项（C-2/C-3/C-5 三条阻塞 + C-4/C-1/Q-1/Q-3/Q-4/Q-5/C-8/C-9）已登记 xiaobao INDEX 待办转 Architect 会话，PM 不代答。

#### C-10 产品半 —— **定案：`sentiment` 本阶段不引入**，第五类回归 `processing`，契约已订正 v1.2

**定性**：与 O-1 完全同类——`news-l1-db` 起草时第五类标签笔误（PM 本人起草失误，认账）。`news-l1` HTTP v1 L133 真源即 `processing`，变更纪律第 5 条适用。

**产品裁定依据**：

1. 代码事实：xiaobao 前后端 `sentiment` **零消费**（grep 前后端源码零命中），不存在任何依赖该标签的展示/筛选场景；
2. 引入 `sentiment` 须先改 `news-l1` HTTP 契约扩展 `L1Output`（ai 侧原则正确：不在 DB 模式偷偷多产字段），属跨项目能力扩展，与两侧当前收尾节奏冲突；
3. 情感标签对财经空间有产品价值，但**按需求成熟度排期**：登记为 xiaobao 后续迭代候选（届时走「先改 HTTP 契约 → ai L1Output 扩展 → DB 契约同步」正规程序）。

**结论**：采纳 ai 方案 A 主体并简化——`tags_v2 = {domain, entity, event, content_type, processing}`（与 HTTP 契约完全一致），**无第六 key、无独立字段**。`processing` 承载运行事实（engine/llm/degraded）的价值 PM 认可，DB 模式排查确需结构化线索。契约已订正（v1.2），CHANGELOG 已记行。**C-10 的 Architect 半随之消解**（无 sentiment 需要安排位置），本条整条闭合；Architect 若对订正后字段表无异议无需另行动作。

#### C-7 —— `language` 取值：**产出内容语种，固定 `'zh'`**

`news-l1` 的输出契约本就是「多语言输入 → 中文输出」（摘要/分析/标签均中文），故 `processed_news.language` 语义为**产出内容语种**，ai_worker 写死 `'zh'` 即可，无需语种检测。原文语种不入此列（原文语种归 `raw_items.language`，且 xiaobao 侧语言检测已显式降级为后续迭代项）。契约 v1.2 已将该行说明补全。`id` 列生成方式与 INSERT 语义耦合，留 Architect 随 C-3 一并明确。

#### Q-2 —— `context` 恒 `[]`：**知悉并接受**

xiaobao 前端 `context` 为**条件渲染**（长度 > 0 才显示「背景补全」区块，R3/R4 复核均已核验），空数组零 UI 副作用，不会被误读为缺陷。PM 承诺：将「DB 模式 context 大概率为空（ai 侧防编造过滤所致，非 bug）」写入 v0.6.1 迭代关闭遗留/已知限制，**不会作为质量问题回流反馈**。不要求 ai 改过滤策略（防 LLM 编造的设计正确，质量优先于填充率）。

#### Q-6 —— rss / jin10_flash：**近期不接入 AI 处理链路**

产品排期口径：AI 处理链路（`process_type='ai'`）近期只有 `x_twitter` 类真实数据；`jin10_flash` 生产虽有直显类（`direct`）历史数据但不走 AI 链路，`rss` 未接入。**同意 ai 侧验收分层**（x_twitter 真实验收 / rss+jin10_flash 单测覆盖）。这两类将来接入 AI 链路时，由 xiaobao 按 ai 侧建议**提报新联调诉求**，不留隐性预期。

#### 附：对 ai 侧承诺的对等回应

上述 4 项中 Q-2/Q-6 属「接受现状」型表态，同意 ai 侧写入 v0.2 交付说明与 summary 已知限制章节；xiaobao 侧同步写入 v0.6.1 迭代关闭记录，双侧留痕对齐。

---

### 2026-07-26 · [REQ-003] 答复清单（按角色分派）：致 xiaobao · Architect（阻塞主力）与 xiaobao · PM（4 项，含 1 项阻塞）

> 本帖是上一帖 C-1~C-10 的**行动化版本 + 按角色分派**，便于各角色打开即知自己要答什么。Owner 直接对接。
> 结论先行：**4 条阻塞项里只有 2 条真正需要「想」，另 2 条是把 xiaobao 已有的实现抄进契约。**

#### 📌 分派索引（各角色照此认领）

**→ xiaobao · Architect**（阻塞项主力：4 条阻塞里 3 条在此）

| 项 | 要答什么 | 优先级 |
|---|---------|--------|
| **C-2** `tasks.status` 枚举 | 提供取值表 + 与 `l1_status` 的对应 + ai 四个时点各写什么值 | **阻塞** |
| **C-3** `processed_news` INSERT vs UPDATE | 确认/推翻 ai 推断 + 幂等键 + `raw_item_id` 唯一约束 + `score_total` 补算时机 | **阻塞** |
| **C-5** 「AI 类入库必建 task」 | 一道是非题（否则是永久漏处理黑洞） | **阻塞** |
| **C-10** 的架构那一半 | `sentiment` 与 `processing` 两个语义在 schema 里**怎么摆**（第六个 key / 独立字段 / 其他） | **阻塞**（与 PM 协作） |
| C-4 | 退避条件是否补进契约 claim SQL | 高（非阻塞但影响面最大） |
| C-1 / Q-5 | `domain_tags`（L0 分类结果）能否补进 ai 可读列；或告知已在 `content` jsonb 内的键名 | 高 |
| Q-1 | 是否给 `needs_context` 补一列（信息损失，详见 Q 表） | 中 |
| Q-4 | 查 fetcher 代码确认 rss 的 `content` 是否真无原文链接键 | 中 |
| Q-3 | `analysis` 空值写 NULL 还是 `''`（影响你方判空写法） | 中 |
| C-8 / C-9 | 重试判定以哪个计数器为真源 / claim join 路径是否有索引支撑 | 中 |

**→ xiaobao · PM**（4 项，其中 1 项阻塞）

| 项 | 要答什么 | 为什么必须 PM 答 | 优先级 |
|---|---------|----------------|--------|
| **C-10** 的产品那一半 | `sentiment`（情感标签）**这个能力要不要**、分几档、由谁产出 | 这是**业务标签的产品决策**。Architect 只能定字段怎么摆，定不了「要不要情感分析」。若决定由 ai 产出，属 `L1Output` 扩展，须**先改 `news-l1` HTTP 契约**再实现 | **阻塞** |
| C-7 | `processed_news.language` 取什么值（原文语种？固定 `zh`？沿用 `translation` 的 key？） | 展示语义 | 中 |
| Q-2 | DB 模式下 `context` 大概率恒为 `[]`，产品侧能否接受 | 涉及产品能否接受该展示效果（否则会被当质量问题反馈回来） | 中 |
| Q-6 | `rss` / `jin10_flash` 两类源**近期是否接入真实数据** | 纯产品排期，决定 ai 侧这两类是否需要真实验收 | 中 |

**→ xiaobao · DevOps**（现在无需动作）

| 项 | 说明 |
|---|------|
| C-6 列级 GRANT 下行锁可行性 | **ai 侧自行实测**并回帖，不占用你方时间 |
| 生产库 `news` 的 GRANT | 届时前置——ai 上生产前再执行 |
| 造数队列补跑 `seed_ai_queue_test.sql` | 届时前置——ai 自测消耗完后会提前打招呼 |

> **若想最快解除 ai 侧阻塞**：先由 Architect 清掉 C-2 / C-3 / C-5 三条（其中两条只是查代码），则 ai 侧仅剩 C-10 一条阻塞；C-10 由 PM 与 Architect 各答一半。

#### 分类与建议归属

| # | 性质 | 需要做的事 | 答复角色 |
|---|------|-----------|-----------|
| C-2 `tasks.status` 枚举 | **照实现补文档**（非新决策） | `tasks` 表是 xiaobao 自己在用的，卡死回收逻辑天天读写它，取值集合必然已存在于代码中——把它抄进契约即可 | **Architect**（或 Developer 查代码） |
| C-5 「AI 类入库必建 task」 | **照实现确认**（非新决策） | 入库代码建不建 `l1_ai_process` task，查证即可。建了 → 写进契约作为承诺；没建 → 属 xiaobao 侧待补实现 | **Architect** |
| C-3 `processed_news` INSERT vs UPDATE | **设计决策**（但已有强提示，见下） | 拍板 + 连带明确幂等与触发器时序 | **Architect** |
| C-10 `tags_v2` 第五类冲突 | **契约笔误性质 + 一处产品取舍** | 与 O-1 同类型；比 O-1 多一步——要给 `sentiment` 与 `processing` 各安排位置 | **PM（要不要/几档/谁产出）+ Architect（字段怎么摆）**，必要时 Owner 拍 |

#### 逐条：希望的答复形式（照填即可）

**C-2 —— 请提供 `tasks.status` 的取值表**

期望形式（与契约 §l1_status 枚举同格式）：

| 值 | 说明 | 谁设置 |
|----|------|--------|
| ... | ... | ai_worker claim 时 / 完成时 / 失败时；xiaobao 建任务时 / 回收时 |

并请一并说明：① 与 `raw_items.l1_status` 的对应关系（是否一一对应、有无只在 tasks 侧出现的中间态）；② ai 在 claim / 成功 / 可重试失败 / 最终失败 四个时点分别该写哪个值。
ai 侧不会自行猜测——`tasks` 是 xiaobao 的业务真源，猜错会污染状态机与卡死回收。

**C-5 —— 请回答一个是非题**

> xiaobao 在把 `process_type='ai'` 的条目写入 `raw_items` 时，是否**必然**同时创建对应的 `l1_ai_process` task 行？

- 若「是」→ 请把这条作为承诺写进契约（ai 侧据此可放心只 claim、不检测孤儿条目）。
- 若「否」或「不确定」→ 这是**永久漏处理黑洞**：条目在 `raw_items` 是 `queued`、但无 task 可 claim，ai 永远领不到，两侧都不报错、监控也看不出来。需要 xiaobao 侧补实现，或明确告知 ai 侧需要额外做孤儿条目探测（后者会增加 ai 的工作量与耦合，不推荐）。

**C-3 —— 请确认或推翻 ai 侧的推断**

ai 侧推断结果是 **ai INSERT**，依据是 xiaobao 自己的触发器设计：`trg_processed_news_auto_link` 是 **`INSERT` 后触发**的。若采用「xiaobao 占位 INSERT + ai UPDATE」，触发器会在**占位时**（此时 `score_dimensions` / `score_total` 均为空）就执行一次 `news_positions` 关联，ai 后续 UPDATE 不再触发——排序位会先按空值排一次且无更新路径。反之若 ai INSERT，触发器在结果齐备后触发，语义自然。

请确认下列四项（ai 侧倾向已标出）：

| 项 | ai 侧倾向 | 请确认 |
|---|---|---|
| 写入方式 | **ai INSERT**（非 UPDATE 占位行） | ☐ 同意 ☐ 改为 UPDATE，理由： |
| 幂等键 | 以 `raw_item_id` 做 upsert（重试不产生第二行） | ☐ 同意 ☐ 其他： |
| `raw_item_id` 唯一约束 | 需要存在（支撑上条 upsert） | ☐ 已有 ☐ 需补 ☐ 不加，改用其他幂等方案： |
| `score_total` 补算时机 | ai 写完 `completed` 后由 xiaobao 触发 | 请说明：由触发器 / 轮询 / 应用层？补算完成前 `news_positions` 排序位是否已对外可见？ |

**C-10 —— 需 PM 与 Architect 各答一半**

> **PM 答**：`sentiment`（情感标签）这个能力要不要？分几档？由谁产出？（业务决策）
> **Architect 答**：两个语义在 schema 里怎么摆？（第六个 key / 独立字段 / 其他）
> 两者都答完这条才算闭合。

冲突事实：`news-l1-db` v1.1 要 `{domain, entity, event, content_type, **sentiment**}`；`news-l1` HTTP v1 与 ai 实现均为 `{..., **processing**}`（`news-l1.md:133` / ai `schemas.py:41-46`）。**ai 从不产 `sentiment`**。

ai 侧立场：**两个语义都要保留**，因为它们不是同一种东西被塞进同一字段位——

- `sentiment` = 业务标签（情感倾向），供 xiaobao 展示/筛选用；
- `processing` = 运行事实（`engine:agent_hub` / `llm:{provider}` / `degraded:{reason}`），是 **worker 模式下判断「这条结果是否经过降级、走的哪个 provider」的唯一结构化线索**。DB 模式没有 HTTP 响应可看，丢了它，联调和线上排查都会瞎。

| 方案 | 内容 | ai 侧评价 |
|---|---|---|
| **A（推荐）** | `tags_v2` 保留 `processing`（与 HTTP 契约对齐，按变更纪律第 5 条订正 DB 契约）；`sentiment` 若确需，作为**第六个 key** 或独立字段，并明确由谁产出 | 语义清晰；但若 `sentiment` 要 ai 产出，属 `L1Output` 扩展，**须先改 `news-l1` HTTP 契约**，ai 不会只在 DB 模式偷偷多产字段 |
| **B** | `tags_v2` 只留 `sentiment`（ai 需新增该能力），`processing` 改由日志承载 | ai 侧可接受但有代价：降级事实退化为非结构化，xiaobao 侧无法按「是否降级」筛选结果。ai 已在 PRD AC-6.6 预留「降级事实冗余进日志」以兜底 |

请一并明确：若 `sentiment` 需要 ai 产出，其取值集合是什么（正/负/中性？几档？），以及是否接受先在 v0.2 留空、后续迭代补。

#### 非阻塞的 6 条（C-1 / C-4 / C-6 / C-7 / C-8 / C-9）

不阻塞 ai 定稿，但影响处理质量或长期正确性，建议同轮一并看（详见上一帖表格）。其中：

- **C-4（退避未进 claim SQL）虽标非阻塞，但影响面最大**——按契约字面实现，失败条目会在 10~30s 后被立刻重领，退避完全失效，几十秒烧完 3 次重试。ai 侧会在实现中自行加判定（`tasks.updated_at + backoff(attempt) <= now()`），但**两侧应同源**，建议补进契约的 claim 规则 SQL。
- **C-6（列级 GRANT 下 `SELECT ... FOR UPDATE` 可行性）由 ai 侧实证**，拿到口令后在 `news_test` 实测并回帖，不占用 xiaobao 时间。

#### 另附：ai 侧原打算「自行消化」但按 Owner 要求提出确认的 6 项（Q-1~Q-6）

> Owner（ai 侧）2026-07-26 明确要求：**没法确认的事不能留成遗留，该沟通的就沟通**。
> 下列 6 项 ai 侧原已在 PRD 中写成「已知差异 / 已知限制 / ai 侧自行决定」——即不阻塞、不上报。但它们都属于**未与 xiaobao 确认过的单方判断，且多数会直接影响 xiaobao 侧的展示或数据解读**，故一并提出。
> 这些不阻塞 ai 定稿，但**请逐条表态**（同意 / 要改 / 需补契约），避免成为上线后才发现的隐性落差。

| # | ai 侧原判断 | 为什么必须让 xiaobao 知道 | 请表态 | 答复角色 |
|---|------------|------------------------|--------|---------|
| **Q-1** | `needs_context` **丢弃**（`L1Output` 有该字段，`processed_news` 无对应列），ai 侧原定「本迭代接受丢弃、不扩契约」 | 这是**信息损失**：HTTP 模式下它是给调用方的质量信号（「上下文不足、本条结果存疑」）。丢弃后 xiaobao **无法区分「证据充分的高分」与「证据不足的高分」**，两者在库里长得一模一样。DB 模式预取上下文消失、证据本就更少，该信号的价值反而比 HTTP 模式更高 | ☐ 接受丢弃 ☐ 补一列（列名与类型请定） | **Architect**（schema 变更） |
| **Q-2** | `context`（背景上下文数组）在 DB 模式**大概率恒为 `[]`** | 非 bug：`normalize_output_node` 会过滤掉 URL 不在证据集内的条目（v0.1 有意设计，防 LLM 编造）；DB 模式证据更少故大概率全被过滤。但 xiaobao 侧看到空 `context` 很可能**误读为「ai 没产出背景」**并当成质量问题反馈 | ☐ 知悉并接受 ☐ 需要 ai 侧改变过滤策略（属算法变更，需另行评估） | **PM**（产品能否接受） |
| **Q-3** | `analysis` 为 None 时**写 NULL**（不写空字符串）——ai 侧单方决定 | 直接影响 xiaobao 侧判空展示逻辑（`NULL` vs `''` 的判断写法不同） | ☐ 同意写 NULL ☐ 要求写 `''` | **Architect**/Developer |
| **Q-4** | `rss` 类**无原文链接可用** → link_read 对 rss 失效，ai 侧原定「记为已知限制，不作为契约缺项」 | 依据仅是 R-5 字段表（`{title, summary, author, categories, source_name}`）中无链接字段。但 **R-5 表未必穷尽** `content` 的实际键——rss 条目通常都有 `link`。若实际存在而只是没写进 R-5，则这是可修复的、不该记为限制 | ☐ 确认确实没有 ☐ 实际有，键名为：______ | **Architect**（查 fetcher 代码） |
| **Q-5** | `domain_tags` 恒空（C-1）ai 侧原标「非阻塞、已知差异」 | 按「不能留遗留」原则**升格为需明确答复**：它同时进 prompt 与 KB 检索查询，恒空会让 DB 模式的处理质量系统性低于 HTTP 模式，且这个差距**不会体现为任何报错**，只会表现为评分与标签变差——是最难归因的一类问题 | ☐ 补进 ai 可读列 ☐ 已在 `content` jsonb 内，键名为：______ ☐ 确认接受质量差异 | **Architect**（schema 变更） |
| **Q-6** | 三类 source type 的验收分层（`x_twitter` 真实验收 / `rss`+`jin10_flash` 仅单测） | 已于 2026-07-26 提出但尚未获答复。若两类源近期接入真实数据，ai 侧需在部署就绪检查阶段补真实验收；若不接入，则顺延至实际接入时由 xiaobao 提报新联调诉求 | ☐ 近期接入（预计时间：____） ☐ 近期不接入 | **PM**（产品排期） |

**ai 侧承诺**：上述任一项若 xiaobao 表态「需要补/需要改」，ai 侧配合调整 PRD 与实现；若表态「接受现状」，ai 侧将其写入 v0.2 交付说明与迭代 summary 的**已知限制**章节，作为正式留痕，不做隐性丢弃。

#### ai 侧并行在做的事（不干等）

R2 三方复审、数据源协议设计、async 地基改造的切分与回归策略——**三项均不依赖上述 4 条**。4 条闭合后即可定稿 PRD 进设计阶段。

### 2026-07-26 · [REQ-003] ai PM 转达契约缺项 C-1~C-10（ai 侧 PRD R1 三方 Review 产出，4 条阻塞 ai PRD 定稿）

ai v0.2 PRD R1 已由 Architect / Developer / DevOps 三方 Review 完毕（三方均判未通过，PM 已改出 R2）。三方在**逐字段核对 `news-l1-db` v1.1 与 ai 侧代码基线**的过程中，发现 10 条契约缺项，汇总转达。

**说明**：① 契约变更须先改 `contracts/news-l1-db.md` 再改两侧代码（变更纪律第 1 条），**ai 侧不自行修改契约**；② 标「**阻塞**」的 4 条未闭合前 ai PRD 不能定稿，但**不阻塞 ai 侧设计阶段启动**；③ C-1~C-9 由 ai Architect 提出并经 ai Developer 对 v1.1 逐条复核，C-10 由 ai Developer 提出。

| # | 缺项 | 后果 | 阻塞 ai 定稿 |
|---|------|------|------------|
| **C-2** | **`tasks.status` 的取值集合与转移规则无任何真源**——契约 §tasks 只写「任务状态 / 可更新」。对比 `l1_status` 有完整枚举表 | ai 要「同步更新 `tasks` 状态」却不知该写什么值，也写不出可验证断言。**ai 不能自行猜一套枚举写进 xiaobao 的业务表**（猜错会污染状态机与卡死回收） | **是** |
| **C-3** | **`processed_news` 是 ai INSERT 还是 UPDATE 占位行未定**——§职责边界表写 xiaobao「占位创建」（暗示 ai 只 UPDATE），§权限矩阵却给了 ai `INSERT` | 两种语义导致 `news_positions` 触发器时序不同（占位时触发则排序位先按空分值排一次，ai 后续 UPDATE 不再触发）、事务范围不同、重试幂等键不同。请一并明确：`raw_item_id` 是否有唯一约束、重试是 upsert 还是插第二行、触发器与 UPDATE 的关系。**ai 侧倾向 ai INSERT + 以 `raw_item_id` 为幂等键 upsert**（触发器在结果齐备后触发，语义最干净）。另：方案 A 落地后请明确 xiaobao 补算 `score_total` 的时机与触发方式，以及补算完成前 `news_positions` 排序位是否对外可见 | **是** |
| **C-5** | **未承诺「AI 类条目入库时必建 `l1_ai_process` task」** | ai 对 `tasks` 只有 SELECT + 部分列 UPDATE、**无 INSERT**，只能 claim xiaobao 预建的 task 行。若 task 缺失，条目在 `raw_items` 里是 `queued` 但无 task 可 claim → **永久漏处理的黑洞**，且两侧都不会报错 | **是** |
| **C-10** | **`tags_v2` 第五类标签两份契约冲突**：`news-l1-db` v1.1 写 `{domain, entity, event, content_type, sentiment}`，而 `news-l1` HTTP v1 与 ai 实现均为 `{..., processing}`（`news-l1.md:133` / ai `schemas.py:41-46`）。**ai 从不产 `sentiment`** | AC「写回 `tags_v2`」无法确定内容：按 ai 现状写会多一个 DB 契约没有的 key、少一个 `sentiment`；按 DB 契约写则 `sentiment` 无来源、且 `processing` 丢失。**`processing` 承载 `engine:agent_hub` / `llm:{provider}` / `degraded:{reason}`，是 worker 模式下判断「这条是否降级、走的哪个 provider」的唯一结构化线索**。与 O-1 同类型（DB 契约未对齐 HTTP 契约），变更纪律第 5 条同样适用 | **是** |
| C-1 | **L0 分类结果字段不在 ai 可读列**——契约只给 `l0_status`（状态），没给分类结果 | HTTP 模式下 `L1Input.domain_tags`「来自 L0 分类」，同时进 prompt 与 KB 查询（ai `news_l1.py:206`）。DB 模式下只能恒为 `[]` → 两模式输入不等价、处理质量下降。ai 侧已将其列为「已知差异」不阻塞，但**建议补入可读列**（或说明它已在 `content` jsonb 内并给出 key 名） | 否 |
| C-4 | **契约定义的退避间隔未进入 claim 规则 SQL**——§task type 定义 `[60s, 300s, 900s]`，但 §claim 规则 SQL 只有 `process_type='ai' AND l1_status IN (...) ORDER BY published_at LIMIT N`，**无任何时间条件** | 按字面实现，刚置 `retryable_failed` 的条目会在下一轮询周期（10~30s）被立刻重领，**退避完全失效**，几十秒烧完 3 次重试进 `final_failed`，而退避本意是给下游故障（LLM 限流 / provider 5xx）恢复窗口。ai 侧将在实现中加判定（`tasks.updated_at + backoff(attempt) <= now()`），但**两侧应同源，建议补进契约 SQL** | 否 |
| C-6 | **列级 GRANT 下 `SELECT ... FOR UPDATE` 的可行性未验证**——PG 对行锁的权限判定按**表级** UPDATE 进行，而契约给的是列级 UPDATE | claim 原子性的可实现性悬空。ai 侧将以 `ai_worker` 身份在 `news_test` 实测并反馈结论；若列级授权不满足行锁权限检查，需要 xiaobao 调整授权方式 | 否（ai 侧实证） |
| C-7 | **`processed_news.language` / `id` 无 AI 输出来源**——契约「对应 AI 输出」列为 `—`，ai 的 `L1Output` 也无 `language` 字段 | 契约要求写入但取值规则未定（原文语种检测？固定 `zh`？沿用 `translation` 的 key？）。请明确取值规则，或把这两列移出 ai 写入清单由 xiaobao 侧填。**若决定由 ai 产出语种标识，属 `L1Output` 扩展，需先改 `news-l1` HTTP 契约**，ai 不会只在 DB 模式偷偷多产一个字段 | 否 |
| C-8 | **`raw_items.l1_attempt` 与 `tasks.attempt` 双计数器，重试上限判定真源未定** | 两者都在 ai 可写范围，但「尝试次数未达上限（3）」按哪个判定没写；xiaobao 卡死回收介入后必然漂移。**ai 侧倾向以 `tasks.attempt` 为真源**（与 §task type 的最大次数 3 和退避表同源），`l1_attempt` 视为镜像值、同事务一起推进。请确认 | 否 |
| C-9 | **claim 的 join 路径可能缺索引支撑**——按 `published_at`（在 `raw_items`）升序取、claim 目标在 `tasks`，两表关联只能走 `tasks.metadata->>'raw_item_id'`（jsonb 表达式） | 若无对应索引，队列增长后 claim 会成为周期性全表扫。建议在测试库确认是否存在支撑该访问路径的索引，必要时补 | 否 |

**一条已撤回**：ai Architect R1 曾提「`source_item_url` 不在可读列 → link_read 整条路径失效」。经 ai Developer 对贵方 R-5 结构说明逐类核对，`x_twitter` 的 URL **可由适配层从 `tweet_id` + `author_username` 构造**（含 `author_username` 为空时走 `x.com/i/status/` 的兜底规则），**不需要契约新增可读列**，已转为 ai 侧适配层的实现要求。感谢 R-5 的字段级交付——正是它让这条从「契约缺项」降为「实现细节」。

**另附 ai 侧一条已知限制（不作为契约缺项）**：`rss` 类的 R-5 字段表无任何原文链接字段，故 rss 类 link_read 无 URL 可用。因 rss 当前无真实数据、按已定的验收分层仅做单测覆盖，本迭代记为已知限制。

**ai 侧当前进度**：PRD R2 已出（按三方 R1 意见 + Owner 核心原则「基础夯实、可扩展性优先」修改，含 async 地基改造进范围），待三方复审。C-1~C-10 中 4 条阻塞项闭合 + R2 三方通过后进设计阶段。

### 2026-07-26 · [REQ-003] ai PM 验收交付物：前置解除确认；因「系统仅有 x_twitter 真实数据」调整验收分层；登记 2 项后续前置

**① 交付验收通过，感谢纠错与实测**。三点特别认可：

- **主动撤回误读**（「契约四处不一致」+「缺锁列/`l1_engine`」两处），并保留「R-1/R-2 实测未就绪」不因核对争议而动摇——事实与判断分离，这份留痕对后续查证很有价值。
- **越权拦截实测**（`SELECT alerts` / `INSERT raw_items` / `UPDATE process_type` 均 `permission denied`）。这正好在数据库层为 ai PRD 的 AC-4「ai 不得写契约未授权的列、不得建表改表」提供了强制保障——ai 侧不必只靠应用层自律，验收时可直接引用本次越权实测。
- **列级 GRANT 逐列对照契约一致** + 触发器 `trg_processed_news_auto_link`(SECURITY DEFINER) 已建（ai 无 `news_positions` 写权、由触发器代写，与契约一致）。

**② R-1/R-2/R-4/R-5 确认解除；R-3 仅剩口令交付（在 Owner 手上）**。连接四要素已收：`host=127.0.0.1` `port=5432` `dbname=news_test` `user=ai_worker`（同机走 localhost）。口令存 `/root/.secrets/ai_worker_news_test.pw`（root only、不入仓）——已上报 ai 侧 Owner 经安全渠道交付，交付后由 ai DevOps 按 O-7 拆字段注入。

**③ 关键新事实处置：系统当前只有 `x_twitter` 真实数据**（生产 757 / test 154；`rss` 与 `jin10_flash` 无 `raw_items`，`sources` 仅 4 个 `x_twitter`）。这与 R-5 结构说明给出的三类 type 形成落差，ai 侧据此**调整 v0.2 验收分层**（将写入 PRD R2，属 PM 职责范围内的验收标准调整，不改契约、不改范围）：

| source type | v0.2 实现 | v0.2 验收要求 |
|---|---|---|
| `x_twitter` | ✅ 做 | **真实数据端到端通过**（claim → 处理 → 写回 `completed`），用 DevOps 预置的 5 条 queued 冒烟 |
| `rss` | ✅ 做（按 R-5 结构说明实现映射 + 缺失兜底） | **仅单测覆盖**（构造符合结构说明的 fixture），标注「待真实数据出现后补真实验收」 |
| `jin10_flash` | ✅ 做（同上） | 同 `rss` |

- 理由：三类映射都只是字段搬运，实现成本低且现在一起做最省（避免将来接入 rss 时再动适配层）；但**没有真实数据就不能声称真实验收通过**，否则是虚假的完成度。分层后边界清楚、不阻塞、不返工。
- **请 xiaobao 侧告知**：`rss` / `jin10_flash` 两类源近期（v0.6.1 灰度期内）是否计划接入真实源？若计划接入，ai 侧在 v0.2 部署就绪检查阶段补真实验收；若无计划，则该两类的真实验收顺延到实际接入时，届时由 xiaobao 提报为新的联调诉求即可。

**④ 登记 2 项后续前置（不阻塞 v0.2 设计与实现，但需到时不被遗漏）**：

1. **生产库 `news` 的 GRANT 尚未执行**（本次仅 `news_test`；角色为 cluster 级已建）。ai 侧登记为**上生产前置**，由 xiaobao DevOps 在 ai 上生产前执行；ai 侧不在 v0.2 部署就绪检查中假定生产已就绪。
2. **造数队列会耗尽**：DevOps 预置 5 条 `queued`，ai 侧冒烟与联调会消耗。耗尽后需 xiaobao 补跑 `seed_ai_queue_test.sql`。ai 侧在联调阶段提前打招呼，不做「静默等待队列」的误判。

**⑤ ai 侧后续动作**：v0.2 PRD R1 剩余两方 Review（Architect / Developer）→ 按三方意见改 R2 定稿 → 设计阶段（适配层分层 + worker 参数 + 事务边界）→ 实现 → 拿到口令后进联调冒烟。**DB 前置已解除，ai 侧不再有对 xiaobao 的阻塞依赖**（除口令交付与上述 2 项登记）。

### 2026-07-25 · [REQ-003] xiaobao DevOps 就绪交付：R-1/R-2 已就绪（迁移执行+端到端验证）+ R-3/R-4/R-5 交付，ai 侧 DB 联调前置解除

已执行 `v0.6.1_ai_contract.sql` 到 `news_test`（`news` 无 CREATEROLE，改用 postgres 超级用户；`CHANGE_ME_IN_PRODUCTION` 已替换强口令、不入仓）。逐项就绪：

#### R-1 ✅ ai_worker 角色 + 列级 GRANT（逐列 verify + 端到端 + 越权拦截）

- 角色 `ai_worker`(LOGIN) 已建；列级 GRANT 逐列对照契约**一致**：raw_items SELECT(8列)+UPDATE(l1_status,l1_error,l1_processed_at,l1_attempt)；processed_news 表级 SELECT/INSERT/UPDATE；sources SELECT(id,type,identity,config)；tasks SELECT+UPDATE(status,locked_by,locked_at,attempt,updated_at,last_error,last_error_kind)。
- 端到端：ai_worker 实连 `news_test`，SELECT raw_items=154、UPDATE l1_status 通过；**越权全部被拒**（SELECT `alerts` / INSERT `raw_items` / UPDATE 无权列 `process_type` 均 `permission denied`）→ 最小权限验证通过。
- 触发器 `trg_processed_news_auto_link`(SECURITY DEFINER) 已建：ai INSERT processed_news 后自动关联 news_positions（ai 无 news_positions 写权，触发器代写，契约一致）。

#### R-2 ✅ schema 列已就绪

契约要 ai 可写的 `l1_status/l1_error/l1_processed_at/l1_attempt` + 只读 `process_type/l0_status/published_at/content/id/source_id` 在 `news_test` **全部存在**（v0.6 + ddl_only 已落）。此前「缺锁列/l1_engine」是我误读契约，已撤回。

#### R-3 连接凭据

- 四要素（按 ai O-7 拆字段、不整串 DSN）：`host=127.0.0.1` `port=5432` `dbname=news_test` `user=ai_worker`（ai worker 与 xiaobao **同机**，走 localhost）。
- **口令**：强口令已存服务器 `/root/.secrets/ai_worker_news_test.pw`（chmod 600 / root only / 不入仓 / 不在本文档）。**请 Owner 经安全渠道交付 ai 侧**。

#### R-4 ✅ 造数脚本（选项①）+ 已预置队列

- 脚本 `server/db/scripts/seed_ai_queue_test.sql`（xiaobao 仓，本次提交），xiaobao 侧执行（ai 无 INSERT 无法自造）。
- **已跑一次，`news_test` 现有 5 条 `process_type='ai'` + `l1_status='queued'` 待 ai claim 冒烟**（真实 x_twitter 数据 reset）；耗尽后 xiaobao 补跑。

#### R-5 样例

- **系统当前只有 x_twitter 数据**（生产 757 + test 154；rss/jin10_flash 无 raw_items、sources 仅 4 个 x_twitter）。x_twitter `content` 脱敏样例（印证 PM 结构说明）：`{tweet_id, text, author_username, author_name, created_at, public_metrics{like/quote/reply/retweet/bookmark/impression_count}, entities{urls[]}}`。
- rss/jin10_flash 无真实样例（系统无此类数据）；ai 适配层按 PM R-5 结构说明实现即可。

#### 说明

- 本次仅 `news_test`（ai 联调库）；**生产 `news` 库级 GRANT 待 ai 上生产前另行执行**（角色 cluster 级已建）。
- 结论：**R-1/R-2 就绪，R-3（口令待 Owner 交付）/R-4/R-5 已交付 → ai 侧 DB 模式联调前置解除，可开工。**

### 2026-07-25 · [REQ-003] xiaobao DevOps 纠错：撤回「契约四处不一致」（我误读参照源），阻塞链简化为直接执行迁移；R-1/R-2 未就绪维持

ai PM 的核对异议**完全成立**。对 coordination 契约 `news-l1-db.md` grep 精确复核，我上一帖的「契约 v1.1 vs `ai_contract.sql` 四处不一致」系**误读参照源**（错把 xiaobao 侧 `v0.6.1-design.md` 当成 coordination 契约），全部撤回：

| 我上帖误称 | 契约实际（grep 实证） | 与 `ai_contract.sql` |
|---|---|---|
| 锁用 `raw_items.l1_locked_*` | `l1_locked_by` **0 次**；`tasks` 4 次（tasks 表锁）| ✅ 一致 |
| 重试字段 `l1_attempts` | `l1_attempts` **0 次** / `l1_attempt` 单数 4 次 | ✅ 一致 |
| processed_news 列级+不给 INSERT | 表级 SELECT/INSERT/UPDATE | ✅ 一致 |
| sources `display_name` | `identity` | ✅ 一致 |
| `l1_engine` 缺列 | `l1_engine` **0 次**（契约本无此列）| ✅ 不涉及 |

**结论修正**：`ai_contract.sql` 与契约一致，**无需 Architect 拍定锁机制、无需改 SQL**，阻塞链省去两环。**维持不变**：R-1/R-2 实测未就绪（角色不存在、脚本未执行、`l1_processed_at`/`l1_attempt` 等未落库）——实测铁证，与契约核对无关。DevOps 现即着手执行迁移：建 `ai_worker`+GRANT+强口令 → 补确认 `l1_processed_at`/`l1_attempt` 列 → 交付凭据（四要素+口令、按 ai O-7 拆字段不整串 DSN）+ 造数脚本（选项①）+ 样例。

### 2026-07-25 · [REQ-003] ai PM 复核：接受 R-1 未就绪结论；但「契约 vs SQL 四处不一致」逐行核对不成立 —— 阻塞链可缩短，请确认参照源

感谢 DevOps 按「逐列 verify、不代下结论」实测，**推翻了 PM 引用的部署留痕**。ai 侧此前明确「不将『大概率就绪』当作已就绪」，本轮实测正好印证该谨慎是必要的。

**① 接受且无异议**：`ai_worker` 角色在 PG 实例不存在、`v0.6.1_ai_contract.sql` 从未执行 → **R-1 ❌ 未就绪是硬事实**，ai 侧当前连库即失败。xiaobao 侧订正自身迭代记录属其内部事，ai 侧不介入。

**② 提出核对异议：那张「契约 v1.1 vs `ai_contract.sql`」对比表，四行与 coordination 契约 v1.1 逐行核对均不成立。** 更关键的是——**表中标为「SQL 实现」的那一列内容，恰好与契约 v1.1 一致**：

| 项 | 对比表称「契约 v1.1」 | **契约 v1.1 实际原文** | 对比表称「SQL 实现」 |
|----|---------------------|----------------------|--------------------|
| 锁机制 | `raw_items.l1_locked_by` / `l1_locked_at` | **契约无此二列**。`raw_items` 可写列仅 `l1_status`/`l1_error`/`l1_processed_at`/`l1_attempt`（§raw_items）；锁在 **`tasks.locked_by`/`locked_at`**（§tasks L117-118、§权限矩阵 L149、§卡死回收 L161 三处一致） | `tasks.locked_by`/`locked_at` ← **与契约一致** |
| 重试字段名 | `l1_attempts`（复数） | **`l1_attempt`（单数）**，L41 / L51 / L145 / L146 四处均单数 | `l1_attempt` ← **与契约一致** |
| `processed_news` 权限 | 列级 SELECT+UPDATE，不给 INSERT | **`SELECT, INSERT, UPDATE`｜全列可读写**（§权限矩阵 L147）。O-1 订正明确「GRANT 保持表级不变，`score_total` 以**语义边界**约束」 | 整表 SELECT,INSERT,UPDATE ← **与契约一致** |
| `sources` SELECT 列 | `id, type, config, display_name` | **`id, type, identity, config`**（§权限矩阵 L148；§sources 字段表 L105 亦为 `identity`） | `id, type, identity, config` ← **与契约一致** |

另：**`l1_engine` 在契约 v1.1 中出现 0 次**，不是本契约要求 ai 侧依赖的列。

**推断**：对比表的「契约」一列疑似参照 **xiaobao 侧 `docs/progress/iterations/v0.6.1-design.md`**（xiaobao 项目内部设计文档），而非 coordination `contracts/news-l1-db.md` v1.1。若属实，两文档本身存在偏差，应由 xiaobao 侧自行对齐——但**跨服务边界的单一真源是 coordination 契约**（契约 §真源说明 + §变更纪律第 1 条），ai 适配层只按契约实现。

**③ 因此阻塞链可缩短两环，请 xiaobao 侧确认**：

- 原阻塞链 ①对齐契约 vs SQL（尤其锁机制需 Architect 拍定）→ ②修正 SQL → ③执行迁移 → ④交付 → ⑤回帖。
- 若确认参照源问题成立，则 **①②可省**：锁机制无分歧（契约与 SQL 均为 `tasks` 表锁）、字段名无分歧（均 `l1_attempt`）、权限无分歧（均表级）、`sources` 列无分歧（均 `identity`）→ **直接进 ③ 执行 `ai_contract.sql`（建角色 + GRANT + 强口令）**，无需 Architect 拍板、无需改 SQL。
- ai 侧对此的立场：**ai 不要求 `raw_items` 内联锁，也不依赖 `l1_engine`**。PRD AC-3/AC-5 本就按 `tasks` 表 claim 与释放写的（与契约、与 SQL 三方一致），无需改动。若 xiaobao 后续出于自身需要改锁机制归属，那是**契约变更**，须先改契约再改两侧代码（契约 §变更纪律第 1 条），ai 侧配合但不主动发起。

**④ 请 DevOps 补确认 2 列（对比表未覆盖，但契约要求 ai 可写）**：`raw_items.l1_processed_at`、`raw_items.l1_attempt` 在 `news_test` 是否存在？契约 §权限矩阵 L146 要求 ai_worker 对二者有 UPDATE，AC-4 的写回依赖它们。对比表只列了 `process_type`/`l1_status`/`l1_error` ✅ 与四个契约未要求的缺列，这两列状态不明。

**⑤ R-3 连接信息已收，但环境变量命名归 ai 侧**：`127.0.0.1:5432` / `news_test`（同机）已记录——顺带确认了 ai worker 与 xiaobao 同机部署这一形态事实。凭据交付方式（安全渠道、非仓库非明文对话、`chmod 600` 仓外文件注入）ai 侧完全认同。但**变量名 `AI_WORKER_DB_URL`（整串 DSN）ai 侧不采用**：ai DevOps 在 v0.2 PRD R1 Review 的 O-7 结论已定为**按字段拆分**（`AI_DB_HOST`/`PORT`/`NAME`/`USER`/`PASSWORD`），理由是整串 DSN 极易被日志、`ps` 输出、驱动异常信息整体带出（该 Review 的问题 3 正是「连接异常把 DSN 整串写进日志」）。变量命名属 ai 侧内部实现，不影响交付——**请只交付连接四要素 + 口令值本身即可，不必封装成 URL**。

**⑥ ai 侧影响判定（供双侧排期）**：R-1/R-2 未就绪**只阻塞实现阶段的联调与冒烟，不阻塞设计阶段**。契约 v1.1 + 已交付的 R-5 结构说明已足够 ai Architect 启动适配层与 worker 设计。ai 侧将并行推进：PRD R1 剩余两方 Review → 改 R2 定稿 → 设计阶段；待 xiaobao 回帖「已就绪」后再进实现联调。**双侧不必互等。**

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
| 3 | R-1 `ai_worker` 角色与列级 GRANT 就绪确认 | xiaobao · DevOps | ✅ **已就绪**（2026-07-25）— 已执行 `ai_contract.sql` 到 `news_test`：`ai_worker` 角色 + 列级 GRANT 逐列对照契约一致 + 端到端连库/越权拦截验证通过。（「契约四处不一致」经复核系我误读参照源，已撤回）|
| 4 | R-2 v0.6.1 schema 迁移测试库落地确认 | xiaobao · DevOps | ✅ **已就绪**（2026-07-25）— 契约要 ai 可写的 `l1_status/l1_error/l1_processed_at/l1_attempt` + 只读列在 `news_test` 全部存在。（「缺锁列/l1_engine」系误读契约，已撤回）|
| 5 | R-3 共享库连接信息与凭据注入渠道 | Owner / DevOps 双侧 | ✅ **已闭合**（2026-07-28）— 连接四要素 + 强口令均已就位；ai DevOps 已按「同机直读」注入部署 `.env`（拆字段、chmod 600、未回显），并以 `ai_worker` 实连 `news_test` 六项验证全过（含越权双拒绝），**权限矩阵与契约 v1.4 一致** |
| 5b | **R-4 测试库造数方式**（ai 只有 `raw_items` SELECT 权限，无造数则 DB 模式冒烟无法执行） | xiaobao · DevOps | ✅ **已交付** — 脚本 `server/db/scripts/seed_ai_queue_test.sql`；已跑一次，`news_test` 现有 5 条 `process_type=ai`+`l1_status=queued` 待 claim 冒烟 |
| 5c | **R-5 `raw_items.content` / `sources.config` jsonb 结构说明 + 各 source type 真实样例**（适配层映射的实现前置，不给结构无法写映射、AC-2 无法验收） | xiaobao · PM / Architect | ✅ **结构说明已交付**（2026-07-25 见上方回应：三类 type 字段表 + 缺失兜底 + config 字段 + renderForLLM 参照）；真实样例：x_twitter 已附（DevOps 就绪回帖）；系统当前只有 x_twitter 数据（生产 757 + test 154），rss/jin10_flash 无 raw_items，按结构说明实现即可 |
| 6 | ai v0.2 PRD R1 三方 Review（Architect/Developer/DevOps） | ai 项目组 | 进行中（DevOps 已交：未通过，4 高 2 中 1 低；Architect / Developer 待做） |
| 6b | **契约缺项 C-1~C-10**（ai PRD R1 三方 Review 产出，2026-07-26 转达）：**C-2 / C-3 / C-5 / C-10 四条阻塞 ai PRD 定稿** | xiaobao · PM / Architect（契约权属方） | **部分回应**（2026-07-27）：PM 名下 4 项已全答——**C-10 整条闭合**（不引入 sentiment，tags_v2 回归 processing，契约订正 v1.2）+ C-7（language 固定 zh）+ Q-2（接受）+ Q-6（近期不接入）。**剩 C-2 / C-3 / C-5 三条阻塞待 xiaobao Architect**（已登记 xiaobao INDEX 转办），另 C-4/C-1/Q-1/Q-3/Q-4/Q-5/C-8/C-9 同转 Architect |
| 6e | 后续迭代候选（xiaobao 侧留痕）：`sentiment` 情感标签能力（届时先改 news-l1 HTTP 契约 → ai L1Output 扩展 → DB 契约同步） | xiaobao · PM | 已登记（不排期） |
| 8 | **测试队列不可领**：seed 脚本未建 `l1_ai_process` task 行,ai 冒烟无货可领（阻塞 AC-10.2 + C-6 实证） | xiaobao · DevOps | 待处置（2026-07-28 转办,PM 倾向补建现有 5 条 + 修脚本双管齐下） |
| 9 | **C-14 `l0_label` 语义与取值域**（+ 是否另有领域分类列;确认后写进契约字段说明） | xiaobao · Architect | ✅ **已闭合**（2026-07-28，契约 **v1.5**）— ① `l0_label` 完整取值域已入契约（`direct_display` / `normal_candidate` / `high_priority_candidate` / `needs_context_candidate` / 规则跳过原因 / `llm_skip` / NULL），**语义是处理决策标记非领域分类**，你方判断正确；实测只见一个值是因 L0 链路当时从未跑通（现已修复，2026-07-29 端到端验证通过，开始产出 `high_priority_candidate`）。② **另有列承载领域标签：`sources.domain_tags`**——`L1Input.domain_tags` 的真源，与 L0 无关（`sources.domain_tags` → `l1-processor.ts:243` → `ai-hub.ts:45`）；**我方 v1.3 答 C-1 的结论错误并已公开撤回**。GRANT 已由 DevOps 执行、ai 已复验可读。数据实况另见 6i |
| 10 | 日增量量级确认（决定 ai v0.3 并发化排期） | xiaobao · PM | ✅ **已回应**（2026-07-28：几十条/天,5~10 倍余量,v0.3 无需前移;量级跃迁时 PM 承诺提前知会） |
| 6c | 生产库 `news` 的 GRANT（本次仅 `news_test`） | xiaobao · DevOps | 登记为 ai 上生产前置，届时执行 |
| 6d | 造数队列耗尽后补跑 `seed_ai_queue_test.sql`（ai 自测阶段即开始消耗——并发 claim 与事务回滚必须对真实 PG 测） | xiaobao · DevOps | 待 ai 提出时执行 |
| 6f | **发现 A（阻塞级）`tasks` 表 `type='l1_ai_process'` 记录数为 0** —— R-4 预置的 5 条 `raw_items` 无配套 task，按契约 claim 规则领不到；且实证 SQL 返回 0 行时 PG 不触发 `FOR UPDATE` 权限检查，**会让 C-6 得出假的「通过」**。请确认：① 造数脚本是否遗漏建 task（需补建或修脚本重跑）② 正式链路中 task 由谁何时创建 ③ **type 字面量是 `l1_ai_process` 还是表中已有的 `l1_process`** | xiaobao · DevOps | ✅ **已闭合**（2026-07-28）— xiaobao DevOps 认领造数脚本缺陷并补建 5 条 `l1_ai_process` task + 订正脚本（幂等）；ai 侧已复验 5 条 `queued`/`run_after<=now()` 可领。**C-6 一并闭合**：贵方验权限侧、ai 方验并发侧（两会话拿到不同行、SKIP LOCKED 生效、越权仍拒、队列未消耗），**列级 GRANT 足以支撑行锁，无需改授表级** |
| 6g | **发现 B（高）`l0_label` 真实数据只有 `direct_display` 一个取值**（test 154 条全是它；生产 637 条 + 120 NULL，无第二个非空值）—— 它是流程标记而非领域分类，**C-1「domain_tags 不再恒空」的闭合结论需重估**；ai 侧 `domain_tags` 会恒为 `['direct_display']` 且因是真值拦不掉，会污染 prompt 与回调贵方 `/v1/kb-search` 的查询。请确认：① 是否还有其它取值 ② 是否另有列承载真正的 L0 领域分类结果 | xiaobao · PM / Architect | ⏳ **部分闭合，转 6i**（2026-07-28）— `l0_label` 语义已澄清（处理决策标记，4 个取值），`domain_tags` 真源改为 `sources.domain_tags` 且 GRANT 已生效、ai 已复验可读。**但数据实况另起一条，见 6i** |
| 6h | **`KB_ADMIN_TOKEN` 是否需要** —— DB 模式下贵方不再预取上下文，KB 检索**只能由 ai 主动回调 `/v1/kb-search`**，该路径从补充变为常用。请确认 ① 是否校验 `x-admin-token`；若校验请**走带外渠道或放同机 `/root/.secrets/` 下告知路径**（与 `ai_worker` 口令同纪律，不贴进文档或 git）② 若不校验请明确说明，ai 侧据此标注「test 环境无需 token」 | xiaobao · DevOps | **待确认（联调前置）** — ai 侧 `.env` 中该值当前为空 | ✅ **已闭合**（2026-07-28）— 采纳**方案 A 同机直连免 token**：实测不带 token 与带错误 token 均 HTTP 200（IP 白名单优先）。ai 已配 `KB_SEARCH_URL=127.0.0.1:8001`。认同不下发全权 `ADMIN_TOKEN`。另订正：ai `.env` 的 `KB_ADMIN_TOKEN` 系 v0.1 臆造键名，贵方后端无此 env |
| 11 | **`locked_by` 格式确认（ai Architect 新提，2026-07-28）** —— ai 将写入 `{worker_id}#{run_token}`（稳定身份 + 本次运行标识），以同时满足「能自愈自己上次进程的残留锁」与「绝不误碰他人的锁」。请确认 ① 该列长度/格式无约束 ② **ai 启动时回收自身上次进程的锁不与贵方 1800s 卡死回收冲突**（ai 只回收 `locked_by` 前缀等于自身 `worker_id` 的行）；若贵方回收逻辑对该列内容有假设请告知，ai 调整格式 | xiaobao · Architect | ✅ **已闭合**（2026-07-30，契约 **v1.6**）— ① `locked_by` 是 `text`，**无长度限制、无格式约束、无 CHECK**，`{worker_id}#{run_token}` 随意。② 我方回收 SQL（`reclaim.ts:8-22`）**完全不读该列内容**（只 `SET NULL`），对其零假设，你方按前缀匹配不受干扰。③ **但你方自愈回收必须多改一列**：我方回收除改 `tasks` 外还会把 `raw_items.l1_status` 从 `processing` 同步回 `queued`（`reclaim.ts:26-35`）；你方只改 `tasks.status` 会留下「前端显示解析中但任务在排队」的不一致（我方 reclaim 每 tick 会兜底修复，但窗口内展示是错的）——请同事务改两列。④ **避雷**：我方写入值是 `config.workerId`（env `WORKER_ID`，**默认字面量 `worker-1`**），请你方 `AI_WORKER_ID` 避开。⑤ **⚠️ 本行原文里的「1800s」已作废**——见 v1.7：该数字系我方契约起草臆定，代码默认 `AI_STALE_TIMEOUT_MS=600s`，实际生效值待我方 DevOps 核实回填；**请勿按 1800s 设计自愈窗口** |
| 6i | **（高）`sources.domain_tags` 类型不统一，且 5 条冒烟数据全为 `{}`** —— 实测 `sources` 4 行中 2 行 `array`（`["AI"]`）、2 行 `object`（`{}`）；而 5 条待冒烟条目 JOIN 其 source **全部返回 `{}`**。后果：① 「与 HTTP 模式完全等价」在当前测试数据上不成立，实际仍恒空 ② `{}` 是 object 非 array，ai 的 `L1Input.domain_tags` 为 `list[str]`，按数组处理会触发校验失败 → 归 `MappingError(client_error)` → **不可重试直接 final_failed，5 条冒烟全报废**。请确认：预期类型是什么（若约定数组则 2 行 `{}` 属脏数据）；能否补几条 `domain_tags` 非空 source 下的待处理条目，否则「有值」路径直到生产才第一次执行 | xiaobao · Architect / DevOps | ⏳ **类型已定性，剩落地项**（2026-07-30，契约 **v1.6**）— **预期类型是数组**；`{}` 系 `schema.ts:67` 列默认值误写（应为 `'[]'::jsonb`），语义等同「未配置」= 空数组，**非另一种有意义的形态**。xiaobao 侧一直做归一化容错（`Array.isArray ? v : typeof object ? Object.values(v) : []`），故「等价」指**取数链路等价**，你方指出的「这批数据实际仍恒空」完全成立，我上帖说「完全等价」时未核数据实况。**ai 兜底方案正确、保持不变**（现有数据上与 xiaobao 行为一致；唯一差异是非空 object 如 `{"0":"AI"}`——xiaobao 取 `["AI"]`、ai 给 `[]`，该形态属脏数据、当前不存在、xiaobao 归一化时会清除，ai 不必对齐)。**剩余 xiaobao 落地项**：① 修列默认值为 `'[]'::jsonb` + 迁移归一存量 2 行（Developer/DevOps）② 补 `domain_tags` 非空 source 下的待处理条目，使冒烟覆盖「有值」路径（DevOps，已转办） |
| 6j | **（提示，不阻塞）`tasks.status` 无 CHECK 约束** —— 实测 `pg_constraint` 中该表 `contype='c'` 为空，`status` 写任何值 DB 都不拦。贵方 C-6 实证 SQL 用的是 `status='processing'`，而 C-2 闭合结论枚举仅 4 个（`queued`/`running`/`succeeded`/`failed`），**ai 将按 C-2 写 `running`**。请确认贵方后端判断「处理中」时读的是 `running` 还是 `processing`——无约束兜底时两侧各写各的不会报错，但状态机会认不出，联调时表现为「任务卡住查不出原因」 | xiaobao · Architect | ✅ **已闭合**（2026-07-30，契约 **v1.6**）— **后端判断「处理中」读的是 `running`**（`reclaim.ts:12,19` 两条回收 SQL 均为 `WHERE status='running'`），C-2 的 4 值枚举不变，**ai 按 `running` 写正确**。你方看到的 `processing` 是**另一个表的值**——契约已新增专节写死：`tasks.status='running'`（执行态）vs `raw_items.l1_status='processing'`（业务态），两表故意不同名但同一次 claim 要都写。风险确认：若给 `tasks.status` 写 `processing`，我方回收的 `WHERE status='running'` 认不出 → **卡死回收永不触发、任务永久卡住且双方无报错**。CHECK 约束本迭代不加（`tasks` 是 5 种 type 共用的通用表，加约束需迁移并约束历史数据），以契约枚举为准；如需 DB 层兜底可在后续迭代评估 |
| 12 | **claim 事务边界确认（xiaobao Architect 新提，2026-07-30，契约 v1.7）** —— xiaobao 将补齐数据库层超时（此前**一项都没有**），其中 `idle_in_transaction_session_timeout=60s` 会终止「事务中空闲超 60s」的会话。**若 ai 在事务内等 LLM（240s 预算），会话会被 DB 终止、事务回滚**，表现为「连接莫名断开、任务反复回滚」。契约已写入硬约束：**claim 短事务立即 COMMIT → 事务外调 LLM → 另开事务写回，事务内不得含 LLM 调用或网络等待**。请 ai 确认当前设计的事务边界是否已符合；**确认前 xiaobao 不执行 `ALTER ROLE ai_worker SET ...`** | **ai · Architect** | **待 ai 确认**（阻塞 xiaobao 侧超时配置落地，不阻塞 ai 实现） |
| 13 | **`AI_STALE_TIMEOUT_MS` 实际生效值核实** —— 契约 v1.6 及以前写的「卡死阈值 1800s」系起草臆定、无实现依据（代码默认 600s），**ai 多轮引用的「贵方 1800s 回收」均源自此处**。需核实 test/prod 两侧 `.env` 实际取值并回填契约，两侧对回收窗口的认知必须一致 | xiaobao · DevOps | 待核实（v1.7 已标注默认值 600s + 待回填） |
| 7 | 端到端联调（正常解析 / 失败重试 / 卡死回收 / ai 不可用时 xiaobao 不阻塞 / 双模式切换） | 双侧 | 待 ai 实现阶段完成后启动 |
