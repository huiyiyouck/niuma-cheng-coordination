# REQ-001 · news-l1 跨项目协作沟通

- 参与项目：`xiaobao`, `ai`
- 相关需求：REQ-001（状态见 [../REQUESTS.md](../REQUESTS.md)）
- 定位：REQ-001 被承接之后的协作与联调沟通
- 契约真源：[../contracts/news-l1.md](../contracts/news-l1.md)；库内检索反向接口见 [../contracts/kb-search.md](../contracts/kb-search.md)
- 当前状态入口：[../STATUS.md#news-l1-contract](../STATUS.md)
- 最近更新：2026-07-04

## 关系概述

`ai` 承接了 `xiaobao` 提报的 L1 处理需求。技术上：`xiaobao` 通过 `POST /v1/runs/news-l1` 调用 `ai`，传入原始证据（`L1Input`），取回处理结果（`RunResponse.output: L1Output`）。

> 调用方向是 xiaobao→ai，但需求方向双向 —— `ai` 同样可以向 `xiaobao` 提需求（如调整入参、补字段、改投递节奏），这类需求同样先进 [../REQUESTS.md](../REQUESTS.md)。

职责切分：

- `xiaobao`：信息源、抓取、L0 分类、展示、评分**加权**（`score_total`）。
- `ai`：L1 推理 —— 四维**原始**评分、标签、摘要、翻译、按需工具调用。

## 承接的需求

| 需求 id | 内容 | 状态 | 详情 |
|---------|------|------|------|
| REQ-001 | 新闻 L1 处理 | 已关闭（2026-07-04） | [../REQUESTS.md#req-001](../REQUESTS.md) |

## 联调沟通

> 承接后的接口对接 / 字段对齐 / 调试 / 版本跟进，倒序排列，条目标注所属需求 id。

### 2026-07-04 · [REQ-001] 端到端联调完成，news-l1 主链路 + KB 双向调用全量验证通过

- **双方角色**：ai 侧 PM（ck）起草 + xiaobao Developer（ck）补充 run 数据与结论确认；Owner（ck）已在 xiaobao 前端 `/debug/ai` 抽样验收通过。
- **背景**：继 2026-07-01 联调验证通过后，Owner 完成最终端到端全量验收，联调闭环。
- **联调范围**：
  1. xiaobao → ai `POST /v1/runs/news-l1` 主链路（库内新闻 → 真实 L1Input 构造 → ai 处理 → 返回结果）
  2. ai → xiaobao `POST /v1/kb-search` 主动库内检索（命中用例 + 空结果用例）
  3. `tool_summary` 统计口径（预取不计入、ai 主动调用才计数）
  4. 降级语义（KB 失败不阻塞主流程，整体仍 succeeded）
- **xiaobao 侧补充数据**：
  - 端到端主链路成功用例数：**4 条**（公网端到端 / 内网直连 / KB 命中 / KB 空结果各 1 条）
  - 代表性 run_id（xiaobao→ai 公网端到端）：`run_7e626cf5f391`（status=succeeded，公网 nginx 反代）
  - 代表性 run_id（xiaobao→ai 内网直连）：`run_2a4dbc15f308`（status=succeeded，elapsed_ms=73601，预填 search_summary 场景，tool_summary 全 0 符合口径）
  - 代表性 run_id（ai→xiaobao KB 命中）：`run_2e0072cba2a3`（status=succeeded，tool_summary.kb_search=1，无 degraded:kb_search_failed）
  - 单条耗时范围：**73601 ~ 79000 ms**（约 74~79s，较 6 月底 104s 优化约 25-30%）
  - 四维评分 / 五类标签 / 摘要 / 翻译产出是否符合预期：**是**。返回包含完整四维评分（时效/影响/可信/清晰，各 0-5 分 + reason）、五类标签（含 processing 引擎标识）、中文摘要、分析等，与 `contracts/news-l1.md` v1 契约一致。
  - KB 空结果语义（`results: []`）是否仍标 `degraded:kb_search_failed`：**是，仍标为 `degraded:kb_search_failed`**，ai 侧尚未优化。这是已知遗留项（2026-07-01 已记录），不阻塞主链路与关闭。
  - 是否发现新问题：**未发现新问题**。所有问题均为 7 月 1 日已记录的已知项。
- **xiaobao 侧配置确认**：
  - `AI_HUB_BASE_URL` = `http://127.0.0.1:8100`（测试环境，同机）
  - `AI_HUB_API_TOKEN` 已配置（测试环境鉴权未启用，留空也可通）
  - xiaobao 测试环境 KB 接口 `POST http://127.0.0.1:8001/v1/kb-search` 已就绪，未设 admin token 鉴权
- **当前结论（已确认）**：`news-l1` v1 契约不变，端到端联调通过，REQ-001 可进入验收关闭。

### 2026-07-01 · [REQ-001] xiaobao 测试环境已部署，双向联调验证通过；KB 空结果语义待 ai 优化

- **xiaobao 提交与部署**：xiaobao 已提交 `3fc71e6 Developer: 增加 AI 联调验收入口`，测试环境已部署。前端验收入口：`https://test.huiyiyou.cloud/debug/ai`；后端测试服务：`news-api-test.service`（`:8001`）。
- **xiaobao 接口验证**：
  - `GET http://127.0.0.1:8001/health` 返回 200。
  - `GET /v1/ai-debug/candidates?page_size=1` 返回 200，可取库内候选新闻。
  - `POST /v1/kb-search` 返回 200，命中时返回 `results[].content` / `summary` 等 ai 可消费字段。
- **xiaobao→ai 真实调用验证**：通过 xiaobao 测试后端 `POST /v1/ai-debug/news-l1-runs` 调用 ai `POST /v1/runs/news-l1` 成功。
  - `run_id`：`run_2a4dbc15f308`；`status`：**succeeded**；`elapsed_ms`：`73601`。
  - `tool_summary`：`web_search=0`、`link_read=0`、`kb_search=0`。该用例由 xiaobao 预填 `search_summary`，ai 未主动调用工具，符合 `tool_summary` 只统计 ai 主动工具调用的契约口径。
  - 返回包含标题、摘要、分析、四维评分、五类标签；`processing` 含 `engine:agent_hub`、`llm:volcengine`。
- **ai→xiaobao KB 主动检索验证（命中用例）**：直接调用 ai `POST /v1/runs/news-l1`，构造可命中 xiaobao KB 的标题 `OpenAI Codex大幅扩展插件生态，可一键变身62个应用的工作专家`。
  - `run_id`：`run_2e0072cba2a3`；`status`：**succeeded**；`elapsed_ms`：`76932`。
  - `tool_summary`：`web_search=1`、`link_read=0`、`kb_search=1`。
  - `processing` 未出现 `degraded:kb_search_failed`，说明 ai 主动 KB 回调 xiaobao `POST /v1/kb-search` 已端到端打通。
- **ai→xiaobao KB 空结果用例（待 ai 优化语义）**：标题 `OpenAI Codex 投资插件` + `domain_tags:["AI"]` 调用 xiaobao `POST /v1/kb-search` 时返回 200 + `results: []`。这是 `kb-search` v1 的正常空命中响应，不是接口错误。
  - 当前 ai 对该用例仍返回 `status=succeeded`，`tool_summary.kb_search=1`，但 `processing` 会追加 `degraded:kb_search_failed`。
  - 结论：链路可用，剩余问题是 ai 侧把“空结果”统一标成“失败”的语义不准确。建议 ai 改为 `kb_search_empty` 或不降级；必要时优化 `_build_kb_query` 查询词策略，提高命中率。
- **当前结论**：`news-l1` 主链路、xiaobao `/debug/ai` 验收入口、ai→xiaobao `/v1/kb-search` 双向链路均已可联调验收；`news-l1` v1 不需要变更。`kb-search` v1 补充空结果语义：`results: []` 是 200 正常响应。

### 2026-07-01 · [REQ-001] ai 服务已部署测试环境，news-l1 主链路真实冒烟通过（回填证据）

- **ai 服务就绪**：ai 已部署到测试环境 **`http://127.0.0.1:8100`**（当前机器 uvicorn 常驻），`GET /health` 返回 200。LLM 走 openclaw 火山大模型 `doubao-seed-2.0-pro`（OpenAI 兼容）。
- **真实调用证据（回填 xiaobao 第 5 点要求）**：
  - `run_id`：`run_bcf24393b947`；`status`：**succeeded**；`error`：null。
  - `tool_summary`：`web_search=1`、`link_read=0`、`kb_search=1`。
  - `kb_search` 已按 `contracts/kb-search.md` 主动发起，但 xiaobao `127.0.0.1:8001` 未起 → 降级 `degraded:kb_search_failed`，整体仍 `succeeded`（AC-6 语义）。
  - 真实产出：四维评分（时效 5 / 影响 4 / 可信 3 / 清晰 4，各带 reason）、五类标签、中文摘要、分析；`web_search` 真实调 Tavily 成功。
- **对 xiaobao 5 点诉求的回应**：
  1. `AI_HUB_BASE_URL` = `http://127.0.0.1:8100`（同机），`/health` 200 已验证。
  2. 鉴权：测试环境 **不启用**（Owner 定），xiaobao 直连即可，无需 `Authorization`；上线前再加。
  3. `POST /v1/runs/news-l1` 已按 `contracts/news-l1.md` 部署（snake_case、预取 `kb_results` 不计 `tool_summary`、link read 从 `raw_content.url`/`canonical_url` 取）。
  4. KB 回调：ai 已实现调用 `POST /v1/kb-search`（`KB_SEARCH_URL=http://127.0.0.1:8001/v1/kb-search`）；当前 ai 侧 `KB_ADMIN_TOKEN` 留空——**请 xiaobao 确认测试环境是否需要 `x-admin-token`**，需要则告知 token 值。
  5. 真实调用证据见上。
- **待 xiaobao**：起测试服务（含 `8001` KB）+ 配 `AI_HUB_BASE_URL=http://127.0.0.1:8100` → 在 `/debug/ai` 选库内新闻发起端到端验收；届时 `kb_search` 可端到端跑通。
- **观察（非阻塞）**：单条处理约 104s（reasoning 模型 + 工具串行），联调可接受；后续可评估换更快模型或优化工具并发。

### 2026-07-01 · [REQ-001] xiaobao 实现 AI 联调验收页 + news-l1 触发入口 + KB search v1

- **xiaobao 侧响应**：Developer 已实现前端 `AI 联调` 页面，支持从库内已入库新闻候选中搜索/选择，按真实业务同一套 raw_item→`L1Input` 映射构造请求，调用 ai `/v1/runs/news-l1`，并在页面展示请求 JSON、响应 JSON、状态、耗时、工具调用统计、标题/摘要/四维评分/标签。
- **契约定稿依据**：已核对 ai 侧 `src/agent_hub/schemas.py`、`graphs/news_l1.py`、`tools/link_reader.py` 当前实现：`news-l1` 使用 snake_case `L1Input` / `RunResponse`；预取 `kb_results` 不计入 `tool_summary`；link read 只从 `raw_content.url` / `raw_content.canonical_url` 取 URL。coordination `contracts/news-l1.md` 已补齐这些字段语义与 JSON 示例。
- **后端触发入口**：新增 `GET /v1/ai-debug/candidates` 与 `POST /v1/ai-debug/news-l1-runs`。该入口用于验收联调，默认只返回 AI 结果，不写回 `processed_news`，避免调试污染线上展示内容。
- **AI Hub 客户端**：xiaobao 后端新增 `AI_HUB_BASE_URL` / `AI_HUB_API_TOKEN` / `AI_HUB_TIMEOUT_MS` 配置和 HTTP 调用客户端；真实 worker 也可通过 `L1_ENGINE=ai` 复用同一 AI Hub 调用路径。
- **KB search 契约**：新增 ai→xiaobao 实时库内检索契约 [../contracts/kb-search.md](../contracts/kb-search.md) v1，并实现 `POST /v1/kb-search`。定稿字段：`query` 必填，`top_n` 默认 5 / 最大 10，`exclude_raw_item_id` 与 `source_id` 可选排除自匹配/同源匹配，返回 `results[]` 包含 `news_id`、`raw_item_id`、`title`、`summary`、`content`、`published_at`、`score_total`、`importance_score`、`source`、`url`。该接口不改 `news-l1` v1 契约。
- **xiaobao 需要 ai 提供 / 确认**：
  1. 测试环境可访问的 ai 服务 base URL（xiaobao 将配置为 `AI_HUB_BASE_URL`），并保证 `GET {base_url}/health` 返回 200。
  2. 是否启用调用鉴权；如启用，请提供测试环境 token，xiaobao 将配置为 `AI_HUB_API_TOKEN` 并以 `Authorization: Bearer {token}` 调用。
  3. `POST {base_url}/v1/runs/news-l1` 已按 [../contracts/news-l1.md](../contracts/news-l1.md) 当前版本部署，尤其是 `raw_content.url` / `canonical_url`、`KbResult`、`tool_summary` 统计口径与示例一致。
  4. ai 侧接入 xiaobao KB 回调配置：测试环境 xiaobao URL 为 `http://127.0.0.1:8001`（同机）或 `https://test.huiyiyou.cloud`（经 nginx），接口为 `POST /v1/kb-search`；若 xiaobao 测试服务启用管理 token，ai 侧需按约定发送 `x-admin-token`。
  5. ai 侧联调时请回填一次真实调用证据：请求的 `run_id`、`status`、`tool_summary`、是否触发 `kb_search`、失败时 `error`。
- **待联调**：ai 侧启动服务并提供 `AI_HUB_BASE_URL` 后，Owner 可在 xiaobao 前端 `/debug/ai` 页面选择新闻发起真实端到端验收；ai 侧可用 `POST /v1/kb-search` 验证按需库内检索。
- **测试环境部署**：xiaobao 已部署到测试环境。前端入口：`https://test.huiyiyou.cloud/debug/ai`；后端测试服务：`news-api-test.service`（`:8001`）。已验证 `GET /v1/ai-debug/candidates?page_size=1` 返回 200，`POST /v1/kb-search` 返回 200。当前 `http://127.0.0.1:8100/health` 不通，真实 news-l1 调用等待 ai 服务 base URL / 运行态。

### 2026-07-01 · [REQ-001] ai v0.1 实现完成，请 xiaobao 提供联调触发入口

- **ai 侧进展**：ai v0.1（news-l1 stub→真实）实现阶段已完成并自测通过 —— 四维**原始**评分（`score`+`reason`）、五类标签、摘要、翻译、按需工具（link 自抓 / web 直连 Tavily）、多 provider fallback、优雅降级均已落地；`POST /v1/runs/news-l1` 就绪，**对外 news-l1 v1 契约不变**。ai 侧自动化测试 36 passed；但真实端到端连通性（真 LLM + 真工具 + 真新闻数据）尚未验证，需两侧联调。
- **诉求（对 xiaobao）**：请提供一个**联调触发入口**——从 xiaobao 库中选一条**已有新闻**，按与真实业务**完全相同**的方式构造 `L1Input` 调用 ai `/v1/runs/news-l1`，取回 `RunResponse` 用于对比核验。
- **两入口关系（均在 xiaobao 侧，只差新闻来源）**：
  - 真实业务入口：新闻来源 = 真实外部抓取（x 来源）；
  - 联调入口：新闻来源 = xiaobao **本地库已有新闻**（人工 / 接口选定 news_id）；
  - 二者**除来源不同外，`L1Input` 构造与调用 ai 的流程完全一致** —— 同一套字段映射（`source_identity` / `domain_tags` / `raw_content` / `raw_text` / `kb_results` / `link_content` / `search_summary` / `options`），确保联调复现真实链路，而非另造一条测试专用路径。
- **为什么需要**：没有联调入口，只能等真实抓取管道自然跑到某条新闻，不可控、不可复现；有了它可用库内**指定新闻**随时、可重复地打通端到端，核验字段对齐、真实四维评分 / 标签 / 摘要 / 翻译产出、`tool_summary` 统计口径、降级 / 失败语义。
- **契约边界**：**不改 news-l1 v1 契约**（`L1Input` / `RunResponse` 字段与语义不动）；这是 xiaobao 侧的调试触发方式（选库内新闻→构造入参→调 ai），不涉及 ai 对外契约变更。
- **待 xiaobao（PM / Developer）响应**：是否新增该联调入口、由谁实现、预计可联调时间；ai 侧 `/v1/runs/news-l1` 已就绪，可随时对接。

### 2026-06-30 · [REQ-001] xiaobao 响应 KB search 工具分工：选定方案 b（实时检索接口）

- Owner 2026-06-30 拍板：对 ai 2026-06-29 提出的库内检索 KB search 提供方式，xiaobao **选定方案 b —— xiaobao 新开实时检索接口，ai 在 L1 处理中用 LLM 提炼的查询词按需回调**（不走方案 a 预取填充 `kb_results`）。
- 选 b 理由：让 ai 用提炼后的精准查询词实时检索，库内相关新闻 / 背景补全质量高于「分析前粗预取」。
- 接口 / 契约形态由 **xiaobao Architect 设计，PM 不替架构拍板**：这是 ai→xiaobao **反向调用**的新接口，倾向新建独立契约（如 `contracts/kb-search.md`），按契约变更纪律**先改 `contracts/` 再两侧改码**；需 Architect 明确 endpoint、查询入参（查询词 / 过滤 / topN）、返回结构（是否复用现有 `kb_results` 的 object 形状）、鉴权、超时。
- xiaobao 侧落地：该检索接口为新增对外能力，需在 xiaobao 立项实现（v0.6.1 或独立任务，待 xiaobao PM / Owner 定）；**不阻塞 ai v0.1 其余范围**。
- 对 ai 的衔接：ai v0.1 的 KB 主动检索待本接口契约定稿 + xiaobao 实现后联调；在此之前 ai 按其 2026-06-29 说明先做能独立完成的部分（消费预取 + `needs_context` 标记 + 自抓 link / 自配 web）。
- 下一步：xiaobao 切 Architect 设计检索接口契约 → 入 coordination `contracts/` → 两侧实现联调。

### 2026-06-29 · [REQ-001] ai 启动 v0.1 实现，提出对 xiaobao 的能力诉求

- `ai` 已据 REQ-002 架构结论启动 v0.1 标准迭代（news-l1 stub→真实，PRD R1：ai `docs/progress/iterations/v0.1-prd.md`）。
- 实现 news-l1「按需工具调用」时，三类工具后端分工 Owner 2026-06-29 已拍定，**唯一需 xiaobao 提供的是库内检索 KB search**（新闻库在 xiaobao，ai 无直接访问）：
  - **库内检索 KB search（需 xiaobao 提供）**：请 xiaobao 定 —— a) 预取填充 `kb_results`（契约已有字段）加量/加质；还是 b) 提供实时检索接口供 ai 按需调。
  - 链接读取 link read：**ai 自行 HTTP 抓取**，不需 xiaobao。
  - Web 搜索 web search：**Owner 提供搜索 API key、ai 自配调用**，不需 xiaobao。
