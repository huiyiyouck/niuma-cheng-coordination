# communications — 承接后的项目间协作沟通

本目录承接 niuma-cheng 生态内，**需求被承接之后**，提出方与承接方之间的协作与联调沟通。每份文档对应一个需求。

> 需求**提报**不在这里 —— 提报在 [../REQUESTS.md](../REQUESTS.md)（需求池，提报时不指定承接方，由项目方认领或 Owner 指派）。需求被某项目承接后，才在本目录建立对应的 `{REQ-id}-{short-name}.md`。

## 模板定位

- **承接后才建立**：一份文档 = 一个已承接需求的协作沟通主线。
- **一份看全**：该需求由谁提出、谁承接、联调到哪。
- **内容两块**：① 需求概述（引用 [../REQUESTS.md](../REQUESTS.md)）；② 联调沟通时间线（接口对接、字段对齐、调试、版本跟进）。
- 契约细节在 [../contracts/](../contracts/)，全局状态在 [../STATUS.md](../STATUS.md)。

## 命名规则

- 格式：`{REQ-id}-{short-name}.md`，如 `REQ-001-news-l1.md`
- `REQ-id` 必须与 [../REQUESTS.md](../REQUESTS.md) 中的需求 id 一致
- `short-name` 使用英文小写短名，用连字符分隔
- 文件名表达**需求**，参与项目写进文档头部

## 索引

| 文档 | 参与项目 | 需求 |
|------|----------|------|
| [REQ-001-news-l1.md](REQ-001-news-l1.md) | `xiaobao`, `ai` | REQ-001 新闻 L1 处理 |

> 每份文档必须被 [../REQUESTS.md](../REQUESTS.md) 对应需求条目的「沟通文档」字段反向链接，避免孤儿沟通记录；[../PROJECTS.md](../PROJECTS.md) 指向 REQUESTS 作为项目级入口。
