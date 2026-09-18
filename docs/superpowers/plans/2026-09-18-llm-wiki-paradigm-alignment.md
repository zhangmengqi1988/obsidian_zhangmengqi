# LLM Wiki 范式对齐 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 依据《✅如何构建LLM Wiki？》（云效 Thoughts）补齐本库缺失的 commands 层与 output 页面类型，并把 CLAUDE.md 的 schema 升级到与该文一致。

**Architecture:** 上一阶段（2026-09-17）搭好了三层结构与样板 wiki；本阶段**只补规则与操作层，不重构已有产物**。文章给的六个约定点中五处本库已符合，需补的是：`.claude/commands/` 四个操作、`知识库/输出/` 目录、可 grep 的日志格式、`tldr`/`related`/`supports`/`contradicts` 字段、callout 冲突标注语法、entity/concept 的「2+ 来源」建页规则。

**依据:** `Clippings/` 中的新剪藏（本计划 Task 0 将其归档为 `原始资料/剪藏/如何构建 LLM Wiki.md`），及与用户确认的四项裁决（见下）。

**上一阶段产物:** `docs/superpowers/plans/2026-09-17-llm-wiki-reorganization.md`、`docs/superpowers/specs/2026-09-17-llm-wiki-reorganization-design.md`

## 用户裁决（本阶段的额外约束，优先于文章原文）

| # | 议题 | 裁决 |
|---|---|---|
| A | 文件名 kebab-case | **保持中文名**。文章自己留了口子（"不一定需要完全和我的分法一致，请根据自己的业务场景和需求来"）。CLAUDE.md 必须**显式写明这是刻意偏离**及理由，避免日后被当成疏漏。 |
| B | 操作日志格式 | **改为标题式 `## [YYYY-MM-DD] 动作 \| 标题` + 标题下 bullet 记细节**，兼顾 grep 与可读性。原有 4 条记录的信息量不得丢失。 |
| C | synthesis / output 拆分 | **新增 `知识库/输出/`**。`综合/` 只放跨源比较与阶段性结论，`输出/` 放有价值的问答归档。同步修正 CLAUDE.md 的 Query 章。 |
| D | 新剪藏 | **移入 `原始资料/剪藏/` 并立即 ingest**。 |

## Global Constraints

本阶段全部任务隐含包含以下约束：

1. 目录名与正文用中文，frontmatter 字段名用英文
2. `原始资料/` 只读。允许移动与重命名，**永不修改其中任何文件的正文**（含 frontmatter）
3. wiki 内部链接一律用全路径形式 `[[摘要/清结算模块设计]]`。仅概念页之间的短链可写 `[[清结算]]`
4. 每步独立 commit，用 `git mv` 保留文件历史。**唯一例外**：`Clippings/` 下的新剪藏尚未被 git 跟踪，首次入库用 `mv` + `git add <具体路径>`
5. **不重命名插件生成的文件**（`原始资料/每日笔记/2026/{7,8,9}月.md`）
6. shell 是 Git Bash（Windows），路径含中文，**所有路径参数必须加双引号**
7. **禁止** `git add -A`、`git add .`、`git commit -a`、`git add .claude/settings.local.json`、`git clean`、`git reset --hard`、`git checkout`、`git stash`
8. 仓库有 `github-sync` 插件产生的无关提交噪音（形如 `LAPTOP-RGRK37M4 2026-9-17:16:35:2`）。评估历史时跳过它们
9. 不得把 messager 插件的 API key 值写进任何被跟踪文件
10. 用 `git ls-tree` 列 CJK 路径时必须加 `-c core.quotePath=false`，否则端锚定 grep（如 `grep "\.md$"`）返回假阴性

---

### Task 0: 归档新剪藏

**Files:**
- Move: `Clippings/✅如何构建LLM Wiki？ · 云效 Thoughts · 企业级知识库.md` → `原始资料/剪藏/如何构建 LLM Wiki.md`

**Interfaces:**
- Produces: `原始资料/剪藏/如何构建 LLM Wiki.md`（Task 5 的 ingest 对象）

- [ ] **Step 1: 记录原文哈希**

```bash
cd "D:/张梦奇ob"
sha256sum "Clippings/✅如何构建LLM Wiki？ · 云效 Thoughts · 企业级知识库.md"
```
记下输出，Step 3 要对比。

- [ ] **Step 2: 移动并精简命名**

文件未被 git 跟踪，故用 `mv` 而非 `git mv`（没有历史需要保留）：

```bash
cd "D:/张梦奇ob"
mv "Clippings/✅如何构建LLM Wiki？ · 云效 Thoughts · 企业级知识库.md" "原始资料/剪藏/如何构建 LLM Wiki.md"
rmdir "Clippings"
```

命名依据：去掉 `✅`、`？`、"· 云效 Thoughts · 企业级知识库" 副标题，保留 `LLM Wiki` 的拉丁词间空格（与既有 `跨库 Join 方案.md`、`Spring Boot 4 API 版本控制.md` 一致）。

- [ ] **Step 3: 验证正文逐字节未变**

```bash
cd "D:/张梦奇ob"
sha256sum "原始资料/剪藏/如何构建 LLM Wiki.md"
ls -a | grep -c "^Clippings$" || echo "Clippings 已移除"
ls "原始资料/剪藏" | wc -l
```
**期望：** 哈希与 Step 1 完全一致；`Clippings` 目录不存在；剪藏文件数 **22**（原 21 + 1）。

- [ ] **Step 4: Commit**

```bash
cd "D:/张梦奇ob"
git add "原始资料/剪藏/如何构建 LLM Wiki.md"
git commit -m "chore: 新剪藏归档入原始资料/剪藏/ 并精简命名"
```

---

### Task 1: schema 升级——CLAUDE.md 与「输出/」目录

**Files:**
- Create: `知识库/输出/.gitkeep`
- Modify: `CLAUDE.md`（全文重写，保留原有有效内容）

