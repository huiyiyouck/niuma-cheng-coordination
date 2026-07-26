# 跨项目状态

> 跨项目当前状态真源：各项目阶段、当前阻塞、谁等谁、下一步。
> 单项目内部细节不在此展开，从「当前入口」链接回各项目 `docs/progress/INDEX.md`。
> 跨项目需求池见 [REQUESTS.md](REQUESTS.md)（提报中心）。
> 最近更新：2026-07-25（**ai · PM 承接 REQ-003**，转入 ai v0.2 主线、PRD R1 待三方 Review；ai 侧提出 1 个 P0 契约冲突（`score_total` 归属）+ 6 项就绪度/资料确认（含 jsonb 结构样例、测试库造数），待 xiaobao 回应，见 [communications/REQ-003-db-boundary-async.md](communications/REQ-003-db-boundary-async.md)）。2026-07-12（REQ-003 R2 更新 + news-l1-db 契约 v1 出稿，待 ai 侧承接）。2026-07-04（REQ-001 news-l1 端到端联调完成，Owner 抽样验收通过，可进入关闭流程；KB 空结果语义待 ai 优化为已知遗留项，不阻塞）。2026-07-01（xiaobao 测试环境已部署 `/debug/ai`、`/v1/ai-debug/news-l1-runs`、`/v1/kb-search`；news-l1 主链路与 ai→xiaobao KB 命中用例已双向验证通过；KB 空结果语义待 ai 优化）。2026-07-01（ai v0.1 实现阶段完成、`/v1/runs/news-l1` 就绪，向 xiaobao 提 news-l1 联调触发入口诉求，见 communications/REQ-001）。2026-06-30（xiaobao v0.6 已部署上线状态同步；REQ-001 KB search xiaobao 选定方案 b 实时接口）

## 各项目当前状态

| 项目 | 阶段 | 当前入口 | 备注 |
|------|------|----------|------|
| `xiaobao` | v0.6 已上线；v0.6.1 实现 R1（设计文档已定稿，PM/Developer/DevOps R2 全部通过） | 项目 `docs/progress/INDEX.md` | REQ-003 数据库边界契约已出稿，**ai 侧已于 2026-07-25 承接**（转 ai v0.2 主线）；`score_total` 冲突已定案（契约订正 v1.1）；PM 侧回应已清，剩 4 项待 xiaobao DevOps（R-3 凭据最优先）；生产 AI 处理默认关闭（X 直显） |
| `ai` | v0.1 已关闭（2026-07-04，REQ-001 news-l1 真实化交付）；**v0.2 进行中 — PRD 阶段 R1 待 Architect/Developer/DevOps 三方 Review**（2026-07-25 范围重排，主线为 REQ-003 数据库边界异步解耦）；已接入团队工作流；PM（ck）已承接 REQ-001、REQ-002、REQ-003；定位为生态内部通用 AI 处理中枢（[decisions/0002](decisions/0002-ai-hub-ecosystem-positioning.md)） | 项目 `docs/progress/INDEX.md` | REQ-003 已承接（2026-07-25），O-1 已解除（契约 v1.1）；PRD R1 Review 中（DevOps 未通过），待改 R2；v0.2 顺延项：托管化 / 工具并发 / RunRecord / 多 provider 生产验证 → v0.3；KB 空结果语义已并入 v0.2 |
| `workboard` | v0.2 已上线（2026-07-07，读写工作台 5 视图）；v0.3 进行中（设计已定稿 2026-07-18，进实现阶段） | 项目 `docs/progress/INDEX.md` | 本行为补登：workboard 立项（2026-06-17）早于「立项登记三处」规则，状态表历史缺行，参谋长 2026-07-18 补齐 |

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

