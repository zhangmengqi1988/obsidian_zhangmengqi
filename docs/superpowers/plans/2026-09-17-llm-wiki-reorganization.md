# LLM-Wiki 知识库重构 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 按 Karpathy 的 LLM Wiki 范式，为本库补上缺失的 wiki 层与 schema 层，把散落的原始资料归入只读的 `原始资料/` 层。

**Architecture:** 三层——`原始资料/`（只读事实源）、`知识库/`（LLM 维护的知识页）、`CLAUDE.md`（规则层）。全靠文件系统移动与新增实现，不引入任何代码或依赖。

**Tech Stack:** Obsidian（Bases、wikilink、core plugins）+ git。无编程语言参与。

**依据 spec：** `docs/superpowers/specs/2026-09-17-llm-wiki-reorganization-design.md`

---

## Global Constraints

以下约束适用于**每一个** task，不再逐条重复：

1. **目录名与正文用中文，frontmatter 字段名用英文。** 理由：现有 `架构/-hovernotes.base` 公式依赖 `title`／`author`／`source`／`tags` 英文字段。
2. **`原始资料/` 只读。** 允许移动位置，**永不修改其中任何文件的正文**。
3. **wiki 内部链接一律用全路径形式** `[[摘要/清结算模块设计]]`。仅概念页之间的短链可写 `[[清结算]]`。
4. **每步独立 commit，全程用 `git mv`** 以保留文件历史。不要用 `mv` + `git add`。
5. **不重命名插件生成的文件。** `原始资料/每日笔记/2026/{7,8,9}月.md` 是 `daily-task-auto-generator` 按 `{年}/{月}月.md` 规则生成的，改名会导致插件重新生成副本。
6. **shell 是 Git Bash（Windows）**，路径含中文，所有路径参数必须加双引号。
7. **仓库被 `github-sync` 插件每 60 秒自动提交一次**，提交信息形如 `LAPTOP-RGRK37M4 2026-9-17:16:35:2`。核对历史时跳过这些噪音提交。

---

## File Structure

### 新建目录

| 路径 | 职责 |
|---|---|
| `原始资料/剪藏/` | 21 篇网页／视频剪藏，只读 |
| `原始资料/截图笔记/` | 5 篇纯截图笔记，只读 |
| `原始资料/每日笔记/` | 日志。`2026/` 子目录由插件写入 |
| `原始资料/古籍/` | 《官智经》原文 |
| `原始资料/附件/` | 108 张图片 |
| `原始资料/归档/` | 停用的 hover-notes Bases |
| `知识库/摘要/` | 每篇原始资料一页 |
| `知识库/概念/` | 想法、框架、模式 |
| `知识库/实体/` | 人物、工具、组织 |
| `知识库/综合/` | 回写的问答结论 |

### 新建文件

| 路径 | 职责 |
|---|---|
| `CLAUDE.md` | schema 规则层：目录权限、页面类型、frontmatter、工作流 |
| `知识库/索引.md` | 读库第一站，按类别列全部知识页 |
| `知识库/操作日志.md` | append-only 操作记录 |
| `知识库/综述.md` | 跨来源整体图景 |
| `.gitignore` | 排除 `messager` 插件密钥 |

### 删除

`未命名.md`、`未命名 1..8.md`（9 个，全 0 字节）、`未命名.base`、`未命名 1..4.base`（5 个空 Base）、`未命名.canvas`（`{}`）、`欢迎.md`（Obsidian 出厂默认文案）、`Clippings/Bilibili/_clean_preview.txt`

---

## Task 0: 安全处置——轮换并移除泄露的 API key

**这是唯一涉及不可逆外部动作的 task，必须先取得用户确认才能执行。**

`Clippings` 无关，但发现于调查阶段：`.obsidian/plugins/messager/data.json` 被 git 跟踪，内含 32 位明文 API key，且本库同步至 GitHub。

**Files:**
- Create: `.gitignore`
- Modify: git index（`.obsidian/plugins/messager/data.json` 从跟踪中移除，本地文件保留）

**Interfaces:**
- Produces: `.gitignore` 文件，Task 1–10 的提交不再包含该密钥

- [ ] **Step 1: 向用户确认**

执行前必须问：这个 key 是否已经轮换？如果仓库是公开的，旧 key 应立即失效。**未获确认则跳过本 task，直接进入 Task 1**，并在最终汇报中再次提醒。

- [ ] **Step 2: 创建 `.gitignore`**

```
# 第三方插件密钥，禁止入库
.obsidian/plugins/messager/data.json

# Obsidian 工作区状态（每次开关窗口都变，无版本价值）
.obsidian/workspace.json

# 系统垃圾
.DS_Store
Thumbs.db
```

- [ ] **Step 3: 从 git 索引移除密钥文件（保留本地副本）**

```bash
cd "D:/张梦奇ob"
git rm --cached ".obsidian/plugins/messager/data.json"
```

- [ ] **Step 4: 验证本地文件仍在、索引中已消失**

```bash
cd "D:/张梦奇ob"
ls -la ".obsidian/plugins/messager/data.json"   # 期望：文件存在
git ls-files ".obsidian/plugins/messager/"      # 期望：不含 data.json
```

