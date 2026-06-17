# 跨项目状态

> 跨项目当前状态真源：各项目阶段、当前阻塞、谁等谁、下一步。
> 单项目内部细节不在此展开，从「当前入口」链接回各项目 `docs/progress/INDEX.md`。
> 跨项目需求池见 [REQUESTS.md](REQUESTS.md)（提报中心）。
> 最近更新：2026-06-17（xiaobao WM）

## 各项目当前状态

| 项目 | 阶段 | 当前入口 | 备注 |
|------|------|----------|------|
| `xiaobao` | v0.6 标准迭代，实现阶段联调精修，Developer 侧已收口 | 项目 `docs/progress/INDEX.md` | 等待 Owner 报具体 bug |
| `ai` | 仓库骨架可运行（冒烟 2 passed），**尚未 Bootstrap 团队工作流** | 项目仓库 README | 无 git remote；未建 `docs/progress/INDEX.md` |

## 跨项目阻塞与谁等谁

<a name="ai-bootstrap"></a>
### 1. ai 项目接入团队工作流

- 状态：待启动
- 谁等谁：无强阻塞；`xiaobao` 不依赖 `ai` Bootstrap 即可继续 v0.6。
- 下一步责任：**Owner / ai 项目会话** —— 为 `ai` 配置 git remote，并在 `ai` 项目目录内进入团队模式、执行 Bootstrap（只复用 baseline/templates，不继承 xiaobao 的 `docs/progress/`）。
- 依据：各项目 `docs/baseline/cross-project-collaboration.md` §新项目复用团队工作流。

<a name="news-l1-contract"></a>
### 2. news-l1 契约

- 状态：v1 生效中，已收编为单一真源 → [contracts/news-l1.md](contracts/news-l1.md)
- 谁等谁：无。两侧当前与 v1 一致。
- 下一步责任：任一侧需改契约时，先改 `contracts/news-l1.md` 再改代码。
- 已验证：`ai` 端 `news-l1` 本机 + `xiaobao` `news_test` 库单条真实 raw_item 端到端通过（2026-06-16，见 xiaobao INDEX 跨任务待办 L1 Agent 化条目）。

## 下一步汇总

1. Owner：为 `niuma-cheng-ai` 配置 GitHub remote。
2. ai 项目会话：进入团队模式 → Bootstrap → 在本仓库 `PROJECTS.md` 补全仓库地址、在本文件登记 ai 阶段。
3. 任一侧改 news-l1 契约：先改 `contracts/news-l1.md`，CHANGELOG 记一行。
