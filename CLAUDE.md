# 知识库维护规约

本库依据 Andrej Karpathy 的 LLM Wiki 范式组织。三层结构，职责边界严格。

## 一、本库定位与角色

**主题与边界**：本库是支付／清结算／分布式系统方向的技术知识库，
兼收少量个人成长与古籍类剪藏。超出此范围的资料默认不摄取，除非用户明确要求。

**LLM 的角色**：LLM 是 wiki 的**维护者**，不是聊天机器人——负责摘要、
交叉引用、记账、一致性维护；用户负责筛选来源、提出问题、判断方向。

## 二、三层架构

1. **`原始资料/`** —— 原始资料层。只读、不可变，是事实源。
2. **`知识库/`** —— 知识层。由 LLM 撰写和维护，用户只读。
3. **本文件（`CLAUDE.md`）** —— 规则层。由人与 LLM **共同演化**（co-evolve）——
   不是一次定死，用着用着往里补约定。

核心原则：知识**编译一次**成持久产物，而不是每次提问从原始资料重新检索拼凑。
Obsidian 是 IDE，LLM 是程序员，`知识库/` 是代码库。

## 三、目录结构

```text
vault/
  CLAUDE.md                    # 本文件，规则层
  docs/                        # 设计与计划文档
  原始资料/                     # 只读层，事实源
    剪藏/                       # Web Clipper 抓取的网页、B站视频转写
    截图笔记/                   # 截图与随手笔记
    每日笔记/                   # daily-task-auto-generator 生成，勿改名
    古籍/                       # 古籍文本
    附件/                       # 全部图片与附件（attachmentFolderPath 指向此处）
    归档/                       # 已停用的 .base 视图
  知识库/                       # 知识层，LLM 全权维护
    摘要/                       # source 页
    概念/                       # concept 页
    实体/                       # entity 页
    综合/                       # synthesis 页
    输出/                       # output 页
    索引.md
    操作日志.md
    综述.md
    lint-report-YYYY-MM-DD.md   # lint 产物
  .obsidian/                   # 只读
  .claude/commands/            # ingest / query / lint / save
```

## 四、目录权限

| 目录 | 权限 |
|---|---|
| `原始资料/` | **只读**。只允许移动位置，永不修改其中文件的正文 |
| `知识库/` | **读写**。全权维护 |
| `CLAUDE.md` | 读写，但改动需用户确认 |
| `docs/` | 读写。设计与计划文档 |
| `.claude/commands/` | 读写 |
| `.obsidian/` | **只读** |

**哪些 raw 已被 ingest，由 `知识库/操作日志.md` 追踪，不在 `原始资料/`
层面做任何标记**——不加已处理标签、不建待处理／已处理文件夹、不改 frontmatter。

## 五、页面类型与 frontmatter

字段名用英文，目录名与正文用中文。

**所有页面必填六个字段**——`title`、`tldr`、`type`、`status`、`created`、
`updated`。（`索引.md`／`操作日志.md`／`综述.md` 除外，用轻 frontmatter 或不加。）

`tldr`：**一句话摘要，不超过 60 字**，同时用作 `索引.md` 中该页的条目文字。
它是检索的承重字段——query 时先靠它判断相关性。

`status` 取值四档：`stub` | `draft` | `stable` | `needs-review`。
`stub` = 仅一个来源、只记定义与出处，等第二来源出现再扩写。

### 1. `type: source` —— 摘要页（`知识库/摘要/`）

每个原始资料一页。**保留剪藏原有的 `title`／`source`／`author`／`tags` 字段**——
`.base` 视图与既有流程依赖这些字段，删改会导致静默失效。营销性或样板性元数据
（如 B 站自动生成的 `description`）可以省略。

```yaml
---
type: source
title: 清结算模块设计
tldr: 从业务需求到落地，拆解清结算模块的对账、清算、账务三层设计
source_title: 一文看懂清结算模块设计：从业务需求和设计落地
source: https://mp.weixin.qq.com/...
source_type: 微信文章          # 微信文章 | B站视频 | 云效知识库 | 其他
author: []
published: 2026-08-20
captured: 2026-08-20
tags: [clippings, 支付]
raw: "[[原始资料/剪藏/清结算模块设计]]"
confidence: high
status: draft
created: 2026-08-20
updated: 2026-08-20
---
```