**Interfaces:**
- Produces: 升级后的 CLAUDE.md，是 Task 2/4/5/6 的规则来源
- Consumes: Task 0 不涉及

- [ ] **Step 1: 建 `知识库/输出/` 目录**

```bash
cd "D:/张梦奇ob"
mkdir -p "知识库/输出"
touch "知识库/输出/.gitkeep"
```
（git 无法跟踪空目录，`.gitkeep` 与既有 `知识库/实体/.gitkeep`、`知识库/综合/.gitkeep` 做法一致。）

- [ ] **Step 2: 重写 CLAUDE.md**

必须包含以下**九章**，顺序即如下顺序。每章的具体要求逐条列出——照写，不要增删章节。

**第一章：本库定位与角色**

写明两件事：
- **主题与边界**：本库是支付/清结算/分布式系统方向的技术知识库，兼收少量个人成长与古籍类剪藏
- **LLM 的角色**：LLM 是 wiki 维护者，不是聊天机器人——负责摘要、交叉引用、记账、一致性维护；用户负责筛选来源、提出问题、判断方向

**第二章：三层架构**

- `原始资料/` = 原始资料层，只读、不可变，是事实源
- `知识库/` = 知识层，由 LLM 撰写和维护
- `CLAUDE.md` = 规则层，由人与 LLM **共同演化**（co-evolve）——不是一次定死，用着用着往里补约定
- 保留原有的一句核心原则：知识**编译一次**成持久产物，而不是每次提问从原始资料重新检索拼凑
- 保留原有的一句：Obsidian 是 IDE，LLM 是程序员，`知识库/` 是代码库

**第三章：目录结构（完整目录树）**

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

**第四章：目录权限**

在原有权限表基础上**加两行**：

| 目录 | 权限 |
|---|---|
| `原始资料/` | **只读**。只允许移动位置，永不修改其中文件的正文 |
| `知识库/` | **读写**。全权维护 |
| `CLAUDE.md` | 读写，但改动需用户确认 |
| `docs/` | 读写。设计与计划文档 |
| `.claude/commands/` | 读写 |
| `.obsidian/` | **只读** |

并写明：**哪些 raw 已被 ingest，由 `知识库/操作日志.md` 追踪，不在 `原始资料/` 层面做任何标记**（不加已处理标签、不建待处理/已处理文件夹、不改 frontmatter）。

**第五章：页面类型与 frontmatter**

先总述：**所有页面必填六个字段**——`title`、`tldr`、`type`、`status`、`created`、`updated`。（`索引.md`／`操作日志.md`／`综述.md` 除外，用轻 frontmatter 或不加。）

`tldr` 的定义必须明确写出：一句话摘要，**不超过 60 字**，同时用作 `索引.md` 中该页的条目文字。它是检索的承重字段——query 时先靠它判断相关性。

`status` 取值扩为四档：`stub` | `draft` | `stable` | `needs-review`。
`stub` = 仅一个来源、只记定义与出处，等第二来源出现再扩写。

五个页面类型各给完整 frontmatter 模板与正文结构：

1. **`type: source`** —— 摘要页（`知识库/摘要/`）。一个来源一页。
   沿用现有模板与「保留剪藏原有的 `title`／`source`／`author`／`tags` 字段，营销性元数据可省」的措辞，**新增 `tldr`**。
   保留 `raw` 字段的两种合法形式说明（本地 wikilink ／线上 URL）。
   正文结构沿用：一句话主旨 → 关键论点（3–7 条）→ 与其他资料的关系 → 值得追问的问题。

2. **`type: concept`** / **`type: entity`** —— 知识页。
   沿用现有模板，**新增 `tldr`、`related`**，并把 `sources`／`related`／`supports`／`contradicts` 四个字段的用途一次性说明白：

   | 字段 | 用途 |
   |---|---|
   | `sources` | 本页依据的摘要页 |
   | `related` | 相关但非依据的页面 |
   | `supports` | 与本页结论互相印证的页面 |
   | `contradicts` | 与本页存在**未解决**冲突的页面 |

   `related`／`supports`／`contradicts` 三项可选，**只在关系确实存在时填**，不要为凑字段而填。

   **建页规则（本阶段新增，必须写清）：**
   - 某概念/实体在 **2 个及以上来源**中出现 → 建完整页
   - **首次出现（仅 1 个来源）→ 建 stub 页**，`status: stub`，正文只写定义与出处
   
   保留 `confidence` 判定标准三条（`high`／`medium`／`low`）原文。
   保留「来源间冲突不得静默抹平」一句，并把原文的"以独立段落并列各方说法及来源"**升级为 callout 语法**，见第六章。

3. **`type: synthesis`** —— 综合页（`知识库/综合/`）。沿用现有模板 + `tldr`。
   定义写清：**跨来源**的比较、阶段性结论、整体图景。

4. **`type: output`** —— 输出页（`知识库/输出/`）。**本阶段新增**，模板：

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
   定义写清：**单次问答**的归档，来自 query 的回填。

   **`synthesis` 与 `output` 的区别必须写明**：synthesis 是跨源综合出的**阶段性判断**，output 是**一次问答**的存档。判断不清时看是否综合了多个来源。

**第六章：核心原则**

五条，逐条写：

1. **raw 不可变**：不改 body、不改 frontmatter、不移动（除归档整理）、不改名（除精简命名）、不删除。ingest 状态由日志追踪，不在 raw 上做标记。
2. **双链双份维护**：正文里写行内 `[[]]` wikilink（给人看、给 Graph View 看）；frontmatter 里填 `sources`、`related`、`supports`、`contradicts`（给 Bases/Dataview 查询用）。**两边都不能偷懒。**
3. **冲突不静默抹平**：用 Obsidian callout 语法标注，原文如下——

