---
math: true
title: "PyTorch 深度学习入门"
date: 2026-09-15
description: "从张量到训练循环"
tags:
  - PyTorch
categories:
  - 机器学习
---

## 张量操作

```python
import torch
x = torch.randn(3, 4)
```

## 自动求导

```python
x = torch.tensor(2.0, requires_grad=True)
y = x ** 2 + 3 * x + 1
y.backward()
print(x.grad)  # 7.0
```
