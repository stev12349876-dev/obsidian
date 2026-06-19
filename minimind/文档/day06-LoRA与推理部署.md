# Day 06：LoRA 高效微调与推理部署

> 今天理解：如何用 2% 参数微调模型，以及模型如何生成回复。

---

## 一、LoRA：低秩分解的微调方法

### 为什么需要 LoRA

全量微调需要更新全部 63.91M 参数。每换一个任务就要保存一份完整权重（128MB/任务），显存压力大，部署也麻烦。

LoRA（Low-Rank Adaptation）的解决方案：**冻结原始权重，只训练两个小矩阵。**

### 核心公式

```
W' = W + B × A

其中：
  W  ∈ R^{768×768}  （原始权重，冻结）
  A  ∈ R^{r×768}    （可训练，高斯初始化）
  B  ∈ R^{768×r}    （可训练，全零初始化）
  r  ≪ 768           （秩，通常取 8）
```

参数量对比（r=8）：
```
全量微调：768×768 = 589,824 参数（100%）
LoRA：    768×8 + 8×768 = 12,288 参数（2.1%）
节省 97.9%
```

### 为什么低秩有效

微调时的"变化量"ΔW 不需要整个 768×768 的空间。大多数适配任务只需要在少数几个方向（约 8 个）上调整权重。比如让模型更擅长医学问答，可能只需调整"术语理解""诊断逻辑"等少数语义轴。

### 为什么 B 初始化为全零

训练开始时 B·A = 0 → ΔW = 0 → 模型行为完全等同于原始模型。这是关键设计：LoRA 从"和原来完全一样"开始，逐渐学到新知识。如果 B 也随机初始化，一开始模型就已经偏离了原方向。

### 只对 Attention 加 LoRA

`apply_lora` 的筛选条件自动命中 Attention 的 Q/K/V/O 投影（都是方阵），不碰 FFN（非方阵）。这是 LoRA 论文的推荐做法：Attention 负责"任务适配"（不同任务关注点不同），FFN 负责"知识存储"（跨任务共享，不应轻易修改）。

### LoRA 的生命周期

```python
# 1. 应用：在 Attention 的 Linear 层上附加 A、B 矩阵
apply_lora(model, rank=8)       # forward 变为 Wx + BAx

# 2. 训练：只更新 A、B
train_lora.py

# 3. 保存：极小的文件（每层 12KB）
save_lora(model, "medical_lora.pth")

# 4. 加载 + 合并（部署用）
merge_lora(model, "medical_lora.pth", "merged.pth")  # W' = W + BA
```

---

## 二、generate()：模型如何生成回复

MiniMind 自实现了完整的生成循环，不依赖 transformers 默认实现，每一步都透明可见。

### 采样参数

```python
model.generate(
    temperature=0.85,         # 温度：控制随机性（低=保守，高=创意）
    top_p=0.85,               # 核采样：累积概率到 85% 后截断
    top_k=50,                 # 只保留概率最高的 50 个候选
    repetition_penalty=1.0,   # 重复惩罚：>1 降低已出现 token 的概率
    max_new_tokens=512,       # 最多生成 512 个新 token
)
```

### Temperature 的作用

```python
logits = logits / temperature     # 缩放
probs = softmax(logits)           # 转为概率
```

| T | 行为 | 适用场景 |
|----|------|---------|
| 0.1~0.3 | 接近确定性输出 | 翻译、代码生成 |
| 0.7~1.0 | 平衡准确与多样 | 通用对话 |
| 1.2~2.0 | 更随机、更有创意 | 写作、头脑风暴 |

### Top-K 和 Top-P 的配合

先 Top-K（硬截断到 50 个候选），再 Top-P（动态截断到 85% 累积概率）。两者互补：Top-K 对长尾分布不够灵活，Top-P 对尖峰分布过于保守。

### 重复惩罚

```python
# 不是禁止重复，而是降低概率
已出现 token 的概率 /= repetition_penalty
```

`repetition_penalty=1.2` 意味着已出现 token 的概率降低约 17%。

### KV Cache：推理加速