```markdown
> [!contradiction] 来源A vs 来源B
> [[摘要/A]] 声称 X，但 [[摘要/B]] 声称 Y。
> 当前状态：未解决 / 已解决（说明理由）
```

   必须写明：**「当前状态：已解决」同样要写**——判定为语义不同的两条标尺、而非同一命题的冲突时，把理由写在 callout 里，而不是删掉 callout。
4. **命名规范**：精简中文短标题，去掉"一文看懂""建议收藏"等营销词与副标题；概念页文件名即概念名。
   保留现有三条跨层重名与全路径链接的规则原文。
   保留「不重命名插件生成的文件」原文。
   **新增一段「与上游范式的刻意偏离」**：文章要求文件名为 kebab-case、中文标题只放 frontmatter 的 `title`；本库**有意不采用**，因为这是中文 vault，kebab-case 意味着拼音 slug（`qing-jie-suan-mo-kuai-she-ji`），会让 `[[清结算模块设计]]` 这类 wikilink 完全丧失可读性。文章原文亦声明目录与命名"根据自己的业务场景和需求来"。同理 `created`／`updated` 有意不改为文章示例中的 `date_created`／`date_modified`。
5. **不确定就标不确定**：不用 `confidence: high` 标注仅凭单一薄弱来源的推断；来源没说的不要用通用知识补全，写明"现有来源未覆盖"。

**第七章：操作流程**

四个操作各给概要，并**注明详细步骤在 `.claude/commands/<名>.md`**。

- **Ingest**：新资料放入 `原始资料/剪藏/` → 与用户确认重点方向 → 写摘要页 → 更新/新建概念页实体页 → 维护交叉引用 → 更新 `索引.md` → 追加一行到 `操作日志.md`。一篇资料通常触及 5–15 个页面。
- **Query**：先读 `索引.md` 靠 `tldr` 定位 → 读相关页面综合作答 → 每个关键论断带 `[[]]` 引用并标置信度 → **wiki 不够用时才回溯读原始资料**。
- **Lint**：六类检查——矛盾 / 过时 / 孤儿页 / 缺失页面 / 断链 / 信息缺口。产出 `lint-report-YYYY-MM-DD.md`。
- **Save**：把有价值的回答回填成页面——跨源综合去 `综合/`，单次问答去 `输出/`。

写明一条硬规矩：**每个操作完成后，`索引.md` 和 `操作日志.md` 必须同步更新。没更新，等于没做。**

**第八章：索引与日志规范**

- `索引.md`：按页面类型分节（摘要／概念／实体／综合／输出／综述），每条 = 链接 + `tldr`。每次 ingest 后更新。
- `操作日志.md`：append-only。每条格式 `## [YYYY-MM-DD] 动作 | 标题`，标题下用 bullet 记涉及页面与备注。写明这个格式的用途：
  ```bash
  grep "^## \[" "知识库/操作日志.md" | tail -5
  ```
  日志的另一个用途：**追踪哪些原始资料已被 ingest，避免重复处理**。

**第九章：语言与风格**

沿用原有内容，并保留「硬性禁止」小节（不修改 `原始资料/` 正文 / 不删除用户笔记 / 不静默抹平来源冲突 / 不用 `confidence: high` 标注薄弱推断）。

- [ ] **Step 3: 验证**

```bash
cd "D:/张梦奇ob"
test -f "知识库/输出/.gitkeep" && echo "输出/ OK"
grep -c "^## " CLAUDE.md
grep -n "tldr\|contradicts\|output\|kebab\|c-evolve\|共同演化" CLAUDE.md | head -20
```
**期望：** 输出目录存在；九章齐备；`tldr`、`contradicts`、`type: output`、kebab-case 偏离说明、co-evolve 说明均在文件中出现。

- [ ] **Step 4: Commit**

```bash
cd "D:/张梦奇ob"
git add "CLAUDE.md" "知识库/输出/.gitkeep"
git commit -m "feat: schema 升级——补齐 tldr/related/contradicts、callout 冲突标注、output 页面类型与 stub 建页规则"
```

---

### Task 2: commands 层

**Files:**
- Create: `.claude/commands/ingest.md`
- Create: `.claude/commands/query.md`
- Create: `.claude/commands/lint.md`
- Create: `.claude/commands/save.md`

**Interfaces:**
- Consumes: Task 1 的 CLAUDE.md（本任务不重复其规则，只写操作步骤）
- Produces: 四个 slash 命令，供用户随时触发

- [ ] **Step 1: 写 `.claude/commands/ingest.md`**

frontmatter 的 `description` 用这一句，逐字：
```
description: "处理 原始资料/ 中的新来源，编译为知识库页面"
```

正文按以下**结构**组织（各节标题即如下标题），步骤逐条照写：

**步骤：**
1. 查 `知识库/操作日志.md`，确认哪些原始资料尚未被 ingest（日志中已有 ingest 记录的跳过）
2. 扫描 `原始资料/剪藏/`、`原始资料/截图笔记/`、`原始资料/古籍/` 查找待处理文件
3. 确定 ingest 单元：用户明确指定的文件优先；否则单个文件为一个单元，一批 1–5 个
4. 完整阅读来源全文
5. **与用户讨论关键发现，确认本次想强调什么方向**
6. 在 `知识库/摘要/` 创建摘要页：填写完整 frontmatter（含 `tldr`、`raw`），核心摘要 200–500 字，列出可抽取的概念与实体（带 `[[]]`），标注与现有页面的连接
7. 创建/更新概念页与实体页（`知识库/概念/`、`知识库/实体/`）：**2+ 来源 → 完整页；首次出现 → `status: stub` 的 stub 页**
8. 维护交叉引用：更新相关页面的 `sources`／`related` 字段，确保无 dangling wikilink
9. 更新 `知识库/索引.md`：添加新条目（链接 + `tldr`）
10. 追加 `知识库/操作日志.md`：

```markdown
## [YYYY-MM-DD] ingest | 标题
- 处理：[[摘要/xxx]]
- 原始资料：原始资料/剪藏/xxx.md
```

