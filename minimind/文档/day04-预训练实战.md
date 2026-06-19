# Day 04：预训练 —— 从零开始训练模型

> 今天理解：训练循环的每一步在做什么，以及为什么要这样做。

---

## 一、预训练的本质

预训练阶段只做一件事：喂给模型海量纯文本，让它反复练习"猜下一个字"。

- 数据：`pretrain_t2t_mini.jsonl`，127 万条纯文本，1.2GB
- 目标：把 loss 从约 8.8（随机状态）降到尽可能低
- 产出：`pretrain_768.pth`（模型权重文件）

---

## 二、数据预处理：从文件到训练样本

`PretrainDataset` 类的 `__getitem__` 方法处理过程：

```python
# 1. 从 JSONL 文件读取一条文本
text = "给我生成一首有关秋天的诗歌。秋日早晨，清风拂面..."

# 2. Tokenize（保留 2 个位置给 BOS 和 EOS）
tokens = tokenizer(text, max_length=max_length - 2, truncation=True)
# → 约 353 个 token

# 3. 添加 BOS（句首）和 EOS（句尾）
tokens = [bos_token_id] + tokens + [eos_token_id]
# → 355 个 token
# BOS 告诉模型"句子开始"，EOS 让模型学会主动停止生成

# 4. Padding 到固定长度
tokens = tokens + [pad_token_id] * (512 - len(tokens))
# → 512 个 token，其中 157 个是 PAD

# 5. 生成 labels（PAD 位置 = -100，不参与损失）
labels = tokens.clone()
labels[labels == pad_token_id] = -100
```

真实数据统计：长度中位数 229 字符 → 约 173 token。Pretrain 的 `max_seq_len=340` 覆盖约 70% 数据不截断。

---

## 三、训练循环的 8 个步骤

```python
for step, (input_ids, labels) in enumerate(loader):
    # 步骤 1：数据搬运（CPU → GPU）
    input_ids = input_ids.to(device)
    labels = labels.to(device)

    # 步骤 2：学习率调度
    lr = cosine_schedule(step, total_steps, base_lr=5e-4)
    for pg in optimizer.param_groups: pg['lr'] = lr

    # 步骤 3：前向传播
    with autocast(dtype=bfloat16):          # 混合精度
        out = model(input_ids, labels=labels)
        loss = (out.loss + out.aux_loss) / accumulation_steps

    # 步骤 4：反向传播
    scaler.scale(loss).backward()           # 梯度计算

    # 步骤 5：参数更新（累积步数到了才执行）
    if step % accumulation_steps == 0:
        scaler.unscale_(optimizer)           # 解除 scaling
        clip_grad_norm_(params, 1.0)         # 梯度裁剪
        scaler.step(optimizer)               # 更新参数
        scaler.update()                      # 更新 scaling 因子
        optimizer.zero_grad()                # 清零梯度

    # 步骤 6：日志
    if step % log_interval == 0:
        print(f'loss: {loss:.4f}, lr: {lr:.8f}')

    # 步骤 7：保存检查点
    if step % save_interval == 0:
        torch.save(model.state_dict(), f'out/pretrain.pth')

    # 步骤 8：释放显存
    del input_ids, labels, out, loss
```

---

## 四、关键机制详解

### Cosine 学习率调度

```python
def get_lr(step, total, base_lr):
    return base_lr * (0.1 + 0.45 * (1 + cos(π * step / total)))
```

数值变化：
```
step  0%：lr = 5.00e-4（100%）
step 50%：lr = 2.75e-4（55%）
step 100%：lr = 5.00e-5（10%）
```

为什么不降到 0？保留 10% 底限确保训练末期仍能继续学习。如果降为 0，最后几百步完全浪费。

### 梯度累积

```python
# 本质：loss/N 后反向 N 次，再统一更新
for i in range(N):
    (loss / N).backward()   # 梯度叠加（不清零）
optimizer.step()             # 一次更新
optimizer.zero_grad()
```

为什么要除以 N？不除以 N，累积的梯度是原来的 N 倍 —— 等价于学习率 × N，训练会炸。