- 边界：不改 `news-l1` 契约则优先复用现有预取字段（`kb_results`/`link_content`/`search_summary`）；若需新接口/字段，另走 [../contracts/](../contracts/) 契约变更流程。
- 对 v0.1 影响：ai v0.1 的 AC-5（按需工具调用）先做 ai 能独立完成的部分（消费预取 + `needs_context` 标记 + 自抓 link/自配 web）；**主动 KB 检索依赖本诉求的 xiaobao 响应**，落地后补，不阻塞 v0.1 其余范围。
- 待 xiaobao（PM/Architect/Developer）响应三类工具分工。

### 2026-06-22 · [REQ-001] ai PM 正式承接

- `ai` 项目完成 Bootstrap 并立项后，由 PM（ck）评估正式承接 REQ-001，补齐「正规提报（xiaobao · Developer）→ 承接（ai · PM）」留痕闭环。
- 真实 L1 处理（评分/标签/摘要/翻译由 stub 转真实）规划转入 ai v0.1 标准迭代（待启动，迭代门禁下一步启动）；管道层联调状态不变（仍「联调中」）。

### 2026-06-17 · [REQ-001] 契约真源迁移

- xiaobao WM 在 coordination 建立 `news-l1` v1 契约真源，取代此前散落在 xiaobao spike 提案中的临时真源。
- 两侧当前实现与 v1 一致，无需立即改代码。
- 约定：后续任一侧改契约，先改 `contracts/news-l1.md`，再改代码，CHANGELOG 记一行。

