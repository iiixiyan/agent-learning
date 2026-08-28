# 42. 图解 Transformer 架构：大模型底层原理

> 阶段：第四阶段 | 为什么要懂原理：理解原理才能更好地使用和优化大模型

## Transformer 是什么？

Transformer 是 2017 年 Google 论文《Attention Is All You Need》提出的架构，是目前所有大模型（GPT、Claude、Llama、Qwen 等）的基础。

核心创新：**完全基于自注意力机制（Self-Attention）**，抛弃了传统的 RNN/LSTM，解决了长距离依赖问题，并且可以高度并行化训练。

## 整体架构图

```
                    ┌─────────────────┐
                    │   输出 (Output)  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   线性层 + Softmax │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  解码器 N× (Decoder) │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                                    │
┌─────────▼─────────┐              ┌──────────▼──────────┐
│  编码器 N× (Encoder)  │              │  编码器输出 (Encoder)  │
└─────────┬─────────┘              └──────────▲──────────┘
          │                                    │
┌─────────▼─────────┐              ┌──────────┴──────────┐
│  输入嵌入 (Input)    │              │  输出嵌入 (Output)    │
│  + 位置编码           │              │  + 位置编码            │
└───────────────────┘              └─────────────────────┘
```

## 核心组件详解

### 1. 位置编码（Positional Encoding）

Self-Attention 本身不考虑顺序，需要位置编码告诉模型词的位置。

```python
import numpy as np

def get_positional_encoding(max_len, d_model):
    pe = np.zeros((max_len, d_model))
    position = np.arange(0, max_len)[:, np.newaxis]
    div_term = np.exp(np.arange(0, d_model, 2) * (-np.log(10000.0) / d_model))
    pe[:, 0::2] = np.sin(position * div_term)
    pe[:, 1::2] = np.cos(position * div_term)
    return pe

# 512维，最大长度1000
pe = get_positional_encoding(1000, 512)
print(pe.shape)  # (1000, 512)
```

### 2. 自注意力机制（Self-Attention）

核心公式：`Attention(Q, K, V) = softmax(QK^T / √d_k) V`

- Q (Query)：查询，"我在找什么"
- K (Key)：键，"我包含什么"
- V (Value)：值，"我的实际内容"

```python
import torch
import torch.nn.functional as F

def self_attention(Q, K, V, mask=None):
    d_k = Q.size(-1)
    # QK^T / √d_k
    scores = torch.matmul(Q, K.transpose(-2, -1)) / torch.sqrt(torch.tensor(d_k, dtype=torch.float32))
    if mask is not None:
        scores = scores.masked_fill(mask == 0, -1e9)
    # softmax
    attn_weights = F.softmax(scores, dim=-1)
    # 乘以 V
    output = torch.matmul(attn_weights, V)
    return output, attn_weights

# 示例：batch=2, seq_len=10, d_model=512
Q = torch.randn(2, 10, 512)
K = torch.randn(2, 10, 512)
V = torch.randn(2, 10, 512)
output, weights = self_attention(Q, K, V)
print(output.shape)  # (2, 10, 512)
```

### 3. 多头注意力（Multi-Head Attention）

把 Q/K/V 分成多个头，每个头学习不同的注意力模式，最后拼接。

```python
class MultiHeadAttention(torch.nn.Module):
    def __init__(self, d_model=512, n_heads=8):
        super().__init__()
        self.n_heads = n_heads
        self.d_head = d_model // n_heads
        self.W_Q = torch.nn.Linear(d_model, d_model)
        self.W_K = torch.nn.Linear(d_model, d_model)
        self.W_V = torch.nn.Linear(d_model, d_model)
        self.W_O = torch.nn.Linear(d_model, d_model)

    def forward(self, x, mask=None):
        batch, seq_len, _ = x.shape
        # 线性变换 + 分头
        Q = self.W_Q(x).view(batch, seq_len, self.n_heads, self.d_head).transpose(1, 2)
        K = self.W_K(x).view(batch, seq_len, self.n_heads, self.d_head).transpose(1, 2)
        V = self.W_V(x).view(batch, seq_len, self.n_heads, self.d_head).transpose(1, 2)
        # 自注意力
        attn_out, _ = self_attention(Q, K, V, mask)
        # 拼接
        attn_out = attn_out.transpose(1, 2).contiguous().view(batch, seq_len, -1)
        return self.W_O(attn_out)
```

### 4. 前馈网络（Feed Forward）

每个注意力层后接一个两层的全连接网络，增加非线性。

```python
class FeedForward(torch.nn.Module):
    def __init__(self, d_model=512, d_ff=2048):
        super().__init__()
        self.linear1 = torch.nn.Linear(d_model, d_ff)
        self.linear2 = torch.nn.Linear(d_ff, d_model)
        self.relu = torch.nn.ReLU()

    def forward(self, x):
        return self.linear2(self.relu(self.linear1(x)))
```

### 5. 残差连接 + LayerNorm

每个子层都有残差连接和层归一化，解决深层网络训练问题。

```python
class EncoderLayer(torch.nn.Module):
    def __init__(self, d_model=512, n_heads=8, d_ff=2048):
        super().__init__()
        self.attn = MultiHeadAttention(d_model, n_heads)
        self.ff = FeedForward(d_model, d_ff)
        self.norm1 = torch.nn.LayerNorm(d_model)
        self.norm2 = torch.nn.LayerNorm(d_model)

    def forward(self, x, mask=None):
        # 自注意力 + 残差 + 归一化
        x = self.norm1(x + self.attn(x, mask))
        # 前馈 + 残差 + 归一化
        x = self.norm2(x + self.ff(x))
        return x
```

## 编码器 vs 解码器

| 组件 | 编码器 (Encoder) | 解码器 (Decoder) |
|------|-----------------|-----------------|
| 自注意力 | ✅ 可以看到所有位置 | ✅ 只能看到前面的位置（因果掩码） |
| 交叉注意力 | ❌ | ✅ 关注编码器输出 |
| 前馈网络 | ✅ | ✅ |
| 用途 | 理解输入 | 生成输出 |

**GPT 系列只用解码器**（Decoder-only），**BERT 只用编码器**（Encoder-only），**T5 用编码器-解码器**（Encoder-Decoder）。

## 学习要点

1. Transformer 核心是 Self-Attention，解决长距离依赖和并行训练问题
2. Q/K/V 是自注意力的三个核心矩阵，分别代表查询、键、值
3. 多头注意力让模型同时关注不同子空间的信息
4. 位置编码让模型知道词的顺序
5. 残差连接 + LayerNorm 让深层网络可训练
6. GPT 是 Decoder-only 架构，用因果掩码实现自回归生成

---
**上一篇**：[LangFuse](./41_LangFuse.md) | **下一篇**：[大模型训练推理全流程](./43_大模型训练推理全流程.md)
