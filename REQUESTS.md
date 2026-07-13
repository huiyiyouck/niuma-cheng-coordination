# 跨项目需求提报中心

> 这是 niuma-cheng 生态的**需求池**。任一项目的任一角色都可以把跨项目需求提报到这里。
> **提报时不指定由哪个项目承接**——各项目的 PM/Architect 按本项目定位主动评估认领，Owner 也可直接指派。
> 承接之后才建立 `communications/{REQ-id}-{short-name}.md`，一份需求一份沟通文档，做协作与联调。
> 生命周期：已提报 → 评估中 → 已承接（/已拒绝）→ 开发中（转入迭代）→ 联调中 → 已关闭。
> 框架基线修正提案见下方「基线修正提案池（BCR）」；BCR 使用独立状态机，不走普通需求的联调生命周期。
> 规则见各项目 `docs/baseline/cross-project-collaboration.md` §跨项目需求流转。

## 需求池

| 需求 id | 提出方 | 内容 | 承接方 | 转入迭代 | 状态 | 沟通文档 |
|---------|--------|------|--------|----------|------|----------|
| REQ-001 | xiaobao · Developer | 新闻 L1 处理：四维原始评分 + 五类标签 + 摘要 + 翻译 + 按需工具调用 | ai · PM（ck） | ai v0.1（已关闭，2026-07-04） | 已关闭 | [communications/REQ-001-news-l1.md](communications/REQ-001-news-l1.md) |
| REQ-002 | xiaobao · Architect | AI 处理架构调研：从 Horizon / ai-news-aggregator 两个参考项目提炼 L0/L1 与 Agent Hub 设计输入，回答 4 个架构岔路口 | ai · PM（ck）承接登记，产出归 Architect | ai v0.1（前置，待启动） | 已承接 | [communications/REQ-002-arch-research.md](communications/REQ-002-arch-research.md) |
| REQ-003 | xiaobao · PM | v0.6.1 集成模式变更：news-l1 AI 解析从 HTTP 同步调用改为数据库契约边界异步解耦（ai 改轮询 worker 模式 + 适配层封装），翻译保留在 ai 侧 | 待 ai · PM 评估承接 | — | 已提报（2026-07-05 初版，2026-07-12 R2 更新） | [contracts/news-l1-db.md](contracts/news-l1-db.md) |

---

## REQ-001 · 新闻 L1 处理

- 提出方：xiaobao · Developer
- 内容：把 L1 推理从新闻平台主进程解耦，由独立服务承载 —— 产出四维**原始**评分（`score`+`reason`）、五类标签、摘要、翻译、按需工具调用（KB 检索 / 链接读取 / Web 搜索）
- 边界：承接方只产原始评分，加权 `score_total` 留 xiaobao
- 承接方：`ai`（AI 处理中枢）· PM（ck）
- 承接说明：本需求先于跨项目需求流转机制既成事实推进并打通管道；2026-06-22 `ai` 项目完成 Bootstrap 并立项后，由其 PM（ck）评估**正式承接**，补齐「正规提报（xiaobao · Developer）→ 承接（ai · PM）」留痕闭环
- 转入迭代：ai v0.1 标准迭代（已关闭，2026-07-04）—— 真实 L1 处理（评分/标签/摘要/翻译由 stub 转真实）已交付，迭代关闭检查通过
- 当前状态：已关闭（2026-07-04）—— 契约 [news-l1 v1](contracts/news-l1.md) 定稿，端到端 4 条成功用例（主链路+KB 双向+空结果），Owner 抽样验收通过，ai v0.1 迭代关闭检查通过。KB 空结果语义为已知遗留项（非阻塞，转入下一迭代）
- 联调记录：见 [communications/REQ-001-news-l1.md](communications/REQ-001-news-l1.md)

---

## REQ-002 · AI 处理架构调研（参考项目借鉴）

- 提出方：xiaobao · Architect
- 提报日期：2026-06-24
- 内容：调研两个外部参考项目的 AI 处理架构，为 `ai`（Agent Hub）把 REQ-001 的 L1 从 stub 转真实、以及未来 L0 分层提供设计输入。请 `ai` 项目组在此基础上继续深入调研并形成自己的架构方案。
- 承接方：`ai`（AI 处理中枢）· PM（ck）—— Owner 指派，2026-06-29 ai PM 评估正式承接（承接登记由 PM 做，架构方案实质产出归 Architect）
- 承接说明：承接背景为 Owner 2026-06-29 拍板 ai 定位升级为生态内部通用 AI 处理中枢（见 [decisions/0002](decisions/0002-ai-hub-ecosystem-positioning.md)）；REQ-002 调研范围相应在「news-l1 真实化架构」基础上增加「生态内部通用骨架」维度
- 转入迭代：作为 ai v0.1 的**前置架构调研** —— v0.1 PRD 待 REQ-002 架构结论后由 ai PM 创建
- 沟通文档：[communications/REQ-002-arch-research.md](communications/REQ-002-arch-research.md)
- 参考项目（本地）：
  - `Horizon`（`/root/Horizon`，Python + asyncio）—— 重度 LLM，与 Agent Hub 几乎同构，最高借鉴价值
  - `ai-news-aggregator`（`/root/ai-news-aggregator`，TS/Node）—— 轻度 LLM（纯规则过滤），一面「少用 LLM」的反向参照
- 关键文件指引：`/root/Horizon/src/ai/{analyzer,enricher,client,prompts}.py`、`/root/Horizon/src/mcp/README.md`、`/root/Horizon/docs/{scoring,horizon-hub-design}.md`；`/root/ai-news-aggregator/src/filters/{ai-related,dedupe}.ts`、`/root/ai-news-aggregator/TECH_SPEC.md`

### xiaobao·Architect 初步提炼的 7 个借鉴点

1. **两段式 LLM pipeline 分离**：Horizon `analyzer`(便宜模型批量打分) + `enricher`(过阈值后才贵处理)，印证 L0/L1 双模型；省钱靠**阈值闸门**而非模型选型。
2. **「伪 Agent」确定性编排 vs 真 Agent 自主循环**：`enricher` 的「按需工具调用」是写死三步（LLM 列概念 → 代码搜索 → 喂回 grounding），无 ReAct 自主循环。对 LangGraph 选型是关键反问。
3. **多 provider 客户端 + 链式 fallback**：`client.py`（`ChainedAIClient` 自动降级 + 大量 provider quirk）是整仓最值得移植的文件。
4. **MCP staged pipeline + 产物落盘可重入**：MCP 层不重实现业务，分阶段 tool + 阶段产物落盘可从中间重入；对 Agent Hub「状态真源」直接有用。
5. **优雅降级链**：enrich 失败→翻译兜底→不丢条目；analyze 失败→score 0，逐级降级而非丢弃。
6. **引用校验防幻觉**：`sources` 只保留真实出现在搜索结果里的 URL，过滤 LLM 编造链接。
7. **确定性预过滤放在 LLM 之前**：aggregator `ai-related.ts` 纯规则做 AI 相关性、`dedupe.ts` 确定性去重；反问 L0 是否必须用 LLM。

### 请 ai 项目组重点回答的 4 个架构岔路口