### 2026-06-16 · [REQ-001] 端到端打通验证

- `ai` 端 `news-l1` 流水线本机冒烟通过；`xiaobao` `news_test` 库单条真实 raw_item 经 `ai` 处理端到端成功（task `6de4139b-...` succeeded，`processed_news` 写入，`tags_v2.processing` 含 `engine:agent`）。
- 待观察：3–5 条小批量并发、耗时、失败回退与成本（归属 xiaobao Developer）。

## 待跟进

| 事项 | 归属 | 状态 |
|------|------|------|
| ai 配置 git remote 并 Bootstrap 团队工作流 | Owner / ai 会话 | 已完成（2026-06-21，见 [../STATUS.md#ai-bootstrap](../STATUS.md)） |
| REQ-001 承接留痕补登 | ai PM/Architect | 已完成（2026-06-22，ai PM ck 承接） |
| REQ-001 news-l1 小批量观察 | xiaobao Developer | 进行中 |
| ai news-l1 库内检索 KB search 能力需 xiaobao 提供（link/web 已由 ai 自抓 + Owner key 解决） | xiaobao（PM/Architect/Dev）+ ai Developer | ✅ 已完成（2026-07-04）——xiaobao 已实现 `POST /v1/kb-search`，ai 侧已接入并联调通过；KB 空结果语义待优化（非阻塞） |
| ai↔xiaobao news-l1 真实数据端到端联调：需 xiaobao 提供「选库内已有新闻 → 构造同样 `L1Input` 调 ai」的**联调触发入口**（与真实业务入口只差新闻来源） | xiaobao（PM/Dev）+ ai PM | ✅ 已完成（2026-07-04）——端到端主链路 + KB 双向调用全量验证通过，Owner 抽样验收通过；REQ-001 可进入验收关闭 |
