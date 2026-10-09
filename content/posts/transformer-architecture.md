---
math: true
title: "Transformer 架构详解：从 Attention 到 LLM"
date: 2026-10-01
cover:
  image: "/img/transformer.jpg"
description: "从 Attention 机制出发，深入解析 Transformer 的核心原理"
tags:
  - Transformer
  - LLM
categories:
  - 大语言模型
---

## Self-Attention 机制

Self-Attention 的核心公式：

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

其中 $Q$、$K$、$V$ 分别是查询、键、值矩阵。

## 多头注意力

$$\text{MultiHead}(Q, K, V) = \text{Concat}(head_1, ..., head_h)W^O$$

## 代码实现

```python
import torch
import torch.nn as nn

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, n_heads):
        super().__init__()
        self.d_k = d_model // n_heads
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)

    def forward(self, x):
        Q = self.W_q(x)
        K = self.W_k(x)
        return Q, K
```

## 嵌入视频示例

{{< alert color="#dbeafe" border="#2563eb" >}}
**提示**：这是一段提示信息，使用 alert shortcode 创建。
{{< /alert >}}