1. L1 第一版用**确定性 staged 编排**，还是**真 agent 自主工具调用**？（借鉴点 2）
2. L0 用 **LLM** 还是**规则预过滤 + LLM 兜底**？（借鉴点 7）
3. LLM 客户端封装：**移植 Horizon `client.py`**（含链式 fallback）还是自研？（借鉴点 3）
4. L1 子阶段产物是否**落盘可重入**，与现有 `tasks` 表调度如何合流？（借鉴点 4）

### 边界与衔接

- 本条是**调研/预研性质**输入，非契约变更；`news-l1` 契约真源仍以 [contracts/news-l1.md](contracts/news-l1.md) 为准，本调研结论若要改契约须另走契约流程。
- 与 REQ-001 的关系：REQ-001 是「L1 真实化」的交付需求，REQ-002 是其前置的架构调研；ai 承接后可自行决定二者合并到同一迭代还是分开。

---

## 基线修正提案池

> 本区登记 target = `agent-workflow` 的基线修正提案（BCR, Baseline Change Request）。
> BCR 由 Owner + agent-workflow 真源维护会话（General）评估、采纳和落地；下游项目只能提报，不直接修改本项目 `docs/baseline/`。
> 状态机：已提报 → 评估中 → 已采纳 / 部分采纳 / 已拒绝 / 转 v2 候选 → 已落地真源 → 回流中 → 已回流下游。