- [ ] **Step 5: Commit**

```bash
cd "D:/张梦奇ob"
git add .gitignore
git commit -m "chore: 排除 messager 插件密钥与 workspace 状态入库"
```

---

## Task 1: 清库——删除垃圾与废弃文件

**Files:**
- Delete: 9 个空 `.md`、5 个空 `.base`、`未命名.canvas`、`欢迎.md`、`Clippings/Bilibili/_clean_preview.txt`

**Interfaces:**
- Produces: 干净的根目录，Task 2 建骨架时不与残留文件混淆

- [ ] **Step 1: 确认待删文件确实为空**

```bash
cd "D:/张梦奇ob"
echo "--- 空 md（期望全部为 0）---"
for f in 未命名.md 未命名\ *.md; do [ -f "$f" ] && printf "%s %s\n" "$(wc -c <"$f")" "$f"; done
echo "--- canvas 内容（期望 {}）---"
cat "未命名.canvas"
```

若任何一个非 0 字节，**停止并报告**——不要删除有内容的文件。

- [ ] **Step 2: 删除空 md 与 canvas**

```bash
cd "D:/张梦奇ob"
git rm "未命名.md" "未命名.canvas"
for n in 1 2 3 4 5 6 7 8; do git rm "未命名 $n.md"; done
```

- [ ] **Step 3: 删除空 Base**

```bash
cd "D:/张梦奇ob"
git rm "未命名.base"
for n in 1 2 3 4; do git rm "未命名 $n.base"; done
```

- [ ] **Step 4: 删除欢迎页与清理脚本残留**

```bash
cd "D:/张梦奇ob"
git rm "欢迎.md" "Clippings/Bilibili/_clean_preview.txt"
```

- [ ] **Step 5: 验证根目录已无 `未命名*`**

```bash
cd "D:/张梦奇ob"
ls 未命名* 2>&1        # 期望：No such file or directory
ls 欢迎.md 2>&1        # 期望：No such file or directory
```

- [ ] **Step 6: Commit**

```bash
cd "D:/张梦奇ob"
git commit -m "chore: 删除空笔记、空 Base、空 canvas 与出厂欢迎页"
```

---

## Task 2: 建目录骨架 + 图片归档

图片迁移安全性已在 spec 阶段验证：108 张图无重名，Obsidian 的 `![[Pasted image X.png]]` 按文件名全局解析，移动后全部嵌入照常渲染。

**Files:**
- Create: `原始资料/{剪藏,截图笔记,每日笔记,古籍,附件,归档}/`、`知识库/{摘要,概念,实体,综合}/`
- Move: 根目录 108 张图片 → `原始资料/附件/`

**Interfaces:**
- Produces: 全部后续 task 依赖的目录骨架

- [ ] **Step 1: 建目录骨架**

```bash
cd "D:/张梦奇ob"
mkdir -p "原始资料"/{剪藏,截图笔记,每日笔记,古籍,附件,归档}
mkdir -p "知识库"/{摘要,概念,实体,综合}
ls "原始资料" "知识库"
```

- [ ] **Step 2: 记录迁移前图片数**

```bash
cd "D:/张梦奇ob"
find . -maxdepth 1 -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" \) | wc -l
```
**期望：108**。记下这个数字，Step 4 要用。

- [ ] **Step 3: 移动全部图片**

```bash
cd "D:/张梦奇ob"
find . -maxdepth 1 -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" \) -exec git mv {} "原始资料/附件/" \;
```

- [ ] **Step 4: 验证数量一致且根目录已清空**

```bash
cd "D:/张梦奇ob"
find "原始资料/附件" -type f | wc -l      # 期望：108
find . -maxdepth 1 -type f \( -iname "*.png" -o -iname "*.jpg" \) | wc -l   # 期望：0
```

- [ ] **Step 5: 抽查嵌入是否仍渲染**

随机取一个引用了图片的笔记，确认图片文件名在 `原始资料/附件/` 中存在：

```bash
cd "D:/张梦奇ob"
grep -oh "!\[\[Pasted image [0-9]*\.png\]\]" "DailyTasks/2026/8月.md" 2>/dev/null | head -3
ls "原始资料/附件/" | grep -c "Pasted image"    # 期望：>0
```

逐个人工比对 grep 出的文件名确实存在于 `原始资料/附件/`。Task 5 迁移日志后，可把上面的
`DailyTasks/2026/8月.md` 换成 `原始资料/每日笔记/2026/8月.md` 再验一次。

- [ ] **Step 6: Commit**

```bash
cd "D:/张梦奇ob"
git commit -m "refactor: 建立三层目录骨架，108 张图片归档至 原始资料/附件"
```

---

## Task 3: 剪藏归位与重命名（21 篇）

重命名规则见 spec 第 5 节：去掉营销词与副标题，保留主题词；完整原标题已在各文件 frontmatter 的 `title` 字段中，不会丢失。

**Files:**
- Move: 21 篇剪藏 → `原始资料/剪藏/`

**Interfaces:**
- Produces: `原始资料/剪藏/` 下 21 篇定名的剪藏，为 Task 8/9 的摘要页提供 `raw:` 链接目标

### 3a. `Clippings/` 根级 9 篇

