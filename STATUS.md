# 跨项目状态

> 跨项目当前状态真源：各项目阶段、当前阻塞、谁等谁、下一步。
> 单项目内部细节不在此展开，从「当前入口」链接回各项目 `docs/progress/INDEX.md`。
> 跨项目需求池见 [REQUESTS.md](REQUESTS.md)（提报中心）。
> 最近更新：2026-07-01（xiaobao 已实现 AI 联调验收页、`/v1/ai-debug/news-l1-runs`、`/v1/kb-search`，新增 `contracts/kb-search.md` v1；待 ai 服务地址配置后端到端验收）。2026-07-01（ai v0.1 实现阶段完成、`/v1/runs/news-l1` 就绪，向 xiaobao 提 news-l1 联调触发入口诉求，见 communications/REQ-001）。2026-06-30（xiaobao v0.6 已部署上线状态同步；REQ-001 KB search xiaobao 选定方案 b 实时接口）。2026-06-29（ai 定位升级 decisions/0002；REQ-002 ai PM 承接；元信息台账 ai 定位变更 → PROJECTS + 生态根索引均已同步）

## 各项目当前状态

| 项目 | 阶段 | 当前入口 | 备注 |
|------|------|----------|------|
| `xiaobao` | v0.6 已实现完成并部署生产上线（2026-06-28 去软链接化隔离部署）；待 DevOps 补登部署记录 + PM 迭代关闭检查 | 项目 `docs/progress/INDEX.md` | REQ-001 KB search 实时接口（方案 b）待 Architect 设计契约后立项；生产 AI 处理默认关闭（X 直显） |
| `ai` | 仓库骨架可运行（冒烟 2 passed），**已接入团队工作流**；PM（ck）已承接 REQ-001、REQ-002；定位升级为生态内部通用 AI 处理中枢（[decisions/0002](decisions/0002-ai-hub-ecosystem-positioning.md)） | 项目 `docs/progress/INDEX.md` | REQ-002 架构调研（前置）→ REQ-001 转 v0.1 标准迭代；各节点真实逻辑仍为占位（stub） |

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

- 状态：**ai 服务已部署测试环境 `http://127.0.0.1:8100`（常驻，`/health` 200）**，news-l1 主链路真实冒烟通过（LLM=火山 `doubao-seed-2.0-pro`，run `run_bcf24393b947` succeeded，Tavily 通；KB 因 xiaobao `8001` 未起降级）。xiaobao 已实现 `/debug/ai` + `POST /v1/ai-debug/news-l1-runs` + `POST /v1/kb-search`（契约 [contracts/kb-search.md](contracts/kb-search.md) v1）；`news-l1` 已补齐 raw_content URL / KbResult / tool_summary 口径。真实**双向**端到端（含 KB 回调）尚未跑通。
- 谁等谁：**ai 已就绪，等 xiaobao 起测试服务（含 `8001` KB）+ 配 `AI_HUB_BASE_URL=http://127.0.0.1:8100`** → 在 `/debug/ai` 选库内新闻发起端到端验收。KB 回调待 8001 起；ai 侧 `KB_ADMIN_TOKEN` 留空，待 xiaobao 确认测试环境是否需 `x-admin-token`。
- 下一步责任：xiaobao 起服务 + 配 `AI_HUB_BASE_URL` → `/debug/ai` 验收 + 确认 KB admin token 需求；ai 侧鉴权测试环境不启用（Owner 定）；**不改 news-l1 v1 契约**。证据见 [communications/REQ-001-news-l1.md](communications/REQ-001-news-l1.md) 2026-07-01 ai 回填条。

## 下一步汇总

1. ~~Owner：为 `niuma-cheng-ai` 配置 GitHub remote~~ —— 已完成（2026-06-21），`PROJECTS.md` 仓库地址已回填。
2. ai 项目会话：PM 已承接 REQ-001；Owner 确认后由 PM 创建 `v0.1-prd.md` 启动标准迭代，实现 news-l1 真实 L1 处理（stub→真实）。
3. 任一侧改 news-l1 契约：先改 `contracts/news-l1.md`，CHANGELOG 记一行。
4. xiaobao 项目会话：已实现 ai 2026-07-01 提的 news-l1 **联调触发入口**诉求；下一步配置 ai 服务地址并用 `/debug/ai` 做真实端到端验收（见 communications/REQ-001 2026-07-01 条）。

## 元信息变更台账

> 用途见各项目 `cross-project-collaboration.md` §项目元信息同步。子项目迭代关闭检查发现定位/名称/技术栈/上线/接入状态变更时登记一行；coordination 会话改 `PROJECTS.md` 后勾「PROJECTS 已同步」；生态索引维护方（本生态=根 `/root/Project/CLAUDE.md`）照真源订正后勾「生态索引已同步」。两列都勾 → 归档/移除。

| 项目 | 字段 | old | new | 来源（commit/迭代） | PROJECTS 已同步 | 生态索引已同步 |
|------|------|-----|-----|----------------------|-----------------|----------------|
| workboard | 技术栈 | 本地 Node + 静态前端 | React 18 + Vite + Tailwind v4 + shadcn + 本地 Node 聚合后端 | v0.1（`v0.1-summary.md`） | ✅ | ✅ |
| workboard | 上线状态 | 待开发 MVP | v0.1 已上线 `workboard.huiyiyou.cloud` | v0.1 关闭 | ✅ | ✅ |
| workboard | 接入版本 | @1b01fba | @66c1e1a | 回流 `4b8e563` | ✅ | ✅ |
| ai | 定位 | AI 处理中枢 / Agent Hub（职责限 xiaobao L1 新闻处理） | niuma-cheng 生态内部通用 AI 处理中枢（多调用方预留，首落地 xiaobao news-l1） | decisions/0002（2026-06-29） | ✅ | ✅ |
