# 深入 Transformer：从 Embedding 到完整前向传播

> 目标：彻底理解 Transformer 的每一个组件，能逐层说出数据怎么流动。不跳步，不简化。

---

## 一、从一个具体例子开始

我们追踪一句话经过整个模型的过程：

```
输入："今天天气真好"
```

这条信息经过以下路径：

```
"今天天气真好"
  → Tokenizer → [5640, 3660, 2131, 658]   (4个数字)
  → Embedding → [4, 768] 矩阵               (每个字变成768维向量)
  → 8层 Transformer → [4, 768]              (shape不变，内容不断深化)
  → LM Head → [4, 6400]                     (每个位置预测6400个字的概率)
  → argmax → 选概率最大的下一个字
```

今天我们详细拆解中间那个"8 层 Transformer"到底做了什么。

---

## 二、为什么需要这么多组件？先看全景

一层 Transformer 包含 4 个核心操作：

```
输入 h ──────────────────────────────────────────────┐
  │                                                    │
  ├─→ RMSNorm ─→ Attention ─→ (+) ─┐                  │
  │                            │    │                  │
  │                            └────┘ (残差：加回 h)   │
  │                                                    │
  ├─→ RMSNorm ─→ FFN ────────→ (+) ─┘                 │
  │                            │                       │
  │                            └── (残差：加回上一步结果)
  │
  └─→ 输出 h'  (shape 完全不变，值变了)
```

**每个组件的职责：**

| 组件 | 职责 | 没有它会怎样 |
|------|------|------------|
| RMSNorm | 稳定数值范围 | 8 层后数值爆炸或消失，梯度无法回传 |
| Attention | 找关系："她"指的是谁？ | 模型不知道字与字之间的关联 |
| FeedForward | 存知识 + 做推理 | Attention 找到了关系但不知道怎么回答 |
| 残差连接 | 梯度"高速路" | 深层网络梯度消失，前几层完全学不动 |

下面逐个深入。

---

## 三、Embedding：字是怎么变成向量的

### 不是什么魔法，就是查表

```python
# model_minimind.py 中的定义
self.embed_tokens = nn.Embedding(6400, 768)
#                          ↑      ↑
#                     词表大小  每个字的维度
```

本质就是一个 **6400 行 × 768 列**的矩阵。当你输入 token ID = 5640（"今"）：

```python
vector = embed_tokens.weight[5640]  # 取出第 5640 行 → 768 维向量
```

### 为什么是 768 维，不是 1 维也不是 10000 维？

```
1 维：   "今" = [0.5]         → 太简单，无法表达任何语义差异
768 维： "今" = [0.03, -0.05, ...]  → 够丰富，能区分微妙含义
10000维："今" = [0.01, 0.02, ...]   → 太多了，浪费参数和计算

768 = 能被 8（注意力头数）和 64（GPU对齐）整除的最小标准维度
```

### 训练前后 Embedding 的变化

```
训练前（随机初始化）：
  "猫"向量 和 "狗"向量 → 余弦相似度 ≈ 0（随机，互相正交）
  "猫"向量 和 "计算机"向量 → 余弦相似度 ≈ 0

训练后（学到语义）：
  "猫"向量 和 "狗"向量 → 余弦相似度 ≈ 0.65（动物，语义相近）
  "猫"向量 和 "计算机"向量 → 余弦相似度 ≈ 0.05（无关）
```

Embedding 不是"定义"出来的，是"学"出来的。训练过程中，梯度反向传播不断调整这个矩阵，最终语义相近的词自然靠近。

### 一个重要的设计：Weight Tying

```python
# model_minimind.py
self.model.embed_tokens.weight = self.lm_head.weight  # 同一个 Tensor！
```

输入 Embedding 和输出 LM Head 共享权重。为什么？

```
Embedding: Token ID → 768 维向量  （用矩阵 W）
LM Head:   768 维向量 → Token ID  （用矩阵 W^T）

如果共享权重：
  "猫"在输入空间中靠近"狗" → 在输出空间中也靠近"狗"
  → 逻辑一致："意思相近的词，被预测出来的概率关系也相近"

如果不共享：
  输入空间里的"近"和输出空间里的"近"是完全独立的
  → 模型需要在两个空间中各自学习，浪费参数且不稳定
```

