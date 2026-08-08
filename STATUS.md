# 跨项目状态

> 跨项目当前状态真源：各项目阶段、当前阻塞、谁等谁、下一步。
> 单项目内部细节不在此展开，从「当前入口」链接回各项目 `docs/progress/INDEX.md`。
> 跨项目需求池见 [REQUESTS.md](REQUESTS.md)（提报中心）。
> 最近更新：**2026-07-28（REQ-003 三件事全部闭合 + C-1~C-14 全闭：`domain_tags` 真源更正为 `sources.domain_tags`、C-6 单会话实证通过、测试队列修复、日增量确认 5~10 倍余量、KB 鉴权定方案 A；契约升 v1.5。ai 侧设计 R2 三方通过、CN-003~007 已落地，待定稿进实现阶段）**。2026-07-27（xiaobao 三方全部答复 ai 转达的契约缺项，4 条阻塞 ai PRD 定稿的项全部闭合；契约连升 v1.2→v1.3、3 列 GRANT 双库执行；ai 侧 PRD 收敛至 R3 待三方复审，无阻塞定稿项）**。2026-07-25（**ai · PM 承接 REQ-003**，转入 ai v0.2 主线、PRD R1 待三方 Review；ai 侧提出 1 个 P0 契约冲突（`score_total` 归属）+ 6 项就绪度/资料确认（含 jsonb 结构样例、测试库造数），待 xiaobao 回应，见 [communications/REQ-003-db-boundary-async.md](communications/REQ-003-db-boundary-async.md)）。2026-07-12（REQ-003 R2 更新 + news-l1-db 契约 v1 出稿，待 ai 侧承接）。2026-07-04（REQ-001 news-l1 端到端联调完成，Owner 抽样验收通过，可进入关闭流程；KB 空结果语义待 ai 优化为已知遗留项，不阻塞）。2026-07-01（xiaobao 测试环境已部署 `/debug/ai`、`/v1/ai-debug/news-l1-runs`、`/v1/kb-search`；news-l1 主链路与 ai→xiaobao KB 命中用例已双向验证通过；KB 空结果语义待 ai 优化）。2026-07-01（ai v0.1 实现阶段完成、`/v1/runs/news-l1` 就绪，向 xiaobao 提 news-l1 联调触发入口诉求，见 communications/REQ-001）。2026-06-30（xiaobao v0.6 已部署上线状态同步；REQ-001 KB search xiaobao 选定方案 b 实时接口）

## 各项目当前状态

| 项目 | 阶段 | 当前入口 | 备注 |
|------|------|----------|------|
| `xiaobao` | v0.6 已上线；v0.6.1 实现 R1（设计文档已定稿，PM/Developer/DevOps R2 全部通过） | 项目 `docs/progress/INDEX.md` | REQ-003 数据库边界契约已出稿，**ai 侧已于 2026-07-25 承接**（转 ai v0.2 主线）；`score_total` 冲突已定案（契约订正 v1.1）；**REQ-003 全部就绪项已交付 ai 侧（2026-07-25）**，仅待答复 rss/jin10_flash 是否接入真实源；生产库 GRANT 待 ai 上生产前执行；生产 AI 处理默认关闭（X 直显） |
| `ai` | v0.1 已关闭（2026-07-04，REQ-001 news-l1 真实化交付）；**v0.2 进行中 — PRD 阶段 R3 待三方复审、无阻塞定稿项**（2026-07-25 范围重排主线为 REQ-003；2026-07-26 按 Owner 核心原则「基础夯实优先」将 async 地基改造纳入范围；2026-07-27 按 xiaobao 三方答复收敛至 R3）；已接入团队工作流；PM（ck）已承接 REQ-001、REQ-002、REQ-003；定位为生态内部通用 AI 处理中枢（[decisions/0002](decisions/0002-ai-hub-ecosystem-positioning.md)） | 项目 `docs/progress/INDEX.md` | REQ-003 已承接（2026-07-25）；**O-1、DB 联调前置、4 条阻塞契约缺项均已闭合**（契约 v1.3 + 3 列 GRANT 双库）；PRD R3 待三方复审；剩余 C-6 待 ai 实证、Q-1 待 xiaobao PM 表态，均不阻塞；v0.2 顺延项：托管化 / 工具并发 / RunRecord / 多 provider 生产验证 → v0.3；KB 空结果语义已并入 v0.2 |
| `workboard` | v0.2 已上线（2026-07-07，读写工作台 5 视图）；v0.3 进行中（设计已定稿 2026-07-18，进实现阶段） | 项目 `docs/progress/INDEX.md` | 本行为补登：workboard 立项（2026-06-17）早于「立项登记三处」规则，状态表历史缺行，参谋长 2026-07-18 补齐 |
| `alpha` | 已立项建壳（2026-08-08），待 Bootstrap 与定位定稿 | 项目 `README.md` | Alpha 管理平台（WorldQuant BRAIN「AI 打工人」Demo 改造）；调研原始材料已入库项目 `docs/research/`；**远端仓库待 Owner 在 GitHub 建 `niuma-cheng-alpha` 空仓后由参谋长首推** |