**质量检查：**
- [ ] 所有新页面含完整 frontmatter（含 `tldr`）
- [ ] 所有 `[[]]` 链接指向已存在的页面（无 dangling）
- [ ] 摘要页含反方观点或数据缺口（如适用）
- [ ] `索引.md` 与 `操作日志.md` 已更新
- [ ] **`原始资料/` 下文件未被修改**（不移动、不改名、不改内容）

**重要约束：**
- `原始资料/` **完全不可变**，不修改、不移动、不重命名
- 处理状态只由 `操作日志.md` 追踪，不在 `原始资料/` 层面做任何标记
- `raw` 字段须含完整子目录路径，如 `[[原始资料/剪藏/filename]]`
- 来源没说的内容不要用通用知识补全；写明"现有来源未覆盖"

- [ ] **Step 2: 写 `.claude/commands/query.md`**

`description` 逐字：
```
description: "基于知识库内容回答问题，附带引用"
```

**步骤：**
1. **读 `知识库/索引.md`**：靠每条的 `tldr` 快速定位与问题相关的页面
2. 拉取相关页面的完整内容
3. 判断是否需要回溯原始资料：知识库信息不足时，看相关摘要页的 `raw` 字段，再去读 `原始资料/` 下的原文；若知识库中存在 `> [!contradiction]` callout，**必须展示双方观点与当前状态**
4. 综合作答：每个关键论断**必须**带 `[[]]` 页面引用；标注置信度（high/medium/low）；存在矛盾时明确展示
5. 建议归档：本次问答若有长期价值，建议用户用 `/save` 归档；若发现缺某个应有页面的概念/实体，记下来供下次 ingest 补充

**注意事项：**
- **永远优先从 `知识库/` 回答，而不是从 `原始资料/` 重新检索**——这正是本库与 RAG 的根本区别
- 知识库完全没有覆盖这个问题时，明确告知用户，建议添加相关来源
- 引用原始资料时用全路径，如 `[[原始资料/剪藏/清结算模块设计]]`

- [ ] **Step 3: 写 `.claude/commands/lint.md`**

`description` 逐字：
```
description: "知识库健康检查：查找孤儿页、断链、矛盾与缺失字段"
```

**检查项：**

*1. 结构完整性*
- [ ] **断链**：扫描所有 `[[]]`，检查目标是否存在。**注意本库链接解析有两个根**——wiki 内部链接（`[[摘要/X]]`）在 `知识库/` 下解析，跨层链接（`[[原始资料/剪藏/X]]`）在 vault 根解析，短链（`[[清结算]]`）需在 `知识库/` 下递归查找。三者都要试，只试一个根会全部误报
- [ ] **孤儿页**：无任何入链的页面（`索引.md`、`操作日志.md`、`综述.md` 除外）
- [ ] **索引同步**：`知识库/` 下所有 `.md` 是否都在 `索引.md` 中列出

*2. frontmatter 合规*
- [ ] **必填字段**：每页是否有 `title`、`tldr`、`type`、`status`、`created`、`updated`
- [ ] **type 合法**：是否为 `source` / `entity` / `concept` / `synthesis` / `output` 之一
- [ ] **status 合法**：是否为 `stub` / `draft` / `stable` / `needs-review` 之一

*3. 内容质量*
- [ ] **矛盾未标注**：是否有两页提出冲突结论但未用 `> [!contradiction]` callout 标注
- [ ] **缺口未标明**：概念页/摘要页是否缺少反方观点或数据缺口说明
- [ ] **stub 僵局**：`status: stub` 的页面是否长期未扩充
- [ ] **积压**：`status: needs-review` 与 `confidence: low` 的页面是否长期未处理
- [ ] **过时内容**：`updated` 超过 6 个月的页面

**输出：**
1. 生成报告到 `知识库/lint-report-YYYY-MM-DD.md`
2. **低风险问题自动修复**：`索引.md` 漏条目、简单 typo 等
3. **高风险问题仅报告**：矛盾标注、内容过时等需用户判断
4. 追加 `知识库/操作日志.md`：`## [YYYY-MM-DD] lint | 检查结果摘要`

**约束：** lint 不得修改 `原始资料/` 下任何文件。

- [ ] **Step 4: 写 `.claude/commands/save.md`**

`description` 逐字：
```
description: "将对话中有价值的内容归档到知识库"
```

**步骤：**
1. **判断类型**：
   - **跨源综合**（综合多个来源的比较、结论）→ `知识库/综合/`
   - **单次问答**（基于知识库回答的一个问题）→ `知识库/输出/`
2. 从内容中提炼精简中文标题作文件名
3. 创建页面：使用对应模板，填写完整 frontmatter（含 `tldr`、`sources`、`related`），正文整理为**结构化的知识库内容，不是聊天记录的复制**
4. 维护交叉引用：在相关页面的 `related` 字段中加入新页面；在 `知识库/索引.md` 添加条目
5. 追加 `知识库/操作日志.md`：`## [YYYY-MM-DD] save | 标题`

**质量检查：**
- [ ] 内容已结构化为知识库格式（标题、列表、表格），不是原始聊天记录
- [ ] 所有关键论断有 `[[]]` 引用
- [ ] frontmatter 完整（含 `tldr`）
- [ ] `索引.md` 与 `操作日志.md` 已更新

- [ ] **Step 5: 验证**

```bash
cd "D:/张梦奇ob"
ls ".claude/commands"
for f in ingest query lint save; do
  echo "--- $f ---"
  head -3 ".claude/commands/$f.md"
  grep -c "^[0-9]\+\." ".claude/commands/$f.md"
done
```
**期望：** 四个文件齐备；每个都有 YAML frontmatter 且带 `description`；步骤编号存在。

```bash
cd "D:/张梦奇ob"
git check-ignore -v ".claude/commands/ingest.md" || echo "未被忽略，可入库"
```
**期望：** 未被忽略。

- [ ] **Step 6: Commit**

