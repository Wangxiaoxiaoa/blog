---
math: true
title: "Prompt Engineering 实战"
date: 2026-09-28
description: "让 LLM 更听话的核心技巧"
tags:
  - Prompt
  - LLM
categories:
  - 大语言模型
---

## 核心技巧

### 明确指令

好的 Prompt = 明确的指令 + 合适的示例 + 结构化的输出要求。

### Few-shot 示例

提供 2-3 个示例，模型会模仿格式和风格。

### Chain-of-Thought

让模型一步步思考，显著提升推理准确率。

## 折叠内容示例

{{< collapse summary="点击展开查看 Prompt 模板" >}}

**系统提示词模板：**

```
你是一个专业的AI助手，请遵循以下规则：
1. 始终用中文回答
2. 回答要简洁清晰
3. 不确定的内容要说明
```

{{< /collapse >}}