## 跨项目阻塞与谁等谁

<a name="ai-bootstrap"></a>
### 1. ai 项目接入团队工作流

- 状态：已完成（2026-06-21）—— `ai` 已同步工作流框架（`agent-workflow@90edee2`）并执行 Bootstrap，建 `docs/progress/`（INDEX/iterations/ad-hoc/archive/roles）、填 `project-context.md`（含 `coordination_root`）。
- 谁等谁：无。
- 剩余：无。`ai` 已配置 git remote（`git@github.com:huiyiyouck/niuma-cheng-ai.git`）并推送 `main`。
- 依据：各项目 `docs/baseline/cross-project-collaboration.md` §新项目复用团队工作流。

<a name="news-l1-contract"></a>
### 2. news-l1 契约

- 状态：v1 生效中，已收编为单一真源 → [contracts/news-l1.md](contracts/news-l1.md)
- 谁等谁：无。两侧当前与 v1 一致。
- 下一步责任：任一侧需改契约时，先改 `contracts/news-l1.md` 再改代码。
- 已验证：`ai` 端 `news-l1` 本机 + `xiaobao` `news_test` 库单条真实 raw_item 端到端通过（2026-06-16，见 xiaobao INDEX 跨任务待办 L1 Agent 化条目）。

<a name="news-l1-integration"></a>
### 3. news-l1 真实数据端到端联调

- 状态：**✅ 已关闭（2026-07-04）**——端到端联调完成，Owner 抽样验收通过，ai v0.1 迭代关闭检查通过。ai 服务已部署测试环境 `http://127.0.0.1:8100`；xiaobao 已部署测试环境 `/debug/ai` + `POST /v1/ai-debug/news-l1-runs` + `POST /v1/kb-search`（契约 [contracts/kb-search.md](contracts/kb-search.md) v1）。4 条成功用例覆盖主链路、KB 命中、KB 空结果等场景；单条耗时 74~79s（较 6 月底优化约 25-30%）；四维评分/五类标签/摘要/翻译产出符合 v1 契约。
- 谁等谁：**无互等，已关闭**。已知遗留项（非阻塞，转入下一迭代）：ai 侧 KB 空结果语义——当 xiaobao `POST /v1/kb-search` 返回 200 + `results: []` 时，ai 仍标 `degraded:kb_search_failed`，建议优化为 `kb_search_empty` 或不降级。
- 下一步责任：ai 侧 v0.2 或独立任务优化 KB 空结果语义；xiaobao 侧维持测试服务配置。**news-l1 v1 契约不变**。完整证据见 [communications/REQ-001-news-l1.md](communications/REQ-001-news-l1.md) 2026-07-04 联调完成条 + 关闭记录。

<a name="req003-db-boundary"></a>
### 4. REQ-003 数据库边界异步解耦（news-l1-db 契约落地）

