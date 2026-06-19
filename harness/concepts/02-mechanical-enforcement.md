# 概念 2：机械化执行

## 核心思想

> 通过强制执行不变量，而非对实施过程进行微观管理

文档会腐烂。人会忘记。但 lint 规则和 CI 检查每次都会执行。

## 两类约束

### 架构约束（结构测试）
- 域内分层顺序：Types → Config → Repo → Service → Runtime → UI
- 依赖方向只能向前
- 横切关注点必须通过 Providers 进入
- 违反 = CI 阻塞合并

### 品味不变式（自定义 linter）
- 结构化日志（禁止 console.log 裸输出）
- Schema/类型的命名约定
- 文件大小限制
- 平台特定的可靠性要求

## 关键设计：lint 错误信息 = 修复指令

```
❌ 普通做法：
Error: File exceeds 500 lines.

✅ Harness 做法：
Error: File exceeds 500 lines.
Fix: Split into domain-specific modules following docs/ARCHITECTURE.md#splitting-guide.
Consider extracting types to <domain>/types/ and service logic to <domain>/service/.
```

错误信息中注入智能体可执行的修复路径 → 自我纠正闭环。

## 哲学

> 在中央层面强制执行边界，在本地层面允许自主权。

类似大型工程平台组织的管理模式：
- **严格的**：边界、正确性、可重复性
- **自由的**：边界内的具体实现方式
- 生成的代码不符合人类风格偏好？没关系。正确 + 可维护 + 智能体可读 = 达标。

## 来自其他文章的补充

### OpenAI Symphony — 给目标，不规定状态转换

OpenAI Symphony（[references/articles.md #16](../references/articles.md)）提供了**机械化执行的反向边界**。OpenAI 的工程师早期把智能体当作状态机里的刚性节点，每个状态规定智能体只能做特定动作。文章原话：

> "把智能体当作状态机里的刚性节点并不好用。模型会变得更聪明，也能解决比我们预设框架更大的问题。"

最终他们转向 **给目标，不规定状态转换**——给智能体目标 + 工具 + 上下文，让推理能力决定路径。

**这与机械化执行不矛盾，而是分工**：
- **机械化层**：约束**结果形态**（不变量、架构、品味），CI 强制
- **目标层**：约束**意图与边界**（要解决什么问题、不能做什么），不规定路径

机械化执行管"产出必须满足什么"，目标层管"为什么做、做到什么程度"。把两者都收紧成"必须按 1-2-3 步骤"会浪费模型的推理能力，也会随模型升级越来越显得笨拙。