| BCR id | 提出方 | 摘要 | 影响范围 | 状态 | 真源评估记录 | 真源落地 commit | 回流清单 | 备注 |
|--------|--------|------|----------|------|--------------|----------------|----------|------|
| BCR-001 | agent-workflow · General（P8 自举） | 基线修正提案从 Owner 人肉带回改为 coordination BCR 池登记与追踪 | agent-workflow baseline；已接入工作流的下游项目同步规则 | 已回流下游 | Owner 已评审通过；agent-workflow PR #4 合并 P8 方案（`fe99ac3`） | `fc22e75`（merge 663f59b） | ai: 已回流（`c8c66ce`, 2026-06-22）；xiaobao: 已回流（`c8c66ce`, 2026-06-22）；workboard: 已回流（接入后随 `66c1e1a` sync，commit `4b8e563`, 2026-06-25） | 自举样例：先登记 BCR，再改真源 baseline；自举例外仅限本次 P8 |
| BCR-002 | xiaobao（对齐真源时工作区留痕，2026-06-22） | communications 沟通文档命名轴：按项目对 vs 按需求一份 | agent-workflow `cross-project-collaboration.md` §communications；回流后影响各下游同名章节 | 已回流下游 | 已采纳：采用「一份需求一份沟通文档」命名轴 | `b5a29a3`（merge `0a76dca`） | ai: 已回流（`1b01fba`, 2026-06-22）；xiaobao: 已回流（`91b442a`, 2026-06-22） | 已在 coordination 执行实体迁移：`xiaobao__ai.md` → `REQ-001-news-l1.md` |
| BCR-003 | workboard · PM（2026-06-24） | 项目元信息（定位/名称/技术栈/上线/接入状态）变更后，生态层真源（`PROJECTS.md` / 根 `/root/Project/CLAUDE.md` 索引 / BCR 回流清单接入状态）无同步跟踪机制 | agent-workflow `mechanisms.md` 迭代关闭检查清单 或 `cross-project-collaboration.md` PROJECTS.md 维护约定；下游同步 | 已回流下游 | 真源会话评估·已采纳（4 轮 review 定稿）：方向 + 执行协议已成草案 v2.1——元信息变更台账落 coordination `STATUS.md`、真源可疑时「上报 + 暂停」转交协议、根 `/root/Project/CLAUDE.md` 重设计；Owner 已通过 5 个拍板点（台账位置 STATUS.md / 台账字段 / 停机转交协议 / 根 CLAUDE.md 两节 + 「不算改子项目」例外 / PROJECTS.md 主从关系）。设计草案见 agent-workflow `docs/progress/ad-hoc/2026-06-25-root-claude-md-redesign.md`（v2.1 `1b07f26` + §九 `ced333b`，分支 `fix/source-repo-bcr-no-role-switch`，未合 main）。合并定稿 spec v4 `fec6135`；baseline 改动 1/1b/2 已落地分支 `b8c7c15`（自检过：双入口一致、sync 通过、回流产物已验）；已合 main（merge `66c1e1a` / PR #7）= 已落地真源；下游 baseline 已回流。**BCR-003 终态前置已满足**：改动 3/4/5 全部完成——STATUS.md 建「元信息变更台账」、PROJECTS.md workboard 订正（技术栈 React+shadcn、v0.1 已上线、接入 @66c1e1a）、根 `/root/Project/CLAUDE.md` 重设计；workboard 过期数据已订正并入台账留证 = 已回流下游（终态） | `66c1e1a`（merge PR #7） | ai `6b1c8b8` / xiaobao `2e41947` / workboard `4b8e563`（baseline 已回流）；生态侧改动 3/4/5 已落（STATUS 台账 + PROJECTS + 根索引） | 触发自 workboard v0.1 定位升级 + 技术栈变更；订正根索引时发现无跟踪机制；本仓 BCR-001 回流清单 workboard「未接入」亦已过期；现归 BCR-005 父框架（元信息同步 facet） |
| BCR-004 | workboard · PM（Owner 要求，2026-06-24） | 删除独立 UI（界面设计师）角色，职责并入 PM | agent-workflow `role-ui.md`（删）、`multi-agent-workflow.md`（角色集/Review 矩阵/UI 阶段）、`runtime.md`（角色路由）、入口 `CLAUDE.md`/`AGENTS.md`（UI 触发/反问）、`standard-iteration-quick.md`（UI 阶段）、模板 `ui-spec.md`；下游各项目 | 已回流下游 | 已采纳（设计 v3 经 3 轮 review + Owner 拍 8 点）并落地：UI 并入 PRD、UI 影响域 Review 改 PM、`ui-spec` 墓碑；合并 spec `docs/progress/ad-hoc/2026-06-25-role-slim-ui-tester.md`；baseline 合 main（PR #8，merge `6f433ca`），下游已回流 = 终态 | `6f433ca`（merge PR #8） | ai `504f7c3` / xiaobao `9bab45d` / workboard `d8e6b74` | workboard v0.1 已实践 PM 兼 UI（见 v0.1.md 角色说明）；落地由真源评估 |
| BCR-005 | agent-workflow · General（真源会话，Owner 驱动 2026-06-25） | 生态参与者拓扑 + 跨界写协议（直写 vs 转交）：把 coordination / 根索引 / 框架真源等非开发参与者系统定义进工作流，消除「Owner 人肉转述」缺口 | agent-workflow `cross-project-collaboration.md` 新增「生态参与者与跨界协议」节；下游同步 | 已回流下游 | 真源会话评估·已采纳（4 轮 review 定稿）：设计 v1，四类参与者（框架真源/协调真源/开发型下游/生态索引根）+ 跨界写协议三条 + 「谁能写什么」矩阵。设计草案见 agent-workflow `docs/progress/ad-hoc/2026-06-25-bcr-005-ecosystem-participant-topology.md`（`1045c9d`，分支 `fix/source-repo-bcr-no-role-switch`，未合 main）。合并定稿 spec v4 `fec6135`；baseline 已落地分支 `b8c7c15`（自检过）；已合 main（merge `66c1e1a` / PR #7）= 已落地真源；下游已回流 = 终态 | `66c1e1a`（merge PR #7） | ai `6b1c8b8` / xiaobao `2e41947` / workboard `4b8e563` | 父框架：收编 BCR-003（元信息同步）+ 根 CLAUDE.md 重设计 + 「授权直写 vs 人肉转述」缺口（回归用例 `748dc22`→`58de4eb`） |
| BCR-006 | agent-workflow · General（真源会话，Owner 驱动 2026-06-25） | 删除 Tester（测试工程师）角色：接口/自动化测试并入 Developer 自测、手动验收由 Owner、取消独立 Tester Review 门禁（Owner 验收升为强制关闭门禁） | agent-workflow `role-tester.md`（删）、`multi-agent-workflow.md`（角色集/流水线/Review 矩阵/关闭检查）、`runtime.md`、入口 `CLAUDE.md`/`AGENTS.md`、`mechanisms.md`、`standard-iteration-quick.md`、模板 `test-report.md`；下游各项目 | 已回流下游 | 已采纳（设计 v3 经 3 轮 review + Owner 拍 8 点）并落地：Tester 删除，接口/自动化测试并入 Developer 自测、手动验收由 Owner（强制关闭门禁）、验收/回归复核由 Architect 或 DevOps、`test-plan` 墓碑 / `test-report`→自测报告；baseline 合 main（PR #8，merge `6f433ca`），下游已回流 = 终态 | `6f433ca`（merge PR #8） | ai `504f7c3` / xiaobao `9bab45d` / workboard `d8e6b74` | 与 BCR-004（删 UI）同类「角色集精简」，合并一份方案/落地 spec |
| BCR-007 | agent-workflow · General（真源会话，Owner 驱动 2026-07-03） | 组织架构定位升级：生态参与者拓扑修订为「指挥官—参谋长制（薄公司）」——撤销 coordination 专职会话并入参谋长、参谋长获回流执行权、元信息同步从三方接力简化为两方接力 | agent-workflow `cross-project-collaboration.md` §生态参与者与跨界协议（四类参与者→三类+场所、字段级白名单矩阵、两方接力）、`README.md` 定位段升级；下游同步 | 已回流下游 | Owner 两轮 review 通过，方案 v3 已定稿；已采纳；落地按 spec §4 九步执行（新身份先于新权限生效）；spec 见 agent-workflow `docs/superpowers/specs/2026-07-03-org-positioning-upgrade-design.md`（commit `b108ea6`） | `4ec68ce`（merge，feat `2de5947`） | ai: 已回流（`2acc305`, 2026-07-03）；xiaobao: 已回流（`107d879`, 2026-07-03）；workboard: 已回流（`9cd17cf`, 2026-07-03） | 自举型，同 BCR-005 先例；矩阵连带修订：框架真源可登记自身元信息变更行 |
| BCR-008 | 参谋长（生态根会话）· Owner 授权代提，2026-07-04 | BCR-007 回流时手动 `cp` 复制 README.md 绕过 `sync-downstream.sh`，把三下游各自 README 覆盖为 agent-workflow 真源 README（脚本本身不同步 README） | 根治优先机制护栏（`sync-downstream.sh` 回流自检兜底，不加条文）；下游影响：xiaobao / ai / workboard 的 `README.md` | 已回流下游（终态） | 参谋长落地（框架维护权 BCR-009） | `3035216`（护栏） | 护栏不下发下游（真源侧工具）；README 恢复全完成：xiaobao `aa28883` / ai `1b2e699` / workboard `c25065b` | 证据：三下游 README 覆盖 commit = BCR-007 三回流 commit（xiaobao `107d879` / ai `2acc305` / workboard `9cd17cf`） |
| BCR-009 | 参谋长（生态根会话）· Owner 拍板，2026-07-04 | 组织定位微调：裁减独立 agent-workflow 维护角色，框架维护权 + 回流执行权归参谋长（身份挂靠、非目录合并）；回流定为半自动收尾，全自动留未来 | 参谋长职责扩含框架维护；agent-workflow 参与者「独立 General 会话」→「参谋长第二工作目录」；「谁能写什么」矩阵参谋长写权扩到 baseline/templates/入口；根 `CLAUDE.md` 参谋长职责表 | 已回流下游（终态） | 参谋长评估落地（自举，Owner 拍板） | `7f68b2e`（框架）+ `a9af637`（根） | ai `cbdea27` / xiaobao `7e97465` / workboard `e240db0` 全部已回流 | 自举型，同 BCR-007；**前置依赖 BCR-008 护栏先行**；全自动留未来的依据 = 组织法则三「半自动是设计选择」 |
| BCR-010 | 参谋长（生态根会话）· Owner 拍板，2026-07-04 | 跨项目定位禁相对路径→绝对路径写死→换机器/上生产即失效（xiaobao/ai 的 `coordination_root`=`/root/Project/...` 已失效）；放开为「显式相对路径（项目根为基准），仍禁无配置猜测」 | agent-workflow `cross-project-collaboration.md` §coordination 发现机制；下游各项目 `coordination_root` 改相对 | 已回流下游（终态） | 参谋长落地（框架维护权 BCR-009） | `1d1f935` | baseline: ai `cbdea27` / xiaobao `7e97465` / workboard `e240db0`；`coordination_root` 改相对：xiaobao `aa28883` / ai `1b2e699`，workboard 无需 | 下游修复：xiaobao/ai 的 `coordination_root` 改 `../niuma-cheng-coordination`（归各项目会话）；workboard 已用相对路径未坏 |
| BCR-011 | workboard · Developer（2026-07-05） | 迭代记录「阶段状态」无标准词表，门禁表结构约定缺失：各项目历史记录写法各异（已定稿 / 部署通过 / 已Review / ✅ 验收通过 / 已跳过…），且有非阶段性内容（如「UI 增强方向」）以三级标题混入阶段门禁区——下游工具（workboard 迭代时间轴）与流程审计只能读取侧逐版本兼容。建议：① `iteration.md` 模板定义封闭状态枚举（如：未开始 / 进行中 / Review中 / 待修正 / 已定稿 / 已完成 / 已跳过）并注明门禁表最新轮次置末行；② 约定「## 阶段门禁」下三级标题只放标准阶段，停放性记录（讨论记录 / 未来方向）移出门禁区或走 ad-hoc | agent-workflow `docs/templates/iteration.md`；或另涉 `standard-iteration-quick.md` 门禁表述；回流后约束各下游**新**迭代记录写法 | 已回流下游 | 参谋长评估：采纳。采用既有 baseline 阶段门禁词表 `待Review / Review中 / 修改中 / 已定稿 / 阻塞 / 已跳过`，不采用提案中跨层级的 `未开始/进行中/已完成`；在 `iteration.md` 模板补状态枚举、部署检查独立枚举、最新轮次置末行、门禁区只放标准阶段。方案游标：agent-workflow `docs/progress/ad-hoc/2026-07-08-bcr-011-012-iteration-gate-spec.md` | `39e7a35` | ai `a53b680` / xiaobao `4dbbddb` / workboard `ad9b706`（2026-07-08） | 与 BCR-012 合并同轮落地；历史迭代记录不回改，仅约束新迭代记录 |
| BCR-012 | workboard · Developer（2026-07-07） | `iteration.md` 模板未随 BCR-004（删 UI 阶段）/ BCR-006（删测试阶段）同步收敛「阶段门禁」的阶段字段集——模板与历史记录里仍保留「测试阶段」「UI 方案阶段」小节，与当前标准流水线（PRD → 设计 → 实现 → 部署就绪检查 → 迭代关闭）不符，下游工具（workboard 时间轴）只能在读取侧把它们降级为「附加记录」兼容。建议：`iteration.md` 模板固定当前标准阶段字段集，**显式删除「测试阶段」「UI 方案阶段」小节**（或保留但字段置空 + 注释「BCR-006 已并入 Developer 自测 / BCR-004 已并入 PRD」），使新迭代记录门禁结构统一、下游可确定性解析、无需逐版本兼容 | agent-workflow `docs/templates/iteration.md`（`## 阶段门禁` 阶段小节）；连带 `standard-iteration-quick.md` 阶段列举；回流后约束下游**新**迭代记录写法 | 已回流下游 | 参谋长评估：部分采纳。当前 `iteration.md` 已无「测试阶段 / UI 方案阶段」小节，原问题描述在现行 main 不复现；采纳其“固定标准阶段字段集”方向，并入 BCR-011 同轮落地，同时清理 `role-architect.md` 中残留 UI 阶段检查口径。方案游标：agent-workflow `docs/progress/ad-hoc/2026-07-08-bcr-011-012-iteration-gate-spec.md` | `39e7a35` | ai `a53b680` / xiaobao `4dbbddb` / workboard `ad9b706`（2026-07-08） | 与 BCR-011 合并同轮落地；历史迭代记录不回改；采纳性质为部分采纳 |
| BCR-013 | workboard · PM（2026-07-12） | 工作流会话未携带结构化「角色+迭代」标识，下游 workboard 精准匹配（会话↔角色↔迭代）只能靠启发式猜（首条「你是XX」+关键词加权，见 `session-meta.js`），信号稀疏（workboard Developer 预研：`vX.Y` 会话开头仅 3%、标题 0 次、全文 71% 但无法定位），`detected_role` 不稳、迭代归属几乎无信号，准确度天花板卡在会话源头 | agent-workflow 角色启动约定（`runtime.md`/入口/`role-*.md` 启动检查）或收尾机制（`mechanisms.md`）；下游 workboard 消费 | 评估关闭·不进框架（转 workboard 解析侧） | 参谋长初评经 Owner review 修订：建议**先走第 0 档**（workboard 纯解析侧扫已入库 assistant 消息、改进启发式探底，零框架改动——缺口未证实前不给框架加永久税）；若缺口显著再进框架，排序 **D 主力（落盘/审计原生）> C 待验证（启动写映射行+时间窗匹配，需验并发消歧）> B 增量（如实标注「高置信度启发式，非 1.0」）**。纠正初版硬伤：B 非确定性 1.0（信号源是 LLM 自述非用户输入）、C 不该枪毙（不依赖 session id，用时间戳）、D 不该贬辅。token 契约须含抗误命中（独占行强定界如 `<!-- wf-meta:… -->`）+ 确定性可正则。收尾级已收敛（Owner 二/三轮 review）：决策点分层（1 前置门，2–4 挂其下，5 可并行）；第 0 档收敛判据**拆角色/迭代两维各设门槛各判进框架**（不用复合"且"——迭代维信号存量近零会拖累够用的角色维）；很可能角色维交解析侧、迭代维进框架，则框架 marker 只承载迭代锚点、改动更小；并明确「D 服务审计不覆盖实时段，提报方实时诉求由第 0 档解析侧满足，BCR 关闭须写清归属」。**Owner 拍板（2026-07-12，采纳参谋长建议）：前置门=先走第 0 档探底、token 契约落点=coordination `contracts/`；决策点 ②B定性/③排序/④C复活 暂挂待探底数据。第 0 档归 workboard 解析侧迭代执行（改进 detectRole 扫已入库 assistant 消息，角色维报 Unknown 率、迭代维报迭代命中率，各对门槛[初值待定：角色维 Unknown ≤10% / 迭代维命中 ≥80%]各判是否进框架），非框架不占本 BCR 落地；参谋长待其数据回来复评。** **参谋长只读基线摸底（2026-07-12，59 会话）：角色维改进前 Unknown 头条 27.1% 但开发会话口径仅 ~7.5%（xiaobao 1/25、workboard 0/5，非开发会话本就应 Unknown）→ 近够用、大概率交解析侧；迭代维 vX.Y 首条 1.7%/全文 66.1%（合预研）→ 源头信号近零、大概率进框架。强烈印证两维分裂，方案结构经数据验证无需改，仅收紧角色维门槛口径（在开发会话上量）。Owner 可二选一：跑正式第0档确认 / 直接采信基线拍板分裂进迭代维 D/C。** **Owner 采信基线、按参谋长推荐拍板（2026-07-12，部分采纳）：角色维交 workboard 解析侧（不进框架、跳过正式第0档）；迭代维进框架。参谋长荐用方案 C（会话启动写"项目+角色+迭代+时间戳"映射行，覆盖进行中会话、直接解决 session↔迭代绑定；窄化到仅迭代维后 C 比总排序的 D 更贴合）。**角色维收尾（Owner 定）：detectRole 兜底 Unknown→General（无角色即 General，框架默认语义）→ 角色维零框架、无孤儿。** **参谋长补充（付框架税前 2 点）：① C 要真确定须靠 SessionStart hook（LLM 启动自写=同 B 的不确定）；② 迭代维可能免框架——会话时间戳×项目 INDEX.md git 历史可重建 session↔迭代（免疫会话不提 vX.Y、零永久税），建议先只读探这条解析侧路，达标则整条 BCR-013 零框架。契约落点改框架 baseline（参谋长黑名单不碰 contracts/）。** **迭代维解析侧探底结果（2026-07-12，只读59会话）：INDEX-git-history 重建 session↔迭代，开发角色会话命中 62.8%（27/43），未命中多为"当时无迭代/老会话"非算错；关键——框架 C 只能标未来新会话、这批历史积压一个都标不了→解析侧绕不开。参谋长改荐：迭代维也走解析侧→整条 BCR-013 零框架/零永久税/免回流，C 降备选（仅解析侧不足才上且须 hook）。待 Owner 确认翻转 C 采纳；确认后 BCR-013=评估后不进框架、转 workboard 解析侧治理（兜底 General + INDEX-git-history）。** **Owner 确认关闭（2026-07-12）：BCR-013 评估关闭·不进框架——两维转 workboard 解析侧治理（角色维 detectRole 兜底=General；迭代维 INDEX-git-history 重建），框架侧零改动、免回流、零永久税；C+hook 留档备选，仅解析侧将来证明不足才重启。** 方案游标：agent-workflow `docs/progress/ad-hoc/2026-07-12-bcr-013-session-role-iteration-marker.md` | — | — | 治本 workboard「会话↔角色↔迭代」精准匹配；workboard v0.2.1 做治标（软匹配：同角色置顶+可手选），本 BCR 治本留 workboard v0.3 收割；详情见下 |
| BCR-014 | workboard · PM（2026-07-13） | 入口「你是 {角色}」角色切换机制无护栏，允许「产出方会话切成 Review 方审自己的产出」= 自审，架空「两方独立 Review 防自审」门禁——纯靠执行者自律，自律会失效 | agent-workflow 入口 `CLAUDE.md`/`AGENTS.md` 角色切换触发；`multi-agent-workflow.md`/`runtime.md` Review 机制；下游各项目 | 已提报 | 待 Owner + 参谋长评估 | — | — | 实例触发：workboard v0.3 R1，PM 写完 PRD 后同会话切 Architect 审自己，Owner 当场拦下。建议护栏：切换到 Review 角色前检查当前会话是否为被审产出的产出方会话，若是则拒绝/警示 + 提示新开会话冷启动。详情见下 |

