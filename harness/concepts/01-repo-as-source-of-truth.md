# 概念 1：仓库即记录系统

## 原文要点

智能体在运行时无法访问的任何内容，对它来说都**不存在**。

知识的存放位置决定了它是否有效：

| 位置 | 对人类 | 对智能体 |
|------|--------|----------|
| Google Docs | ✅ | ❌ |
| Slack 讨论 | ✅ | ❌ |
| 团队成员脑中 | ✅ | ❌ |
| 仓库内 Markdown | ✅ | ✅ |
| 代码 + 注释 | ✅ | ✅ |
| Lint 规则 | 间接 ✅ | ✅（强制） |

## 文档结构（原文方案）

```
AGENTS.md              ← 入口目录 (~100行)
ARCHITECTURE.md        ← 域和包分层的顶层地图
docs/
├── design-docs/       ← 设计决策，带验证状态
├── exec-plans/        ← 执行计划，带进度和决策日志
│   ├── active/
│   └── completed/
├── product-specs/     ← 产品规格
├── references/        ← 外部参考（llms.txt）
├── generated/         ← 自动生成（DB schema 等）
├── QUALITY_SCORE.md   ← 每个领域的质量评分
├── RELIABILITY.md
├── SECURITY.md
└── ...
```

## 关键实践

1. **AGENTS.md 是目录，不是百科** — ~100行，只指路
2. **专职 linter + CI 验证** — 知识库是否更新、是否交叉链接、结构是否正确
3. **doc-gardening 智能体** — 定期扫描过时文档，自动发起修复 PR
4. **执行计划是一等工件** — 提交到仓库，版本控制，带进度日志

## 来自其他文章的补充

### OpenAI Symphony — 任务跟踪器也是记录系统

OpenAI Symphony（[references/articles.md #16](../references/articles.md)）把"记录系统"的边界**从仓库扩展到任务跟踪器**：每个打开的 Linear ticket 是一个"在飞工作"的记录单元，状态机映射到 ticket 状态字段（Backlog → In Progress → Review → Merging → Done）。

这给"仓库即记录系统"加了一条对称命题：

> **代码与文档放仓库；在飞工作放跟踪器。两者都对智能体可见，缺一不可。**

如果跟踪器（Linear、Jira、GitHub Issues）的状态对智能体不可读，智能体就只能看到"已合并的过去"，看不到"正在进行的现在"——它可能重复劳动、抢工作、或者在已经被人放弃的任务上继续推进。Symphony 的 `WORKFLOW.md` 文件本质上就是把"在飞工作的状态语义"也变成仓库内可版本化的文本。