MiniMind 默认设置：`batch_size=32, accumulation_steps=8`，等效 batch = 256。这样在单张显卡上就能模拟大 batch 的稳定训练效果。

### 混合精度训练

```python
# bfloat16（推荐，新 GPU 和 Apple Silicon 支持）
with torch.cuda.amp.autocast(dtype=torch.bfloat16):
    loss = model(x)

# float16（老 GPU 需要 GradScaler）
scaler = torch.cuda.amp.GradScaler()
scaler.scale(loss).backward()  # loss 放大防止梯度下溢
```

bfloat16 的优势：指数位数和 float32 相同（8 位），因此值的范围一致，不需要 loss scaling。float16 指数位只有 5 位，范围窄很多，需要 GradScaler 动态调整放大倍数。

CPU 训练时使用 `nullcontext()`——全精度 float32，不需要混合精度。

### 梯度裁剪

```python
clip_grad_norm_(model.parameters(), max_norm=1.0)
```

将所有参数的梯度的 L2 范数限制在 1.0 以内。相当于一个"安全阀"：某次异常数据产生的超大梯度被自动压缩，不会破坏前期训练成果。

什么时候梯度会异常大？训练初期（参数完全随机）、数据中极罕见 token 组合、batch 太小导致方差大。

---

## 五、Pretrain vs SFT 参数对比

| 参数 | Pretrain | SFT | 原因 |
|------|----------|-----|------|
| learning_rate | 5e-4 | 1e-5 | SFT 在已收敛权重上微调，大步会破坏已有能力（灾难性遗忘） |
| max_seq_len | 340 | 768 | 预训练文本短（中位数 229 字），对话长（多轮 + think） |
| batch_size | 32 | 16 | 预训练 127 万条数据用大批次稳定；SFT 数据量小 |
| accumulation_steps | 8 | 1 | 预训练需要大等效 batch(=256)，SFT 不需要 |
| from_weight | none | pretrain | SFT 必须基于预训练权重 |

**灾难性遗忘**：如果 SFT 用 Pretrain 的学习率（5e-4），模型会在几个 epoch 内丧失语言能力。因为大学习率相当于"跳出"了预训练找到的最优区域，落到一个完全不同的地方。

---

## 六、动手实验

### 实验 1：可视化 Cosine 学习率

```bash
python3 << 'EOF'
import math
total, base = 2000, 5e-4
for step in range(0, total+1, 400):
    lr = base * (0.1 + 0.45 * (1 + math.cos(math.pi * step / total)))
    bar = '#' * int(lr/base*50)
    print(f"{step:4d} {lr:.6f} {bar} {lr/base*100:.0f}%")
EOF
```

### 实验 2：查看预训练数据长什么样

```bash
cd /Users/xiaoxiaoma/Documents/minimind
python3 << 'EOF'
import json
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("model")
with open("dataset/pretrain_t2t_mini.jsonl") as f:
    for i, line in enumerate(f):
        if i >= 3: break
        obj = json.loads(line)
        text = obj['text']
        n_tokens = len(tokenizer.encode(text))
        print(f"[样本{i+1}] {len(text)}字 → {n_tokens}token (压缩比 {len(text)/n_tokens:.1f})")
        print(f"  内容：{text[:80]}...\n")
EOF
```

---

## 今日总结

| 概念 | 公式/原理 | 为什么 |
|------|----------|--------|
| Cosine LR | `lr×(0.1+0.45×(1+cos))` | 平滑从 100% 降到 10%，比阶梯式稳定 |
| 梯度累积 | `(loss/N).backward()` 重复 N 次 | 小显存模拟大 batch |
| 混合精度 | bfloat16 前向 + float32 更新 | 速度 1.5×，精度几乎无损 |
| 梯度裁剪 | `clip_grad_norm(limit=1.0)` | 安全阀：一次坏 batch 不毁训练 |
| 灾难性遗忘 | 大 lr 破坏已收敛的权重 | SFT 必须用 Pretrain 的 1/50 学习率 |