### BCR-001 · 基线修正提案走 coordination 管理

- 目标：把下游 `[基线修正提案]` 从口头带回真源，改为写入 coordination 的 BCR 池，形成有状态、可追溯、可回流的治理流程。
- 采纳依据：agent-workflow P8 方案已通过 PR #4 合并（`fe99ac3`）。
- 下一步：已闭环。真源已落地（`fc22e75`，merge 663f59b，PR #5）；xiaobao、ai 均已 sync 回流至 `c8c66ce`（2026-06-22），workboard 未接入不适用——全部已接入下游回流完成，BCR-001 置「已回流下游」终态。

---

## BCR-002 · communications 命名轴（按项目对 vs 按需求）

- 提出方：xiaobao —— 对齐真源（P7 第 3 步）时，工作区有一处未提交改动把 `communications/` 命名轴从「按项目对」翻成「按需求一份」，覆盖前抢救留痕。
- 目标：评议是否调整 `cross-project-collaboration.md` §communications 的命名轴。
- 背景：真源现状为**按项目对** `communications/{a}__{b}.md`（一对项目共用一份，承载该对所有需求）；xiaobao 提案为**按需求** `communications/{REQ-id}-{短名}.md`（一个需求一份）。该改动带的「创建时机 / 职责分工」措辞与真源新版一致，说明是看过真源新思路后有意翻转命名轴，非旧版残留。
- 评估结论：已采纳；真源已落地；已接入下游均已回流，状态置为「已回流下游」。
- 真源落地：agent-workflow commit `b5a29a3`（merge `0a76dca`）。
- coordination 实体迁移：`communications/xiaobao__ai.md` → `communications/REQ-001-news-l1.md`，并同步更新 `REQUESTS.md` / `PROJECTS.md` / `communications/README.md`。