- 状态：**ai 侧已承接（2026-07-25），转入 ai v0.2 主线**，PRD R1 待 Architect/Developer/DevOps 三方 Review。xiaobao 侧前置已就绪（v0.6.1 PRD R2 定稿、设计 R2 三方通过、`contracts/news-l1-db.md` v1 出稿）。
- **已闭合（2026-07-25 同日一轮往返解决 3 项）**：
  1. ✅ **O-1 `score_total` 归属 —— 定案方案 A，P0 阻塞解除**。系契约起草笔误（xiaobao `l1-processor.ts` `calcScoreTotal` 加权已部署为既成事实），非边界变更意图。`contracts/news-l1-db.md` 已订正 **v1.1**（职责边界表 + `processed_news` 字段表 + 状态置生效中），CHANGELOG 记行；ai 侧已核对到位并采纳「GRANT 保持表级、以语义边界约束替代权限收紧」。
  2. ✅ **O-5** `l1_status` 枚举 `completed` 已合并单行。
  3. ✅ **R-5 结构说明已全量交付**（三类 source type `x_twitter`/`rss`/**`jin10_flash`** 的 `content` 字段表 + 缺失兜底 + `sources.config` 字段 + `renderForLLM` 参照点）。ai 侧确认足够启动适配层设计、无需等样例；已收编「三类而非两类」「不依赖 `raw_items.language`」两项事实。真实样例与 R-3 凭据同一前置。
- **R-1/R-2 实测未就绪（2026-07-25 xiaobao DevOps 逐列 verify，推翻 PM 引用的部署留痕）**：`ai_worker` 角色在 PG 实例不存在、`v0.6.1_ai_contract.sql` 从未执行 → ai 侧当前连库即失败；schema 仅 `ddl_only` 落地。
- **ai PM 复核提出核对异议（2026-07-25）**：DevOps 所列「契约 v1.1 vs `ai_contract.sql` 四处不一致」经逐行核对**均不成立**——契约实际为 `tasks` 表锁、`l1_attempt` 单数、`processed_news` 表级 SELECT/INSERT/UPDATE、`sources` 取 `identity`，**四项与 SQL 一致**；`l1_engine` 在契约中出现 0 次。疑参照源为 xiaobao 侧 `v0.6.1-design.md` 而非 coordination 契约。**若确认成立，阻塞链省去「Architect 拍定锁机制 + 修正 SQL」两环，可直接执行迁移。**
- 谁等谁：**xiaobao 欠 ai 4 项，全部归 xiaobao DevOps / Owner**（PM 侧已清）：
  1. **执行 `v0.6.1_ai_contract.sql`：建 `ai_worker` 角色 + GRANT + 强口令（替换 `CHANGE_ME_IN_PRODUCTION`）** —— R-1 的唯一出路，角色未建则后续全部无法开始。先确认上述核对异议，成立则无需改 SQL、无需 Architect 拍板。
  2. 补确认 `raw_items.l1_processed_at` / `l1_attempt` 两列在 `news_test` 是否存在（契约 §权限矩阵要求 ai 可写、AC-4 写回依赖，DevOps 对比表未覆盖）。
  3. **R-3 凭据交付** —— 连接信息已给（同机 `127.0.0.1:5432` / `news_test`，即 ai worker 与 xiaobao 同机部署）；口令待角色建成后经安全渠道交付。ai 侧只需连接四要素 + 口令值，**不必封装成 `AI_WORKER_DB_URL` 整串 DSN**（ai DevOps O-7 已定按字段拆分，防 DSN 被日志/`ps`/驱动异常整串带出）。
  4. R-4 测试库造数方式（双侧 PM 均倾向**选项①造数脚本**，不倾向选项③临时放开 INSERT）。
- 迟滞记录：REQ-003 于 2026-07-05 提报、07-12 R2 更新，ai 侧 07-25 才响应，约 20 天无人接（响应端可见性缺口，正是 REQ-004 要解的问题）。
- 下一步责任：xiaobao 侧先裁定 ai PM 的核对异议（参照源问题），成立则直接执行迁移建角色；ai 项目组并行推进 PRD Review → R2 定稿 → 设计阶段。**R-1/R-2 未就绪只阻塞实现阶段联调与冒烟，不阻塞设计阶段（契约 v1.1 + R-5 结构说明已足够），双侧不必互等。**契约变更须先改 `contracts/news-l1-db.md` 再改两侧代码，CHANGELOG 记一行。
- 完整往来见 [communications/REQ-003-db-boundary-async.md](communications/REQ-003-db-boundary-async.md)。

## 下一步汇总

1. ~~Owner：为 `niuma-cheng-ai` 配置 GitHub remote~~ —— 已完成（2026-06-21），`PROJECTS.md` 仓库地址已回填。
2. ai 项目会话：PM 已承接 REQ-001；Owner 确认后由 PM 创建 `v0.1-prd.md` 启动标准迭代，实现 news-l1 真实 L1 处理（stub→真实）。
3. 任一侧改 news-l1 契约：先改 `contracts/news-l1.md`，CHANGELOG 记一行。
4. xiaobao 项目会话：联调证据已齐，维持测试服务配置待命。
6. **xiaobao 项目会话（最高优先）**：① 裁定 ai PM 的核对异议（「契约 vs SQL 四处不一致」经逐行核对不成立，疑参照源为 xiaobao 侧 design 文档）② 成立则直接执行 `v0.6.1_ai_contract.sql` 建 `ai_worker` 角色 + GRANT + 强口令 ③ 补确认 `l1_processed_at`/`l1_attempt` 两列 ④ 交付凭据（四要素+口令，非整串 DSN）+ 造数脚本。见[本文件 §4](#req003-db-boundary) 与 [communications/REQ-003-db-boundary-async.md](communications/REQ-003-db-boundary-async.md)。
7. **ai 项目会话（2026-07-25 新增）**：v0.2 PRD R1 三方 Review（Architect/Developer/DevOps）；**O-1 已解除**（xiaobao 定案方案 A、契约 v1.1），三方齐后改 R2 定稿→进设计阶段。
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
