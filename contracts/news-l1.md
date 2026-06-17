# 契约：news-l1（新闻 L1 处理）

- 契约 id：`news-l1`
- 版本：v1
- 状态：生效中
- 调用方：`xiaobao`（Node 新闻平台）
- 服务方：`ai`（Python Agent Hub）
- 最近更新：2026-06-17
- 真源说明：本文件是该契约的**单一真源**。修改 `L1Input` / `L1Output` / `RunResponse` 任一字段前，先改本文件，再改两侧代码，并在 [../CHANGELOG.md](../CHANGELOG.md) 记一行。
- 实现参考：`ai` 项目 `src/agent_hub/schemas.py`（Pydantic 定义需与本文件保持一致）

## Endpoint

```text
POST /v1/runs/news-l1
Content-Type: application/json

请求体：L1Input
响应体：RunResponse
```

健康检查：`GET /health`。

## 职责边界（重要）

- `ai` 只产出**四维原始评分**（`score` + `reason`）、标签、摘要、翻译、上下文与分析。
- **评分加权 `score_total` 留在 `xiaobao`**，不在本契约内传输，也不由 `ai` 计算。
- `tasks` 表为 `xiaobao` 侧业务真源；一次 run 仅处理单条证据，不持有任务状态。

## 请求：L1Input

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `source_identity` | string | 必填 | 信息源身份标识 |
| `domain_tags` | string[] | `[]` | 领域标签（来自 L0 分类） |
| `raw_content` | object | `{}` | 原始结构化内容 |
| `raw_text` | string | `""` | 原始文本 |
| `kb_results` | object[] | `[]` | 调用方预取的库内检索结果 |
| `link_content` | string \| null | `null` | 调用方预取的链接正文 |
| `search_summary` | string \| null | `null` | 调用方预取的搜索摘要 |
| `options` | RunOptions | 见下 | 运行选项 |

### RunOptions

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `max_tool_calls` | int | `4` | 单次 run 最大工具调用次数 |
| `timeout_ms` | int | `180000` | 单次 run 超时（毫秒） |

## 响应：RunResponse

| 字段 | 类型 | 说明 |
|------|------|------|
| `run_id` | string | 本次 run 标识 |
| `status` | `"succeeded"` \| `"failed"` | 处理结果 |
| `elapsed_ms` | int | 处理耗时（毫秒） |
| `tool_summary` | ToolSummary | 工具调用统计 |
| `output` | L1Output \| null | 成功时的处理结果；失败为 `null` |
| `error` | string \| null | 失败原因；成功为 `null` |

### ToolSummary

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `web_search` | int | `0` | Web 搜索调用次数 |
| `link_read` | int | `0` | 链接读取次数 |
| `kb_search` | int | `0` | 库内检索次数 |

### L1Output

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `title` | string | 必填 | 标题 |
| `summary` | string | 必填 | 摘要 |
| `translation` | object | `{}` | 翻译，如 `{"zh": "..."}` |
| `context` | array | `[]` | 背景上下文条目 |
| `analysis` | string \| null | `null` | 分析 |
| `score_dimensions` | ScoreDimensions | 必填 | 四维评分 |
| `tags` | Tags | 见下 | 五类标签 |
| `needs_context` | bool | `false` | 是否需要补充上下文 |

### ScoreDimensions（四维，每维 ScoreDimension）

四个维度：`timeliness`（时效）、`impact`（影响力）、`confidence`（可信度）、`clarity`（清晰度）。

每个维度结构 ScoreDimension：

| 字段 | 类型 | 说明 |
|------|------|------|
| `score` | int | 0–5 |
| `reason` | string | 评分理由（默认 `""`） |

### Tags（五类，每类 string[]，默认 `[]`）

| 字段 | 说明 |
|------|------|
| `domain` | 领域 |
| `entity` | 实体 |
| `event` | 事件 |
| `content_type` | 内容类型 |
| `processing` | 处理标签（如 `engine:agent`） |

## 变更纪律

1. 任一字段增删改、类型变化、语义变化前，先改本文件并更新版本/最近更新日期。
2. breaking change（删字段、改类型、改必填性）必须在 [../CHANGELOG.md](../CHANGELOG.md) 标记 **BREAKING** 并点名需要跟进的项目。
3. 两侧代码改完后，各自在本项目 `docs/progress/INDEX.md` 或相关记录中写明已跟进的本契约版本。
4. 不允许 `xiaobao` 与 `ai` 各自维护一份互相冲突的契约说明。
