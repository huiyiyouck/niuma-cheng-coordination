# niuma-cheng-coordination

niuma-cheng 多项目生态的**协调仓库**：承载跨项目契约和状态的单一真源。

本仓库只处理跨项目边界，不替代任何单项目内部的 `docs/progress/INDEX.md`、迭代记录或角色日志。规则见各业务项目 `docs/baseline/cross-project-collaboration.md`。

## 生态成员

| 项目 id | 名称 | 技术栈 | 仓库 |
|---------|------|--------|------|
| `xiaobao` | 牛马成新闻平台 | Node.js + Vue | `git@github.com:huiyiyouck/niuma-cheng-xiaobao.git` |
| `ai` | AI 处理中枢（Agent Hub） | Python + FastAPI + LangGraph | 待 Owner 配置 remote |

详见 [PROJECTS.md](PROJECTS.md)。

## 关系

`xiaobao` 通过 HTTP 调用 `ai` 的新闻 L1 处理能力：

```text
xiaobao (调用方)  ──HTTP──▶  ai (服务方)
     POST /v1/runs/news-l1
```

契约单一真源：[contracts/news-l1.md](contracts/news-l1.md)。

## 目录

| 路径 | 作用 |
|------|------|
| [PROJECTS.md](PROJECTS.md) | 项目目录：仓库、职责、当前入口、关联沟通文档 |
| [REQUESTS.md](REQUESTS.md) | 跨项目需求提报中心（需求池）：提报时不指定承接方，由项目方认领或 Owner 指派 |
| [STATUS.md](STATUS.md) | 跨项目当前状态：谁等谁、阻塞项、下一步 |
| [CHANGELOG.md](CHANGELOG.md) | 跨项目重大事件、契约 breaking change、迁移提醒 |
| `contracts/` | 跨项目契约单一真源（HTTP schema、字段语义） |
| `communications/` | 承接需求**之后**的项目间协作与联调沟通，按参与项目命名 |
| `decisions/` | 影响两个以上项目的决策记录 |

## 协作约定（摘要）

- 任何跨项目工作**开工前 `git pull` 本仓库**，先看 `STATUS.md`。
- 改契约：**先改 `contracts/`**，再各自改代码，并在 `CHANGELOG.md` 记一行。
- 一边完成影响另一边的事：更新 `STATUS.md` 并点名对方项目和下一步责任。
- 本仓库为跨项目真源；各项目 `docs/progress/INDEX.md` 仍是各自项目级真源，二者不重复。

完整规则以各项目 `docs/baseline/cross-project-collaboration.md` 为准。