```bash
cd "D:/张梦奇ob"
git add ".claude/commands"
git commit -m "feat: 建立 commands 层——ingest / query / lint / save 四个操作"
```

---

### Task 3: 操作日志改为可 grep 格式

**Files:**
- Modify: `知识库/操作日志.md`（全文重写，信息量不得减少）

**Interfaces:**
- Consumes: Task 1 第八章的格式规定
- Produces: Task 5 要追加记录的日志文件

- [ ] **Step 1: 重写 `知识库/操作日志.md`**

原有 4 条记录的信息必须**全部保留**（含 lint 那条里列出的待补概念页清单）。新格式：

```markdown
# 操作日志

append-only，不修改历史条目。每条格式 `## [YYYY-MM-DD] 动作 | 标题`，标题下记细节。

```bash
grep "^## \[" "知识库/操作日志.md" | tail -5
```

日志同时用于追踪哪些原始资料已被 ingest，避免重复处理。

## [2026-09-17] ingest | 清结算模块设计、支付系统技术决策、跨库 Join 方案
- 产出：[[摘要/清结算模块设计]]、[[摘要/支付系统技术决策]]、[[摘要/跨库 Join 方案]]

## [2026-09-17] 建立 | 概念页
- [[清结算]]、[[支付网关]]、[[跨库 Join]]、[[分库分表]]

## [2026-09-17] 建立 | 索引与综述
- [[索引]]、[[综述]]

## [2026-09-17] lint | 首次全库体检
- 孤儿页 0、失效链接 0
- 待补概念页：对账、幂等性、分布式系统、数据异构、轧差
- 页面矛盾 0
- 详见阶段验收报告
```

注意：嵌套代码块在文件里是正常的 markdown 围栏，实现时直接写 ```` ```bash ```` 围栏即可，不要写成 ```` ````bash ````。

- [ ] **Step 2: 验证**

```bash
cd "D:/张梦奇ob"
grep "^## \[" "知识库/操作日志.md" | wc -l
grep -c "对账\|幂等性\|分布式系统\|数据异构\|轧差" "知识库/操作日志.md"
```
**期望：** 4 条记录；待补概念页清单 5 个词全部出现（信息未丢失）。

- [ ] **Step 3: Commit**

```bash
cd "D:/张梦奇ob"
git add "知识库/操作日志.md"
git commit -m "refactor: 操作日志改为可 grep 的标题式格式"
```

---

### Task 4: 现有页面回填新字段

> **计划修订 2026-09-18（Task 1 完成后）**：Task 1 把第五章的必填字段定为六个
> （`title`/`tldr`/`type`/`status`/`created`/`updated`），但 4 个摘要页目前**这四个新字段一个都没有**。
> 原 Task 4 只回填 `tldr`，会让 Task 6 的 frontmatter 合规检查必然失败。本任务因此扩为
> **回填全部六个必填字段**，并追加 Step 0 补齐 schema 自身的两处欠定义。

**Files:**
- Modify: `CLAUDE.md`（仅 Step 0 的两处定义补齐）
- Modify: `知识库/摘要/清结算模块设计.md`、`知识库/摘要/支付系统技术决策.md`、`知识库/摘要/跨库 Join 方案.md`、`知识库/摘要/MIT 6.824 分布式系统.md`
- Modify: `知识库/概念/清结算.md`、`知识库/概念/支付网关.md`、`知识库/概念/跨库 Join.md`、`知识库/概念/分库分表.md`
- Modify: `知识库/综述.md`

**Interfaces:**
- Consumes: Task 1 第五章的字段规范与第六章的 callout 语法
- Produces: 8 个页面的 frontmatter 与 schema 一致；Task 6 的 lint 依赖此一致性

- [ ] **Step 0: 补齐 CLAUDE.md 的两处欠定义**

Task 1 的审查发现两处：某个字段在模板里被用了，但第五章从没定义它。

**（a）`sources` 的取值范围。** 第五章第 2 节的定义写的是「本页依据的摘要页」，但第五章第 4 节的 `output` 模板里 `sources` 填的是概念页 `[[支付网关]]`。两者矛盾。把定义改为：

> | `sources` | 本页依据的页面（摘要页、概念页、实体页均可，不限摘要页） |

**（b）`question` 字段。** `synthesis` 与 `output` 两个模板都用了 `question`，但全篇没有定义。在第五章第 4 节 `output` 模板之后补一句：

> `question`：本页要回答的那个问题原文，用于让检索者判断这页是否对得上自己的问题。`type: output` 与 `type: synthesis` 必填。

**（c）`status` 四档只定义了 `stub`。** 第五章第 5 节补全其余三档：

> - `stub` —— 仅一个来源，只记定义与出处，等第二来源出现再扩写
> - `draft` —— LLM 已写完，但用户尚未确认
> - `stable` —— 用户已通读并确认
> - `needs-review` —— 发现疑问或来源可能已过时，待查

- [ ] **Step 1: 给 8 个页面补齐六个必填字段**

每个页面检查 `title`、`tldr`、`type`、`status`、`created`、`updated` 六项，缺哪个补哪个。

- `title`／`type`：8 页均已有，不动。
- `tldr`：**新建**。≤60 字，且**与其在 `知识库/索引.md` 中已有的一行描述对齐**——索引条目的文字就是 tldr，不要出现两份互相打架的摘要。从索引里该页已有的一行描述精简到 60 字以内；若索引描述本身已 ≤60 字则直接沿用。统一插在 `title` 下一行。
- `status`：8 页统一 `draft`（本次是 LLM 自行回填，用户尚未逐页确认；`stable` 留给用户确认后再改）。
- `created`：摘要页取该页已有的 `captured` 值（即资料抓取日）；概念页保留各自已有的 `created: 2026-09-17`。
- `updated`：8 页统一改为 `2026-09-18`（本次确实改了它们）。

- [ ] **Step 2: 给 4 个概念页加 `related`**

