# Meta-Harness 与现有 Harness Engineering 体系的五个张力

> 读完 Anthropic "Scaling Managed Agents" 后，对比 references/ 中 7 篇文章的质疑与思考。

## 背景

Managed Agents 引入了一个新的抽象层次：不再讨论"如何设计好的 harness"，而是追问"如何让 harness 本身成为可替换的基础设施"。这与已有 6 篇文章建立的框架存在多处张力。

---

## 张力 1：Meta-harness 是否消解了 Harness Engineering 的投入价值？

**现有共识：** Agent = Model + Harness，工程师应精心设计 harness（AGENTS.md、linter、Sprint 合同）。

**Managed Agents 的隐含主张：** Harness 是牲畜，应该可替换。投入应放在接口层（session、sandbox 的标准化），而非 harness 细节。

**矛盾点：** 你花两周打磨 AGENTS.md 和验证闭环，但下个模型发布后这些可能全部过时（Anthropic #4 已有实证：Sonnet → Opus 升级后 context reset 变死重）。那 harness engineering 的 ROI 如何计算？

**我的判断：** 两者不矛盾，但作用在不同时间尺度上。Harness engineering 是**当前模型的战术优化**，meta-harness 是**跨模型代际的战略设计**。问题是大多数团队只有战术预算。

---

## 张力 2：Session 外部存储 vs Context Rot 的"笨办法"

**LangChain 的方案：** 在 harness 层解决 context rot——compaction、Ralph Loop、渐进式披露。这些是不可逆的启发式方法。

**Managed Agents 的方案：** 别做不可逆决策。把完整事件流存到 session，`getEvents()` 按需切片取回。

**质疑：这是在用存储换智能吗？** 如果模型在 200k token 的事件流中挑选相关切片的能力不够强，这个设计就退化为"全都存、全都丢给模型"——和不做上下文管理没区别。

**当前的实际情况：**
- Anthropic #6 数据：Opus 4.6 的 compaction 质量达 84%——说明模型已经能做较好的上下文筛选
- 但 84% 不是 100%，16% 的信息丢失在长时间任务中可能累积成致命错误
- LangChain 的"笨办法"在当前模型能力下可能更稳健，因为它们是确定性的

**开放问题：** `getEvents()` 的切片策略由谁决定？如果是模型决定，那就回到了"模型能否做好上下文管理"的老问题。如果是 harness 决定，那 session 外部存储的灵活性就被 harness 的启发式策略限制了——和 LangChain 方案有什么本质区别？

---

## 张力 3："多大脑、多双手"的认知负担

**文章的乐观叙事：** "Claude 必须推理多个执行环境并决定将工作发送到哪里"，随着智能扩展，这不再是问题。

**现有证据的悲观信号：**
- Anthropic #4：Sonnet 4.5 连单容器内的 self-evaluation 都做不好
- HumanLayer："便宜模型做子任务，贵模型做编排"——编排消耗大量认知预算
- Anthropic #6：上下文焦虑在单环境下就存在

**质疑：** 多双手意味着模型需要同时维护多个执行环境的心智模型。这和"上下文焦虑"直接矛盾——模型在单环境下都焦虑，多环境下能不焦虑？

**可能的反驳：** Opus 4.5/4.6 已经消除了上下文焦虑。但消除焦虑 ≠ 具备多环境编排能力。文章用 TTFT 数据证明了基础设施收益，但完全没给出多环境下的任务完成率数据。

---

## 张力 4：安全边界的转移，而非消除

**Managed Agents 的安全方案：** 令牌不进沙箱，通过 MCP proxy 和 vault 隔离。

**质疑：** `execute(name, input) → string` 本身就是能力入口。如果 Claude 能通过工具调用 provision 新容器、传递任意 input，那攻击面只是从"沙箱内读环境变量"转移到了"说服 harness 路由恶意工具调用"。

文章没有讨论：
- Harness 层的工具调用审计机制
- 对 `provision({resources})` 的权限限制
- 大脑传递双手给其他大脑时的信任链

**与 HumanLayer 的对比：** HumanLayer 明确提出了 MCP 的信任边界问题（"别连不信任的 MCP"）。Managed Agents 的架构假设所有 MCP proxy 都是可信的，但没有论证为什么。

---

## 张力 5：缺失的成本维度

**已有成本数据：**
- Anthropic #4：Solo $9 → V1 $200 → V2 $125
- YDD："AI 写得快但交付没变"的效率悖论

**Managed Agents 给了什么：** TTFT 性能数据，但零成本数据。

**质疑：** 多大脑 + 多双手 + 持久化 session + MCP proxy + vault = 显著的基础设施和推理成本。如果 meta-harness 的运营成本高于它节省的工程师时间，那它只适用于 Anthropic 自己这个量级的组织。

对于 YDD 讨论的中小团队，meta-harness 可能是过度工程。HumanLayer 的"简单开始，按需添加"可能更务实。

---

## 总体判断

Managed Agents 的价值不在具体方案，而在于它标记了 harness engineering 演进的方向：**从设计 harness 到设计容纳 harness 的系统**。

但这篇文章比 repo 中已有的 6 篇更"愿景导向"——它展示的是 Anthropic 希望世界变成的样子，而不是世界现在的样子。已有文章反而更诚实地面对了模型的局限性：
- LangChain 承认 context rot 没有银弹
- Anthropic #4 承认 self-evaluation 会失败
- HumanLayer 承认大多数问题是 skill issue 而非架构问题
- YDD 用数据证明效率提升不等于交付提升

**Managed Agents 回避的核心问题：模型的认知能力是否真的跟上了架构的野心。**