- [ ] **Step 1: 迁移并重命名**

```bash
cd "D:/张梦奇ob"
git mv "Clippings/AI时代，如何让自己越活越值钱？——对话《令人心动的ffer》带教律师史欣悦.md" "原始资料/剪藏/AI时代的个人价值.md"
git mv "Clippings/Matt Pocock 直播实战：Wayfinder 从想法到 spec 全流程.md" "原始资料/剪藏/Wayfinder 从想法到 spec.md"
git mv "Clippings/✅大模型微调的方法 · 云效 Thoughts · 企业级知识库.md" "原始资料/剪藏/大模型微调方法.md"
git mv "Clippings/手捏一个 vibe coding 面试专用方法论.md" "原始资料/剪藏/vibe coding 面试方法论.md"
git mv "Clippings/真正不缺钱的人，都做对了一件事——对话74岁台湾商业大佬金惟纯.md" "原始资料/剪藏/金惟纯访谈.md"
git mv "Clippings/顽主杯大神比赛找感觉：周期论1（位置、逻辑）.md" "原始资料/剪藏/炒股周期论.md"
```

### 3b. `高志凯谈判课` 三篇——按 `source:` URL 的 `p=` 参数定序

三个文件的 `title` 字段完全相同，文件名后缀与集数**不对应**。定序依据是 `source:` URL：

| 现文件名 | `source:` 中的 `p=` | 目标名 |
|---|---|---|
| `高志凯谈判课 全56讲.md` | 无（默认第1集） | `高志凯谈判课 01.md` |
| `高志凯谈判课 全56讲 1.md` | `p=2` | `高志凯谈判课 02.md` |
| `高志凯谈判课 全56讲 2.md` | `p=3` | `高志凯谈判课 03.md` |

- [ ] **Step 2: 先核验集数，再迁移**

```bash
cd "D:/张梦奇ob"
for f in "高志凯谈判课 全56讲.md" "高志凯谈判课 全56讲 1.md" "高志凯谈判课 全56讲 2.md"; do
  printf "%s -> %s\n" "$f" "$(grep -m1 '^source:' "Clippings/$f")"
done
```
**期望**：依次看到无 `p=`、`p=2`、`p=3`。若实际不符，**按实际 `p=` 值调整目标名**，不要照抄本表。

- [ ] **Step 3: 迁移**

```bash
cd "D:/张梦奇ob"
git mv "Clippings/高志凯谈判课 全56讲.md"   "原始资料/剪藏/高志凯谈判课 01.md"
git mv "Clippings/高志凯谈判课 全56讲 1.md" "原始资料/剪藏/高志凯谈判课 02.md"
git mv "Clippings/高志凯谈判课 全56讲 2.md" "原始资料/剪藏/高志凯谈判课 03.md"
```

### 3c. Bilibili 视频剪藏 1 篇

- [ ] **Step 4: 迁移**

```bash
cd "D:/张梦奇ob"
git mv "Clippings/Bilibili/2026-08-01-【硬核】官宦世家的不传之秘，为什么圣人的书拿来办事百无一用？.md" "原始资料/剪藏/官宦世家与圣人之说.md"
```

### 3d. 库根目录 8 篇

- [ ] **Step 5: 迁移并重命名**

```bash
cd "D:/张梦奇ob"
git mv "一文看懂清结算模块设计从业务需求和设计落地.md" "原始资料/剪藏/清结算模块设计.md"
git mv "不做老好人三步打造有效人脉关系.md" "原始资料/剪藏/有效人脉关系.md"
git mv "从128只红利基金中精选15只分红频率和估值水平一次看清.md" "原始资料/剪藏/红利基金筛选.md"
git mv "做支付系统要注意的几个技术决策.md" "原始资料/剪藏/支付系统技术决策.md"
git mv "吃透SpringBoot自动装配的核心原理约定大于配置30版本.md" "原始资料/剪藏/Spring Boot 自动装配原理.md"
git mv "请设计一个Java项目组件SpringBootStarter全解析与案例应用建议收藏.md" "原始资料/剪藏/Spring Boot Starter 全解析.md"
git mv "SpringBoot4终于原生支持API版本控制了我把项目里那套v1v2重新改了一遍.md" "原始资料/剪藏/Spring Boot 4 API 版本控制.md"
git mv "红利估值的误区99的人都搞错了简单说红利10.md" "原始资料/剪藏/红利估值误区.md"
```

### 3e. `架构/` 3 篇

- [ ] **Step 6: 迁移并重命名**

```bash
cd "D:/张梦奇ob"
git mv "架构/DDD概念与架构从分层到六边形整洁.md" "原始资料/剪藏/DDD 分层到六边形架构.md"
git mv "架构/从0到1建设美团数据库容量评估系统.md" "原始资料/剪藏/美团数据库容量评估系统.md"
git mv "架构/大厂面试官连环追问跨库Join你的方案代价是什么.md" "原始资料/剪藏/跨库 Join 方案.md"
```

- [ ] **Step 7: 验证共 21 篇，且无重名**

```bash
cd "D:/张梦奇ob"
ls "原始资料/剪藏" | wc -l                                  # 期望：21
ls "原始资料/剪藏" | sort | uniq -d                          # 期望：无输出
ls "Clippings"                                               # 期望：只有 Bilibili
```