| 维度 | 按项目对（真源现状） | 按需求（xiaobao 提案） |
|------|---------------------|----------------------|
| 命名 | `{a}__{b}.md` | `{REQ-id}-{短名}.md` |
| 粒度 | 一对项目共用一份 | 一个需求一份 |
| 文件数 | 有界（最多 C(N,2) 对） | 随需求量线性增长 |
| 与 REQUESTS.md | 一份多 REQ，靠条目标注区分 | 与需求 id 一一对应，反查直达 |
| 长期可读性 | 单份越积越长、跨需求混杂 | 需求关闭即归档，边界清晰 |

- 回流进展：ai 已回流至 `agent-workflow@1b01fba`（2026-06-22）；xiaobao 已回流至 `agent-workflow@1b01fba`（commit `91b442a`，2026-06-22）。
- 下一步：已闭环；后续新增下游接入时按当前真源版本安装即可。

---

## BCR-003 · 生态层项目元信息同步机制缺失

- 提出方：workboard · PM（2026-06-24）
- 问题：项目在迭代中**定位 / 名称 / 技术栈 / 上线状态 / 工作流接入状态**发生变更后，没有机制保证生态层真源被同步订正，导致以下真源易过期、无人跟踪：
  - coordination `PROJECTS.md`（项目目录：当前入口、技术栈、状态）
  - 根 `/root/Project/CLAUDE.md` 生态索引（定位、技术栈、状态、已知不一致）
  - 本仓 BCR 回流清单中各项目的「接入 / 未接入」状态
- 实例（本次触发）：
  - workboard v0.1 定位升级为「项目管理工作台 / 统一管理中枢」、技术栈由「静态 HTML/CSS/JS」改为「React + shadcn」、已上线 `workboard.huiyiyou.cloud`，但根索引仍写「跨项目 Agent 工作看板 / 本地 Node + 静态前端 / 待开发 MVP」。
  - workboard 已接入 agent-workflow，但 BCR-001 回流清单仍标 workboard「不适用（未接入）」。
  - 根索引「已知不一致」里 `../claude-workflow` 路径条目已修复但未清理。
- 拟改方向（供真源评估，非定稿）：
  - 方案 A：在 `mechanisms.md` 迭代关闭检查清单**新增一项**——「核对本迭代是否变更项目定位 / 名称 / 技术栈 / 上线 / 接入状态；若是，登记同步 coordination `PROJECTS.md` 并提示 Owner 订正根索引」。
  - 方案 B：在 `cross-project-collaboration.md` 规定「项目元信息变更时由该项目同步 coordination `PROJECTS.md` 对应行」，并明确根 `/root/Project/CLAUDE.md` 的维护责任归属（根目录会话）。
- 不在下游落地：本提案仅提报，落地由 Owner + agent-workflow 真源会话评估。

---

## BCR-004 · 删除 / 合并 UI（界面设计师）角色到 PM

- 提出方：workboard · PM（Owner 明确要求，2026-06-24）
- 提案：删除独立的 **UI（界面设计师）角色**，其职责并入 **PM（产品经理）**。
- 理由：
  - 一人公司 + AI 出原型（Figma Make / Claude Design）模式下，UI 与 PM 高度重叠；页面结构 / 信息架构本就贴近 PM 的需求与流程定义。
  - workboard v0.1 已实际以「PM 兼任 UI」跑通整条迭代（UI 方案 + 原型 + 数据规格均 PM 产出），见 `niuma-cheng-workboard/docs/progress/iterations/v0.1.md` 角色说明与 `v0.1-ui.md`。
- 拟改范围（供真源评估，非定稿）：
  - `role-ui.md`：删除；核心方法（用户流程映射 / 交互状态枚举 / 视觉约束 / UI 验收）择要并入 `role-pm.md`。
  - `multi-agent-workflow.md`：角色集移除 UI；Review 矩阵「流程 / 视觉」影响领域改由 PM 承担；标准迭代「UI 方案阶段」并入 PRD 或作为 PM 可选阶段。
  - `runtime.md`：角色加载路由移除 UI。
  - 入口 `CLAUDE.md` / `AGENTS.md`：角色切换触发表移除「UI / 界面设计师」精准触发与模糊反问（或将「UI / 界面 / 设计」关键词指向 PM）。
  - `standard-iteration-quick.md`：流水线 UI 阶段处理。
  - 模板 `ui-spec.md`：保留为 PM 可选产出，或并入 PRD 模板。
- 风险提示（供评估）：xiaobao 历史迭代有独立 UI 角色产出与日志（`roles/ui.md`、`vX.Y-ui-spec.md`），删角色后历史产物的归属与后续维护需在真源方案中明确（保留为历史 / 转 PM 名下）。
- 不在下游落地：本提案仅提报，落地由 Owner + agent-workflow 真源会话评估。

---

## BCR-005 · 生态参与者拓扑 + 跨界协议（父框架）

