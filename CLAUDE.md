# niuma-cheng-coordination · 协调真源台账入口

## 我在这个仓库的角色

本仓库是 niuma-cheng 生态的**跨项目协调真源台账**：契约、需求池、跨项目状态、决策、基线修正提案池（BCR）的单一真源。它**只承载跨项目边界事实**，不承载任何项目的业务代码，也不替代各项目自己的 `docs/progress/`。

## 默认角色：General（通用助手），直接维护台账

- 默认且仅以 **General** 工作：直接读写本仓的 Markdown 台账文件（`PROJECTS.md` / `REQUESTS.md` / `STATUS.md` / `CHANGELOG.md` / `contracts/` / `decisions/` / `communications/`）。
- **不进开发工作流**：本仓不做软件开发，不切 PM/Architect/Developer/Tester/DevOps 角色，不创建 `docs/progress/`，不走 PRD / 标准迭代 / Review 门禁 / Bootstrap。
- 默认且必须使用中文。

## 为什么不接入 agent-workflow 工作流

本仓是**协调台账**，不是开发项目（性质同 `workboard`）。agent-workflow 那套「一人公司开发团队」机制对一个纯 Markdown 真源台账几乎全是空转。因此：

- **不把 agent-workflow sync 进本仓**，本仓不出现在 `PROJECTS.md` 的「已接入 agent-workflow」集合里，**不进任何 BCR 回流清单**——本仓是 BCR 的登记处，不是 BCR 的回流对象。
- 本仓自身规则（台账结构、BCR 状态机、契约格式）需要演进时，General 直接改本仓 Markdown 即可，无需走工作流门禁。

## 维护纪律（以已有规则为准，这里只列最高频红线）

完整规则见本仓 `README.md`「协作约定」、`REQUESTS.md` 顶部说明，以及各业务项目 `docs/baseline/cross-project-collaboration.md`。最常踩的几条：

- **跨项目边界**：只在本仓记跨项目事实；**不在本仓写某项目内部进度**——那是各项目自己仓里 `docs/progress/` 的事。
- **开工先 `git pull`**，先看 `STATUS.md`。
- **改契约**：先改 `contracts/`，再各自改代码，并在 `CHANGELOG.md` 记一行。
- **BCR 池**：BCR 由 Owner + agent-workflow 真源会话评估/采纳/落地；下游只能提报。状态机 `已提报 → 评估中 → 已采纳/部分采纳/已拒绝/转 v2 候选 → 已落地真源 → 回流中 → 已回流下游`；下游全部 sync 完才置「已回流下游」终态。