- [ ] **Step 8: Commit**

```bash
cd "D:/张梦奇ob"
git commit -m "refactor: 21 篇剪藏归入 原始资料/剪藏 并精简命名"
```

---

## Task 4: 截图笔记、古籍与 hover-notes 归档

**Files:**
- Move: `学习/**` 4 篇 + `架构/数据基础概念.md` → `原始资料/截图笔记/`
- Move: `Clippings/Bilibili/官宦世家的不传之秘.md` → `原始资料/古籍/官智经.md`
- Move: `架构/*.base` 3 个 → `原始资料/归档/`

**Interfaces:**
- Produces: `学习/` 与 `架构/` 目录清空，Task 6 可安全移走 `Untitled.md`

- [ ] **Step 1: 迁移截图笔记（拉平深层目录树）**

`学习/` 的 `股票理论/牵牛/道氏理论/` 三层嵌套是为一段从未写出的内容搭的骨架，底层是 21 张截图。全部拉平。

```bash
cd "D:/张梦奇ob"
git mv "学习/AI使用/matt AI编程.md" "原始资料/截图笔记/matt AI编程.md"
git mv "学习/中级思考力/框架.md" "原始资料/截图笔记/框架.md"
git mv "学习/中级思考力/框架思维.md" "原始资料/截图笔记/框架思维.md"
git mv "学习/股票理论/牵牛/道氏理论/道氏理论.md" "原始资料/截图笔记/道氏理论.md"
git mv "架构/数据基础概念.md" "原始资料/截图笔记/数据基础概念.md"
```

- [ ] **Step 2: 迁移古籍**

```bash
cd "D:/张梦奇ob"
git mv "Clippings/Bilibili/官宦世家的不传之秘.md" "原始资料/古籍/官智经.md"
```

注意：此文件与 Task 3d 迁移的 `官宦世家与圣人之说.md` **不是同一份内容**。前者是《官智经》古文原文（125 KB），后者是渤海小吏的视频讲稿。

- [ ] **Step 3: 归档 hover-notes Bases**

```bash
cd "D:/张梦奇ob"
git mv "架构/-hovernotes.base" "原始资料/归档/-hovernotes.base"
git mv "架构/-video-screenshots.base" "原始资料/归档/-video-screenshots.base"
git mv "架构/-video-screenshots-without-notes.base" "原始资料/归档/-video-screenshots-without-notes.base"
```

> **已知副作用，需在汇报中告知用户**：这三个 Base 的全局过滤器含 `file.folder == this.file.folder`（文件夹作用域）。归档后其所在文件夹只剩它们自己，且 Task 6 会把 `Untitled.md` 移走，因此 `-hovernotes.base` 将不再列出任何笔记。归档即停用，符合 spec 决定；若日后想恢复，需把它移回笔记所在目录。

- [ ] **Step 4: 清理空目录**

```bash
cd "D:/张梦奇ob"
rmdir "学习/股票理论/牵牛/道氏理论" "学习/股票理论/牵牛" "学习/股票理论" "学习/中级思考力" "学习/AI使用" "学习" 2>&1
rmdir "Clippings/Bilibili" "Clippings" 2>&1
rmdir "架构/hover-notes-images" "架构" 2>&1
rmdir "日复盘" 2>&1
```

`rmdir` 只删空目录，若目录非空会报错——报错说明还有文件没迁走，**应停下来检查**而不是加 `-r`。

- [ ] **Step 5: 验证**

```bash
cd "D:/张梦奇ob"
ls -d 学习 架构 日复盘 Clippings 2>&1     # 期望：全部 No such file
ls "原始资料/截图笔记" | wc -l            # 期望：5
ls "原始资料/古籍" | wc -l                # 期望：1
ls "原始资料/归档" | wc -l                # 期望：3
```

- [ ] **Step 6: Commit**

```bash
cd "D:/张梦奇ob"
git commit -m "refactor: 截图笔记拉平归档，官智经入古籍，停用 hover-notes Bases"
```

---

## Task 5: 日志归位 + 修复路径耦合配置

**本 task 修复两处会让功能静默失效的路径耦合，是迁移风险最高的部分。**

**Files:**
- Move: `DailyTasks/2026/` → `原始资料/每日笔记/2026/`，根目录 2 篇日期笔记 → `原始资料/每日笔记/`
- Modify: `.obsidian/plugins/daily-task-auto-generator/data.json`（`rootDir`）
- Modify: `.obsidian/app.json`（新增 `attachmentFolderPath`）

**Interfaces:**
- Consumes: Task 2 建好的 `原始资料/每日笔记/`
- Produces: 插件与附件行为恢复正常，Task 2 Step 5 的抽查可重做

- [ ] **Step 1: 迁移日志**

**不重命名** `7月.md`／`8月.md`／`9月.md`——见 Global Constraints 第 5 条。

```bash
cd "D:/张梦奇ob"
git mv "DailyTasks/2026" "原始资料/每日笔记/2026"
git mv "2026-07-28.md" "原始资料/每日笔记/2026-07-28.md"
git mv "2026-07-29.md" "原始资料/每日笔记/2026-07-29.md"
rmdir "DailyTasks"
```