- 提出方：agent-workflow · General（真源会话，Owner 驱动，2026-06-25）；自举型，同 BCR-001。
- 问题：coordination / 根索引 / 框架真源等**非开发参与者**一直游离定义，"谁能改谁"靠临场判断，退化成 Owner 人肉转述（回归用例：BCR-003 评估记录转录走样 `748dc22`→`58de4eb`）。
- 方案（v1，详见设计草案）：在 agent-workflow 产品层系统定义——
  - **四类参与者**：框架真源(agent-workflow) / 协调真源台账(coordination) / 开发型下游(xiaobao·ai·workboard) / 生态索引根(`/root/Project/CLAUDE.md`)，各钉性质·边界·机制职责·跨仓写权限。
  - **跨界写协议**：①授权直写优先（有权威 + [P0] 允许→直写，不经 Owner）②转交仅限跨权限边界 ③转交走「读文件 / 精确制品」，禁止 Owner retype 状态真源。
  - **「谁能写什么」矩阵**。
- 收编：BCR-003（元信息同步=本拓扑「元信息变更」数据流机制）、根 CLAUDE.md 重设计（=「生态索引根」节点落地）、直写vs转述（=§三协议）；三者各保留自己游标节奏，定位归本 BCR。
- 不装门禁：coordination 仍不接 dev 工作流（门禁空转），只被定义为一等参与者。
- 设计草案：agent-workflow `docs/progress/ad-hoc/2026-06-25-bcr-005-ecosystem-participant-topology.md`（`1045c9d`）。
- 落地由 Owner + agent-workflow 真源会话评估。

---

## BCR-006 · 删除 Tester（测试工程师）角色

- 提出方：agent-workflow · General（真源会话，Owner 驱动，2026-06-25）。
- 提案：删除独立 Tester 角色，6 角色 → 4 角色（PM/Architect/Developer/DevOps）。与 BCR-004（删 UI）同属「角色集精简」，合并一份方案与落地 spec。
- 理由（Owner）：Tester 只能调接口测试、没法代人手动点击，与 Developer 自测高度重叠；后续接口/自动化测试由 Developer 自测覆盖，手动点击验收由 Owner 完成、提 bug 给 Developer 修；独立 Tester 角色空转。
- 职责再分配：接口/自动化测试 → Developer 自测；手动验收 → Owner；Tester Review 门禁取消 → **Owner 验收升为强制关闭门禁** + Developer 自测纪律强化。
- 风险（供评估）：移除独立质量门禁——缓解见设计 v1 §五（Owner 验收强制 + 自测证据必留）；历史 Tester 产物保留存档不追溯。
- 设计草案 v1（与 BCR-004 合并）：agent-workflow `docs/progress/ad-hoc/2026-06-25-role-slim-ui-tester.md`。
- 落地由 Owner + agent-workflow 真源会话评估；精确到行 spec 待 review 通过后出。

---

## BCR-007 · 组织架构定位升级（指挥官—参谋长制）

- 提出方：agent-workflow · General（真源会话，Owner 驱动，2026-07-03）；自举型，同 BCR-005。
- 目标：将「一人公司 AI 开发团队工作流」升级为「一人公司 AI 组织操作架构：指挥官—参谋长制（薄公司）」。
- 核心变更：
  1. 撤销 coordination 专职会话，维护职责并入参谋长（生态根会话升格）
  2. 参谋长获得第二个跨界例外：工作流回流执行权（sync 脚本）
  3. 生态根会话身份重写为「参谋长」
- 影响范围：
  - 生态参与者从四类（框架真源/协调真源台账/开发型下游/生态索引根）→ 三类参与者 + 一个场所（框架真源/参谋长/开发型下游 + 公告板）
  - 「谁能写什么」矩阵：参谋长对协调台账从「只读+回执」→ 字段级白名单直写
  - 元信息同步从「三方接力」→「两方接力」（子项目登记 → 参谋长一站式）
  - 框架真源写权扩为「BCR 池 + 登记自身元信息变更行」
- 方案文档：agent-workflow `docs/superpowers/specs/2026-07-03-org-positioning-upgrade-design.md`（v3，commit `b108ea6`）
- 落地步骤：spec §4 九步执行，排序原则「新身份先于新权限生效」
- 评估结论：已采纳；已落地真源（commit `4ec68ce`，feat `2de5947`）；已回流下游（终态）
- 真源落地内容：
  - `cross-project-collaboration.md` §生态参与者与跨界协议：四类→三类+场所、字段级白名单矩阵、两方接力
  - `README.md`：加内部架构定位段（指挥官—参谋长制·薄公司）
- 回流范围：ai / xiaobao / workboard 三下游（均已接入 agent-workflow）
- 回流完成：ai `2acc305` / xiaobao `107d879` / workboard `9cd17cf`（2026-07-03），BCR-007 置「已回流下游」终态

---

## BCR-008 · 回流误带 README（手动 cp 绕过 sync 脚本）

- 提出方：参谋长（生态根会话）· Owner 授权代提，2026-07-04
- 现象：三个下游 `xiaobao` / `ai` / `workboard` 的 `README.md` 被整体覆盖为 agent-workflow 真源的 README（一字不差，且保留了「本仓库性质 / 真源仓库 / single source of truth」段），各自项目自述丢失。
- 根因（提报方观察，待真源确认）：
  - `sync-downstream.sh` **不同步 README.md** —— 框架文件仅 `CLAUDE.md`/`AGENTS.md`（对入口还 `strip_block` 剥离 SOURCE-REPO-ONLY）+ `docs/baseline/`（除 `project-context.md`）+ `docs/templates/`；脚本另有「目标工作区不干净则拒绝同步」保护。
  - 但三下游 README 被**原样整体覆盖、SOURCE-REPO-ONLY 段完整保留**（若经脚本必被 strip）→ 不可能是脚本所为，只能是 BCR-007 回流时**手动 `cp` 复制**了 README（BCR-007 影响域含「agent-workflow `README.md` 定位段升级」，操作者顺手把 README 也带到了下游），**绕过了 `sync-downstream.sh`**。
  - 违反纪律：参谋长职责③「只准经 `sync-downstream.sh` 同步框架文件，**禁止手改下游任何文件**」。
- 证据：三下游 README 覆盖 commit = BCR-007 三个回流 commit（xiaobao `107d879` / ai `2acc305` / workboard `9cd17cf`）；coordination README 未被波及（本仓不在 sync 回流对象内）。
- 双侧修复（一条 BCR 覆盖）：
  - **根治侧（agent-workflow 真源，供真源评估，非定稿）**：
    - 取向（Owner）：**规则宜精简，优先机制护栏、不新增纪律条文**。已有边界「回流只经 `sync-downstream.sh`、禁止手改下游」本身不加码，改用机制落实。
    - 首选（护栏）：给 `sync-downstream.sh` 加回流后自检 —— 本次 sync 改动的下游文件必须都在框架白名单（`CLAUDE.md`/`AGENTS.md` + `docs/baseline` + `docs/templates`）内，越界（如 README / 项目专属文件）即告警拦下。让脚本兜底，而不是靠人记纪律。
    - 兜底备选（仅护栏不足时再考虑）：`docs/regression-cases.md` 补一条回归用例；能用护栏解决就不加条文。
  - **影响修复侧（各下游恢复自己的 README，挂本 BCR 修复清单）**：
    - xiaobao：`git checkout 635d558 -- README.md`
    - ai：`git checkout 0ee6c9a -- README.md`
    - workboard：`git checkout b5b18d9 -- README.md`
    - 承接：由各项目 **PM 承接修改**（Owner 指派），恢复后随下方修复清单勾销。
