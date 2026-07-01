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
| `raw_content` | object | `{}` | 原始结构化内容；若原始条目有 URL，调用方必须提供 `url` 或 `canonical_url` |
| `raw_text` | string | `""` | 原始文本 |
| `kb_results` | KbResult[] | `[]` | 调用方预取的库内检索结果；预取消费不计入 `tool_summary.kb_search` |
| `link_content` | string \| null | `null` | 调用方预取的链接正文 |
| `search_summary` | string \| null | `null` | 调用方预取的搜索摘要 |
| `options` | RunOptions | 见下 | 运行选项 |

### raw_content 约定

`raw_content` 是弱结构对象，但以下 key 已进入两侧契约语义：

| key | 类型 | 说明 |
|-----|------|------|
| `title` | string \| null | 原始标题或短标题，ai 可用于搜索 query |
| `text` | string \| null | 原始正文，优先级高于 `body` / `description` |
| `body` | string \| null | 原始正文候选 |
| `description` | string \| null | 摘要/描述候选 |
| `url` | string \| null | 原文 URL；ai 侧 link read 只认 `url` / `canonical_url` |
| `canonical_url` | string \| null | 规范原文 URL；与 `url` 二选一即可 |
| `author_username` | string \| null | X/Twitter 等社交源作者标识 |
| `author_name` | string \| null | 作者显示名 |

xiaobao 构造 `L1Input` 时，若 `raw_items.source_item_url` 存在而 `raw_content.url` / `canonical_url` 不存在，必须补入 `raw_content.url = source_item_url`。

### KbResult（预取库内结果）

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `news_id` | string \| null | `null` | 库内新闻 id |
| `raw_item_id` | string \| null | `null` | 原始条目 id |
| `title` | string | 必填 | 标题 |
| `summary` | string | `""` | 摘要 |
| `content` | string | `summary` | 可直接喂给 ai 的上下文正文 |
| `published_at` | string \| null | `null` | ISO 时间 |
| `source` | object \| null | `null` | `{ id, identity, name }` |
| `url` | string \| null | `null` | 原文 URL |

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

统计口径：只统计 **ai 主动发起** 的工具调用。调用方传入的 `kb_results` / `link_content` / `search_summary` 属于预取上下文，ai 消费它们时不增加 `tool_summary`。

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

## 调用示例

### Request

```json
{
  "source_identity": "openai",
  "domain_tags": ["AI"],
  "raw_content": {
    "title": "OpenAI 发布新能力",
    "text": "OpenAI 发布新的模型路由能力，开发者可以更细粒度控制成本。",
    "url": "https://x.com/openai/status/1",
    "author_username": "openai"
  },
  "raw_text": "OpenAI 发布新的模型路由能力，开发者可以更细粒度控制成本。",
  "kb_results": [
    {
      "news_id": "00000000-0000-0000-0000-000000000001",
      "raw_item_id": "00000000-0000-0000-0000-000000000002",
      "title": "模型路由能力背景",
      "summary": "此前已有多家平台提供按成本与延迟路由模型的能力。",
      "content": "此前已有多家平台提供按成本与延迟路由模型的能力。",
      "published_at": "2026-07-01T00:00:00.000Z",
      "source": { "id": "source-1", "identity": "example", "name": "Example" },
      "url": "https://example.com/news/1"
    }
  ],
  "link_content": null,
  "search_summary": null,
  "options": {
    "max_tool_calls": 4,
    "timeout_ms": 180000
  }
}
```

### Response

```json
{
  "run_id": "run_20260701_001",
  "status": "succeeded",
  "elapsed_ms": 38000,
  "tool_summary": {
    "web_search": 1,
    "link_read": 1,
    "kb_search": 0
  },
  "output": {
    "title": "OpenAI 发布模型路由能力",
    "summary": "OpenAI 发布新的模型路由能力，开发者可更细粒度控制成本。",
    "translation": { "zh": "OpenAI 发布新的模型路由能力，开发者可更细粒度控制成本。" },
    "context": [],
    "analysis": "该信息对开发者成本控制有直接影响。",
    "score_dimensions": {
      "timeliness": { "score": 5, "reason": "近期发布" },
      "impact": { "score": 4, "reason": "影响开发者成本和模型选择" },
      "confidence": { "score": 4, "reason": "来自官方账号" },
      "clarity": { "score": 4, "reason": "信息表达明确" }
    },
    "tags": {
      "domain": ["AI"],
      "entity": ["OpenAI"],
      "event": ["产品发布"],
      "content_type": ["社交动态"],
      "processing": ["engine:agent_hub"]
    },
    "needs_context": false
  },
  "error": null
}
```
