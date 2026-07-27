# REQ-003 数据库边界异步解耦 跨项目沟通

- 需求：REQ-003（状态见 [../REQUESTS.md](../REQUESTS.md)）
- 参与项目：xiaobao（提出方 / schema 权属方）, ai（承接方 / worker 方）
- 契约真源：[../contracts/news-l1-db.md](../contracts/news-l1-db.md) v1.2（数据库边界）；[../contracts/news-l1.md](../contracts/news-l1.md) v1（HTTP 模式，继续有效，灰度/回滚用）
- 最近更新：2026-07-27

## 关系概述

REQ-003 是 xiaobao · PM 提报的集成模式变更：news-l1 的 AI 解析从「HTTP 同步调用」改为「数据库契约边界异步解耦」——xiaobao 入库标记状态，ai 轮询 claim 取数、处理、写回共享库。翻译职责保留在 ai 侧（与 REQ-001 一致）。

对 ai 侧是**运行形态级变更**：HTTP 常驻服务 → 常驻轮询 worker，且需新增数据源适配层以保住 [../decisions/0002](../decisions/0002-ai-hub-ecosystem-positioning.md) 的多调用方定位（不被 xiaobao 库 schema 焊死）。

## 承接的需求

| 需求 id | 内容 | 状态 | 详情 |
|---------|------|------|------|
| REQ-003 | v0.6.1 集成模式变更（数据库契约边界异步解耦） | 已承接，开发中（ai v0.2 主线） | [../REQUESTS.md](../REQUESTS.md) §REQ-003 |

## 联调沟通

> 倒序排列。

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
| 5 | R-3 共享库连接信息与凭据注入渠道 | Owner / DevOps 双侧 | ✅ **连接四要素已交付**（`127.0.0.1:5432/news_test/ai_worker`，按字段拆分非整串 DSN）；强口令已存服务器 `/root/.secrets/ai_worker_news_test.pw`（chmod 600），**待 Owner 经安全渠道交付 ai** |
| 5b | **R-4 测试库造数方式**（ai 只有 `raw_items` SELECT 权限，无造数则 DB 模式冒烟无法执行） | xiaobao · DevOps | ✅ **已交付** — 脚本 `server/db/scripts/seed_ai_queue_test.sql`；已跑一次，`news_test` 现有 5 条 `process_type=ai`+`l1_status=queued` 待 claim 冒烟 |
| 5c | **R-5 `raw_items.content` / `sources.config` jsonb 结构说明 + 各 source type 真实样例**（适配层映射的实现前置，不给结构无法写映射、AC-2 无法验收） | xiaobao · PM / Architect | ✅ **结构说明已交付**（2026-07-25 见上方回应：三类 type 字段表 + 缺失兜底 + config 字段 + renderForLLM 参照）；真实样例：x_twitter 已附（DevOps 就绪回帖）；系统当前只有 x_twitter 数据（生产 757 + test 154），rss/jin10_flash 无 raw_items，按结构说明实现即可 |
| 6 | ai v0.2 PRD R1 三方 Review（Architect/Developer/DevOps） | ai 项目组 | 进行中（DevOps 已交：未通过，4 高 2 中 1 低；Architect / Developer 待做） |
| 6b | **契约缺项 C-1~C-10**（ai PRD R1 三方 Review 产出，2026-07-26 转达）：**C-2 / C-3 / C-5 / C-10 四条阻塞 ai PRD 定稿** | xiaobao · PM / Architect（契约权属方） | **部分回应**（2026-07-27）：PM 名下 4 项已全答——**C-10 整条闭合**（不引入 sentiment，tags_v2 回归 processing，契约订正 v1.2）+ C-7（language 固定 zh）+ Q-2（接受）+ Q-6（近期不接入）。**剩 C-2 / C-3 / C-5 三条阻塞待 xiaobao Architect**（已登记 xiaobao INDEX 转办），另 C-4/C-1/Q-1/Q-3/Q-4/Q-5/C-8/C-9 同转 Architect |
| 6e | 后续迭代候选（xiaobao 侧留痕）：`sentiment` 情感标签能力（届时先改 news-l1 HTTP 契约 → ai L1Output 扩展 → DB 契约同步） | xiaobao · PM | 已登记（不排期） |
| 6c | 生产库 `news` 的 GRANT（本次仅 `news_test`） | xiaobao · DevOps | 登记为 ai 上生产前置，届时执行 |
| 6d | 造数队列耗尽后补跑 `seed_ai_queue_test.sql`（ai 自测阶段即开始消耗——并发 claim 与事务回滚必须对真实 PG 测） | xiaobao · DevOps | 待 ai 提出时执行 |
| 7 | 端到端联调（正常解析 / 失败重试 / 卡死回收 / ai 不可用时 xiaobao 不阻塞 / 双模式切换） | 双侧 | 待 ai 实现阶段完成后启动 |