---

## 四、RMSNorm：为什么需要归一化

### 问题：数值不稳定

```
假设没有归一化，一层 Transformer 的计算：
  h₁ = h₀ + Attention(h₀)    数值范围 ≈ [0.5, 0.5] → [0.5, 5.0]
  h₂ = h₁ + FFN(h₁)          数值范围 ≈ [0.5, 5.0] → [0.5, 50.0]
  ...
  h₈ = ...                   数值范围 ≈ [0.5, 5000] → 梯度爆炸！

或者反过来：
  h₁ = h₀ + Attention(h₀)  数值 ≈ [0.5, 0.5] → [0.5, 0.05]
  h₈ = ...                  数值 ≈ [0.5, 0.0001] → 梯度消失！
```

### RMSNorm 的解决：每一步都把数值拉回稳定范围

```python
class RMSNorm(nn.Module):
    def __init__(self, dim=768, eps=1e-6):
        self.weight = nn.Parameter(torch.ones(768))  # 可学习参数

    def forward(self, x):
        # 1. 计算 RMS（均方根）
        rms = (x.pow(2).mean(-1, keepdim=True) + self.eps).sqrt()
        # 2. 除以 RMS → 此时 RMS = 1
        x_normed = x / rms
        # 3. 乘以可学习的权重
        return x_normed * self.weight
```

**操作拆解（以一个 768 维向量为例）：**

```
原始向量: [0.03, -0.05, 0.12, 0.01, -0.02, ...]  (768个数)
                                            RMS ≈ 0.05
步骤 1：除以 RMS
        [0.03/0.05, -0.05/0.05, 0.12/0.05, ...]
      = [0.6, -1.0, 2.4, 0.2, -0.4, ...]       RMS = 1.0 ✓

步骤 2：乘以 weight
      = [0.6×1.85, -1.0×1.85, 2.4×1.85, ...]   RMS ≈ 1.85
                              ↑
                        weight 训练后 ≈ 1.85
```

**为什么 weight 不等于 1？** 初始化为全 1，但训练后会被调整到最优值。这个例子中模型发现"放大约 1.85 倍后续效果最好"。

### 为什么不用 BatchNorm，而用 RMSNorm？

```
BatchNorm: 按"列"归一化（跨 batch 中所有样本）
  需要足够大的 batch → 文本生成中 batch 经常很小或为 1 → 不稳定

RMSNorm: 按"行"归一化（每个样本自己算自己的）
  不需要其他样本 → batch=1 也完全稳定
  不需要"减均值"步骤 → 比 LayerNorm 快约 15%
```

---

## 五、Attention：三个角色一台戏

### Q、K、V 不是凭空想出来的

把 Attention 想象成**搜索引擎**：

```
你去图书馆找"机器学习"相关的书：

  Q（Query，你的需求）: "我想了解机器学习"  → 一个表示你需求的向量
  K（Key，书的标签）:  每本书的标题和关键词  → 所有书的标签向量
  V（Value，书的内容）: 每本书的正文         → 所有书的内容向量

检索过程：
  1. Q 和每个 K 算相似度 → 找到最相关的几本书
  2. 根据相似度比例 → 对这几本书的 V 做加权平均
  3. 返回"融合了所有相关书籍信息"的结果
```

**在 Transformer 中，"图书馆"是同一个句子的所有字：**

```
句子："小明把苹果给了小红，她很开心"

对于"她"这个字（Q）：
  和"小明"（K）的相似度 = 0.2（小明是男的 → 不太像）
  和"小红"（K）的相似度 = 0.8（小红是女的 → 很像！）
  和"苹果"（K）的相似度 = 0.05（物 → 不像）
  和"开心"（K）的相似度 = 0.1（状态 → 不太像）

相似度 = Q · K 的内积。内积越大 → 两个向量方向越一致 → 越相关。
```

### 为什么需要三个矩阵（W_Q, W_K, W_V）？