概念页之间已通过正文短链互引，按实际存在的关系填 `related`：

| 页面 | `related` |
|---|---|
| `清结算` | `["[[支付网关]]", "[[跨库 Join]]"]` |
| `支付网关` | `["[[清结算]]"]` |
| `跨库 Join` | `["[[分库分表]]", "[[清结算]]"]` |
| `分库分表` | `["[[跨库 Join]]"]` |

**先读每个页面正文确认这些关系在正文里确实存在**，若某条关系正文未提及则不要填（`related` 是实质关系，不是礼貌性互链）。

- [ ] **Step 3: 综述.md 的张力改 callout**

把「一处张力（并列，不抹平）」小节下的正文，改为按第六章的 callout 语法呈现。要点：
- callout 标题：`来源A vs 来源B` 替换为两篇摘要页的简称
- 必须写「当前状态」，且**此处的状态是「已解决」**——综述原文已论证这不构成事实矛盾（两者针对的数据语义不同），理由要写进 callout
- callout 之后保留原有的追问段落（金额/状态类字段冗余时几小时容忍窗口是否成立的缺口）

- [ ] **Step 4: 验证**

```bash
cd "D:/张梦奇ob"
echo "--- 8 页六字段是否齐全（期望无 MISSING 行）---"
python -c "
import re, glob, os
REQ = ['title','tldr','type','status','created','updated']
for p in sorted(glob.glob('知识库/摘要/*.md') + glob.glob('知识库/概念/*.md')):
    fm = re.match(r'^---\n(.*?)\n---', open(p, encoding='utf-8').read(), re.S)
    if not fm: print('NO FRONTMATTER:', p); continue
    miss = [k for k in REQ if not re.search(rf'^{k}:', fm.group(1), re.M)]
    if miss: print('MISSING', miss, 'in', os.path.basename(p))
print('field check done')
"
echo "--- tldr 超 60 字检查（按字符数，非字节）---"
python -c "
import re, glob, os
for p in sorted(glob.glob('知识库/摘要/*.md') + glob.glob('知识库/概念/*.md')):
    m = re.search(r'^tldr:\s*(.+)$', open(p, encoding='utf-8').read(), re.M)
    if not m: continue
    t = m.group(1).strip().strip('\"')
    if len(t) > 60: print('TOO LONG', len(t), os.path.basename(p), t)
print('tldr length done')
"
echo "--- callout ---"
grep -n "\[!contradiction\]" "知识库/综述.md"
```
**期望：** `field check done` 前无 MISSING/NO FRONTMATTER 行；`tldr length done` 前无 TOO LONG；综述含 contradiction callout。

（用 Python 而非 `awk length()` 数字符——awk 按字节计，一个中文字算 3，会全部误报。）

⚠️ 中文字符在 awk `length()` 下可能按字节计数，若出现 TOO LONG 需人工核对是否为误报（一个中文字 3 字节）。

- [ ] **Step 5: Commit**

```bash
cd "D:/张梦奇ob"
git add "CLAUDE.md" "知识库/摘要" "知识库/概念" "知识库/综述.md"
git commit -m "feat: 补齐六个必填字段与 related，综述张力改用 contradiction callout"
```

---

### Task 5: ingest 新剪藏

**Files:**
- Create: `知识库/摘要/如何构建 LLM Wiki.md`
- Create: `知识库/概念/LLM Wiki.md`
- Modify: `知识库/索引.md`
- Modify: `知识库/操作日志.md`

**Interfaces:**
- Consumes: Task 0 的 `原始资料/剪藏/如何构建 LLM Wiki.md`；Task 1 的页面模板与建页规则；Task 3 的日志格式
- Produces: Task 6 的 lint 对象

- [ ] **Step 1: 读来源全文**

`原始资料/剪藏/如何构建 LLM Wiki.md`，**只读，不修改**。

- [ ] **Step 2: 写 `知识库/摘要/如何构建 LLM Wiki.md`**

frontmatter（来源自身的 frontmatter 有 `title` / `source` / `author`（空）/ `published`（空）/ `created: 2026-09-18` / `tags: [clippings]` / `description`（空））：

```yaml
---
type: source
title: 如何构建 LLM Wiki
tldr: <预制 ≤60 字：三步搭起 LLM Wiki——建目录、写 CLAUDE.md、配四个命令>
source_title: "✅如何构建LLM Wiki？ · 云效 Thoughts · 企业级知识库"
source: https://thoughts.aliyun.com/workspaces/6963289eb0fc2e001bb052eb/docs/6a40f9fdc71a890001618034
source_type: 云效知识库
author: []
published: null
captured: 2026-09-18
tags: [clippings, LLM Wiki]
raw: "[[原始资料/剪藏/如何构建 LLM Wiki]]"
---
```

`tldr` 与正文一句话主旨须自己写，不要照抄占位。

正文按既有摘要页结构写：一句话主旨 → **关键论点（3–7 条）** → 与其他资料的关系 → 值得追问的问题。

关键论点须覆盖来源的真实内容，至少含：三步走（建目录／写 CLAUDE.md／配 commands）；raw-wiki-schema 三层与各自权限；五种页面类型及其建页时机；双链双份维护（行内 + frontmatter）；冲突用 callout 标注不静默抹平；四个命令 ingest/query/lint/save 各自的作用；index 的 tldr 是检索承重字段、log 可 grep 且用于追踪 ingest 状态；co-evolve。

「与其他资料的关系」：本页与其他摘要页**没有主题关联**（题材正交），但它是本库 `CLAUDE.md` 与 `.claude/commands/` 的直接依据——**如实写明这一点，不要为了连上图谱而编造关联**。

「值得追问的问题」：从来源的留白里提，例如 kebab-case 命名是否适用于中文 vault、stub 页何时该升级、save 归档的页面由谁复核等。

- [ ] **Step 3: 写 `知识库/概念/LLM Wiki.md`**

