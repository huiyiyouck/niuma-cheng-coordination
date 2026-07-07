# 跨项目状态

> 跨项目当前状态真源：各项目阶段、当前阻塞、谁等谁、下一步。
> 单项目内部细节不在此展开，从「当前入口」链接回各项目 `docs/progress/INDEX.md`。
> 跨项目需求池见 [REQUESTS.md](REQUESTS.md)（提报中心）。
> 最近更新：2026-07-04（REQ-001 news-l1 端到端联调完成，Owner 抽样验收通过，可进入关闭流程；KB 空结果语义待 ai 优化为已知遗留项，不阻塞）。2026-07-01（xiaobao 测试环境已部署 `/debug/ai`、`/v1/ai-debug/news-l1-runs`、`/v1/kb-search`；news-l1 主链路与 ai→xiaobao KB 命中用例已双向验证通过；KB 空结果语义待 ai 优化）。2026-07-01（ai v0.1 实现阶段完成、`/v1/runs/news-l1` 就绪，向 xiaobao 提 news-l1 联调触发入口诉求，见 communications/REQ-001）。2026-06-30（xiaobao v0.6 已部署上线状态同步；REQ-001 KB search xiaobao 选定方案 b 实时接口）

## 各项目当前状态

| 项目 | 阶段 | 当前入口 | 备注 |
|------|------|----------|------|
| `xiaobao` | v0.6 已实现完成并部署生产上线（2026-06-28 去软链接化隔离部署）；待 DevOps 补登部署记录 + PM 迭代关闭检查 | 项目 `docs/progress/INDEX.md` | REQ-001 KB search 实时接口（方案 b）待 Architect 设计契约后立项；生产 AI 处理默认关闭（X 直显） |
| `ai` | v0.1 已关闭（2026-07-04，REQ-001 news-l1 真实化交付）；**已接入团队工作流**；PM（ck）已承接 REQ-001、REQ-002；定位升级为生态内部通用 AI 处理中枢（[decisions/0002](decisions/0002-ai-hub-ecosystem-positioning.md)） | 项目 `docs/progress/INDEX.md` | REQ-001 已交付关闭；REQ-002 架构调研结论已用于 v0.1 设计；KB 空结果语义待下一迭代优化 |

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

## 下一步汇总

1. ~~Owner：为 `niuma-cheng-ai` 配置 GitHub remote~~ —— 已完成（2026-06-21），`PROJECTS.md` 仓库地址已回填。
2. ai 项目会话：PM 已承接 REQ-001；Owner 确认后由 PM 创建 `v0.1-prd.md` 启动标准迭代，实现 news-l1 真实 L1 处理（stub→真实）。
3. 任一侧改 news-l1 契约：先改 `contracts/news-l1.md`，CHANGELOG 记一行。
4. xiaobao 项目会话：联调证据已齐，维持测试服务配置待命。
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
| workboard | 名称/定位 | 跨项目 Agent 工作看板 | 项目管理工作台（Web 端项目迭代 IDE） | v0.2 迭代关闭（2026-07-07，`v0.2-summary.md`） | ⬜ | ⬜ |
| workboard | 功能范围 | 只读聚合看板（4 视图） | 读写工作台（5 视图：新增项目会话 + 角色映射 + 对话查看 + 迭代时间轴） | v0.2 迭代关闭（2026-07-07，`v0.2-summary.md`） | ⬜ | ⬜ |