**因为"找什么"、"怎么描述自己"、"存了什么信息"是三个不同的问题。**

如果用同一个投影矩阵：
- Q 和 K 会非常相似 → 每个字"找"的结果就是"自己是自己"
- 无法体现"她"去找"小红"这种跨越的行为

三个独立矩阵让 Q、K、V 在三个不同的空间中运作，互不干扰。

### 完整计算：从一个 token 的视角

```
设句子长度为 L=6，每个 token 768 维，head_dim=96

步骤 1：Q、K、V 投影
  "她"这个 token (768维)
    → W_Q (768×768) → Q_她 [96维]   "她在找什么？"
    → W_K (768×384) → K_她 [96维]   "她怎么描述自己？"
    → W_V (768×384) → V_她 [96维]   "她有什么信息？"

步骤 2：计算注意力分数
  scores = Q_她 · [K₁, K₂, K₃, K₄, K₅, K₆]^T
        = [0.2, 0.15, 0.8, 0.05, 0.1, 0.3]  (6个分数)

步骤 3：缩放
  scaled = scores / √96 = scores / 9.8

步骤 4：softmax → 概率
  weights = [0.05, 0.04, 0.75, 0.01, 0.02, 0.13]
           (关注"小红"最多，权重 75%)

步骤 5：加权求和
  output = 0.05×V_1 + 0.04×V_2 + 0.75×V_3 + 0.01×V_4 + 0.02×V_5 + 0.13×V_6
         ≈ 主要是 V_3（"小红"）的信息 + 少量其他字的信息
```

### 为什么要除以 √d_k？

```
假设 Q 和 K 的每个维度是独立随机变量（均值 0，方差 1）：
  Q·K = q₁k₁ + q₂k₂ + ... + q₉₆k₉₆

每一项 q_i*k_i 的方差 ≈ 1，96 项加起来的方差 ≈ 96。
标准差 = √96 ≈ 9.8

如果不除以 9.8：
  Q·K 的值在 -30 到 +30 之间
  softmax(-30, 5, 10) → [0.000..., 0.007, 0.993]  ← 几乎只选最大的
  → 梯度几乎为 0（softmax 饱和）

除以 9.8 后：
  Q·K 的值在 -3 到 +3 之间
  softmax(-1.5, 0.5, 1.0) → [0.09, 0.38, 0.53]  ← 平滑分布
  → 梯度正常流动
```

### 多头的真正含义

不是"多头 = 多次计算然后平均"。而是**每个头有不同的 W_Q、W_K、W_V**，各自学会关注不同的模式：

```
8 个头的 W_Q 矩阵各不相同：
  头 0 的 W_Q 学到：关注"名词和动词"的关系
  头 1 的 W_Q 学到：关注"代词和指代对象"的关系
  头 2 的 W_Q 学到：关注"修饰词和被修饰词"的关系
  ...
  头 7 的 W_Q 学到：关注"标点和句子边界"

最后拼起来：8 个头 × 96 维 = 768 维 → 再投影一次（o_proj）
```

### GQA：K/V 头为什么可以少一半

实验发现：8 个 Q 头之间有冗余——很多头关注的内容是相似的。所以让 2 个 Q 头共享 1 组 K/V：

```
Q₀  Q₁  Q₂  Q₃  Q₄  Q₅  Q₆  Q₇   (8 个 Q 头，参数不变)
 ↕   ↕   ↕   ↕   ↕   ↕   ↕   ↕
 K₀      K₁      K₂      K₃        (4 个 K 头，每个被 2 个 Q 共享)
 V₀      V₁      V₂      V₃        (4 个 V 头)

好处：推理时 KV Cache (用来加速的缓存) 从 2×8×96×2bytes = 3KB/token
                                           降到 2×4×96×2bytes = 1.5KB/token
      对于大模型（70B），KV Cache 可能占几十 GB → 减半 = 省一半显存
```

### 因果掩码：为什么不能看未来

