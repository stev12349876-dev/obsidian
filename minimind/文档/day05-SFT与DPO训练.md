# Day 05：SFT 与 DPO —— 让模型学会对话与偏好

> 今天理解：模型如何从"会说话"变成"会聊天"。

---

## 一、SFT 的核心问题：选择性学习

Pretrain 阶段，模型学会了语言能力——但它不知道自己是"助手"。你问它"你是谁"，它可能接着编一段故事，而不是自我介绍。

SFT 的目标：让模型学会对话格式和指令跟随。

但对话数据中有很多"不该学"的部分：

```
<|im_start|>system
你是MiniMind<|im_end|>               ← 系统提示，不应学习
<|im_start|>user
你是谁<|im_end|>                      ← 用户输入，不应学习
<|im_start|>assistant
我是MiniMind<|im_end|>                ← 这是模型要学习的内容
```

**关键设计：只让模型学习 assistant 的回复，其他部分全部跳过。**

---

## 二、选择性 Label 的生成算法

`SFTDataset.generate_labels` 的核心逻辑：找到每一段 `<|im_start|>assistant...<|im_end|>`，只把这一段标记为有效学习目标，其余标记为 -100（忽略）。

```python
def generate_labels(self, input_ids):
    labels = [-100] * len(input_ids)    # 全部初始化为忽略

    bos = tokenizer('<|im_start|>assistant\n', ...).input_ids  # 5 个 token
    eos = tokenizer('<|im_end|>\n', ...).input_ids              # 2 个 token

    i = 0
    while i < len(input_ids):
        # 检测到 assistant 开始标记
        if input_ids[i:i+5] == bos:
            start = i + 5           # 回复内容开始
            end = start
            # 找到最近 EOS
            while end < len(input_ids):
                if input_ids[end:end+2] == eos: break
                end += 1
            # 整段回复（含 EOS）标记为有效
            for j in range(start, end + 2):
                labels[j] = input_ids[j]
            i = end + 2
        else:
            i += 1
```

真实数据：一条 7 轮对话的 SFT 数据，549 个 token 中 414 个（75%）为有效 label（7 段 assistant 回复），135 个（25%）被忽略（7 段 user 输入 + 格式标记）。

---

## 三、为什么 SFT 学习率是 Pretrain 的 1/50

| 阶段 | 学习率 | 原因 |
|------|--------|------|
| Pretrain | 5e-4 | 从零开始，大步探索 |
| SFT | 1e-5 | 在已收敛的权重上微调，不能跑偏 |

灾难性遗忘的机制：预训练权重在 loss 曲面上处于一个"深谷"——最优语言能力区域。用大学习率做 SFT 相当于"跳出深谷"——模型会忘记基本语言能力。

小学习率只在谷底做微调——改变对话风格，不改变语言基础。

---

## 四、DPO：为什么需要偏好对齐

SFT 后模型会对话了，但可能：
- 啰嗦：简单问题回答过长
- 机械：语气像模板
- 不一致：类似问题回答风格迥异

DPO 的解法：给模型看"同一问题的好回答和差回答"，让它学会偏好。

### DPO 损失函数

```
DPO 的目标：让"好回答"的概率上升、"差回答"的概率下降，
           但不能偏离原始（SFT）模型太远。

ratio_chosen  = log(π_θ(好回答)) - log(π_ref(好回答))
ratio_rejected = log(π_θ(差回答)) - log(π_ref(差回答))

diff = ratio_chosen - ratio_rejected
loss = -log(sigmoid(β × diff))
```

- β = 0.15（默认）：控制学习强度。β 越大→越激进追求偏好，β 越小→越保守接近参考模型
- `π_ref`：完全冻结的 SFT 模型，作为"参照系"。没有它，DPO 可能无差别提升所有回答的概率（包括差回答）

### DPO 不需要单独训练奖励模型

经典 RLHF 需要三步：训练奖励模型 → 用强化学习优化 → KL 约束。

DPO 通过数学推导发现：最优策略可以直接用"好回答 vs 差回答的对数概率比"来表达，跳过奖励模型训练和 RL 算法环节。这大大简化了训练流程。

---

## 五、完整训练链路

```
Pretrain（lr=5e-4）→ pretrain_768.pth
    ↓
SFT（lr=1e-5）→ full_sft_768.pth
    ↓
DPO（lr=1e-6, β=0.15）→ dpo_768.pth
```

学习率从 5e-4 → 1e-5 → 1e-6，每个阶段下降 10~50 倍。这反映了从"建立基础"到"精细打磨"的递进过程。

---

## 六、动手实验

### 实验 1：观察 SFT 数据的 Label 分布

```bash
cd /Users/xiaoxiaoma/Documents/minimind
python3 << 'EOF'
import json
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("model")
with open("dataset/sft_t2t_mini.jsonl") as f:
    sample = json.loads(f.readline())

prompt = tokenizer.apply_chat_template(sample['conversations'], tokenize=False, add_generation_prompt=False)
ids = tokenizer(prompt)['input_ids']
bos = tokenizer('<|im_start|>assistant\n', add_special_tokens=False).input_ids
eos = tokenizer('<|im_end|>\n', add_special_tokens=False).input_ids

labels = [-100] * len(ids)
i = count = 0
while i < len(ids):
    if ids[i:i+len(bos)] == bos:
        start = i + len(bos)
        end = start
        while end < len(ids):
            if ids[end:end+len(eos)] == eos: break
            end += 1
        for j in range(start, min(end+len(eos), len(ids))):
            labels[j] = ids[j]
        count += 1
        text = tokenizer.decode(ids[start:min(end+len(eos), len(ids))])
        print(f"第{count}段 assistant: {text[:60].replace(chr(10),' ')}")
        i = end + len(eos) if end < len(ids) else len(ids)
    else:
        i += 1

valid = sum(1 for l in labels if l != -100)
print(f"\n总 token: {len(ids)}, 有效: {valid} ({valid/len(ids)*100:.0f}%)")
EOF
```

---

## 今日总结

| 概念 | 核心 |
|------|------|
| SFT Label | 只标记 assistant 回复为有效学习目标 |
| 忽略机制 | `ignore_index=-100`，PyTorch 自动跳过 |
| 灾难性遗忘 | SFT lr = Pretrain lr / 50 |
| DPO | 不需要奖励模型，直接用对比数据做对齐 |
| 参考模型 | 冻结的 SFT 模型，防止 DPO 退化 |