`raw` 字段有两种合法形式：有对应本地原始文件时写 wikilink（如上，
`raw: "[[原始资料/剪藏/清结算模块设计]]"`）；若该摘要页没有对应的本地文件
（只有线上视频或网页），`raw` 直接写原始 URL 字符串，
如 `raw: "https://www.bilibili.com/video/BV1uD3v6LER1/"`。

正文结构：一句话主旨 → 关键论点（3–7 条）→ 与其他资料的关系 → 值得追问的问题。

### 2. `type: concept` / `type: entity` —— 知识页

```yaml
---
type: concept                  # concept | entity
title: 清结算
tldr: 支付链路的资金核算与划拨环节，含清算、结算、对账三段职责
aliases: [清算, 结算, settlement]
tags: [支付, 金融]
sources: ["[[摘要/清结算模块设计]]", "[[摘要/支付系统技术决策]]"]
related: ["[[支付网关]]"]       # 可选，只在关系确实存在时填
confidence: medium             # high | medium | low
status: draft                  # stub | draft | stable | needs-review
created: 2026-09-17
updated: 2026-09-17
---
```

四个链接字段的用途一次说清：

| 字段 | 用途 |
|---|---|
| `sources` | 本页依据的摘要页 |
| `related` | 相关但非依据的页面 |
| `supports` | 与本页结论互相印证的页面 |
| `contradicts` | 与本页存在**未解决**冲突的页面 |

`related`／`supports`／`contradicts` 三项可选，**只在关系确实存在时填**，
不要为凑字段而填。

**建页规则**：

- 某概念/实体在 **2 个及以上来源**中出现 → 建完整页
- **首次出现（仅 1 个来源）→ 建 stub 页**，`status: stub`，正文只写定义与出处

`confidence` 判定标准：

- `high` —— 单一来源直接陈述，或多来源完全一致
- `medium` —— 多来源综合，细节有出入但不矛盾
- `low` —— 仅凭薄弱来源推断，或来源间存在冲突

**来源间冲突不得静默抹平。** 用 Obsidian callout 语法标注，见第六章第 3 条。

### 3. `type: synthesis` —— 综合页（`知识库/综合/`）

跨来源的比较、阶段性结论、整体图景。

```yaml
---
type: synthesis
title: 支付系统架构决策对比
question: 做支付系统时哪些技术决策最关键？
tldr: 横向对比各来源在一致性、幂等、对账上的技术选型与取舍
tags: [支付]
sources: ["[[摘要/清结算模块设计]]"]
confidence: medium
status: draft
created: 2026-09-17
updated: 2026-09-17
---
```

### 4. `type: output` —— 输出页（`知识库/输出/`）

**单次问答**的归档，来自 query 的回填。

```yaml
---
type: output
title: 支付系统幂等方案怎么选
question: 支付回调的幂等应该做在哪一层？
tldr: 对比 Redis 分布式锁、唯一索引、状态机三种幂等实现及其代价
tags: [支付]
sources: ["[[摘要/支付系统技术决策]]", "[[支付网关]]"]
confidence: medium
status: draft
created: 2026-09-18
updated: 2026-09-18
---
```

正文结构：问题 → 结论 → 依据（带 `[[]]` 引用）→ 未解决的部分。

**`synthesis` 与 `output` 的区别**：synthesis 是跨源综合出的**阶段性判断**，
output 是**一次问答**的存档。判断不清时看是否综合了多个来源。

## 六、核心原则

1. **raw 不可变**：不改 body、不改 frontmatter、不移动（除归档整理）、
   不改名（除精简命名）、不删除。ingest 状态由日志追踪，不在 raw 上做标记。
2. **双链双份维护**：正文里写行内 `[[]]` wikilink（给人看、给 Graph View 看）；
   frontmatter 里填 `sources`、`related`、`supports`、`contradicts`
   （给 Bases/Dataview 查询用）。**两边都不能偷懒。**