- [ ] **Step 2: 改插件 rootDir**

`.obsidian/plugins/daily-task-auto-generator/data.json` 中：

```json
"rootDir": "DailyTasks",
```
改为：
```json
"rootDir": "原始资料/每日笔记",
```

只改这一个字段，其余（含 `customTemplate`）原样保留。

**验证：**
```bash
cd "D:/张梦奇ob"
python -c "import json;d=json.load(open('.obsidian/plugins/daily-task-auto-generator/data.json',encoding='utf-8'));print(d['rootDir'])"
```
**期望：** `原始资料/每日笔记`

- [ ] **Step 3: 补 attachmentFolderPath**

`.obsidian/app.json` 当前只有 `{"showLineNumber": false}`。改为：

```json
{
  "showLineNumber": false,
  "attachmentFolderPath": "原始资料/附件",
  "userIgnoreFilters": ["docs/"]
}
```

`userIgnoreFilters` 把 `docs/` 从 Obsidian 的搜索、图谱与 Bases 中排除——本设计文档与实施计划放在那里，属工具产物而非知识内容（spec 第 9 节列为需处理的副作用）。

**验证：**
```bash
cd "D:/张梦奇ob"
python -c "
import json;d=json.load(open('.obsidian/app.json',encoding='utf-8'))
print(d['attachmentFolderPath']); print(d['userIgnoreFilters'])
"
```
**期望：** 打印 `原始资料/附件` 与 `['docs/']`。

- [ ] **Step 4: 人工验证插件仍工作**

请用户在 Obsidian 中重启（或重载）后触发一次每日笔记生成，确认：
1. 新内容写进了 `原始资料/每日笔记/2026/9月.md`，而**没有**新建 `DailyTasks/`
2. 粘贴一张图片，确认落在 `原始资料/附件/`

**这一步不能由脚本代替**——插件行为只能在 Obsidian 运行时验证。若用户暂不方便，标记为待验证并继续。

- [ ] **Step 5: Commit**

```bash
cd "D:/张梦奇ob"
git add .obsidian/plugins/daily-task-auto-generator/data.json .obsidian/app.json
git commit -m "fix: 修正每日笔记 rootDir 与附件目录，避免迁移后静默失效"
```

---

## Task 6: hover-note 升格为摘要页

**与 spec 的一处偏离，需说明：** spec 第 3.1 节原定 `架构/Untitled.md` 升格为 `知识库/概念/`。读取实际内容后改判为 **`type: source`**——该文件带 `source:`（B站视频 URL）与 `author: AI中英文字幕课程`，是**单一来源**的加工品。概念页的定义是**跨来源**综合，单视频笔记属摘要页。

**Files:**
- Move: `架构/Untitled.md` → `知识库/摘要/MIT 6.824 分布式系统.md`
- Modify: 同文件的 frontmatter

**Interfaces:**
- Produces: `知识库/摘要/` 的第一篇，其 frontmatter 结构是 Task 8 三篇摘要页的模板

- [ ] **Step 1: 迁移并改名**

```bash
cd "D:/张梦奇ob"
git mv "架构/Untitled.md" "知识库/摘要/MIT 6.824 分布式系统.md"
```

- [ ] **Step 2: 改写 frontmatter**

把文件开头原有的：

```yaml
---
title: AI时代为什么必须懂分布式？MIT 6.824带你掌握大模型背后的系统架构与工程能力_哔哩哔哩_bilibili
description: AI时代为什么必须懂分布式？MIT 6.824带你掌握大模型背后的系统架构与工程能力共计20条视频，包括：1_Introduction、2_RPC and Threads、3_GFS等，UP主更多精彩视频，请关注UP账号。
author: AI中英文字幕课程
source: https://www.bilibili.com/video/BV1uD3v6LER1/?spm_id_from=333.1007.tianma.1-2-2.click&vd_source=f6e401a2285124f0171b20a22d0b87be
created: "2026-08-01"
tags:
  - hover-notes
  - bilibili
---
```

替换为：

```yaml
---
type: source
title: MIT 6.824 分布式系统
source_title: AI时代为什么必须懂分布式？MIT 6.824带你掌握大模型背后的系统架构与工程能力
source: https://www.bilibili.com/video/BV1uD3v6LER1/
source_type: B站视频
author: [AI中英文字幕课程]
published: 2026-08-01
captured: 2026-08-01
tags: [clippings, bilibili, 分布式]
raw: "https://www.bilibili.com/video/BV1uD3v6LER1/"
---
```

说明：
- 去掉 URL 里的 `?spm_id_from=...&vd_source=...` 跟踪参数
- `raw` 指向原始视频而非本地文件——此笔记没有对应的本地原始资料，视频本身就是源头
- 正文（`### Distributed System Concepts` 起）**一字不改**

- [ ] **Step 3: 验证 frontmatter 合法**

```bash
cd "D:/张梦奇ob"
python -c "
import re,sys
t=open('知识库/摘要/MIT 6.824 分布式系统.md',encoding='utf-8').read()
m=re.match(r'^---\n(.*?)\n---\n', t, re.S)
print('frontmatter found:', bool(m))
print(m.group(1) if m else 'MISSING')
"
```
**期望：** 打印出 `type: source` 及全部字段。

