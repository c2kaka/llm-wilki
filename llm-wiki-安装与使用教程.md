# LLM Wiki 安装与使用教程

> **LLM Wiki** 是一个 Claude Code 插件，在 Obsidian 知识库中构建持久化、可复合增长的知识库。它基于 Andrej Karpathy 提出的 LLM Wiki 模式：LLM 读取原始资料、撰写交叉链接的 wiki 文章、将问答答案归档回知识库、并定期检查知识空白与矛盾。

---

## 目录

- [一、工作原理](#一工作原理)
- [二、前置条件](#二前置条件)
- [三、安装插件](#三安装插件)
- [四、验证安装](#四验证安装)
- [五、核心概念](#五核心概念)
- [六、操作指南](#六操作指南)
  - [6.1 init — 创建新 Wiki](#61-init--创建新-wiki)
  - [6.2 ingest — 摄入原始资料](#62-ingest--摄入原始资料)
  - [6.3 compile — 编译为 Wiki 页面](#63-compile--编译为-wiki-页面)
  - [6.4 query — 查询知识库](#64-query--查询知识库)
  - [6.5 lint — 完整性检查](#65-lint--完整性检查)
  - [6.6 remove — 删除 Wiki](#66-remove--删除-wiki)
- [七、Obsidian 集成技巧](#七obsidian-集成技巧)
- [八、进阶用法](#八进阶用法)
- [九、常见问题排查](#九常见问题排查)
- [十、完整工作流示例](#十完整工作流示例)

---

## 一、工作原理

LLM Wiki 的核心循环：

```
原始资料 → 摄入(ingest) → 编译(compile) → 查询(query) → 归档 → 循环增长
                                                    ↓
                                              完整性检查(lint)
```

1. **摄入（Ingest）**：从 URL 或文件获取原始资料，保存到 `raw/articles/`（不可修改）
2. **编译（Compile）**：读取原始资料，提取实体和概念，创建/更新 wiki 页面，建立交叉引用
3. **查询（Query）**：基于 wiki 内容回答问题，带有 `[[wikilinks]]` 引用，答案自动归档
4. **检查（Lint）**：检查死链、孤立页面、矛盾信息、索引漂移

每次操作后自动 git commit，保留完整变更历史。

---

## 二、前置条件

| 条件 | 说明 | 验证命令 |
|------|------|----------|
| **Node.js 18+** | 用于自动安装 qmd 和 Marp 依赖 | `node --version` |
| **Git** | 自动提交 wiki 变更，需配置好用户信息 | `git --version` |
| **Git 用户配置** | 需设置 user.name 和 user.email | `git config user.name` |
| **Obsidian vault** | 需有 `03-Resources/` 目录 | `ls ~/ObsidianVault/03-Resources/` |

### 验证步骤

```bash
# 检查 Node.js 版本（需要 18+）
node --version

# 检查 Git 版本
git --version

# 检查 Git 用户配置
git config user.name     # 应返回你的名字
git config user.email    # 应返回你的邮箱

# 检查 Obsidian vault 目录结构
ls ~/ObsidianVault/03-Resources/
```

> **注意**：如果你的 vault 不在默认路径 `~/ObsidianVault/`，插件也能正常工作——它通过从当前工作目录向上查找 `CLAUDE.md` + `wiki/` 子目录来定位活动 wiki。

---

## 三、安装插件

### 方式一：从 Claude Code marketplace 安装（推荐）

在 Claude Code 会话中执行：

```
/plugin marketplace add ekadetov/llm-wiki
/plugin install llm-wiki@llm-wiki
```

### 方式二：从本地路径安装

先将仓库克隆到本地：

```bash
git clone https://github.com/ekadetov/llm-wiki.git
```

然后在 Claude Code 中执行：

```
claude plugin install /path/to/llm-wiki
```

### 依赖自动安装

安装插件后，首次启动新的 Claude Code 会话时，SessionStart hook 会自动运行 `npm install` 安装以下依赖：

- **qmd**：混合搜索引擎（BM25 + 向量搜索）
- **marp-cli**：将 Markdown 导出为幻灯片

验证依赖是否安装成功：

```bash
ls ~/.claude/plugins/data/llm-wiki/node_modules/.bin/qmd
ls ~/.claude/plugins/data/llm-wiki/node_modules/.bin/marp
```

> 如果以上文件不存在，重新启动一个 Claude Code 会话以触发自动安装。

---

## 四、验证安装

在 Claude Code 会话中输入：

```
/llm-wiki:wiki
```

如果安装成功，你将看到参数提示：

```
init <name> | ingest <path|url> | compile [<path>] | query <question> | lint | remove <name>
```

---

## 五、核心概念

### 5.1 目录结构

每个 wiki 位于 `~/ObsidianVault/03-Resources/<名称>/` 下：

```
<名称>/
├── raw/                  ← 不可变的原始资料（LLM 永远不会修改此目录）
│   ├── articles/         ← 文本源文件（文章、论文、转录）
│   └── attachments/      ← 图片和二进制附件
├── wiki/                 ← LLM 管理的页面（LLM 拥有完整写入权限）
│   ├── index.md          ← 目录索引（操作前首先阅读此文件）
│   ├── queries/          ← 已归档的查询答案
│   └── <概念>.md         ← 实体/概念页面
├── outputs/
│   └── reports/          ← 带日期的检查报告
├── CLAUDE.md             ← wiki 的 schema 定义和约定
├── log.md                ← 只追加的操作日志
├── .gitignore
└── qmd.yml               ← qmd 搜索集合配置
```

**关键原则**：
- `raw/` 是神圣的——资料进去后不可修改
- `wiki/` 由 LLM 管理——你有完整写入权限
- `log.md` 只能追加——永远不编辑已有条目

### 5.2 活动 Wiki 检测

运行任何操作时，插件从当前工作目录向上查找同时包含 `CLAUDE.md` 和 `wiki/` 子目录的文件夹。第一个匹配即为活动 wiki。

这意味着你可以：
- `cd` 到 wiki 根目录后运行命令（推荐）
- 在任意位置运行，让插件自动检测或提示选择

### 5.3 CLAUDE.md — Wiki 的 Schema

`CLAUDE.md` 是你与 LLM 之间的契约，定义了：

- 目录布局和各文件夹用途
- 实体模板（concept、person、source-summary）及其 frontmatter 格式
- 命名约定（文件名用 lowercase-kebab-case，内部链接用 `[[wikilinks]]`）
- 日志格式、索引格式、交叉引用规则

**Schema 会随 wiki 共同演进。** 如果你想添加新的实体类型或修改约定，编辑 `CLAUDE.md`，LLM 会在下次操作时遵循新规则。

### 5.4 实体类型

三种页面类型，各有标准化的 frontmatter：

**概念页（concept.md）**：

```yaml
---
date: 2026-05-10
tags: [领域]
type: concept
status: active
---
# 概念名称
<一段话概述>

## 详情
...

## 另见
- [[相关概念]]

## 反论与空白
...
```

**人物页（person.md）**：

```yaml
---
date: 2026-05-10
tags: [领域, person]
type: person
status: active
---
# 人物姓名
角色 / 所属机构

## 主要贡献
...

## 另见
- [[相关概念]]
```

**源文件摘要页（source-summary.md）**：

```yaml
---
date: 2026-05-10
tags: [领域]
type: source-summary
source-url: https://...
---
# 源文件标题
<一段话摘要>

## 要点
...

## 提及的实体
- [[人物或概念]]
```

### 5.5 Wikilinks 与复合增长

每个页面通过 `[[wikilinks]]` 链接到相关页面。每次编译后，插件运行**反向链接审计**：在现有页面中搜索新建页面标题的出现位置，自动补充缺失的 wikilinks。

**复合增长效应**：当查询 wiki 时，LLM 会跟随 wikilinks 向下展开一层以收集上下文。一个良好链接的 wiki 能发现任何单一资料都不包含的关联。

### 5.6 qmd 搜索引擎（可选）

qmd 提供混合搜索（BM25 关键词 + 向量语义搜索）。对于小型 wiki（约 50 页以下），直接读取 `index.md` 足够快。对于更大的 wiki，qmd 是高效查询的关键。

> **重要**：始终通过 `env -u BUN_INSTALL` 调用 qmd，强制使用 Node.js 运行时。如果环境中设置了 `BUN_INSTALL`，qmd 会在 Bun 下运行，其 SQLite 构建不支持扩展加载。

---

## 六、操作指南

### 6.1 init — 创建新 Wiki

**命令**：

```
/llm-wiki:wiki init <名称>
```

**示例**：

```
/llm-wiki:wiki init machine-learning
```

**执行步骤**：

1. 检查 `~/ObsidianVault/03-Resources/<名称>/` 是否已存在，如存在则中止并提示
2. 创建目录结构：`raw/articles/`、`raw/attachments/`、`wiki/queries/`、`outputs/reports/`
3. 写入 `CLAUDE.md`（完整 schema 定义）
4. 写入 `wiki/index.md`（空目录模板）
5. 写入 `log.md`（空日志模板）
6. 写入 `.gitignore` 和 `qmd.yml`
7. Git commit：`init: <名称> wiki`
8. 如果 qmd 可用：创建搜索集合并生成嵌入
9. 打印 Web Clipper 配置说明

**验证**：

```bash
ls ~/ObsidianVault/03-Resources/<名称>/
# 应看到: raw/  wiki/  CLAUDE.md  log.md  .gitignore  qmd.yml

cat ~/ObsidianVault/03-Resources/<名称>/wiki/index.md
# 应看到空的目录模板
```

### 6.2 ingest — 摄入原始资料

**命令**：

```
/llm-wiki:wiki ingest <路径或URL>
```

**示例**：

```bash
# 从文件摄入
/llm-wiki:wiki ingest raw/articles/my-article.md

# 从 URL 摄入
/llm-wiki:wiki ingest https://example.com/interesting-article
```

**执行步骤**：

1. 检测活动 wiki，读取 `CLAUDE.md`
2. 获取源文件内容（URL 用 WebFetch，文件直接读取）
3. 分类源文件类型：`article` | `paper` | `transcript` | `conversation` | `image-set`
4. 保存到 `raw/articles/YYYY-MM-DD-<slug>.md`，附带标准 frontmatter：

```yaml
---
date: YYYY-MM-DD
source-type: article
source-url: <原始URL或文件路径>
title: <提取的标题>
compiled: false
---
```

5. 追加到 `log.md`
6. Git commit
7. 提示："源文件已保存，运行 `wiki compile` 将其整合到 wiki 中。"

> **注意**：ingest **不会**创建 wiki 页面，只是将原始资料存入仓库。需要用 `compile` 来生成 wiki 页面。

### 6.3 compile — 编译为 Wiki 页面

**命令**：

```bash
# 编译所有未编译的源文件
/llm-wiki:wiki compile

# 编译指定的源文件
/llm-wiki:wiki compile raw/articles/2026-05-10-my-article.md
```

**执行步骤**：

1. 检测活动 wiki，读取 `CLAUDE.md` 获取模板
2. **识别未编译源文件**：
   - 如果指定了路径：编译该文件
   - 否则：扫描 `raw/articles/`，找出在 `wiki/` 中没有对应源文件摘要页的资料
   - 如果全部已编译："所有源文件均已编译，无需操作。"
3. **对每个源文件执行编译**：
   - 读取原始资料内容
   - **写入源文件摘要页**：使用 `source-summary` 模板
   - **实体提取**：识别提及的实体（人物、概念、事件）
     - 已有页面 → 用新信息更新，保留已有内容
     - 无页面 → 使用 `concept.md` 或 `person.md` 模板创建
     - 在双向添加 `[[wikilinks]]`
   - **反向链接审计**：在现有页面中搜索新页面标题，补充缺失的 wikilinks
4. 更新 `wiki/index.md`
5. 追加到 `log.md`
6. Git commit
7. 如果 qmd 可用：重新生成嵌入

> 一个源文件通常会涉及 5-15 个页面的创建或更新，这是正常的。

### 6.4 query — 查询知识库

**命令**：

```
/llm-wiki:wiki query "<你的问题>"
```

**示例**：

```
/llm-wiki:wiki query "什么是交换子？它在魔方中有什么应用？"
```

**执行步骤**：

1. 检测活动 wiki，读取 `CLAUDE.md`
2. **查找相关页面**：
   - 如果 qmd 可用：运行混合搜索查询
   - 否则：读取 `index.md`，通过标题/描述匹配识别相关页面
3. **阅读所有相关页面**，跟随一层 wikilinks 获取更多上下文
4. **合成答案**，使用 `[[wikilinks]]` 作为引用。输出格式规则：
   - **默认**：散文式叙述，内联 wikilink 引用
   - **问题含 "表格"**：Markdown 表格，单元格带 wikilink
   - **问题含 "幻灯片"**：Marp Markdown，frontmatter 加 `marp: true`
5. **自动归档**答案到 `wiki/queries/<slug>.md`（强制归档，不提示）
6. **询问**是否将答案提升为 `wiki/<slug>.md` 的正式概念页
7. 追加到 `log.md`，Git commit

**示例输出**：

> 交换子（Commutator）写作 **A B A⁻¹ B⁻¹**，是[[group-theory]]中的核心概念。在[[rubiks-cube]]中，交换子可以实现"只改变特定块而不影响其他块"的效果。最基础的原子动作是 R U R' U'，详见 [[commutator]]。

### 6.5 lint — 完整性检查

**命令**：

```
/llm-wiki:wiki lint
```

**执行步骤**：

1. 读取 `wiki/` 下所有文件
2. 从所有 `[[wikilinks]]` 构建链接图
3. 运行确定性检查脚本（如果可用）
4. **报告并修复**：

| 检查项 | 操作 |
|--------|------|
| **孤立页面**（无入站链接） | 列出并建议添加链接的位置 |
| **死链**（指向不存在文件的 wikilinks） | 用适当模板创建占位页面 |
| **未链接的概念提及** | 扫描正文中出现但未加 wikilink 的专有名词，补充链接或标记为新页面候选 |
| **矛盾信息** | 扫描 `[!WARNING]` 标记并列出 |
| **缺失"反论与空白"章节** | 添加空的 `## 反论与空白` 章节 |
| **过期页面** | 标记 frontmatter 中 `status: stale` 的页面 |
| **索引漂移** | 对比 `index.md` 条目与实际文件，增删修正 |

5. **生成增长建议**：
   - 3-5 个 wiki 目前无法回答的问题（查询候选）
   - 2-3 个最能加强 wiki 的主题领域或资料来源
6. 写入检查报告到 `outputs/reports/YYYY-MM-DD-lint.md`
7. 追加到 `log.md`，Git commit

### 6.6 remove — 删除 Wiki

**命令**：

```
/llm-wiki:wiki remove <名称>
```

**执行步骤**：

1. 确认 wiki 路径存在
2. 列出目录内容，询问确认："这将永久删除 '<名称>' wiki 及其所有内容。确认？(y/n)"
3. 移除 qmd 集合（如可用）
4. 从 git 和文件系统中删除
5. 确认："Wiki '<名称>' 已删除。"

---

## 七、Obsidian 集成技巧

### 7.1 Web Clipper 浏览器插件

安装 [Obsidian Web Clipper](https://obsidian.md/clipper) 浏览器扩展后，配置：

- **目标文件夹**：`03-Resources/<wiki名称>/raw/articles`
- **文件名模板**：`{{date:YYYY-MM-DD}}-{{title}}`

剪藏网页后运行：

```
/llm-wiki:wiki ingest raw/articles/<剪藏文件名>.md
```

### 7.2 图谱视图作为可视化检查

在 Obsidian 中按 `Ctrl/Cmd+G` 打开图谱视图：

- **孤立节点** = 没有入站链接的页面（lint 会标记）
- **密集连接的节点群** = 最强的知识领域
- **集群之间的细桥** = 新概念页面的候选位置

### 7.3 Dataview 动态查询

在任意笔记中添加以下代码块来查询 wiki：

**列出所有 wiki 页面**：

````markdown
```dataview
TABLE date, type FROM "03-Resources/<名称>/wiki"
SORT date DESC
```
````

**筛选特定类型**：

````markdown
```dataview
TABLE date, status FROM "03-Resources/<名称>/wiki"
WHERE type = "concept" AND status = "stale"
SORT date ASC
```
````

**最近的源文件摘要**：

````markdown
```dataview
LIST FROM "03-Resources/<名称>/wiki"
WHERE type = "source-summary"
SORT date DESC
LIMIT 10
```
````

### 7.4 Marp 幻灯片导出

任何 wiki 页面都可以导出为幻灯片。在 frontmatter 中添加 `marp: true`，然后运行：

```bash
~/.claude/plugins/data/llm-wiki/node_modules/.bin/marp wiki/<页面>.md -o output.html
```

---

## 八、进阶用法

### 8.1 多个 Wiki

每个主题在 `03-Resources/` 下有独立目录。按需创建：

```
/llm-wiki:wiki init machine-learning
/llm-wiki:wiki init company-research
/llm-wiki:wiki init book-notes
```

活动 wiki 检测根据当前工作目录自动选择。Wiki 之间不共享页面。

### 8.2 大型 Wiki（200+ 页面）

此时直接读取 `index.md` 会变慢，qmd 的混合搜索必不可少。可以将 `index.md` 按领域分区，每节保持在 50 条以内。

验证 qmd 集合状态：

```bash
~/.claude/plugins/data/llm-wiki/node_modules/.bin/qmd collection list
```

### 8.3 矛盾处理

当 LLM 发现跨资料的冲突信息时，会在页面中插入标记：

```markdown
> [!WARNING] 与 [[另一页面]] 矛盾
> 来源 A 说 X，但 [[另一-page]] 说 Y。需要核实。
```

lint 会在报告中汇总所有 `[!WARNING]` 标记。阅读两个来源后更新页面即可解决。

### 8.4 查询答案的晋升机制

查询工作流有两个归档步骤：

1. **归档到 `wiki/queries/`**：保存为查询记录（自动执行）
2. **晋升到 `wiki/`**：提升为正式概念页面（需确认）

晋升适用于查询合成了真正新的、任何单一资料都不包含的知识。随着时间推移，查询成为 wiki 增长的主要方式之一。

---

## 九、常见问题排查

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| `/llm-wiki:wiki` 未找到 | 插件未安装 | 运行 `/plugin install llm-wiki@llm-wiki` |
| qmd 未找到 | 依赖未安装 | 检查 `~/.claude/plugins/data/llm-wiki/node_modules/.bin/qmd`；启动新会话触发自动安装 |
| Git commit 失败 | 未配置 git 用户信息 | `git config --global user.name "名字"` 和 `git config --global user.email "邮箱"` |
| init 失败"目录已存在" | 上次测试未清理 | 运行 `wiki remove <名称>` 删除后重试 |
| Hook 未在会话启动时运行 | hooks.json 格式错误 | 重新安装插件 |
| compile 未生成实体页面 | 源文件过短或结构不清 | 为源文件添加更多内容 |
| query 返回模糊答案 | wiki 太小或 qmd 未索引 | 摄入更多源文件；运行 `qmd embed --collection <名称>` |
| URL 摄入网络错误 | 网络问题 | 重试一次；如仍失败，手动保存内容到 `raw/articles/` |
| 源文件过大（>50KB） | 内容太多 | 警告：实体提取可能不完整，建议拆分 |
| log.md 丢失 | 意外删除 | 插件会自动从模板创建新的，并发出警告 |

---

## 十、完整工作流示例

以下是一个从零开始的完整工作流示例：

### 第一步：创建 wiki

```
/llm-wiki:wiki init my-research
```

### 第二步：摄入资料

```bash
# 从 URL 摄入
/llm-wiki:wiki ingest https://en.wikipedia.org/wiki/Transformer_(deep_learning_architecture)

# 从本地文件摄入
/llm-wiki:wiki ingest raw/articles/my-paper-notes.md
```

### 第三步：编译

```
/llm-wiki:wiki compile
```

这一步会为每个未编译的源文件创建源文件摘要页、提取实体并创建/更新概念页和人物页。

### 第四步：查询

```
/llm-wiki:wiki query "Transformer 架构与 RNN 相比有什么优势？"
```

答案自动归档到 `wiki/queries/`，可选择晋升为正式概念页。

### 第五步：定期检查

```
/llm-wiki:wiki lint
```

修复死链、孤立页面、矛盾信息，获取增长建议。

### 循环

持续摄入新资料 → 编译 → 查询 → 检查，wiki 会随时间复合增长，每次新资料都会丰富已有页面，每次查询都能发现跨越所有已摄入资料的关联。

---

## 附录：卸载

```bash
claude plugin uninstall llm-wiki
```

这会移除插件及其依赖缓存。`~/ObsidianVault/` 中的 wiki 数据不受影响，完整保留。

---

## 参考链接

- GitHub 仓库：https://github.com/ekadetov/llm-wiki
- Obsidian Web Clipper：https://obsidian.md/clipper
- 完整技能定义：插件源码中的 `skills/wiki/SKILL.md`