- 落地分工：根治侧由 agent-workflow 会话改真源、改完 sync 回流（Owner 安排）；影响修复侧由各项目 PM 承接恢复。参谋长（本仓）只负责本 BCR 登记/铺写与后续回流状态推进。
- 修复清单（承接后回填 commit）：
  - [x] xiaobao README 恢复 `aa28883`
  - [x] ai README 恢复 `1b2e699`
  - [x] workboard README 恢复 `c25065b`
  - [x] agent-workflow 回流护栏落地 `3035216`（真源侧工具，不下发下游）

---

## BCR-009 · 框架维护权与回流权归参谋长（裁减独立 workflow 维护角色）

- 提出方：参谋长（生态根会话）· Owner 拍板 2026-07-04；自举型，同 BCR-005/007。
- 背景：BCR-007 刚把回流执行权给参谋长；本 BCR 顺势把 agent-workflow 的**框架维护**也收归参谋长——让「把控部门的人」一手持有制度制定权（SOP 真源），并让「改框架的人顺手回流」少一环。设计经 Owner × 参谋长碰撞定案。
- 核心变更：
  1. **裁角色 · 身份挂靠**（非目录合并）：取消「独立 agent-workflow 维护会话（General）」，框架维护身份改挂**参谋长**。参谋长 `cd` 进 agent-workflow 目录即以**参谋长身份**维护 `docs/baseline`/`docs/templates`/入口、在该目录 commit。**守住「结构跟着物理走」**：仍在 agent-workflow 目录操作，不在生态根跨仓手写——是「一个头衔、两个目录」，非「一个会话塞两个仓」。
  2. **回流半自动收尾**：参谋长改完 baseline 合 main 后，收尾接着跑 `sync-downstream.sh` + 逐仓 commit/push + 回填 BCR 回流清单（改的人顺手回流）。
  3. **全自动留未来**：CI/hook 全自动回流挂为未来升级项，依据组织法则三「半自动是设计选择而非缺陷，全自动留下次定位升级」；升级前置 = BCR-008 护栏跑稳 + 下游数量增长。
- 影响 / 待落地：
  - 参谋长职责：从「协调＋立项＋回流执行＋索引」扩为「＋ 框架真源维护」；由「薄协调层」变「协调＋框架研发」一肩挑（**Owner 已认领此代价**，正向裁剪）。
  - agent-workflow 参与者定义：「独立 General 维护会话」→「参谋长第二工作目录」。
  - BCR 承接：下游 BCR 提报后由参谋长评估＋改真源＋回流（原「真源会话评估」并入参谋长）；含 BCR-008 护栏落地随之归参谋长。
  - 待改文件：agent-workflow `docs/baseline/cross-project-collaboration.md`（参与者拓扑 /「谁能写什么」矩阵）、`README.md` 定位段、agent-workflow `CLAUDE.md` 自述（真源会话→参谋长）、根 `CLAUDE.md` 参谋长职责表。
- 前置依赖：**BCR-008 回流护栏先落地**（半自动/自动回流前，护栏兜底防错误扩散）。
- 自举落地排序：同 BCR-007「新身份先于新权限生效」——先在文档确立参谋长的框架维护身份，再据此由参谋长落地其余改动。
- 状态：已提报，待 Owner 定落地节奏（现在落地 or 先留提案）；本 BCR 自举，落地由参谋长承接。

---

## BCR-010 · 跨项目定位改显式相对路径（禁绝对写死）

- 提出方：参谋长（生态根会话）· Owner 拍板 2026-07-04。
- 现象：xiaobao / ai 的 `docs/baseline/project-context.md` `coordination_root` 写死绝对路径 `/root/Project/niuma-cheng-coordination`，而实际树已在 `/Users/ck/Project/niuma-cheng/`——路径失效；workboard 用相对 `../niuma-cheng-coordination` 未坏。**印证绝对路径不可移植、整棵树搬家即失效**。
- 根因两层：
  1. 配置层：xiaobao/ai 的 `coordination_root` 写死绝对路径；
  2. 规则层：baseline §「coordination 仓的发现」明文**禁止** sibling/相对路径（理由「换机器即错」），把可移植的相对路径也堵死了——该理由只对「非统一树」成立，对 niuma-cheng（`setup.sh` 强制兄弟树）恰恰相反。
- 方案（Owner 拍板 · 显式相对，非自动推导）：
  - baseline 放开：`coordination_root` 允许绝对 / URL / **相对项目根的相对路径**；统一兄弟树生态优先相对；相对以项目根（`git rev-parse --show-toplevel`）为基准。
  - **仍禁无配置的目录猜测**（显式相对 ≠ 运行时扫上级找同级）——保住「不靠猜」。
- 双侧修复：
  - **根治（参谋长落地）**：改 `cross-project-collaboration.md` §发现机制 + 随 BCR-009 回流各下游。
  - **影响修复（各项目会话）**：xiaobao / ai 的 `coordination_root` 改 `../niuma-cheng-coordination`；workboard 已合规。
- 落地：参谋长以框架维护权（BCR-009）评估落地 baseline；下游配置改归各项目会话，挂本 BCR 修复清单。
- 修复清单：
  - [x] baseline 发现机制放开 + 回流三下游（`cbdea27` / `7e97465` / `e240db0`）
  - [x] xiaobao `coordination_root` 改相对 `aa28883`
  - [x] ai `coordination_root` 改相对 `1b2e699`
---

## BCR-013 · 工作流会话自带「角色 + 迭代」结构化标识

- 提出方：workboard · PM（2026-07-12）
- 问题：workboard 要做「会话 ↔ 角色 ↔ 迭代」精准匹配，但工作流会话**本身不携带可靠的结构化标识**。现状 workboard 侧只能靠启发式推断（首条「你是 XX」精确匹配 + 前 5 条关键词加权，见 `niuma-cheng-workboard/src/server/sync/session-meta.js`），信号稀疏——workboard Developer 预研实测：靠会话开头提 `vX.Y` 仅 **3%**（4/115）、标题 **0 次**、全文任意处出现 71%（82/115）但无法定位归属。导致：
  - `detected_role`（角色识别）不稳，跨角色误判；
  - 迭代归属几乎无可靠信号，「某会话属于哪个迭代」无法确定；
  - **准确度天花板卡在会话源头**，workboard 侧再优化算法也是治标。
- 期望（方向供 Owner + 参谋长评估，不定死方案）：让工作流在会话生命周期注入可被下游可靠读取的「角色 + 迭代」标识。候选方向：
  1. 约定角色切换**开场白结构化格式**（如统一首句「你是 {角色}，当前迭代 {vX.Y}」），下游可稳定解析；
  2. 或在项目 `docs/progress/` 登记「会话 id ↔ 角色 ↔ 迭代」**映射真源**（启动/收尾机制写入），下游读真源而非猜。
