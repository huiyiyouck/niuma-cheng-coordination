# communications — 承接后的项目间协作沟通

本目录承接 niuma-cheng 生态内，**需求被承接之后**，提出方与承接方之间的协作与联调沟通。每份文档对应一组项目。

> 需求**提报**不在这里 —— 提报在 [../REQUESTS.md](../REQUESTS.md)（需求池，提报时不指定承接方，由项目方认领或 Owner 指派）。需求被某项目承接后，才在本目录建立对应的 `{a}__{b}.md`。

## 模板定位

- **承接后才建立**：一份文档 = 一组项目承接关系确立后的协作沟通主线。
- **一份看全**：这组项目承接了哪些需求、各自联调到哪。
- **内容两块**：① 承接的需求列表（引用 [../REQUESTS.md](../REQUESTS.md)）；② 联调沟通时间线（接口对接、字段对齐、调试、版本跟进，条目标注所属需求 id）。
- 契约细节在 [../contracts/](../contracts/)，全局状态在 [../STATUS.md](../STATUS.md)。

## 命名规则

- 两个项目：`{project-a}__{project-b}.md`，如 `xiaobao__ai.md`
- 三个及以上：`{project-a}__{project-b}__{project-c}.md`
- 项目 id 用 [../PROJECTS.md](../PROJECTS.md) 中的稳定短名
- 文件名只表达**参与方**，不表达临时议题；议题写进文档内章节

## 索引

| 文档 | 参与项目 | 承接的需求 |
|------|----------|------------|
| [xiaobao__ai.md](xiaobao__ai.md) | `xiaobao`, `ai` | REQ-001 新闻 L1 处理 |

> 每份文档必须被 [../PROJECTS.md](../PROJECTS.md) 反向链接，避免孤儿沟通记录。