```
训练时，我们一次性输入完整句子 "今天天气真好"
模型同时处理 6 个字。

但它的任务是：用前面的字预测后面的字。

如果不加掩码：
  位置 0（"今"）可以看到位置 5（"好"） → 作弊！
  → 模型不需要"猜"了，直接偷看答案

加了因果掩码：
  位置 0（"今"）：只能看 [0]
  位置 1（"天"）：只能看 [0, 1]
  ...
  位置 5（"好"）：只能看 [0,1,2,3,4,5]
  → 每个位置只能看自己 + 所有前面的字
 
掩码实现：上三角设为 -inf
  [  0, -∞, -∞, -∞, -∞, -∞]
  [  0,   0, -∞, -∞, -∞, -∞]
  [  0,   0,   0, -∞, -∞, -∞]
  [  0,   0,   0,   0, -∞, -∞]
  [  0,   0,   0,   0,   0, -∞]
  [  0,   0,   0,   0,   0,   0]

softmax(-inf) = 0 → 看不见的位置完全不关注
```

---

## 六、RoPE：怎么把位置信息注入Attention

### 问题：Attention 不关心顺序

```
"我打你" → Attention 看到 {"我", "打", "你"}  ← 一个集合
"你打我" → Attention 看到 {"我", "打", "你"}  ← 完全一样的集合

没有位置信息 → 无法区分这两句话！
```

### RoPE 的做法：转圈圈

```
把 96 维向量分成 48 对（每对是二维平面上的一个点）：

位置 0（"今"）：48 对都不转（角度 = 0）
位置 1（"天"）：48 对各自转一个固定角度
  · 第 0 对转 1/1000000^0 = 1 弧度     ← 高频，区分相邻位置
  · 第 1 对转 1/1000000^(2/96) ≈ 0.75
  · 第 24 对转 1/1000000^(48/96) = 1/1000  ← 中频
  · 第 47 对转 1/1000000^(94/96) ≈ 1/1000000 ← 低频，看全局
位置 2（"天"）：每对转 2 倍角度
位置 3（"气"）：每对转 3 倍角度
```

### 为什么设计成高频和低频？

```
高频维度（转得快）：位置 0 和 位置 1 旋转差很大 → QK 内积差大
  → 相邻位置容易区分 → 负责"局部信息"（这个词前后的几个词）

低频维度（几乎不转）：位置 0 和 位置 100 旋转差也很小 → QK 内积差小
  → 远距离保持关联 → 负责"全局信息"（整段话的结构）
```

### 为什么是旋转而不是加法？

```
加法位置编码：x' = x + pos_encoding
  改变了向量的值 → 可能改变语义（"猫"+"位置3"的向量完全变了）

旋转位置编码：x' = rotate(x, pos)
  旋转是正交变换 → 不改变向量长度 → 不改变"语义强度"
  只改变 Q 和 K 之间的角度 → 不影响 V 的内容
```

**代码实现（核心就一行）：**

```python
# 对 Q 和 K 做旋转（V 不做！因为 V 只需要内容，不需要位置）
query = query * cos + rotate_half(query) * sin
key   = key   * cos + rotate_half(key)   * sin

# rotate_half：把每对 (a, b) 变成 (-b, a)
# 这正好是二维平面上的 90° 旋转
# 最终 query*cos + rotate_half(query)*sin 实现了任意角度的旋转
```

---

## 七、FeedForward：存 70% 参数的地方

### 为什么需要 FFN？

Attention 找到"她 → 小红"这个关联后，还需要：
1. 知道"小红"是什么（一个专有名词，指代人）
2. 知道"开心"是什么（一种情绪状态）
3. 推断出"给了苹果 → 所以开心"的因果

这些**常识和知识**不存储在 Attention 中，而是存储在 FFN 的权重矩阵里。

### SwiGLU：三个矩阵 + 门控

```python
class FeedForward(nn.Module):
    def __init__(self):
        # 768 → 展开到 2432 → 缩回 768
        self.gate_proj = nn.Linear(768, 2432)   # 门控
        self.up_proj   = nn.Linear(768, 2432)   # 信息
        self.down_proj = nn.Linear(2432, 768)   # 压缩

    def forward(self, x):
        gate = silu(self.gate_proj(x))   # [B,L,2432]  门控信号
        up   = self.up_proj(x)           # [B,L,2432]  信息内容
        return self.down_proj(gate * up) # [B,L,768]   门控 × 信息 → 压缩
```

