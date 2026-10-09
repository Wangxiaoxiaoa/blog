---
math: true
title: "RAG：检索增强生成的原理与实践"
date: 2026-09-25
description: "通过结合外部知识库来增强 LLM 的能力"
tags:
  - RAG
  - LLM
categories:
  - 大语言模型
---

## 为什么需要 RAG？

LLM 的知识有截止日期，且可能产生幻觉。RAG 通过检索相关文档来提供最新信息、减少幻觉、支持私有知识库。

## RAG 架构

用户提问 → Embedding → 向量检索 → Top-K 文档 → LLM 生成回答