按新建页规则：`LLM Wiki` 是**单一来源**概念（本库仅此一篇涉及），故建 **stub** 页。

```yaml
---
type: concept
title: LLM Wiki
tldr: <预制 ≤60 字：以 LLM 为维护者、把原始资料编译成双链知识库的三层范式>
aliases: [LLM 知识库]
tags: [方法论]
sources: ["[[摘要/如何构建 LLM Wiki]]"]
related: []
confidence: high
status: stub
created: 2026-09-18
updated: 2026-09-18
---
```

`confidence: high` 的依据：CLAUDE.md 规定 `high` = **单一来源直接陈述**或多来源完全一致——本页内容是该来源的直接陈述，符合。

正文只写定义与出处（stub 的定义）：三层结构（raw / wiki / schema）、四个操作（ingest / query / lint / save）、五种页面类型，并写明"本页暂无第二来源，待后续资料扩充"。

- [ ] **Step 4: 更新 `知识库/索引.md`**

在「## 摘要」节追加一行，在「## 概念」节追加一行。**注意排序**：摘要节现在按主题排（支付三篇 + MIT），新条目题材正交，追加在末尾即可。

「## 输出」「## 实体」节保持原样（均为「（暂无）」）。

- [ ] **Step 5: 追加 `知识库/操作日志.md`**

```markdown
## [2026-09-18] ingest | 如何构建 LLM Wiki
- 处理：[[摘要/如何构建 LLM Wiki]]、[[概念/LLM Wiki]]
- 原始资料：原始资料/剪藏/如何构建 LLM Wiki.md
- 备注：本页是本库 CLAUDE.md 与 .claude/commands/ 的直接依据
```

- [ ] **Step 6: 验证**

```bash
cd "D:/张梦奇ob"
echo "--- 新页存在且有 tldr ---"
grep -l "^tldr:" "知识库/摘要/如何构建 LLM Wiki.md" "知识库/概念/LLM Wiki.md"
echo "--- raw 回指有效 ---"
test -f "原始资料/剪藏/如何构建 LLM Wiki.md" && echo "raw 目标存在"
echo "--- 原始资料未被修改 ---"
git -C "D:/张梦奇ob" status --short "原始资料" | head
echo "--- 索引已收录 ---"
grep -c "如何构建 LLM Wiki\|LLM Wiki" "知识库/索引.md"
```
**期望：** 两页均有 `tldr`；raw 目标存在；`原始资料` 无改动输出；索引命中 ≥2 处。

- [ ] **Step 7: Commit**

```bash
cd "D:/张梦奇ob"
git add "知识库/摘要/如何构建 LLM Wiki.md" "知识库/概念/LLM Wiki.md" "知识库/索引.md" "知识库/操作日志.md"
git commit -m "feat: ingest 如何构建 LLM Wiki——摘要页、LLM Wiki 概念 stub、索引与日志"
```

---

### Task 6: lint 首跑与阶段验收

**Files:**
- Create: `知识库/lint-report-2026-09-18.md`
- Modify: `知识库/操作日志.md`（追加 lint 记录）
- Modify: `知识库/索引.md`（仅当 lint 发现遗漏）

**Interfaces:**
- Consumes: Task 0–5 的全部产物
- Produces: 验收结论。通过后用户决定阶段 4（其余 20 篇剪藏的全量编译）

- [ ] **Step 1: 断链检查（两根解析）**

```bash
cd "D:/张梦奇ob"
python -c "
import re, os, glob
missing = []
for p in glob.glob('知识库/**/*.md', recursive=True):
    t = open(p, encoding='utf-8').read()
    for link in re.findall(r'\[\[([^\]|#]+)', t):
        link = link.strip()
        if not link: continue
        ok = os.path.exists(link + '.md') \
             or os.path.exists(os.path.join('知识库', link + '.md')) \
             or bool(glob.glob('知识库/**/' + link + '.md', recursive=True))
        if not ok:
            missing.append(f'{p} -> [[{link}]]')
print('MISSING:' if missing else 'ALL LINKS OK')
for m in missing: print(' ', m)
"
```
**期望：** `ALL LINKS OK`。三个解析根缺一不可——只试 vault 根会漏掉 `[[摘要/X]]`，只试 `知识库/` 会漏掉 `[[原始资料/剪藏/X]]`。

- [ ] **Step 2: 孤儿页检查**

```bash
cd "D:/张梦奇ob"
python -c "
import re, glob, os
pages, inbound = {}, {}
for p in glob.glob('知识库/**/*.md', recursive=True):
    name = os.path.splitext(os.path.basename(p))[0]
    pages[name] = p
    inbound.setdefault(name, set())
for p in glob.glob('知识库/**/*.md', recursive=True):
    t = open(p, encoding='utf-8').read()
    for link in re.findall(r'\[\[([^\]|#]+)', t):
        link = link.strip().split('/')[-1]
        if link in pages: inbound.setdefault(link, set()).add(p)
for n, p in sorted(pages.items()):
    if n in ('索引','操作日志','综述'): continue
    if not inbound.get(n): print('ORPHAN:', n)
print('done')
"
```
**期望：** 除 `索引`／`操作日志`／`综述` 外无输出。新页 `如何构建 LLM Wiki` 应被 `概念/LLM Wiki` 入链；`LLM Wiki` 应被摘要页与索引入链。

- [ ] **Step 3: frontmatter 合规检查**

```bash
cd "D:/张梦奇ob"
python -c "
import re, glob, os
REQ = ['title','tldr','type','status','created','updated']
for p in glob.glob('知识库/**/*.md', recursive=True):
    n = os.path.basename(p)
    if n in ('索引.md','操作日志.md','综述.md'): continue
    t = open(p, encoding='utf-8').read()
    m = re.match(r'^---\n(.*?)\n---', t, re.S)
    if not m: print('NO FRONTMATTER:', p); continue
    fm = m.group(1)
    miss = [k for k in REQ if not re.search(rf'^{k}:', fm, re.M)]
    if miss: print('MISSING', miss, 'in', p)
print('done')
"
```
**期望：** 无输出（除 `done`）。