**三个矩阵的分工（类比）：**

```
gate_proj：决定哪些神经元被激活
  → 像"管理层"：判断这件事属于什么类型的问题

up_proj：提供实际的神经元计算值
  → 像"员工"：动手处理具体信息

down_proj：把结果压缩回 768 维
  → 像"汇报"：把 2432 维的细节提炼成 768 维的摘要

gate * up = "管理层说这个员工不用干活" → 输出 = 0
          = "管理层说这个员工很重要" → 输出 = 员工的计算结果
```

### SiLU 的门控特性

```
silu(x) = x / (1 + e^(-x))

x 很大正值（比如 5）：
  sigmoid(5) ≈ 0.993 → silu(5) ≈ 4.97 → 几乎完全通过

x 很大负值（比如 -5）：
  sigmoid(-5) ≈ 0.007 → silu(-5) ≈ -0.035 → 几乎阻断

x = 0：
  sigmoid(0) = 0.5 → silu(0) = 0 → 完全阻断

和 ReLU 的区别：
  ReLU(-5) = 0（硬截断，梯度为 0 → "死神经元"）
  silu(-5) ≈ -0.035（平滑过渡，梯度不为 0 → 始终可学习）
```

### 为什么中间维度是 2432（不是 2048 也不是 3072）？

```python
intermediate_size = ceil(768 × π / 64) × 64
                  = ceil(37.7) × 64
                  = 2432

传统 FFN 扩展比：2~2.5 倍
SwiGLU 推荐扩展比：2.5~4 倍（因为有 3 个矩阵，实际活跃参数量被门控压缩了）
π ≈ 3.14 是"中间但偏大"的选择

如果选 2048（≈2.67倍）：FFN 总参数 4.72M → 容量不足
如果选 3072（=4倍）   ：FFN 总参数 7.08M → 模型太大，64M 不够

2432（≈3.17倍）刚好卡住 64M 总参数量 + 足够容量
```

### 为什么 FFN 占 70% 参数

```
Attention 的参数：4×768² ≈ 4 × 0.59M ≈ 2.36M
（Q: 768×768, K: 768×384, V: 768×384, O: 768×768）

但这只是没考虑 GQA 的情况。实际每层 Attention = 1.77M。

FFN 的参数：3×768×2432 ≈ 5.60M

一层中：1.77M (Attention) vs 5.60M (FFN)
        ≈ 24% vs 76%

Attention 负责"关系检索"——找到谁跟谁相关（不需要很大容量）
FFN 负责"知识存储"——存所有语法规则、常识、事实（需要大容量）

类比：
  Attention = 图书馆的检索系统（软件，很轻量）
  FFN       = 图书馆的所有藏书（知识，很庞大）
```

---

## 八、残差连接：8 层能堆起来的秘密

### 公式

```python
h = h + Attention(RMSNorm(h))   # 残差 1
h = h + FFN(RMSNorm(h))         # 残差 2
```

### 为什么没有残差就训不动

```
反向传播时，梯度从第 8 层传到第 1 层：

没有残差：
  ∂L/∂h₀ = ∂L/∂h₈ × ∂h₈/∂h₇ × ... × ∂h₁/∂h₀
  每一项 ≈ 0.9（小于 1）
  → 经过 8 次乘法 → 0.9^8 ≈ 0.43 → 梯度衰减为原来的 43%
  → 经过 50 层 → 0.9^50 ≈ 0.005 → 几乎消失了

有残差：
  h_{i+1} = h_i + F(h_i)
  ∂h_{i+1}/∂h_i = 1 + ∂F/∂h_i  ← 注意这个 "1 +"
  → 只要 ∂F/∂h_i > -1，梯度就不会衰减
  → 即使 100 层，梯度也能完整回传
```

