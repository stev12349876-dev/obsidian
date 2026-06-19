# Guides × Sensors 框架的产品化检验：claude-code-harness v4.2 暴露的五个张力

> 把 Böckeler 的理论分类（前馈/反馈 × 计算性/推理性）和一个开源产品级实现（[Chachamaru127/claude-code-harness](https://github.com/Chachamaru127/claude-code-harness/tree/v4.2.0) v4.2 "Hokage"）放在一起对照，看产品里发生了什么是框架没说的。
> 日期：2026-04-21
> **被引版本**：所有指向上游的链接 pin 到 `v4.2.0` tag（commit 范围对应 README 中"v4.2 Update — Claude Code 2.1.99-110 + Opus 4.7"小节）。上游已迭代到 v4.3.x，引用差异以 v4.2.0 tag 为准。

---

## 为什么这个对照值得做

Böckeler 在 [Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html) 里给出了一个干净的 2×2 矩阵（[works/fowler-harness-engineering-full-translation.md:30-46](../works/fowler-harness-engineering-full-translation.md)），但她明确说"这是分类学，不是设计模板"。她列举的例子（AGENTS.md、ArchUnit、Skills）都是组件级的——没有一个完整产品被她拿来做端到端剖析。

claude-code-harness v4.2 恰好提供了这样一个样本：

- **公开**——MIT License，全部代码和配置可读
- **完整**——Setup → Plan → Work → Review → Release 五个动词全覆盖
- **成熟**——已经迭代到 v4.2，自身走过了 v3 (bash + Node.js) → v4 (Go-native) 的重写
- **self-referential**——用 harness 改进 harness（[CLAUDE.md#L9](https://github.com/Chachamaru127/claude-code-harness/blob/v4.2.0/CLAUDE.md#L9)），是 cross-article-insights 洞见 1 提出的 "Harness Gardening" 的活样本

本文不写"这个产品好不好用"，而问：**当 Böckeler 的框架被一个真实产品全量实现后，框架和产品互相暴露了什么裂缝？**

---

## 先把 v4.2 装进 2×2 矩阵

| | 计算性（确定性，CPU） | 推理性（语义，LLM） |
|---|---|---|
| **引导器（前馈）** | `harness.toml`、`Plans.md` 的 status 标记体系（`pm:依頼中` → `cc:WIP` → `cc:完了` → `pm:確認済`）、CLAUDE.md 的 pointer 结构 | 5 verb skills（`/harness-plan` `/harness-work` `/harness-review` `/harness-release` `/harness-setup`）、`agents/scaffolder.md`、`agents/worker.md` 的 prompt |
| **传感器（反馈）** | **R01–R13 guardrail 引擎**（Go 原生,<10ms）、`harness doctor --residue`（残留检测）、`monitors/monitors.json` 的健康监控、`PreCompact` hook | reviewer agent 的 4 视角（Security / Performance / Quality / Accessibility）、`/harness-review` xhigh effort、Advisor 咨询 |

按 Böckeler 的字面分类先填一遍——四个象限**初看都有候选实现**。但凑近看，分类已经在拉伸：R01–R13 写在"反馈"侧只是因为它们运行在 PreToolUse hook 这个工程位置，**张力 2 会论证它们实际上同时承担前馈和反馈职责**；Advisor 看似在推理性反馈侧，**张力 1 会论证它根本不属于矩阵的任何一格**（它是按条件激活，不是按时机激活）。

**所以这张矩阵的价值不是"证明 2×2 全象限可工程化"，而是作为一个起点暴露出问题。** 真正的发现在下面五个张力里——是框架的分类边界被产品级实现拉伸后渗出的东西。

---

## 张力 1：Advisor Strategy 是 Böckeler 矩阵装不下的"条件触发推理性传感器"

**Böckeler 的预设：** 推理性传感器（如 LLM-as-judge）昂贵、不是每次都跑，所以放在反馈侧。

**v4.2 的实际做法：** 引入 Advisor agent（`docs/advisor-strategy.md`），它**既不在每次提交跑（不像 reviewer），也不预先注入（不像 skills）**，而是按三类触发条件被动唤起：
- 高风险任务的 preflight consultation
- 重复失败的 corrective consultation
- 检测到 plateau 时的 escalation consultation

这是一种**条件激活的推理性控制**——既不算前馈（行动前注入），也不算标准反馈（行动后审查）。它更像一个**"按需上线的备用推理回路"**。

Böckeler 的矩阵把控制按"何时执行"切了一刀（前/后），但 Advisor 揭示了第三个时间维度：**「按条件何时唤起」**。在 long-running harness（`docs/long-running-harness.md`）里这是关键设计——便宜模型持续推进，贵推理只在难处启动。

**质疑：** Advisor 的存在意味着**控制成本本身需要被调度**。Böckeler 框架默认"控制是开关"，v4.2 证明"控制是仪表盘"。框架需要一个新轴：**控制的激活策略**（always-on / per-commit / conditional / human-summoned）。

---

## 张力 2：guardrail 越严，前馈和反馈的边界越模糊

**Böckeler 的清晰二分：** 前馈在行动前引导，反馈在行动后纠正。

**v4.2 的 R01–R13：** 13 条规则在 PreToolUse hook 层拦截——`R06 git push --force` 直接 deny，`R05 rm -rf` 触发 ask，`R12 push to main/master` 给 warn。

问题是：**当一条规则的反应是 "deny"（直接拒绝执行），它到底是前馈还是反馈？**
- 从 Claude 的视角：它没行动就被打回，类似前馈
- 从系统视角：它已经发出了 tool call，被传感器拦截后才回滚——类似反馈

更微妙：**R13（Protected file edits）只 warn 不阻止**——这又退化成纯反馈。同一套引擎里，13 条规则的反应严格度形成了**前馈/反馈的连续光谱**，不是离散象限。

**张力的根源：** Böckeler 的矩阵假设了"engine 层的拦截 = 前馈"和"linter 层的提示 = 反馈"是分离的。但当机械化执行做到 hook 层时，这条边界消失了。**v4.2 的 guardrail engine 同时承担前馈和反馈职责**——这恰好是 OpenAI 原文"机械化执行"概念的成熟形态，但 Böckeler 框架没给它留位置。

---

## 张力 3：行为 Harness（房间里的大象）在 v4.2 里被绕过了，没被解决

**Böckeler 的诊断：** 三类调控对象里，可维护性最成熟、架构适应度中等、行为正确性"最弱、是房间里的大象"（[works/fowler-harness-engineering-full-translation.md:112-118](../works/fowler-harness-engineering-full-translation.md)）。

**v4.2 的应对：** 看似认真——reviewer agent 的 4 视角（Security/Performance/Quality/Accessibility）、`xhigh` effort（[CLAUDE.md#L17](https://github.com/Chachamaru127/claude-code-harness/blob/v4.2.0/CLAUDE.md#L17)）、Plans.md 强制四态状态机（`pm:依頼中 → cc:WIP → cc:完了 → pm:確認済`）。

**但仔细看：** 这些机制全部是**结构性**的——4 视角检查的是"代码是否符合 X 类问题模式"，状态机检查的是"流程是否走完"。**没有任何一个机制能回答"这段代码做对了用户真正想要的事吗"**。

Plans.md 的 acceptance criteria 是用户写的自然语言；reviewer 验收的是代码是否触犯 4 类常见模式问题；闭环里**唯一握有"行为是否正确"裁决权的依然是人类**——状态从 `cc:完了` 推进到 `pm:確認済` 的那一步，没有 harness 替你做。

**v4.2 的实际策略不是"解决行为 Harness"，而是"承认行为 Harness 不可解,所以把人类批准做成强制 gate"。** 这是务实的——但和 Böckeler 框架想暗示的"未来三类都能 harness 化"不一致。

**我的判断：** Böckeler 把行为 Harness 列为"最弱但同维度"是一种**乐观的分类**。v4.2 的实践表明它**根本不在同一维度**——前两类是工程问题，第三类是认识论问题。一个产品级 harness 能做的不是覆盖它，而是**为人类的最终判断保留高质量的输入**（清晰的 plan、可追溯的 reviewer 记录、明确的状态门控）。这其实是 Böckeler 自己说的"将人类输入引导到最重要的地方"——但她说得太轻，没强调这是**结构性绕过而非渐进解决**。

---

## 张力 4：Harness 的自我演化压力在框架里被低估

**Böckeler 提到的开放挑战：** "harness 模板会面临版本管理和贡献问题"——但她把这当作未来风险。

**v4.2 的实证：** 这不是未来风险，是**当下日常**。README 里直白记录：

> `harness sync` 可能悄悄删除你的 `monitors`/`agents` 块（**前 4 次事故**）。v4.2 修复：双层 guard（shell 幂等性测试 + Go struct 测试）。

**4 次事故。** 一个迭代到 v4.2 的成熟产品，仅"sync 命令悄悄破坏自身配置"这一类问题就发生过 4 次。这对应 cross-article-insights 洞见 1 提出的 "Harness Gardening"——**harness 不是设计完就稳定的产物，它处于持续的自我侵蚀压力下**。

更尖锐的是触发点：把 v4.2 的 7 项主要变更（README v4.2 update 表格）逐项归类：

| # | 变更 | 触发源 |
|---|---|---|
| 1 | PreCompact hook 阻止任务被压缩切断 | 上游：CC 2.1.105 新增 PreCompact hook |
| 2 | 改用 `monitors/monitors.json` 公共 schema | 上游：plugin spec 收紧 |
| 3 | `harness sync` 双层 guard | **自修：4 次自伤事故修复** |
| 4 | 1 小时 prompt cache opt-in | 上游：CC v2.1.108 新能力 |
| 5 | Reviewer/Advisor 升 `xhigh` effort | 上游：CC v2.1.111 + Opus 4.7 新能力 |
| 6 | Agent prompts 重调以匹配 Opus 4.7 字面遵循 | 上游：Opus 4.7 语义变化 |
| 7 | R01–R13 重新对齐 CC 2.1.110 hook 协议 | 上游：hook 协议演化 |

**6/7 是被动追赶上游，1/7 是修自己的 bug——零项是"harness 主动添加新能力"。**

**这意味着：** Böckeler 框架默认 harness 是工程师的**主动设计产物**。v4.2 揭示了 harness 实际上是**对宿主运行时变化的被动追赶产物**——95% 的迭代精力花在"不被宿主升级搞坏"或"修自伤"，几乎没有花在"主动改善能力"。

**框架的盲点：** Guides × Sensors 把 harness 当作**封闭系统**分析。但 harness 真正的成本中心是**与宿主（Claude Code）和模型（Opus 4.x）的接口维护**。这层在框架里完全不可见。

---

## 张力 5：harness 必然带价值观锁定——这一点框架完全没讨论

**Böckeler 的中性预设：** Guides 和 Sensors 是技术工具，可移植、可替换。

**v4.2 的实际状态：** [`CLAUDE.md#L38`](https://github.com/Chachamaru127/claude-code-harness/blob/v4.2.0/CLAUDE.md#L38) 硬编码 "All responses must be in **Japanese**"——任何用户装上这个 harness，Claude 立刻切日语回应。

这不是 bug，是设计决策。它揭示了：**任何成熟的 harness 必然携带其设计者的价值观/约定/语言/审美**。R01–R13 编码了"什么不该做"，5 verb 编码了"什么是合理工作流"，强制日文编码了"谁是预期用户"。

**Böckeler 框架的盲点：** 她讨论了 harness 模板会"分叉"，但没讨论 harness 的**价值观锁定**——一个 harness 不是中立的运行时层，它是一份**带强主张的工作宣言**。装上 v4.2，你接受的不仅是 13 条 guardrail，还接受了"日文为母语""Conventional Commits""GitFlow 分支命名"等一整套日本企业开发文化的隐式假设。

**对应到 cross-article-insights 洞见 7（技术栈收敛风险）：** 如果未来出现"主流 harness"，它一定不是技术中立的——它会带着设计组织的全部约定。这放大了单一栽培风险——**不只是技术单一，而是工程文化单一**。

**Harnessability（Böckeler 的概念）应该被扩展：** 一个仓库的 harnessability 不仅取决于强类型/模块边界/框架成熟度（技术维度），还取决于**仓库的文化约定与可用 harness 的预设是否对齐**（社会维度）。Java/Spring 不仅技术上可 harness，文化上也可 harness——因为它们已经把企业开发约定编码进了框架。

---

## 浮现的开放问题（连回 cross-article-insights）

1. **控制的"激活策略"应该如何分类？** Advisor 揭示了 always-on / per-commit / conditional / human-summoned 的连续光谱——Böckeler 的 2×2 应该升维成 2×2×N？
2. **当 guardrail engine 同时做前馈和反馈，"机械化执行"和"传感器"还是两个概念吗？** 是否应该合并为"运行时控制"这个第三类？
3. **Harness 的接口维护成本如何衡量？** v4.2 显示 95% 的迭代花在追上游——这个比例是否是所有成熟 harness 的常态？
4. **行为 Harness 的"结构性绕过"是否应该被正式承认为一种合法策略？** 即"harness 的目标不是覆盖行为正确性，而是把行为判断高质量地交还给人类"。
5. **harness 的价值观锁定如何披露？** 类似软件包的 license——是否应该有 "Harness Manifesto"，明示这套 harness 隐含的工作文化假设？
6. **延伸到洞见 1 的 "Harness Gardening"：** v4.2 的 4 次 sync 事故 + 6/7 项追上游变更，能否提炼出 harness 的**故障模式分类法**？

---

## 总结

claude-code-harness v4.2 给 Böckeler 的 Guides × Sensors 框架做了三件事：

| | 验证 | 补充 | 反驳 |
|---|---|---|---|
| 矩阵 | 四象限可作为分析起点（不是覆盖证明——见张力 1、2） | 需要加"激活策略"维度 | 前馈/反馈在 hook 层会融合 |
| 三类调控 | 可维护性最成熟 ✓ | 行为 Harness 是结构性绕过，不是同维度的弱项 | — |
| Harnessability | 强类型/模块边界 ✓ | 应包含**文化对齐**维度 | 技术中立性是幻觉 |
| Harness 演化 | 模板会分叉 ✓ | 95% 维护成本在追宿主升级 | 框架低估了被动追赶压力 |

**最锐利的判断：** Böckeler 的框架是一份**优秀的体检表**，但不是**手术台**。把一个真实产品放上手术台后，会看到框架没标的解剖结构——尤其是 harness 与宿主、与人类、与文化的三层接口。下一版框架若要做出来，应该把这三层接口画进去。