- [ ] **Step 4: 写 `知识库/lint-report-2026-09-18.md`**

报告须含：
- 检查项与结果（断链 / 孤儿 / frontmatter / 矛盾标注 / 缺口）
- 页面清单（各类型页数）
- **缺失概念页候选**：沿用上次 lint 的 5 个（对账、幂等性、分布式系统、数据异构、轧差），并核对 `知识库/` 现状后修正
- 高风险问题仅报告，低风险已自动修复的列出修了什么

- [ ] **Step 5: 追加 lint 记录**

```markdown
## [2026-09-18] lint | 范式对齐后全库体检
- 断链 0、孤儿页 0、frontmatter 合规
- <实际发现的缺失概念页候选>
```

- [ ] **Step 6: 一致性验收**

```bash
cd "D:/张梦奇ob"
echo "--- 根目录 ---"
ls -a | grep -v "^\.$\|^\.\.$"
echo "--- 知识库 ---"
find "知识库" -name "*.md" | sort
echo "--- commands ---"
ls ".claude/commands"
echo "--- 原始资料自上一阶段后有无内容改动 ---"
git log --oneline dcf8961..HEAD -- "原始资料" | head
echo "--- 工作树 ---"
git status --short
```

逐项对照本计划的 Global Constraints 与用户裁决 A–D：

- [ ] 裁决 A：CLAUDE.md 含 kebab-case 刻意偏离说明；文件名仍为中文
- [ ] 裁决 B：`操作日志.md` 可被 `grep "^## \["` 提取
- [ ] 裁决 C：`知识库/输出/` 存在；CLAUDE.md 的 Query 章指向它而非 `综合/`
- [ ] 裁决 D：新剪藏在 `原始资料/剪藏/`，且已 ingest
- [ ] Global Constraint 2：`原始资料/` 自 `dcf8961` 起无内容改动（只有 Task 0 的移动）
- [ ] 根目录无 `Clippings/` 残留
- [ ] 工作树干净

- [ ] **Step 7: Commit**

```bash
cd "D:/张梦奇ob"
git add "知识库/lint-report-2026-09-18.md" "知识库/操作日志.md"
git commit -m "chore: 范式对齐后首次 lint 与阶段验收"
```

- [ ] **Step 8: 向用户汇报**

汇报内容：本次产出清单、裁决 A–D 的落实情况、lint 结果、**仍需用户在 Obsidian 内验证的运行时项**（若上一阶段的每日笔记插件验证仍未做，一并提示）、以及阶段 4（其余 20 篇剪藏全量编译）的批次建议。

---

## Self-Review

**1. Spec coverage（对照文章逐条）**

| 文章约定 | 覆盖任务 |
|---|---|
| 建目录骨架 | 已符合（上一阶段）+ Task 1 Step 1 补 `输出/` |
| CLAUDE.md 核心定位与 LLM 角色 | Task 1 第一章 |
| 三层架构 + 完整目录树 + 权限 | Task 1 第二/三/四章 |
| ingest 状态由 log 追踪、不在 raw 标记 | Task 1 第四章 |
| 五种页面类型与模板 | Task 1 第五章 |
| raw 不可变 / 双链双份 / 命名 / 冲突 callout | Task 1 第六章 |
| ingest/query/lint/save 流程 | Task 1 第七章 + Task 2 全文 |
| index tldr / log 可 grep | Task 1 第八/五章 + Task 3 |
| commands 层 | Task 2 |
| entity/concept 2+ 来源规则 | Task 1 第五章 + Task 2 Step 1 第 7 步 |
| lint 报告产物 | Task 2 Step 3 + Task 6 |
| output 页面类型 | Task 1 第五章 + Task 5 无（本期无问答归档） |

**2. Placeholder scan**

计划中出现的 `<...>` 均为**实现者应当自行撰写的正文内容**（如 `tldr` 文案），且每处都给出了明确的内容要求与字数约束，不是"TBD"式空缺。

**3. Type consistency**

- 字段名 `tldr` / `related` / `supports` / `contradicts` / `status` 在 Task 1、4、5、6 中一致
- `status` 取值 `stub|draft|stable|needs-review` 在 Task 1 第五章与 Task 2 Step 3 的 lint 检查项中一致
- 目录名 `知识库/输出/` 在 Task 1、2、5、6 中一致
- 日志格式 `## [YYYY-MM-DD] 动作 | 标题` 在 Task 1 第八章、Task 2 的 ingest/lint/save、Task 3、Task 5、Task 6 中一致

---

## 修订记录

### 修订 1（2026-09-18，Task 1 完成后）

Task 1 的审查与协调者各自独立发现同一处缺陷，外加审查员查到一处能力丢失。三处一并修正：

| # | 问题 | 修法 |
|---|---|---|
| 1 | **Task 4 覆盖面不足（计划缺陷）**。Task 1 把必填字段定为六个，但 4 个摘要页的 `tldr`／`status`／`created`／`updated` **一个都没有**；原 Task 4 只回填 `tldr`，Task 6 的 frontmatter 合规检查必然失败 | Task 4 扩为回填全部六个字段，Step 4 的验证脚本改为逐页检查六字段 |
| 2 | **schema 自身两处欠定义（审查员发现）**。`sources` 定义说「本页依据的摘要页」，但 `output` 模板里填了概念页；`question` 被两个模板使用却全篇未定义 | Task 4 新增 Step 0，在 CLAUDE.md 里补齐 `sources` 取值范围、`question` 定义，并补全 `status` 四档中未定义的另外三档 |
| 3 | **lint 能力丢失（审查员发现）**。上一版 CLAUDE.md 的 lint 清单含「`status: needs-review` 与 `confidence: low` 的积压」，Task 1 的六类检查里没有它的落点 | Task 2 的 `lint.md` 检查项补一条「积压」 |