- [ ] **Step 4: Commit**

```bash
cd "D:/张梦奇ob"
git commit -m "feat: Untitled 升格为知识库摘要页并规范化 frontmatter"
```

---

## Task 7: 编写 `CLAUDE.md`

**Files:**
- Create: `CLAUDE.md`

**Interfaces:**
- Produces: 规则层。Task 8/9 的编译工作按此规范执行；后续会话中 LLM 靠它判断权限

- [ ] **Step 1: 写入 `CLAUDE.md`**

```markdown
# 知识库维护规约

本库依据 Andrej Karpathy 的 LLM Wiki 范式组织。三层结构，职责边界严格。

## 三层架构

1. **`原始资料/`** —— 原始资料层。只读、不可变，是事实源。
2. **`知识库/`** —— 知识层。由 LLM 撰写和维护，用户只读。
3. **本文件** —— 规则层。定义下面的全部约定。

核心原则：知识**编译一次**成持久产物，而不是每次提问从原始资料重新检索拼凑。
Obsidian 是 IDE，LLM 是程序员，`知识库/` 是代码库。

## 目录权限

| 目录 | 权限 |
|---|---|
| `原始资料/` | **只读**。只允许移动位置，永不修改其中文件的正文 |
| `知识库/` | **读写**。全权维护 |
| `CLAUDE.md` | 读写，但改动需用户确认 |
| `docs/` | 读写。设计与计划文档 |
| `.obsidian/` | **只读** |

## 页面类型与 frontmatter

字段名用英文，目录名与正文用中文。

### `type: source` —— 摘要页（`知识库/摘要/`）

每个原始资料一页。保留剪藏原有的 `title`／`source`／`author` 等字段，仅增不改。

```yaml
---
type: source
title: 清结算模块设计
source_title: 一文看懂清结算模块设计：从业务需求和设计落地
source: https://mp.weixin.qq.com/...
source_type: 微信文章          # 微信文章 | B站视频 | 云效知识库 | 其他
author: []
published: 2026-08-20
captured: 2026-08-20
tags: [clippings, 支付]
raw: "[[原始资料/剪藏/清结算模块设计]]"
---
```

`raw` 字段指向本地原始资料。若该摘要页没有对应的本地文件（只有线上视频或网页），
`raw` 直接写原始 URL 字符串，如 `raw: "https://www.bilibili.com/video/BV1uD3v6LER1/"`。

正文结构：一句话主旨 → 关键论点（3–7 条）→ 与其他资料的关系 → 值得追问的问题。

### `type: concept` / `type: entity` —— 知识页

```yaml
---
type: concept                  # concept | entity
title: 清结算
aliases: [清算, 结算, settlement]
tags: [支付, 金融]
sources: ["[[摘要/清结算模块设计]]", "[[摘要/支付系统技术决策]]"]
confidence: medium             # high | medium | low
status: draft                  # draft | stable | needs-review
created: 2026-09-17
updated: 2026-09-17
---
```

`confidence` 判定标准：

- `high` —— 单一来源直接陈述，或多来源完全一致
- `medium` —— 多来源综合，细节有出入但不矛盾
- `low` —— 仅凭薄弱来源推断，或来源间存在冲突

**来源间冲突不得静默抹平。** 在正文中以独立段落并列各方说法及来源。

### `type: synthesis` —— 综合页（`知识库/综合/`）

```yaml
---
type: synthesis
title: 支付系统架构决策对比
question: 做支付系统时哪些技术决策最关键？
tags: [支付]
sources: ["[[摘要/清结算模块设计]]"]
confidence: medium
updated: 2026-09-17
---
```

### 索引与日志

`知识库/索引.md`、`知识库/操作日志.md`、`知识库/综述.md` 用轻 frontmatter 或不加。

## 命名规范

1. 精简中文短标题。去掉"一文看懂"、"建议收藏"、"吃透"等营销词与副标题
2. 概念页文件名即概念名（`清结算.md`），不加"摘要"、"笔记"后缀
3. **跨层允许重名，但 wiki 内部链接一律写全路径：**
   - 引用摘要页：`[[摘要/清结算模块设计]]`
   - 引用原始资料：`[[原始资料/剪藏/清结算模块设计]]`
   - 概念页之间可写短链：`[[清结算]]`
4. **不重命名插件生成的文件**。`原始资料/每日笔记/2026/{7,8,9}月.md` 由
   `daily-task-auto-generator` 按 `{年}/{月}月.md` 生成，改名会导致重复生成

## 工作流

### Ingest（摄取）

新资料放入 `原始资料/剪藏/` → 阅读 → 写 `知识库/摘要/` 下的摘要页 → 更新 `知识库/索引.md`
→ 更新相关概念页与实体页 → 追加一行到 `知识库/操作日志.md`。
一篇资料通常触及 5–15 个知识库页面。

### Query（查询）

先读 `知识库/索引.md` → 再读摘要页、概念页及相关页面综合作答。
有价值的回答**回写**成 `知识库/综合/` 下的新页面，使结果复利。

### Lint（体检）

定期检查：页面间矛盾 / 被新资料推翻的旧结论 / 孤儿页 / 多处提及但无独立页的概念
/ 缺失的交叉引用 / `status: needs-review` 与 `confidence: low` 的积压。

## 硬性禁止

- 不修改 `原始资料/` 下任何文件的正文
- 不删除用户笔记（清理垃圾须先列入计划并获确认）
- 不在页内静默抹平来源冲突
- 不用 `confidence: high` 标注仅凭单一薄弱来源的推断

## 语言与风格

中文正文。技术术语保留英文原词（如 Raft、PEFT、B+ 树）。不用营销腔。
```

