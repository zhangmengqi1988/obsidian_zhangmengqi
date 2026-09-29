---
type: source
title: 如何构建 LLM Wiki
tldr: LLM Wiki 搭建三步：建 raw/wiki 目录骨架、写 CLAUDE.md 立规矩、配 ingest 等四个命令
source_title: "✅如何构建LLM Wiki？ · 云效 Thoughts · 企业级知识库"
source: https://thoughts.aliyun.com/workspaces/6963289eb0fc2e001bb052eb/docs/6a40f9fdc71a890001618034
source_type: 云效知识库
author: []
published: null
captured: 2026-09-18
tags: [clippings, LLM Wiki]
raw: "[[原始资料/剪藏/如何构建 LLM Wiki]]"
status: draft
created: 2026-09-18
updated: 2026-09-18
---

本文把 Karpathy 的 LLM Wiki 范式拆成可落地的搭建步骤：Obsidian 是后端开发的 IDEA，LLM 是被指挥构建代码仓（wiki）的程序员——按"建目录 → 写 CLAUDE.md → 配 commands"三步搭起骨架，再用 ingest / query / lint / save 四个操作让 wiki 持续增长。

## 关键论点

- **三步走**：先建目录骨架（raw/ 按资料类型分 articles/videos/assets，wiki/ 按页面类型分 sources/entities/concepts/synthesis/outputs），再写 CLAUDE.md 立规矩，最后配 `.claude/commands/` 四个命令。目录先于规矩，因为 CLAUDE.md 要指明每种子页面去哪个目录；骨架建空目录即可，页面等第一次 ingest 时由 LLM 自己创建。
- **三层架构与权限**：raw/ 完全不可变、LLM 只读——raw 是事实源，一旦允许改就分不清"原文说了什么"和"LLM 理解成了什么"；wiki/ 由 LLM 完全拥有；CLAUDE.md（schema）由人与 LLM **共同演化（co-evolve）**，不是一次定死。哪些 raw 已 ingest 由 log 追踪，不在 raw 层面做任何标记。
- **五种页面类型及建页时机**：source（证据节点，每来源一页，每次 ingest 必建）、entity / concept（知识节点，2+ 来源出现建完整页、首次出现建 stub）、synthesis（跨源比较与阶段性结论）、output（有价值问答的归档）。
- **双链双份维护**：正文写行内 `[[]]`（给人看、给 Graph View 看），frontmatter 填 `sources`／`related`／`supports`／`contradicts`（给 Dataview 查询用），两边都不能偷懒；来源冲突**永不静默抹平**，用 `> [!contradiction]` callout 写清双方观点与当前状态。
- **四个命令**：ingest 是 wiki 增长的唯一入口，一份资料可能牵动 10–15 页；query 从已编译的 wiki 综合答案而非从 raw 重检索（与 RAG 的根本区别）；lint 六类检查——矛盾、过时、孤儿页、缺失页面、断链、信息缺口，低风险自动修、高风险仅报告待人判断；save 把好答案回填成页面，跨源综合去 synthesis、单次问答去 outputs。
- **index 与 log 的承重作用**：index 每条 = 链接 + tldr，query 第一步靠 tldr 判相关性，它直接决定检索效率；log 为 append-only，统一格式 `## [YYYY-MM-DD] op | title` 使其可 grep，兼作 ingest 状态追踪。每个操作完成后 index 与 log 必须同步更新——"没更新，等于没做"。
- **人机分工**：人负责筛选来源、提出问题、判断方向；LLM 负责摘要、交叉引用、记账、一致性维护——schema 的作用正是让 LLM 成为"守规矩的 wiki 维护者而非普通聊天机器人"。

## 与其他资料的关系

- 与本库其余摘要页（[[摘要/清结算模块设计]]、[[摘要/支付系统技术决策]]、[[摘要/跨库 Join 方案]]、[[摘要/MIT 6.824 分布式系统]]）**没有主题关联**——那些讲支付与清结算业务，本篇讲知识库维护方法论，题材正交，如实记此而不为连接图谱强凑关系。
- 它是本库 `CLAUDE.md` 与 `.claude/commands/`（ingest/query/lint/save）的直接依据：本库的三层目录、页面模板、status 四档、日志格式均可回溯到本文；相关概念页见 [[概念/LLM Wiki]]。

## 值得追问的问题

- 来源要求文件名用 kebab-case（理由：URL 友好、grep 方便），但中文 vault 用拼音 slug 会让 wikilink 丧失可读性——本库已刻意偏离。该规则是硬约束还是应按语言环境裁剪？来源自己也说目录分法"根据自己的业务场景和需求来"，命名是否同理？
- stub 页的升级时机未展开：首次出现建 stub、2+ 来源建完整页，但谁来发现"第二来源出现了"？靠 lint 的"stub 僵局"检查项还是人主动触发？
- save 归档的页面（synthesis/output）由谁复核？来源给出 `explored` 门控（AI 不应置 true），本库用 status 四档代替，draft → stable 需用户通读确认，这一差异是否会造成回填页面的长期滞留？
- 来源 lint 清单引用 `explored`／`date_modified`／`pending_review` 等字段，与本库 schema 的 `status`／`updated` 字段名不一致——这份清单显然来自另一套实现，落地到本库时需要逐条翻译，翻译清单尚未写。