"1" 就是残差连接的关键——它给了梯度一条**直通高速**，不用经过任何非线性变换。就像从 8 楼坐直达电梯下到 1 楼，而不是每层都换乘。

### Pre-Norm vs Post-Norm

```
Post-Norm（原始 Transformer 论文，已淘汰）:
  h = Norm(h + Attention(h))  ← 残差先加，然后归一化
  问题：残差可能很大，归一化后信息被压缩

Pre-Norm（现代大模型的标准做法）:
  h = h + Attention(Norm(h))  ← 先归一化，然后残差加
  优势：残差直接加了归一化后的干净值，梯度路径更短
```

MiniMind 使用的是 **Pre-Norm + RMSNorm** 组合，和 GPT-3、LLaMA 完全一致。

---

## 九、完整前向传播：MiniMindModel.forward 走读

```python
def forward(self, input_ids):
    B, L = input_ids.shape                          # [1, 50]

    # ---- 步骤 1：Embedding ----
    h = self.embed_tokens(input_ids)                # [1, 50, 768]
    h = self.dropout(h)                             # 训练时随机丢弃，推理时不丢

    # ---- 步骤 2：准备 RoPE ----
    cos = self.freqs_cos[:L]                        # [50, 96]
    sin = self.freqs_sin[:L]                        # [50, 96]

    # ---- 步骤 3~10：8 层 Transformer ----
    for layer in self.layers:                       # 8 次
        # 子层 1：Pre-Norm + Attention + 残差
        normed = layer.input_layernorm(h)           # [1, 50, 768]
        attn_out, kv = layer.self_attn(             # [1, 50, 768]
            normed, (cos, sin),                     # 带 RoPE
            past_key_value=None,
            use_cache=False,
            attention_mask=None
        )
        h = h + attn_out                            # 残差连接 1

        # 子层 2：Pre-Norm + FFN + 残差
        normed = layer.post_attention_layernorm(h)  # [1, 50, 768]
        ffn_out = layer.mlp(normed)                 # [1, 50, 768]
        h = h + ffn_out                             # 残差连接 2

    # ---- 步骤 11：最终归一化 ----
    h = self.norm(h)                                # [1, 50, 768]

    # ---- 步骤 12：输出头 ----
    logits = self.lm_head(h)                        # [1, 50, 6400]

    # ---- 步骤 13：计算损失（训练时） ----
    if labels is not None:
        # 移位：用位置 0..48 预测 1..49
        shift_logits = logits[:, :-1, :]            # [1, 49, 6400]
        shift_labels = labels[:, 1:]                # [1, 49]
        loss = F.cross_entropy(
            shift_logits.reshape(-1, 6400),          # [49, 6400]
            shift_labels.reshape(-1),                 # [49]
            ignore_index=-100                         # 跳过 PAD
        )
        return loss, logits
    return logits
```

### 损失计算的"移位"逻辑

```
输入 tokens:   [BOS, 今, 天, 天, 气, EOS]
                 0    1   2   3   4    5

位置 0 的输出 → 预测位置 1 应该是"今"
位置 1 的输出 → 预测位置 2 应该是"天"
位置 2 的输出 → 预测位置 3 应该是"天"
位置 3 的输出 → 预测位置 4 应该是"气"
位置 4 的输出 → 预测位置 5 应该是"EOS"
位置 5 的输出 → 不用于训练（没有位置 6 的标签）

所以 logits 取 [0:5] ←→ labels 取 [1:6]
```

这是语言模型训练的核心：用"前面的字"预测"后面的字"。

---

## 十、动手：追踪真实数据的流动