- 理由：治本 workboard 精准匹配（支撑「角色 Tab 只放该角色会话」等强匹配能力）；同时利于流程审计与跨项目会话追溯。
- 影响范围：agent-workflow 角色启动约定（`runtime.md` / 入口 `CLAUDE.md`·`AGENTS.md` / `role-*.md` 启动检查）或收尾机制（`mechanisms.md`）；回流后影响各下游会话行为；消费方 workboard。
- 与 workboard v0.2.1 的关系：v0.2.1 做**治标**——`detected_role` 算法优化 + 页内选会话「同角色置顶 + 可手选」软匹配（不硬限制，人工兜底）；本 BCR **治本**，落地后 workboard v0.3 再上强匹配 / 强限制。
- 落地：本提案仅提报，评估 · 采纳 · 落地由 Owner + 参谋长（框架维护权 BCR-009）。
---

## BCR-014 · 角色切换机制无护栏防「产出方自审」

- 提出方：workboard · PM（2026-07-13）
- 问题：入口「你是 {角色}」的角色切换机制，技术上允许在**同一会话**从产出方角色切成 Review 方角色。于是「产出方写完产出、同会话切成 Review 方审自己」成为可能——本质是**自审**：Review 方带着产出方的全部上下文与倾向，无法独立客观，且污染会话，**架空了「两方独立 Review 防自审」门禁**。整条规则**只靠执行者自律**，而自律会失效。
- 实例触发：workboard v0.3 R1 —— PM 写完 v0.3 PRD 后，Owner 一句「你是架构」，AI 在同一会话直接从 PM 切成 Architect 去审自己刚写的 PRD；Owner 当场识破拦下。
- 建议方向（供 Owner + 参谋长评估，不定死）：框架层加护栏——切换到 Review 角色（被指定为某产出的 Review 方）前，检查当前会话是否为该产出的**产出方会话**；若是则**拒绝切换 / 强警示**并提示新开独立会话冷启动做 Review。可落在入口 `CLAUDE.md`/`AGENTS.md` 角色切换触发，或 `multi-agent-workflow.md` Review 机制约定。
- 理由：独立 Review 的前提是审的人不带产出方上下文；靠自律守的门迟早破，需机制护栏兜底。
- 影响范围：agent-workflow 入口 `CLAUDE.md`/`AGENTS.md`、`runtime.md` / `multi-agent-workflow.md` Review 机制；回流后影响各下游角色切换行为。
- 落地：本提案仅提报，评估 · 采纳 · 落地由 Owner + 参谋长（框架维护权 BCR-009）。
---

## REQ-003 · v0.6.1 集成模式变更（数据库契约边界异步解耦）

- 提出方：xiaobao · PM
- 提报日期：2026-07-05（初版）；2026-07-12（R2 更新）
- 关联迭代：xiaobao v0.6.1（PRD R2 已定稿，设计 R2 PM 复审通过）
- 当前状态：已提报，待 ai · PM 评估承接
- 数据库边界契约：[contracts/news-l1-db.md](contracts/news-l1-db.md) v1

### 背景

xiaobao v0.6 上线了 AI 处理能力（默认关闭），采用 HTTP 同步调用模式（契约 [news-l1 v1](contracts/news-l1.md)），单条处理耗时 74~79 秒，调用方需挂起等待，且重试、超时、解耦能力弱。

v0.6.1 决定：**news-l1 AI 解析集成模式从 HTTP 同步调用改为数据库契约边界异步解耦**。xiaobao 入库标记状态 → ai 轮询取数处理 → 写回数据库。翻译职责保留在 ai 侧（与 REQ-001 一致），不剥离。

### 变更内容（请 ai 承接的事项）

#### 变更 1：AI 解析从 HTTP 同步改数据库契约边界异步解耦

- ai 服务从「HTTP 服务等请求」改为「轮询 worker 模式」：定期从共享数据库取待处理的 AI 类新闻，处理完写回
- ai 侧需新增**数据源适配层**：把「从 xiaobao 库取数」封装为独立适配层，AI 处理核心不感知数据来源（数据库 / HTTP 请求）
- 此要求源自 [decisions/0002](decisions/0002-ai-hub-ecosystem-positioning.md)（ai 生态定位为多调用方预留），ai 直连 xiaobao 库读 schema 会把 ai 焊死在 xiaobao 上，必须用适配层隔离

#### 变更 2：翻译保留在 ai 侧（不变更）

- 翻译职责仍由 ai 承担（与 REQ-001 一致），输入为原文，输出含翻译字段
- 数据库契约中 `processed_news.translation` 字段仍由 ai_worker 写入
- 说明：2026-07-05 初版曾计划将翻译剥离到 xiaobao，后经 v0.6.1 PRD Review 阶段 Owner 决策调整，改为保留在 ai 侧以简化架构（两层入库替代三层）

### ai 侧需做的工作

1. **架构改造**：HTTP 服务模式 → 轮询 worker 模式（claim → 处理 → 写回）
2. **适配层封装**：数据获取逻辑独立分层，处理核心不感知取数方式
3. **承接新契约**：按 [news-l1-db v1](contracts/news-l1-db.md) 数据库边界契约实现
   - 读：`raw_items` 指定列 + `sources` 指定列 + `tasks` claim
   - 写：`raw_items.l1_status` 等状态字段 + `processed_news` 写入结果 + `tasks` 状态更新
4. **保留灰度能力**：支持 HTTP 模式与数据库模式切换的配置开关，便于灰度验证和回滚
5. **卡死回收**：配合 xiaobao 侧的卡死回收机制（claim 时 `locked_by` + `locked_at`，超时未释放由 xiaobao 回收）

### 边界与衔接

- **契约变更**：本变更是 [news-l1 v1](contracts/news-l1.md) HTTP 契约的**新增并行模式**，非替换。HTTP 模式契约继续有效（灰度 / 回滚用）。数据库边界契约见 [contracts/news-l1-db.md](contracts/news-l1-db.md) v1
- **schema 权属**：共享库 schema 归 xiaobao 所有，ai 只能读指定表列、写回指定结果字段，不得建表改表。权限由数据库角色 GRANT 控制
- **与 REQ-001 的关系**：REQ-001 已关闭（ai v0.1 已交付 HTTP 模式 + 翻译），本 REQ 是其后继变更，ai 侧需在下一个迭代中承接
- **与 REQ-002 的关系**：REQ-002 架构调研结论若与本变更有关联，由 ai PM 自行判断是否合入本 REQ 的承接迭代
- **多调用方定位**：本变更要求 ai 做适配层封装正是为了保住 decisions/0002 的多调用方定位，不与此决策冲突

### xiaobao 侧已完成的前置产出

1. ✅ v0.6.1 PRD R2 已定稿（含两层入库架构和 5 态状态机）
2. ✅ coordination `contracts/news-l1-db.md` 数据库边界契约 v1 已出稿
3. ✅ xiaobao 侧设计文档 R2（含 schema 设计、权限模型、迁移方案、前端展示分层）
   - 设计文档：xiaobao `docs/progress/iterations/v0.6.1-design.md`

### 联调预期

- ai 侧改造完成后，xiaobao 与 ai 在共享数据库上做端到端联调
- 联调用例覆盖：正常解析路径 / 解析失败重试 / 卡死回收 / ai 服务不可用时 xiaobao 不阻塞 / HTTP 模式与数据库模式切换
- Owner 验收通过后切换灰度开关

### 沟通文档

待 ai · PM 评估承接后建立 `communications/REQ-003-db-boundary-async.md`。
