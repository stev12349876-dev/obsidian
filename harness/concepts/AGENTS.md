# concepts/ — 概念笔记

原文六大核心概念的拆解与整理（01-06）+ 后续阅读引出的延伸概念（07-）。每个概念一个文件，编号排序。

## 文件约定

- 文件名：`{编号}-{英文短名}.md`，如 `01-repo-as-source-of-truth.md`
- `00-overview.md` 是总览，先读这个

### 结构（建议，非强制）

早期文件（01–03）来自 OpenAI 单一原文，结构倾向于：核心要点 → 关键实践 / 方案。
后期文件（04–06）已演化为多源综合（OpenAI + LangChain + HumanLayer + Fowler），允许加入：

- "来自其他文章的补充" / "来自 X 的补充"：跨源对照
- "关键洞察" / "类比" / "哲学"：作者层加工
- 06 是定义综合型，结构（定义 → 组件清单 → 跨源对照）与前 5 篇不同，属于合法变体

约束只有两条：
1. 第一段说清楚这个概念**是什么**（无论叫"原文要点"还是"核心思想"还是"问题"）
2. 跨源内容必须**标注来源**（哪篇文章、哪个作者）

## 已有内容

| 文件 | 概念 | 来源 |
|------|------|------|
| [00-overview.md](00-overview.md) | 六大核心概念总览 | OpenAI 原文 |
| [01-repo-as-source-of-truth.md](01-repo-as-source-of-truth.md) | 仓库即记录系统 | OpenAI 原文 |
| [02-mechanical-enforcement.md](02-mechanical-enforcement.md) | 机械化执行 | OpenAI 原文 |
| [03-entropy-and-garbage-collection.md](03-entropy-and-garbage-collection.md) | 熵管理与垃圾回收 | OpenAI 原文 |
| [04-agent-readability.md](04-agent-readability.md) | 智能体可读性 | OpenAI + LangChain + HumanLayer + Fowler |
| [05-throughput-changes-merge.md](05-throughput-changes-merge.md) | 吞吐量改变合并理念 | OpenAI + HumanLayer + LangChain + Fowler |
| [06-harness-definition.md](06-harness-definition.md) | Harness 的精确定义与组件清单 | LangChain + HumanLayer + Fowler |
| [07-spec-as-product.md](07-spec-as-product.md) | 约束即产品（Spec as Product）| OpenAI Symphony + Fowler + HumanLayer + Meta-Harness |

## 下一步

读完概念后，去 [thinking/](../thinking/) 写你自己的理解和质疑。