```bash
cd /Users/xiaoxiaoma/Documents/minimind
python3 << 'EOF'
import torch, torch.nn.functional as F
from model.model_minimind import MiniMindForCausalLM, MiniMindConfig

# 创建随机初始化模型（用训练好的权重会更精确）
model = MiniMindForCausalLM(MiniMindConfig())
model.eval()

# 输入 6 个 token
ids = torch.randint(0, 6400, (1, 6))
L = 6

with torch.no_grad():
    # 步骤 1：Embedding
    emb = model.model.embed_tokens(ids)
    print(f"Embedding 后：     shape={list(emb.shape)}, 范数≈{emb.norm(dim=-1).mean():.1f}")

    cos = model.model.freqs_cos[:L]
    sin = model.model.freqs_sin[:L]
    h = emb.clone()

    # 步骤 2-9：逐层追踪
    for i in range(8):
        layer = model.model.layers[i]

        # Attention
        normed = layer.input_layernorm(h)
        attn_out, _ = layer.self_attn(normed, (cos, sin))
        h = h + attn_out

        # FFN
        normed = layer.post_attention_layernorm(h)
        h = h + layer.mlp(normed)

        print(f"第 {i} 层后：         shape={list(h.shape)}, 范数≈{h.norm(dim=-1).mean():.1f}")

    # 步骤 10-11
    h = model.model.norm(h)
    logits = model.lm_head(h)
    print(f"最终 logits：      shape={list(logits.shape)}")

    # 损失
    labels = ids.clone()
    x = logits[:, :-1, :].contiguous()
    y = labels[:, 1:].contiguous()
    loss = F.cross_entropy(x.view(-1, 6400), y.view(-1), ignore_index=-100)
    print(f"CrossEntropy Loss： {loss.item():.4f}")
    print(f"(随机权重下 loss ≈ ln(6400) = {torch.tensor(6400.0).log().item():.1f})")
EOF
```

---

## 十一、面试速查：Transformer 21 问

| 问题                           | 答案                                             |
| ---------------------------- | ---------------------------------------------- |
| Transformer 为什么能并行？          | 每个 token 独立计算 QKV，不依赖前一个 token                 |
| RNN 为什么不能并行？                 | 必须按顺序：第 n 步依赖第 n-1 步的结果                        |
| Q、K、V 的本质？                   | 三个不同的投影矩阵，让"提问""定位""提取"在独立空间运作                 |
| 除以 √d_k 的作用？                 | 防止内积过大 → softmax 饱和 → 梯度消失                     |
| Multi-Head 怎么理解？             | 8 组不同的 W_Q/W_K/W_V = 8 种"关注模式"                 |
| GQA 省了什么？                    | K/V 头数减半 → KV Cache 减半 → 推理显存省一半               |
| 残差连接的公式和作用？                  | `h = h + F(h)`，梯度有"1"保底，深层可训练                  |
| Pre-Norm vs Post-Norm？       | Pre-Norm = 先 Norm 再算；Post-Norm = 先算再 Norm（已淘汰） |
| RMSNorm vs LayerNorm？        | RMSNorm 没有"减均值"，更快，效果几乎一样                      |
| RoPE 怎么编码位置？                 | 按位置旋转 Q/K，高频区分局部，低频关联全局                        |
| SwiGLU 为什么 3 个矩阵？            | gate 做门控 + up 传值 + down 压缩                     |
| SiLU vs ReLU？                | SiLU 平滑（负值不死），ReLU 硬截断（负值=0，梯度消失）              |
| FFN 为什么占 70%？                | 存知识需要大容量；Attention 只管检索                        |
| Weight Tying 是什么？            | Embedding 和 LM Head 共享权重，4.92M 参数省一次           |
| Embedding 为什么 768 维？         | 能整除 8（头数）和 64（GPU 对齐）的标准维度                     |
| 因果掩码怎么实现？                    | 注意力矩阵上三角设为 -inf → softmax 后 = 0                |
| 训练时为什么要"移位"？                 | 位置 i 的输出预测位置 i+1，logits[:-1] ↔ labels[1:]      |
| positional encoding 必须吗？     | 必须。没有它，"我打你"="你打我"（顺序无关）                       |
| Decoder-only 是什么？            | 只用 Decoder 部分，GPT/LLaMA/Qwen 都属此类              |
| 为什么 MiniMind 是 8 层？          | 64M 参数的预算内，≥6 层才有深度，≤12 才不会参数超标                |
| intermediate_size 为什么是 2432？ | π 公式：ceil(768×π/64)×64，保证 GPU 对齐 + 合理扩展比       |
