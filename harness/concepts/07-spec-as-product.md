## 概念 7：约束即产品（Spec as Product）

> 来源：OpenAI Symphony（[references/articles.md #16](../references/articles.md)）。在六大原文概念之外的第七个概念。它把 04 智能体可读性从"内部"推到"对外发布"，同时把六大概念中的"地图而非手册"原则（[00-overview.md §2](00-overview.md)）推到极致——map 不再止于本仓库导航，而成为可分发给社区使用者的种子文件。

## 核心思想

当编码智能体能从规范生成实现时，**可分发的产品形态从"代码"反转为"规范"**。

软件交付的传统形态是"代码 + 文档"——使用者拿到的是已编译的实现，文档只是辅助理解。Spec as Product 的反转：使用者拿到的是**约束、目标与工作流的形式化描述**，自己用编码智能体生成符合本地环境的实现。

OpenAI Symphony 是当前最干净的范式样本：仓库主体只是 `SPEC.md` + `WORKFLOW.md`，参考实现（Elixir）的地位明确为"参考"，社区被鼓励**自己拿规范跑一份**。

## 三个支撑结构

### SPEC.md：定义问题，不定义实现

`SPEC.md` 描述三件事：

1. **要解决的问题**（"对每一个打开的任务，保证都有一个智能体在自己的工作区中运行"）
2. **目标解法的形态**（控制平面、状态机映射、生命周期保证）
3. **取舍的边界**（什么不在范围内）

刻意**不写**：使用什么语言、什么库、什么部署方式。这些是实现问题，留给本地智能体决定。

### WORKFLOW.md：把隐式人类流程显式化

许多团队的"开发流程"是部落知识——只有资深成员知道 issue 进 In Progress 之前要先 checkout 仓库、PR 提交后要附演示视频、Review 状态前要附自我反思。这些步骤过去靠文化传承，从未文档化。

Symphony 把这些步骤压进 `WORKFLOW.md`，由编排器保证智能体每一步都执行。**当文化必须显式化才能被智能体执行时，团队规范也被迫从口头传承升级为可审计文本**。

### 多语言验证：用实现差异反向打磨规范

OpenAI 的工程师让 Codex 用 Elixir、TypeScript、Go、Rust、Java、Python 各自实现一遍 `SPEC.md`，然后**用实现差异定位规范歧义**：哪一段被不同语言的智能体理解成了不同的东西，那段规范就有问题。

这是 spec engineering 的"压力测试"环节，把"规范是否清晰"从主观判断变成了可重复实验。

## 与既有概念的关系

### 04 智能体可读性 — 从内部走向对外

04 关心的是**让智能体能在本仓库工作**——AGENTS.md、skills、文档结构都是给本地智能体读的。

07 是 04 的对外发布版：**让其他团队的智能体也能基于这份规范工作**。当 04 做到极致——所有上下文都被结构化、所有约束都被显式化——那么"分发规范让别人复刻"就变成自然结果。

### "地图而非手册"原则 — 推到跨仓库

六大原文概念中的"地图而非手册"（[00-overview.md §2](00-overview.md)）主张给本地智能体的是**导航**而非全量手册：AGENTS.md ≈ 目录页（~100 行），渐进式披露指向更深层文档。

07 把这个原则**推到跨仓库版本**：SPEC.md 不仅是给本仓库智能体的导航，还是分发给社区使用者的种子文件。Map 从"本仓库内部产物"升级为"指引外部智能体复刻整个系统的入口"。这与 04 不是同一件事——04 是"让智能体能在这个仓库里干活"，"地图而非手册"是"导航替代手册"作为表达形式，07 同时延伸了这两条。

## 与其他文章的关联

### Martin Fowler — 控制论的双向延伸

Fowler 的 Guides×Sensors 框架（[references/articles.md #2](../references/articles.md)）提供了**前馈+反馈**的二维分类。Spec as Product 是把"前馈"维度（Guides）抽出来作为可独立分发的工件——SPEC 是给智能体的最高层 Guide，WORKFLOW 是过程性 Guide。反馈维度（Sensors，CI/lint/eval）通常仍由本地实现。

### HumanLayer — AGENTS.md 杠杆的扩展

HumanLayer（[references/articles.md #5](../references/articles.md)）的"AGENTS.md 60 行规则"是**单仓库**层面的智能体引导。Spec as Product 把这套思路推到**跨仓库**：你不只是给本地智能体写一份精炼的规则文件，而是写一份"任何智能体都能拿去复刻你这套系统"的规则文件。

### Meta-Harness 论文 — 从搜索到分发

Meta-Harness 论文（[references/articles.md #11](../references/articles.md)）讨论"如何自动搜索最优 harness 设计"。Spec as Product 是它的**社会化版本**：你不是在算法空间里搜索，而是在不同团队、不同语言、不同环境下让人/智能体各自实现一遍，社区演化筛选出更好的规范。

## 关键洞察

- **代码免费 → 语言可以按特性选**：参考实现选 Elixir 不是因为团队熟悉，而是因为并发是 Symphony 的核心需求，Elixir 的 OTP 模型天然契合。当代码生成成本接近零，技术栈选择从"工程师熟悉度"约束中释放，回归"问题适配度"。这与 [thinking/cross-article-insights.md 洞见 7](../thinking/cross-article-insights.md) 的"技术栈收敛"形成对称——前者预测同质化，后者预测分化，未来需要观察哪条路径占主导。
- **规范的脆弱性**：当规范成为产品时，规范本身的质量缺陷会被放大。一处歧义可能让 100 个使用者的智能体生成 100 种不一致的实现。多语言交叉验证是必要的硬性环节。
- **维护责任的转移**：传统开源项目维护实现；Spec as Product 项目维护规范本身——OpenAI 明确表示"不打算把 Symphony 作为独立产品维护"。这改变了开源治理模型：贡献者贡献的是规范的修订意见，不是 PR 代码。

## 哲学

> Linus 说 "good programmers worry about data structures and their relationships"。
>
> Spec as Product 时代的对应陈述可能是：**"good engineers worry about constraints and their composability"** —— 当代码免费时，工程的核心价值集中在约束设计，而约束的可组合性（一份 SPEC 能否拼接到不同环境）成为新的衡量标准。