- [ ] **Step 2: 验证七章齐全**

```bash
cd "D:/张梦奇ob"
grep -c "^## " CLAUDE.md
```
**期望：** ≥ 6（目录权限、页面类型、命名规范、工作流、硬性禁止、语言与风格、三层架构）

- [ ] **Step 3: Commit**

```bash
cd "D:/张梦奇ob"
git add CLAUDE.md
git commit -m "feat: 新增 CLAUDE.md 规则层，定义三层架构与工作流"
```

---

## Task 8: 样板编译——3 篇支付剪藏摘要页

样板选题理由见 spec 第 7 节：三篇主题收敛、能展示完整扇出效果、属用户工作技术栈。

**Files:**
- Create: `知识库/摘要/清结算模块设计.md`
- Create: `知识库/摘要/支付系统技术决策.md`
- Create: `知识库/摘要/跨库 Join 方案.md`

**Interfaces:**
- Consumes: `知识库/摘要/MIT 6.824 分布式系统.md` 的 frontmatter 结构（Task 6）
- Produces: 三篇摘要页，Task 9 的概念页靠它们的全路径链接建立 `sources:` 引用

- [ ] **Step 1: 阅读三篇原文**

```
原始资料/剪藏/清结算模块设计.md       (25 KB)
原始资料/剪藏/支付系统技术决策.md      (3.4 KB)
原始资料/剪藏/跨库 Join 方案.md        (10 KB)
```

- [ ] **Step 2: 写 `知识库/摘要/清结算模块设计.md`**

frontmatter 按 `CLAUDE.md` 的 `type: source` 模板填。`source_title` 从原文 frontmatter 的 `title` 字段取，`raw:` 写 `"[[原始资料/剪藏/清结算模块设计]]"`。

正文结构：

```
一句话主旨（1 句）

## 关键论点
- （3–7 条，每条一个具体结论，不要泛泛而谈）

## 与其他资料的关系
- （指向 [[摘要/支付系统技术决策]] 或 [[摘要/跨库 Join 方案]] 的呼应或冲突）

## 值得追问的问题
- （1–3 条原文没回答、但影响实践的）
```

- [ ] **Step 3: 写 `知识库/摘要/支付系统技术决策.md`**

同上结构，`raw:` 写 `"[[原始资料/剪藏/支付系统技术决策]]"`。

- [ ] **Step 4: 写 `知识库/摘要/跨库 Join 方案.md`**

同上结构，`raw:` 写 `"[[原始资料/剪藏/跨库 Join 方案]]"`。

- [ ] **Step 5: 验证三篇 frontmatter 合法且 raw 指向存在的文件**

```bash
cd "D:/张梦奇ob"
python -c "
import re, os
for n in ['清结算模块设计','支付系统技术决策','跨库 Join 方案']:
    p = f'知识库/摘要/{n}.md'
    t = open(p, encoding='utf-8').read()
    m = re.match(r'^---\n(.*?)\n---\n', t, re.S)
    ok_type = 'type: source' in (m.group(1) if m else '')
    raw = re.search(r'raw:\s*\"\[\[(.+?)\]\]\"', m.group(1) if m else '')
    target = raw.group(1) + '.md' if raw else None
    print(f'{n}: type={ok_type} raw={target} exists={os.path.exists(target) if target else False}')
"
```
**期望：** 三行全部 `type=True ... exists=True`。

- [ ] **Step 6: Commit**

```bash
cd "D:/张梦奇ob"
git add "知识库/摘要"
git commit -m "feat: 样板编译——3 篇支付剪藏摘要页"
```

---

## Task 9: 样板编译——概念页、索引与综述

**Files:**
- Create: `知识库/概念/清结算.md`、`知识库/概念/支付网关.md`、`知识库/概念/跨库 Join.md`、`知识库/概念/分库分表.md`
- Create: `知识库/索引.md`、`知识库/综述.md`、`知识库/操作日志.md`

**Interfaces:**
- Consumes: Task 8 的 3 篇摘要页（全路径链接）
- Produces: 可导航的 wiki 层，Task 10 的 lint 对象

- [ ] **Step 1: 写 4 篇概念页**

每篇按 `CLAUDE.md` 的 `type: concept` 模板。`sources:` 用**全路径**形式，如 `"[[摘要/清结算模块设计]]"`。

四个主题的界定：

| 概念页 | 涵盖 |
|---|---|
| `清结算.md` | 清算与结算的业务流程、账务模型、对账 |
| `支付网关.md` | 渠道隔离、渠道路由、回调一致性 |
| `跨库 Join.md` | 分库分表下的关联查询方案与各自代价 |
| `分库分表.md` | 拆分策略、扩容、全局 ID |

