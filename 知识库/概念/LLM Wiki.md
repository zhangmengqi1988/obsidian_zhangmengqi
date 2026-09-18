---
type: concept
title: LLM Wiki
tldr: LLM 担任 wiki 维护者、把原始资料编译成双链知识库的三层范式
aliases: [LLM 知识库]
tags: [方法论]
sources: ["[[摘要/如何构建 LLM Wiki]]"]
related: []
confidence: high
status: stub
created: 2026-09-18
updated: 2026-09-18
---

LLM Wiki 是 Karpathy 提出、由 [[摘要/如何构建 LLM Wiki]] 展开的知识库维护范式：Obsidian 是 IDE，LLM 是程序员，`知识库/` 是代码库——把原始资料**编译一次**成持久的双链知识页面，而不是每次提问从原始资料重新检索拼凑。

**三层结构**：raw（原始资料，只读不可变，事实源）／ wiki（LLM 撰写维护的知识层）／ schema（CLAUDE.md，人与 LLM 共同演化）。

**四个操作**：ingest（把新资料编译进 wiki）、query（从已编译的 wiki 综合回答）、lint（全库体检）、save（把有价值的回答回填归档）。

**五种页面类型**：source / entity / concept / synthesis / output。

> [!note] 本页为 stub
> 本页暂无第二来源，仅据 [[摘要/如何构建 LLM Wiki]] 单一来源建立，待后续资料出现后扩充。