- 状态：**ai 侧已承接（2026-07-25），转入 ai v0.2 主线；PRD 已收敛至 R3 待三方复审、无阻塞定稿项**。契约由 v1 连升至 **v1.3**（v1.1 `score_total` 归属 / v1.2 `tags_v2` 第五类 + `language` / v1.3 Architect 事实订正 6 项），DevOps 已执行 3 列 GRANT（`source_item_url`/`l0_label`/`run_after`，test + prod 双库对称），**契约可升 v1.4 待 xiaobao 补权限矩阵**。
- **已闭合（2026-07-25 同日一轮往返解决 3 项）**：
  1. ✅ **O-1 `score_total` 归属 —— 定案方案 A，P0 阻塞解除**。系契约起草笔误（xiaobao `l1-processor.ts` `calcScoreTotal` 加权已部署为既成事实），非边界变更意图。`contracts/news-l1-db.md` 已订正 **v1.1**（职责边界表 + `processed_news` 字段表 + 状态置生效中），CHANGELOG 记行；ai 侧已核对到位并采纳「GRANT 保持表级、以语义边界约束替代权限收紧」。
  2. ✅ **O-5** `l1_status` 枚举 `completed` 已合并单行。
  3. ✅ **R-5 结构说明已全量交付**（三类 source type `x_twitter`/`rss`/**`jin10_flash`** 的 `content` 字段表 + 缺失兜底 + `sources.config` 字段 + `renderForLLM` 参照点）。ai 侧确认足够启动适配层设计、无需等样例；已收编「三类而非两类」「不依赖 `raw_items.language`」两项事实。真实样例与 R-3 凭据同一前置。
- **✅ R-1/R-2/R-4/R-5 已就绪交付（2026-07-25 xiaobao DevOps 执行迁移 + 实测验证）**：`ai_worker` 角色已建、列级 GRANT 逐列对照契约一致、端到端实连通过、**越权全部被拒**（`SELECT alerts` / `INSERT raw_items` / `UPDATE process_type` 均 `permission denied`）、`SECURITY DEFINER` 触发器已建；契约要求的列在 `news_test` 全部存在；造数脚本已交付且已预置 5 条 `queued` 待 claim；`x_twitter` 脱敏样例已给。
- **DevOps 已撤回两处误读**：「契约 vs SQL 四处不一致」与「缺锁列 / `l1_engine`」系误读参照源（错把 xiaobao 侧 `v0.6.1-design.md` 当 coordination 契约），经 grep 实证全部撤回 → ai PM 核对异议成立，阻塞链省去「Architect 拍定锁机制 + 修正 SQL」两环。R-1/R-2 的实测未就绪结论未因此动摇，事实与判断分离，留痕完整。
- **关键新事实：系统当前只有 `x_twitter` 真实数据**（生产 757 / test 154；`rss` 与 `jin10_flash` 无 `raw_items`、`sources` 仅 4 个 `x_twitter`）。ai 侧据此调整 v0.2 验收分层：三类 type 均实现映射，但 `x_twitter` 要求真实数据端到端验收，`rss`/`jin10_flash` 仅要求单测覆盖并标注「待真实数据补验」——**待 xiaobao 告知两类源近期是否接入真实源**。
- **答复角色分派（2026-07-26 写入沟通文档顶帖，各角色打开即知自己要答什么）**：**Architect = 阻塞主力**（C-2 枚举 / C-3 写入方式 / C-5 必建 task 三条阻塞，另 C-4·C-1·C-8·C-9·Q-1·Q-3·Q-4·Q-5）；**PM = 4 项含 1 项阻塞**（**C-10 的产品那一半**——`sentiment` 能力要不要/几档/谁产出，Architect 定不了；另 C-7·Q-2·Q-6）；**DevOps 现在无需动作**（C-6 由 ai 侧实测；生产库 GRANT 与造数补跑为届时前置）。**C-10 需 PM 与 Architect 各答一半才闭合。** 最快路径：Architect 先清 C-2/C-3/C-5（其中两条只需查代码）→ ai 侧仅剩 C-10 一条阻塞。
- **2026-07-26 给 xiaobao Architect 的答复清单**：4 条阻塞项已行动化——**C-2 / C-5 属「照实现补文档」非新决策**（`tasks.status` 枚举与「AI 类入库必建 task」均可从 xiaobao 现有代码查证）；**C-3 / C-10 需拍板**（C-3 已有强提示：xiaobao 的 `trg_processed_news_auto_link` 是 INSERT 后触发 → 若占位 INSERT + ai UPDATE，触发器会在分值为空时先关联一次且无更新路径，故 ai 推断为 ai INSERT）。每条均附「希望的答复形式」与 ai 侧倾向，可勾选确认。Owner 直接对接 xiaobao Architect。
- **另附 Q-1~Q-6（2026-07-26，按 ai 侧 Owner「没法确认的不能留成遗留」要求提出）**：ai 原打算自行消化为「已知差异/已知限制」的 6 项单方判断，全部提出请 xiaobao 表态——`needs_context` 丢弃（信息损失：无法区分证据充分与不足的高分）/ `context` 恒空易被误读 / `analysis` 空值写 NULL / rss 是否真无原文链接（R-5 表未必穷尽）/ `domain_tags` 恒空升格为需明确答复 / 三类源验收分层待答。不阻塞定稿，但逐条留痕不隐性丢弃。
- **新增：契约缺项 C-1~C-10（2026-07-26 ai PM 转达，ai PRD R1 三方 Review 逐字段核对产出）** —— **C-2**（`tasks.status` 枚举无真源）/ **C-3**（`processed_news` INSERT vs UPDATE 未定 + 幂等键 + 触发器时序）/ **C-5**（未承诺「AI 类入库必建 task」→ 永久漏处理黑洞）/ **C-10**（`tags_v2` 第五类 `sentiment` vs `processing` 与 HTTP 契约冲突）**四条阻塞 ai PRD 定稿**；C-1/C-4/C-6~C-9 为建议补齐项。另撤回一条（`source_item_url` 可由适配层构造，R-5 已给规则）。详见沟通文档 2026-07-26 转达帖。
- 谁等谁：**ai 侧 DB 联调前置已解除**；剩余项无一阻塞 v0.2 设计与实现：
  1. ~~口令交付~~ —— **已不是双方在途项**：ai worker 与 xiaobao 同机部署，ai DevOps 在部署阶段直接从服务器 `/root/.secrets/ai_worker_news_test.pw` 读入 ai 部署目录 `.env`，无需 Owner 转交、不经对话传递。xiaobao 侧无需跟进。
  2. **生产库 `news` 的 GRANT 尚未执行**（本次仅 `news_test`，角色 cluster 级已建）→ 登记为 **ai 上生产前置**，由 xiaobao DevOps 届时执行；ai 侧不假定生产已就绪。
  3. **造数队列会耗尽**（预置 5 条），耗尽后由 xiaobao 补跑 `seed_ai_queue_test.sql`；ai 侧联调前打招呼，不做「静默等待队列」误判。
- **✅ 2026-07-28 三件事全部闭合 + C-11~C-14 全闭（xiaobao 三方同日答复，ai 已回执）**：
  1. **`domain_tags` 真源找到 —— xiaobao Architect 主动撤回其上轮对 C-1 的错误答复并认账**：真源是 **`sources.domain_tags`**（信息源级静态领域标签），**不是** `raw_items.l0_label`。GRANT 已执行、契约升 **v1.5** → **ai 与 HTTP 模式在该字段上完全等价**，一个「已知限制」直接消失。ai 侧据此撤回 CN-004 变更 1（CN-007）、重写设计 §3.3（CN-006）。`l0_label` 语义确认为「是否值得送 AI + 优先级 + 是否需补上下文」的处理决策标记，**ai 侧不再使用该列**。
  2. **C-6 行锁实证通过（xiaobao DevOps 顺手实测）** → ai claim 采用写法 A。⚠️ **但双方均不得视作彻底关闭**——本次仅验证**单会话可行性**，**多 worker 并发不重复的验证仍在 ai 侧待做**；ai 侧 PRD「v0.3 多实例前必须先解决 C-6」前置**不解除**，完成后再回帖。
  3. **测试队列已修复**：xiaobao DevOps 认领系其造数脚本缺陷（「正是 C-5 讨论过的形态，这次是我造出来的」），已补建 5 条 task + 订正脚本为幂等 → **ai 的 AC-10.2 真实数据冒烟阻塞解除**。
  4. **日增量已答**：活跃期日均 **15~30 条**（757 条系 50+ 天累积），逐项排查无「上千条/天」场景 → 对照 ai 能力上界 340~920 条/天**有 5~10 倍余量**，**ai v0.3 并发化无需排期前移**（O-11 由 P1 降 P2）。xiaobao PM 承诺量级跃迁时**提前经本文档知会**。
  5. **C-11/C-12/C-13 三条答复均与 ai 假设一致，ai 无需改实现**；且 xiaobao 应用层判重试上限**已改读 `tasks.max_attempts` 行内值**（两侧同源，`AI_MAX_RETRIES` 改动不再漂移）。
  6. **KB 检索鉴权定案：方案 A（同机内网直连 + IP 白名单，无需 token）**。双方一致**不采纳**方案 B——唯一可用的是 xiaobao **全权 `ADMIN_TOKEN`**，下发即授予改源/删空间/同步规则等全部 admin 写权限，**违反最小权限**。**部署约束**：方案 A 唯一前提是**同机**；任一侧迁机则 KB 全失败且**主流程不中断只持续降级**，故 ai 已将「核对同机前提」写入部署就绪检查；迁机正解为 xiaobao 新增**独立只读 KB token**，ai 侧承诺迁机前提前提报。
  > **`C-1 ~ C-14` 全部闭合**（ai 侧已闭合计数 18）；**仅剩 Q-1（`needs_context` 是否补列）待 xiaobao PM 表态**，不阻塞 ai 定稿与实现。
- ~~2026-07-28 ai 新提三件事~~ —— 已全部闭合，见上。原文：
  1. **C-14** `l0_label` 完整取值域与语义——实测两库**只有 `direct_display` 一个非空取值**（test 154/154，生产 637 非空全同 + 120 NULL），**是流程标记非领域分类**，**推翻双方此前对 C-1 的闭合结论**。ai 已订正 PRD（`domain_tags` 在 DB 模式实际恒为 `[]`，CN-004）并用**排除集**处置——对方将来启用真实分类时 ai 无需改代码即自动生效。请确认取值域 + 是否另有列承载真实 L0 分类。
  2. **`tasks` 中 `l1_ai_process` 记录为 0** —— 2026-07-25 交付的 5 条预置 `raw_items` 没有对应 task 行，ai 按契约**只 claim tasks**（C-5 既定边界）故**永远领不到**。**阻塞 ai 的 AC-10.2 真实数据冒烟与 C-6 实证**。请修正 `seed_ai_queue_test.sql` 或补建 task 行。ai 侧按 C-5 结论**不做孤儿探测**，只新增空转可观测性（报告自己领不到活，不去查 `raw_items`）。
  3. **请提供 `process_type='ai'` 的日增量量级** —— ai v0.2 处理能力上界约 **340~920 条/天**（`N=1` + 单实例 + 批内串行，**无横向扩展余地**），生产已有 757 条历史而日增量双方从未确认。该数决定 **ai v0.3 并发化是否需排期前移**；若日增超过处理能力，队列将持续单调增长且**不报错**，REQ-003 只兑现一半动机。
  > **角色分派（已写入沟通文档顶帖）**：**Architect** 答 C-14（`l0_label` 取值域与语义 / 是否另有列承载真实分类；答「就是流程标记、暂无分类列」亦为完整答案，只需写进契约字段说明）；**DevOps** 处置测试队列不可领（修造数脚本或补建 task 行）——**这件最急，是 ai 唯一被卡住的动作**；**PM** 答日增量量级（属产品投放节奏与开关策略，非数据库统计）。三件互不依赖，可并行。
  > 本轮已是**第四次「文档表述与实现不符」**（C-9 `metadata` 列不存在 / C-4 GRANT 缺列 / Q-4 rss 链接在一级列 / 本次 `l0_label` 语义），四次均靠实读实测才发现。
- 迟滞记录：REQ-003 于 2026-07-05 提报、07-12 R2 更新，ai 侧 07-25 才响应，约 20 天无人接（响应端可见性缺口，正是 REQ-004 要解的问题）。
- 下一步责任：① **`ai_worker` 口令注入**（归 ai DevOps 部署阶段执行：同机直读服务器文件写入 `.env`，无需 Owner 转交）；② **xiaobao 回应契约缺项 C-1~C-10**（4 条阻塞 ai PRD 定稿）+ 答复「rss/jin10_flash 近期是否接入真实源」；③ ai 项目组推进 PRD R2 三方复审 → 定稿 → 设计 → 实现 → 联调。生产库 GRANT 与造数补跑为届时前置。契约变更须先改 `contracts/news-l1-db.md` 再改两侧代码，CHANGELOG 记一行。
- 完整往来见 [communications/REQ-003-db-boundary-async.md](communications/REQ-003-db-boundary-async.md)。

## 下一步汇总

1. ~~Owner：为 `niuma-cheng-ai` 配置 GitHub remote~~ —— 已完成（2026-06-21），`PROJECTS.md` 仓库地址已回填。
2. ai 项目会话：PM 已承接 REQ-001；Owner 确认后由 PM 创建 `v0.1-prd.md` 启动标准迭代，实现 news-l1 真实 L1 处理（stub→真实）。
3. 任一侧改 news-l1 契约：先改 `contracts/news-l1.md`，CHANGELOG 记一行。
4. xiaobao 项目会话：联调证据已齐，维持测试服务配置待命。
6. ~~xiaobao DevOps：执行迁移建角色 + GRANT + 交付凭据/造数/样例~~ —— **✅ 已完成（2026-07-25）**，ai 侧 DB 联调前置解除。剩 xiaobao 一问：`rss`/`jin10_flash` 近期是否接入真实源（决定 ai 侧这两类是否需真实数据验收）。
8. ~~Owner 交付 `ai_worker` 口令~~ —— **已简化为 ai DevOps 部署阶段同机直读注入**，不再是跨项目在途项（2026-07-27）。
9. **xiaobao（2026-07-27 新增）**：① 契约升 **v1.4** 补权限矩阵 3 列（本轮 C-9/C-4/Q-4 三条前提错误均源于契约文本与实现分叉，勿遗漏）；② **Q-1 `needs_context` 是否补列**待 PM 表态（其 Architect 已倾向补列）；③ 其 PM 帖「原文语种归 `raw_items.language`」该列不存在（其 Architect 已指出），内部两份表述待对齐；④ `score_total` 在 database 模式无触发点，待其 PM 决策补。
10. **ai（2026-07-27 新增）**：① PRD R3 三方复审 → 定稿 → 设计阶段；② **C-6 行锁可行性实证**（口令注入后以 `ai_worker` 在 `news_test` 实测 `SELECT ... FOR UPDATE` 在列级 GRANT 下是否可行，结论回帖 coordination）。
7. **ai 项目会话（2026-07-25 新增）**：v0.2 PRD R1 三方 Review（Architect/Developer/DevOps）；**O-1 已解除 + DB 前置已解除**，三方齐后改 R2 定稿→设计→实现→联调（仅待 Owner 交付口令）。
5. ~~ai 项目会话：执行 v0.1 迭代关闭检查（联调证据已齐）；后续优化 KB 空结果语义（非阻塞，可入 v0.2 或独立任务）。~~ —— 已完成（2026-07-04，v0.1 已关闭）

## 元信息变更台账

> 用途见各项目 `cross-project-collaboration.md` §项目元信息同步。子项目迭代关闭检查发现定位/名称/技术栈/上线/接入状态变更时登记一行；coordination 会话改 `PROJECTS.md` 后勾「PROJECTS 已同步」；生态索引维护方（本生态=根 `/root/Project/CLAUDE.md`）照真源订正后勾「生态索引已同步」。两列都勾 → 归档/移除。

| 项目 | 字段 | old | new | 来源（commit/迭代） | PROJECTS 已同步 | 生态索引已同步 |
|------|------|-----|-----|----------------------|-----------------|----------------|
| workboard | 技术栈 | 本地 Node + 静态前端 | React 18 + Vite + Tailwind v4 + shadcn + 本地 Node 聚合后端 | v0.1（`v0.1-summary.md`） | ✅ | ✅ |
| workboard | 上线状态 | 待开发 MVP | v0.1 已上线 `workboard.huiyiyou.cloud` | v0.1 关闭 | ✅ | ✅ |
| workboard | 接入版本 | @1b01fba | @66c1e1a | 回流 `4b8e563` | ✅ | ✅ |
| ai | 定位 | AI 处理中枢 / Agent Hub（职责限 xiaobao L1 新闻处理） | niuma-cheng 生态内部通用 AI 处理中枢（多调用方预留，首落地 xiaobao news-l1） | decisions/0002（2026-06-29） | ✅ | ✅ |
| agent-workflow | 名称 | AI 助手团队工作流真源 | AI 助手团队工作流真源（内部定位：一人公司 AI 组织操作架构·SOP 真源） | BCR-007（commit `4ec68ce`，2026-07-03） | ✅ | ✅ |
| agent-workflow | 接入版本 | @6ebc119 | @4ec68ce | BCR-007 回流（2026-07-03） | ✅ | ✅ |
| ai | 实现状态 | 骨架占位 | v0.1 已交付 | v0.1 迭代关闭（2026-07-04，`v0.1-summary.md`） | ✅ | ✅ |
| workboard | 名称/定位 | 跨项目 Agent 工作看板 | 项目管理工作台（Web 端项目迭代 IDE） | v0.2 迭代关闭（2026-07-07，`v0.2-summary.md`） | ✅ | ✅ |
| workboard | 功能范围 | 只读聚合看板（4 视图） | 读写工作台（5 视图：新增项目会话 + 角色映射 + 对话查看 + 迭代时间轴） | v0.2 迭代关闭（2026-07-07，`v0.2-summary.md`） | ✅ | ✅ |
