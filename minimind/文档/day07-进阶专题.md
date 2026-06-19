# Day 07：进阶专题 —— MoE、强化学习、蒸馏

> 今天概览 MiniMind 的进阶能力，理解每个技术的核心思想和适用场景。

---

## 一、MoE（混合专家）

传统模型中，每个 token 都要经过全部 FFN 参数（5.60M×8=44.83M）。MoE 的洞见：不是所有参数都要参与计算。

```
传统：token → 整个 FFN(5.60M) → 输出
MoE： token → 路由器 → 专家2(1.87M) → 输出  （只激活 1/4）
                专家1、3、4 不激活
```

总参数变多了（4 个专家），但计算量不变（只走 1 个）。相当于用更多显存放更大的模型容量。

### 路由机制

```python
scores = softmax(gate(x))           # [B, L, 4]  对 4 个专家的亲和度
topk_weight, topk_idx = topk(scores, k=1)  # 选最高分
# 只把 token 送给选中的专家
for i, expert in enumerate(experts):
    if topk_idx == i:
        output += expert(x) * topk_weight
```

### 辅助损失

路由器可能退化——只选专家 0，忽视其他人。辅助损失惩罚不均衡分配：

```python
load = one_hot(topk_idx, 4).mean(0)     # 每个专家的被选概率
aux_loss = (load * scores.mean(0)).sum() * 4 * 5e-4
```

---

## 二、强化学习：PPO 与 GRPO

### PPO（需要 Critic）

四大组件：Actor（生成回答）、Critic（估计分数）、Reward Model（实际打分）、Reference Model（KL 约束）。PPO 同时训练 Actor 和 Critic，复杂但功能强大。

### GRPO（不需要 Critic）

GRPO 是对同一个 prompt 生成多条回答（如 4 条），取组内平均作为"基准线"。高于基准线→增加概率，低于→减少概率。组内归一化让训练更稳定，免去了训练 Critic 的负担。

### MiniMind 的混合奖励

```python
# 规则奖励（快、可解释）
if 20 <= len(response) <= 800: reward += 0.5    # 长度合适
if response.count('</think>') == 1: reward += 0.25  # 标记正确
reward -= rep_penalty(response)                  # 惩罚重复

# 模型奖励（精确）
reward += reward_model.get_score(response)
```

---

## 三、知识蒸馏

让大模型（Teacher）教小模型（Student）的不只是"答案是什么"，而是"怎么思考"。

```python
soft_loss = KL(softmax(student_logits/T), softmax(teacher_logits/T)) * T^2
total_loss = α * hard_loss + (1-α) * soft_loss
```

高温度 T（如 4）让概率分布更"软"——次优选项的概率被放大，Teacher 的"偏好"信息更明显。Student 通过学习这些相对关系，比单纯学硬标签能获得更多知识。

---

## 四、Agent RL：工具调用

让模型学会使用工具（计算器、搜索引擎等）：

```
<tool_call>{"name": "calc", "args": [123, 456]}</tool_call>
```

训练数据包含完整的多步交互：用户提问 → 模型决定调用工具 → 系统返回结果 → 模型给出最终答案。Agent RL 的奖励信号来自最终结果是否正确。

---

## 五、技术选型速查

| 需求 | 方案 |
|------|------|
| 快速微调特定任务 | LoRA (r=8) |
| 提升对话质量 | SFT → DPO |
| 扩大模型容量 | MoE (num_experts=4) |
| 复杂对齐 | GRPO |
| 模型压缩 | 知识蒸馏 |
| 工具使用 | Agent RL |

---

## 六、七天总结

| 天数 | 如果只记住一个概念 |
|------|-------------------|
| Day 01 | 语言模型 = 自回归概率预测；BPE = 频率合并 |
| Day 02 | Attention = Q问K答V提取；RoPE = 旋转编码位置 |
| Day 03 | FFN = 70%参数存知识；残差 = 梯度保底通道 |
| Day 04 | 梯度累积 + Cosine LR + AMP = 高效稳定训练 |
| Day 05 | SFT 只学 assistant；DPO 对比对齐 |
| Day 06 | LoRA = 2% 参数微调；Temperature = 创意旋钮 |
| Day 07 | MoE 参数多计算不变；GRPO 不需要 Critic |
