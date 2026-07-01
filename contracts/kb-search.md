# 契约：kb-search（库内新闻实时检索）

- 契约 id：`kb-search`
- 版本：v1
- 状态：生效中
- 调用方：`ai`（Python Agent Hub）
- 服务方：`xiaobao`（Node 新闻平台）
- 最近更新：2026-07-01
- 真源说明：本文件是 ai→xiaobao 库内检索接口的单一真源。修改 endpoint、字段、鉴权或错误语义前，先改本文件，再改两侧代码，并在 [../CHANGELOG.md](../CHANGELOG.md) 记一行。

## Endpoint

```text
POST /v1/kb-search
Content-Type: application/json
x-admin-token: {xiaobao 管理 token，如生产环境启用}

请求体：KbSearchRequest
响应体：KbSearchResponse
```

## 职责边界

- `ai` 在 L1 处理中根据模型提炼出的查询词按需调用本接口。
- `xiaobao` 只返回库内已入库新闻的检索候选，不执行 LLM 分析，不计算新评分。
- 本接口不改 `news-l1` v1 入参；`news-l1` 的 `tool_summary.kb_search` 由 `ai` 侧统计。

## 请求：KbSearchRequest

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `query` | string | 必填 | 查询词或短查询句，1–300 字符 |
| `top_n` | int | `5` | 返回条数，1–10；ai 不传时 xiaobao 返回 5 条 |
| `exclude_raw_item_id` | string(uuid) \| null | `null` | 排除当前正在处理的原始条目 |
| `source_id` | string(uuid) \| null | `null` | 排除同一 source，避免只命中同源重复内容 |
| `domain_tags` | string[] | `[]` | 领域标签提示；v1 不保证强过滤，服务方可用于排序或忽略 |

### 参数约定

- `query` 由 ai 根据当前新闻提炼，建议 3–30 个中文字符或 2–8 个英文 token；不传整段原文。
- `top_n` 默认 5，最大 10。超过最大值按 `400` 处理，不静默截断。
- `exclude_raw_item_id` 用于排除当前正在处理的新闻，避免自匹配。
- `source_id` 用于排除同一来源；如果 ai 希望同源也参与背景检索，可以不传。

## 响应：KbSearchResponse

| 字段 | 类型 | 说明 |
|------|------|------|
| `query` | string | 回显查询词 |
| `results` | KbSearchResult[] | 命中的库内新闻 |

### 空结果语义

`results: []` 是合法的 200 响应，表示本次查询没有命中库内候选；调用方不得将其等同于接口失败或传输失败。ai 侧可以选择继续 web_search / link_read / LLM 主流程，但处理标签应区分“空命中”和“工具失败”，例如 `kb_search_empty`，不应标记为 `kb_search_failed`。

### KbSearchResult

| 字段 | 类型 | 说明 |
|------|------|------|
| `news_id` | string(uuid) | 新闻 id |
| `raw_item_id` | string(uuid) | 原始条目 id |
| `title` | string | 标题 |
| `summary` | string | 摘要 |
| `content` | string | 可直接喂给 LLM 的上下文正文；v1 与 `summary` 相同 |
| `published_at` | string \| null | ISO 时间 |
| `score_total` | number \| null | v0.6 综合分，可能为空 |
| `importance_score` | number | v0.5 兼容重要性分 |
| `source` | object | `{ id, identity, name }` |
| `url` | string \| null | 原文 URL，如有 |

结果排序：服务方按文本匹配度优先，其次按 `score_total` / `importance_score` 与发布时间排序。v1 不返回总数，不做分页；ai 如需更多候选，必须用更精确 query 再次调用。

## 错误语义

- `400`：请求体校验失败。
- `403`：鉴权失败（生产环境启用管理 token 或 IP 白名单时）。
- `500`：服务端异常。

## 变更纪律

1. breaking change（删字段、改类型、改必填性、改鉴权方式）必须在 [../CHANGELOG.md](../CHANGELOG.md) 标记 **BREAKING** 并点名需要跟进的项目。
2. 两侧代码改完后，各自在本项目 `docs/progress/INDEX.md` 或相关记录中写明已跟进的本契约版本。
3. 不允许 `xiaobao` 与 `ai` 各自维护一份互相冲突的库内检索契约说明。

## 调用示例

### Request

```json
{
  "query": "OpenAI 模型路由 成本控制",
  "top_n": 5,
  "exclude_raw_item_id": "00000000-0000-0000-0000-000000000002",
  "source_id": "00000000-0000-0000-0000-000000000003",
  "domain_tags": ["AI"]
}
```

### Response

```json
{
  "query": "OpenAI 模型路由 成本控制",
  "results": [
    {
      "news_id": "00000000-0000-0000-0000-000000000001",
      "raw_item_id": "00000000-0000-0000-0000-000000000004",
      "title": "模型路由能力背景",
      "summary": "此前已有多家平台提供按成本与延迟路由模型的能力。",
      "content": "此前已有多家平台提供按成本与延迟路由模型的能力。",
      "published_at": "2026-07-01T00:00:00.000Z",
      "score_total": 6.8,
      "importance_score": 0,
      "source": {
        "id": "00000000-0000-0000-0000-000000000005",
        "identity": "example",
        "name": "Example"
      },
      "url": "https://example.com/news/1"
    }
  ]
}
```