概念页之间用短链互引：如 `清结算.md` 正文中写 `[[支付网关]]`。

**`confidence` 赋值要求：** 只有当某结论在至少两篇摘要页中一致出现时才可标 `high`；仅单篇来源陈述标 `medium`；来源间有出入标 `low` 并在正文并列双方说法。

- [ ] **Step 2: 写 `知识库/索引.md`**

```markdown
# 索引

读库第一站。检索时先读本页，再决定深入哪些页面。

## 摘要
- [[摘要/清结算模块设计]] —— 清结算的业务流程与账务模型
- [[摘要/支付系统技术决策]] —— 支付系统的关键技术取舍
- [[摘要/跨库 Join 方案]] —— 分库分表下跨库关联的五种方案及代价
- [[摘要/MIT 6.824 分布式系统]] —— 分布式系统的动机、挑战与容错

## 概念
- [[清结算]]
- [[支付网关]]
- [[跨库 Join]]
- [[分库分表]]

## 实体
（暂无）

## 综合
（暂无）

## 综述
- [[综述]]
```

上面每行的一行摘要必须**真实反映**对应页面的内容，不要照抄本模板的占位描述。

- [ ] **Step 3: 写 `知识库/综述.md`**

跨来源的整体图景：这批支付/架构资料共同勾勒出什么？哪些问题反复出现？哪些结论互相支撑、哪些存在张力？

- [ ] **Step 4: 建 `知识库/操作日志.md`**

append-only。首条记录本次样板编译：

```markdown
# 操作日志

| 日期 | 动作 | 涉及页面 |
|---|---|---|
| 2026-09-17 | ingest | [[摘要/清结算模块设计]]、[[摘要/支付系统技术决策]]、[[摘要/跨库 Join 方案]] |
| 2026-09-17 | 建立 | [[清结算]]、[[支付网关]]、[[跨库 Join]]、[[分库分表]] |
| 2026-09-17 | 建立 | [[索引]]、[[综述]] |
```

- [ ] **Step 5: 验证——全部 wikilink 目标存在**

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
        cands = [link + '.md', os.path.join(link, '') ] if '/' in link else [f'知识库/**/{link}.md', f'知识库/{link}.md']
        if '/' in link:
            found = os.path.exists(link + '.md')
        else:
            found = bool(glob.glob(f'知识库/**/{link}.md', recursive=True))
        if not found:
            missing.append(f'{p} -> [[{link}]]')
print('MISSING:' if missing else 'ALL LINKS OK')
for m in missing: print(' ', m)
"
```
**期望：** `ALL LINKS OK`。有 MISSING 就逐个修，这是样板质量的门槛。

- [ ] **Step 6: Commit**

```bash
cd "D:/张梦奇ob"
git add "知识库"
git commit -m "feat: 样板编译——概念页、索引、综述与操作日志"
```

---

## Task 10: 首次 lint 与阶段验收

**Files:**
- Modify: `知识库/索引.md`（如 lint 发现遗漏）
- Modify: `知识库/操作日志.md`（追加 lint 记录）

**Interfaces:**
- Consumes: Task 8/9 的全部产物
- Produces: 验收结论。通过后用户决定是否进入阶段 4 全量编译

- [ ] **Step 1: 检查孤儿页（无入链）**

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
**期望：** 除 `索引`／`操作日志`／`综述` 外无输出。

- [ ] **Step 2: 检查缺失的概念页**

通读概念页与摘要页，找出被多次提及但尚无独立页的概念，记入待办。

- [ ] **Step 3: 核对 spec 第 10 节验收标准**

```bash
cd "D:/张梦奇ob"
echo "--- 根目录残留（期望只剩 .git .obsidian docs CLAUDE.md package-lock.json）---"
ls -a | grep -v "^\.$\|^\.\.$"
echo "--- 原始资料/ 六类目录 ---"
ls "原始资料"
echo "--- 图片数（期望 108）---"
find "原始资料/附件" -type f | wc -l
```

**逐项对照 spec 第 10 节：**

- [ ] 根目录无残留（除 `CLAUDE.md`、`.obsidian`、`.git`、`docs/`）
- [ ] `原始资料/` 下六类目录各就各位
- [ ] 每日笔记插件仍能正常生成（Task 5 Step 4 由用户确认）
- [ ] 随机抽查 5 处图片嵌入正常渲染
- [ ] `CLAUDE.md` 含全部章节
- [ ] `知识库/` 有 3 个摘要页 + 不少于 4 个概念页
- [ ] `索引.md` 可导航，链接全部有效
- [ ] 概念页之间双向可跳转
- [ ] Obsidian 图谱视图中该簇不再是孤儿

- [ ] **Step 4: 追加 lint 记录并 Commit**

```bash
cd "D:/张梦奇ob"
git add -A "知识库"
git commit -m "chore: 首次 lint 与阶段验收"
```

- [ ] **Step 5: 向用户汇报并请示阶段 4**

汇报内容：本次产出清单、Task 5 Step 4 的插件验证结果、Task 0 是否执行、Task 4 Step 3 的 Base 停用副作用。

用户确认样板效果后，再决定阶段 4（其余 18 篇剪藏的全量编译）的优先级与批次。
