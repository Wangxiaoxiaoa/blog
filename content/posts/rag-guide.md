---
title: "RAG：检索增强生成的原理与实践"
date: "2026-09-25T00:00:00.000Z"
description: "为什么需要 RAG？

LLM 的知识有截止日期，且可能产生幻觉。RAG 通过检索相关文档来提供最新信息、减少幻觉、支持私有知识库。


RAG 架构

用户提问 → Embedding → 向量检索 → Top-K 文档 → LLM 生成回答"
tags: ["RAG", "LLM", "大语言模型"]
math: true
draft: false
---

## 为什么需要 RAG？

LLM 的知识有截止日期，且可能产生幻觉。RAG 通过检索相关文档来提供最新信息、减少幻觉、支持私有知识库。

## RAG 架构

用户提问 → Embedding → 向量检索 → Top-K 文档 → LLM 生成回答
