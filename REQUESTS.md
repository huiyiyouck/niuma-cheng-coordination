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
| REQ-001 | xiaobao · Developer | 新闻 L1 处理：四维原始评分 + 五类标签 + 摘要 + 翻译 + 按需工具调用 | ai · PM（ck） | ai v0.1（待启动） | 联调中 | [communications/REQ-001-news-l1.md](communications/REQ-001-news-l1.md) |
| REQ-002 | xiaobao · Architect | AI 处理架构调研：从 Horizon / ai-news-aggregator 两个参考项目提炼 L0/L1 与 Agent Hub 设计输入，回答 4 个架构岔路口 | ai · PM（ck）承接登记，产出归 Architect | ai v0.1（前置，待启动） | 已承接 | [communications/REQ-002-arch-research.md](communications/REQ-002-arch-research.md) |

---

## REQ-001 · 新闻 L1 处理

- 提出方：xiaobao · Developer
- 内容：把 L1 推理从新闻平台主进程解耦，由独立服务承载 —— 产出四维**原始**评分（`score`+`reason`）、五类标签、摘要、翻译、按需工具调用（KB 检索 / 链接读取 / Web 搜索）
- 边界：承接方只产原始评分，加权 `score_total` 留 xiaobao
- 承接方：`ai`（AI 处理中枢）· PM（ck）
- 承接说明：本需求先于跨项目需求流转机制既成事实推进并打通管道；2026-06-22 `ai` 项目完成 Bootstrap 并立项后，由其 PM（ck）评估**正式承接**，补齐「正规提报（xiaobao · Developer）→ 承接（ai · PM）」留痕闭环
- 转入迭代：ai v0.1 标准迭代（待启动）—— 真实 L1 处理（评分/标签/摘要/翻译由 stub 转真实）规划进 v0.1，迭代门禁下一步启动
- 当前状态：联调中 —— 契约 [news-l1 v1](contracts/news-l1.md) 定稿，端到端单条已通过，3–5 条小批量观察进行中
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