不缓存时，每一步生成都需要重新计算整个序列的 K 和 V——token 越多越慢。KV Cache 把历史 K、V 存起来，每一步只算新 token，速度提升约 N 倍（N = 生成步数）。

---

## 三、chat_terminal.py：完整对话链路

```python
# 1. 加载（一次性）
model = AutoModelForCausalLM.from_pretrained("weights/minimind-3")
tokenizer = AutoTokenizer.from_pretrained("weights/minimind-3")

# 2. 初始化对话历史
messages = [{"role": "system", "content": "你是MiniMind，一个有用的AI助手。"}]

while True:
    # 3. 用户输入
    user_input = input("你: ")
    messages.append({"role": "user", "content": user_input})

    # 4. Chat Template → Tokenize
    text = tokenizer.apply_chat_template(messages, add_generation_prompt=True)
    # → "<|im_start|>system\n...<|im_end|>\n<|im_start|>user\n...<|im_end|>\n<|im_start|>assistant\n"
    inputs = tokenizer(text, return_tensors="pt")

    # 5. 生成
    outputs = model.generate(**inputs, max_new_tokens=512)

    # 6. 解码（只取新生成部分，跳过输入）
    response = tokenizer.decode(
        outputs[0][inputs['input_ids'].shape[1]:],
        skip_special_tokens=True
    )
    print(f"MiniMind: {response}")

    # 7. 加入历史（多轮对话的基础）
    messages.append({"role": "assistant", "content": response})
```

关键细节：`add_generation_prompt=True` 在文本末尾追加 `<|im_start|>assistant\n<think>\n\n</think>\n\n`，这是告诉模型"轮到你了，开始回答"。空的 `<think></think>` 表示"不需要特殊思考过程"。

---

## 四、动手实验

### 实验 1：验证 LoRA 等价性

```bash
cd /Users/xiaoxiaoma/Documents/minimind
python3 << 'EOF'
import torch
from model.model_lora import LoRA

lora = LoRA(768, 768, 8)
W = torch.randn(768, 768)
x = torch.randn(2, 768)
BA = lora.B.weight @ lora.A.weight

out1 = x @ (W + BA).T      # 全量: (W+BA)x
out2 = x @ W.T + lora(x)   # LoRA: Wx + BAx

print(f"等价性验证差异: {(out1-out2).abs().max():.10f}")
print(f"全量参数: {768*768:,}")
print(f"LoRA参数: {768*8*2:,}  (节省 {(1-768*8*2/(768*768))*100:.0f}%)")
EOF
```

### 实验 2：看模型下一个 token 的预测分布

```bash
cd /Users/xiaoxiaoma/Documents/minimind
python3 << 'EOF'
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch.nn.functional as F

tokenizer = AutoTokenizer.from_pretrained("model")
model = AutoModelForCausalLM.from_pretrained("weights/minimind-3", trust_remote_code=True)
model.eval()

msg = [{"role": "user", "content": "1+1="}]
text = tokenizer.apply_chat_template(msg, tokenize=False, add_generation_prompt=True)
inputs = tokenizer(text, return_tensors="pt")

with torch.no_grad():
    logits = model(**inputs).logits[0, -1, :] / 0.85
    probs = F.softmax(logits, -1)
    top5 = torch.topk(probs, 5)
    print("下一个 token 的 Top-5 预测：")
    for i in range(5):
        print(f"  {tokenizer.decode([top5.indices[i].item()])!r:10s} 概率 {top5.values[i].item():.4f}")
EOF
```

---

## 今日总结

| 概念 | 公式/原理 | 关键 |
|------|----------|------|
| LoRA | W' = W + BA, r=8 | 2% 参数，B 初始化为 0 保证平稳起步 |
| Temperature | logits/T → softmax | 创意旋钮 |
| Top-K + Top-P | 先硬截断，再动态截断 | 互补过滤 |
| KV Cache | 复用历史 K/V | 推理加速 N 倍 |
| `add_generation_prompt` | 追加 `<\|im_start\|>assistant` | 信号：轮到你了 |