3. **冲突不静默抹平**：用 Obsidian callout 语法标注——

   ```markdown
   > [!contradiction] 来源A vs 来源B
   > [[摘要/A]] 声称 X，但 [[摘要/B]] 声称 Y。
   > 当前状态：未解决 / 已解决（说明理由）
   ```

   **「当前状态：已解决」同样要写**——判定为语义不同的两条标尺、
   而非同一命题的冲突时，把理由写在 callout 里，而不是删掉 callout。
4. **命名规范**：精简中文短标题，去掉"一文看懂"、"建议收藏"、"吃透"等
   营销词与副标题；概念页文件名即概念名（`清结算.md`），不加"摘要"、"笔记"后缀。

   - **跨层允许重名，但 wiki 内部链接一律写全路径：**
     - 引用摘要页：`[[摘要/清结算模块设计]]`
     - 引用原始资料：`[[原始资料/剪藏/清结算模块设计]]`
     - 概念页之间可写短链：`[[清结算]]`
   - **不重命名插件生成的文件**。`原始资料/每日笔记/2026/{7,8,9}月.md` 由
     `daily-task-auto-generator` 按 `{年}/{月}月.md` 生成，改名会导致重复生成。

   **与上游范式的刻意偏离**：文章要求文件名为 kebab-case、中文标题只放
   frontmatter 的 `title`；本库**有意不采用**，因为这是中文 vault，kebab-case
   意味着拼音 slug（`qing-jie-suan-mo-kuai-she-ji`），会让
   `[[清结算模块设计]]` 这类 wikilink 完全丧失可读性。文章原文亦声明目录与命名
   "根据自己的业务场景和需求来"。同理 `created`／`updated` 有意不改为文章示例中的
   `date_created`／`date_modified`。
5. **不确定就标不确定**：不用 `confidence: high` 标注仅凭单一薄弱来源的推断；
   来源没说的不要用通用知识补全，写明"现有来源未覆盖"。

## 七、操作流程

四个操作的详细步骤均在 `.claude/commands/<名>.md`，此处只给概要。

- **Ingest（摄取）**：新资料放入 `原始资料/剪藏/` → 与用户确认重点方向 →
  写摘要页 → 更新／新建概念页实体页 → 维护交叉引用 → 更新 `索引.md` →
  追加一行到 `操作日志.md`。一篇资料通常触及 5–15 个页面。
- **Query（查询）**：先读 `索引.md` 靠 `tldr` 定位 → 读相关页面综合作答 →
  每个关键论断带 `[[]]` 引用并标置信度 → **wiki 不够用时才回溯读原始资料**。
- **Lint（体检）**：六类检查——矛盾 / 过时 / 孤儿页 / 缺失页面 / 断链 /
  信息缺口。产出 `lint-report-YYYY-MM-DD.md`。
- **Save（回填）**：把有价值的回答回填成页面——跨源综合去 `综合/`，
  单次问答去 `输出/`，使结果复利。

硬规矩：**每个操作完成后，`索引.md` 和 `操作日志.md` 必须同步更新。
没更新，等于没做。**

## 八、索引与日志规范

- **`索引.md`**：按页面类型分节（摘要／概念／实体／综合／输出／综述），
  每条 = 链接 + `tldr`。每次 ingest 后更新。
- **`操作日志.md`**：append-only。每条格式：

  ```markdown
  ## [YYYY-MM-DD] 动作 | 标题
  ```

  标题下用 bullet 记涉及页面与备注。这个格式便于机器检索：

  ```bash
  grep "^## \[" "知识库/操作日志.md" | tail -5
  ```

  日志的另一个用途：**追踪哪些原始资料已被 ingest，避免重复处理**。

## 九、语言与风格

中文正文。技术术语保留英文原词（如 Raft、PEFT、B+ 树）。不用营销腔。

### 硬性禁止

- 不修改 `原始资料/` 下任何文件的正文
- 不删除用户笔记（清理垃圾须先列入计划并获确认）
- 不在页内静默抹平来源冲突
- 不用 `confidence: high` 标注仅凭单一薄弱来源的推断
