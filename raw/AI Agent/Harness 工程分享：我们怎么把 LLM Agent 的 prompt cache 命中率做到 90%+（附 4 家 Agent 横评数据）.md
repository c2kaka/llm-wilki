---
title: "Harness 工程分享：我们怎么把 LLM Agent 的 prompt cache 命中率做到 90%+（附 4 家 Agent 横评数据）"
source: "https://www.v2ex.com/t/1212780#reply29"
author:
  - "[[yafeilee]]"
published: 2026-05-14
created: 2026-05-15
description: "程序员 - @yafeilee - 接上一篇 https://v2ex.com/t/1211434 很多朋友好奇我们的工程实践，我把这二年的坑和 Ruby 重写的思考放出来，大家一起看看离 ClaudeCode 这种顶级 Harness"
tags:
  - "clippings"
---
接上一篇 [https://v2ex.com/t/1211434](https://v2ex.com/t/1211434) 很多朋友好奇我们的工程实践，我把这二年的坑和 Ruby 重写的思考放出来，大家一起看看离 ClaudeCode 这种顶级 Harness 工程还有多远。

## 开篇

为了让新朋友重新了解一下我们的评测结果，我再列一下。

**成本极优** ：3 项任务实测，4 家 Agent 横评，OpenRouter CSV 逐请求核算：

| Agent | 总成本 | 请求数 | Cache 命中率 |
| --- | --- | --- | --- |
| **OpenClacky** | **$5.10** | **51** | **90.6%** |
| Claude Code | $5.49 | 70 | 95.2% |
| OpenClaw | $15.70 | 81 | 88.7% |
| Hermes | $30.14 | 218 | 60.3% |

![](https://i.v2ex.co/G5XEe7ne.png)

完整数据和产物对比： [openclacky.com/benchmark](https://www.openclacky.com/benchmark)

51 个请求 + 90.6% 命中率 → $5.10 。218 个请求 + 60.3% 命中率 → $30.14 。成本差距的直接原因就两个： **请求数** 和 **cache 命中率** 。

不要忘了， **OpenClacky 是一个全功能 Agent** ：WebUI + 命令行、长期记忆、Skill 技能库、定时任务、IM 接入（飞书/企微/微信）、浏览器自动化、子 Agent 、运行时切模型、Skill 自进化与动态加载。

而很多开源 Agent 也许有较好的 Token 消耗，或功能不全，或命中率不高。

在实践中最大的问题是： **这些功能里很多跟"高 cache 命中率"是结构性冲突的。**

举例：

- 切模型 → 模型 ID 写在哪？写进 system prompt 就 cache 失效一次。
- 中途装 skill → skill 列表写在哪？写进 system prompt 就 cache 失效一次。
- 知道"今天日期" → 写进 system prompt ？跨天就失效。
- 加"读 PDF"能力 → 最容易的实现是再加一个工具 → 工具 schema 变了 → cache 失效面变大，模型选错工具的概率也变大。
- 上下文不爆 → 最容易的做法是开一次独立 LLM call 做压缩 → 压缩本身 100% miss ，压完之后主对话 cache 也凉了。

单看任意一头都不难做：少做功能，命中率自然高；不管账单，功能可以堆得很猛。难的是两头同时做。这篇文章讲我们在每个冲突点上具体怎么取舍。

![](https://i.v2ex.co/3S4j29c6.png) 效果已经不是当前 Agent 的主要矛盾，成本才是。

## 起步：两年失败史

第三代之前还有两代，失败的很严重。但我感觉现在还有很多人在踩坑，估计很多人有争议，但我 100% 站我自己的观点。

### 第一代（ 2024-2025 上半）：RAG / 知识库

把用户 codebase 、文档、历史会话全 embedding 进向量库，hybrid 检索 + 重排 + query rewrite 。Agent 流程是"先查上下文，再答"。

实际跑下来的问题：

- 成本高，每次更新的 codebase ，需要同步更新向量，实时性无法保证。
- 准确率有限，例如听起来 90% 的召回率是不是还不错，但是对不起，不仅没有用，还可能有害，我预测，97%的召回率可能才刚刚够用。
- 多了一个会失败的部件（向量库），增加了很多延迟。

结论：千万不要搞任何 RAG 、知识库分片。如果你要上 Agent ，请直接上 Agent ，外加一个适合 AI 去阅读的网站就可以了。（参考我们自反思 Skill product-help 的实现）

### 第二代（ 2025 中期）：SWEBench / 多 Agent 工作流

Planner / Coder / Reviewer / Tester 各一个 agent ，消息总线 + 角色 prompt 编排。

实际跑下来的问题：

- 每个 sub-agent 各有 system prompt ，各有 cache 命名空间。Agent 间交接靠消息序列化状态， **每次交接 = 一次 cache miss** 。
- 一个单 agent 4 分钟能完成的任务，多 agent 编排到 14 分钟，成本 6×。
- SWEBench 分数能刷上去，但榜单跑分跟用户实际感受脱节得很厉害。

结论：

- **不要做工作流编排。** 多 Agent 在结构上就是 cache 灾难。人类的分工不对 AI 有任何价值。AI 是万能的。
- **不要被 benchmark 绑架。** 模型每 6 个月跨一个台阶，用今天的二流模型 + 工作流堆出来的分数，会被半年后顶级模型 + 朴素 harness 直接抹平。把工程预算花在 harness 上，不要花在编排上。对 Agent 工程来说，Benchmark 本身也并不重要，一个朴素的 Agent 思想打败一切：站在 AI 的角度思考你的上下文。

### 第三代（ 2025 年底至今）

Ruby 从零重写，4 个月。围绕"cache 局部性"和"工具集稳定性"来组织。后面讲的所有决策都属于这一代。

## 核心决策 1：双 cache 标记 + 允许失败回退

OpenClacky 同时跑在 Claude / OpenAI 兼容这两条主线上，两边的 prompt cache 行为不同，但工程上我们只关心一个共性： **cache 是按"前缀"匹配的——前缀里改一个字节，从那里往后全部失效** 。

所以前缀的"层次"和"标记位置"，决定了你下一轮还能 hit 到哪里。我们把请求前缀分成几段考虑：

- **session-stable 段** ：system prompt 、工具 schema 。session 内绝不变。
- **append-only 段** ：历史消息。只追加、不修改。
- **session-volatile 段** ：当前轮新消息（用户输入、工具结果、模型回复）。

前两段交给"系统提示词层"的天然断点，后续每轮都能 hit 。 **真正需要工程的是"append-only 段"——它每轮都在长尾部，标记打哪儿、打几个，决定了下一轮还认不认得它。**

### 朴素做法为什么不够

最直觉的做法是"每轮在 messages 末尾打一个 marker"。它在以下场景都会失效：

1. **history 单调追加** ：第 N 轮在 `messages[-1]` 打 marker ，第 N+1 轮 messages 又长了一条，原 marker 的位置内容已经不一样了——服务端找不到匹配，整段 history 上 cache miss 。
2. **模型回退一次工具调用** ：工具报错、用户 Ctrl-C 重试、或者模型自己决定换一种 tool call——这一刻"原本的最后一条"被丢弃，单 marker 直接作废。
3. **运行时切模型** ：用户在 session 中途从 Sonnet 切到 Opus ，请求路由到新 endpoint ，最理想情况下我们希望两个模型共享尽可能多的前缀。任何不必要的 marker 抖动都会让"切换"成为新的 cache miss 事件。

我们一开始就栽在 (1) 上。修复链能从 git log 里看出节奏：

```
8ff66cc fix: cache
6ea99fe fix: prompt cache
e9a3602 feat: prompt cache works fine
7734c97 feat: try 2 point cache
```

前三个 commit 是逐步逼近，最后一个是结构性正解。

### 双标记是怎么工作的

每轮我们标 **两条** 连续消息，不是一条：

```
第 N 轮：    [..., msg_A, msg_B(*), msg_C(*)]
                                ↑       ↑
                          marker 1   marker 2

第 N+1 轮：  [..., msg_A, msg_B(*), msg_C(*), msg_D(*)]
                                ↑       ↑          ↑
                          (仍在)   (仍在)     新 marker
```

第 N+1 轮发出请求时：

- 服务端尝试匹配 `msg_C` 的 marker → **命中到 `msg_C` 之前的所有内容** （ system prompt + 工具 + 整段历史除最后一条）。
- 我们在 `msg_D` 上加新 marker ，建立新的尾部断点供下一轮使用。

这是一个滚动双缓冲：任何时刻都持有两个断点——一个"刚建立的"（写）和一个"上一轮建立的"（读）。下一轮把"读"再读一次，把"写"扔掉，再在新尾部写一个。 **永远不会出现两个 buffer 同时失效的瞬间。**

### 为什么是 2 ，不是 3 或 4

主流大模型的 cache 都允许多个标记位（上限不一），但 **更多并不更好** ：

- 每多一个 marker ，那一轮就多一次 cache write ，按写入费率收。
- 双标记要解决的失败模式只有一个位置：\*\*"昔日尾部 / 今日尾部"这个边界 **。两个 marker 正好覆盖。第三个 marker 落在更靠前的位置，对应的 cache 段在下一轮** 仍然会被前两个 marker 之一覆盖\*\*——它写的是一段永远不会被独立读到的前缀。
- 标记多了之后，部分 endpoint 上服务端的候选前缀匹配代价也会涨。

简单说：2 是覆盖尾部边界的最小数量，3 多余，4 浪费。

### 允许失败：单步回退仍然命中

这是双标记的第二个好处，也是当时 `7734c97` 的真正动机。

模型偶尔需要回退一次 tool call：工具返回错误、用户 Ctrl-C 重试、或者上游 streaming 断了一半。这种情况下"昨天的最后一条"被丢弃了，但 **倒数第二个 marker 通常仍然落在仍存在的消息上** ——单步回退后还能命中。

单 marker 在回退时直接作废；双标记是能扛住单步回退的最小数量。我们没继续往上加（三标记也能扛两步回退，但成本不划算）——回退超过一步的概率已经低到可以接受全 miss 一次。

### 模型切换：为什么要 marker 不动

OpenClacky 支持在 session 中途换模型。工程上要保证两件事：

1. **新模型的请求前缀和老模型尽量一致。** 我们不在 system prompt 里写当前模型 ID （写在 `[session context]` 块里，见决策 2 ），换模型不动 system prompt 。
2. **marker 位置不变。** 切完模型的下一轮，前两个 marker 落在和切换前完全相同的 message 上。新 endpoint 第一次请求会因为"换了上游账号 / 区域"产生一次 cache write ，但 **前缀的几何结构是连续的** ，warm-up 只发生一轮。

这个细节不做的话，每次切模型一定要都要付完整 cache 重建的钱，用户会很不开心。

### 不能标的位置

marker 选择逻辑里有一条硬规则： **跳过 `system_injected: true` 的消息** 。

`[session context]` 块就是典型例子——它是一次性信息，下一轮尾部已经变了，落在它身上的 marker 是一笔永远读不回来的写入。压缩指令注入也是同样的处理（决策 5 会展开）。

marker 选择从尾部往前走， `system_injected` 的跳过，凑够两个真实对话消息为止。

### 本节总结

- system prompt + 工具 schema：靠 system prompt 段的天然断点 hit 。
- history 滚动：靠双标记。
- 单步回退：靠双标记容错。
- 模型切换：靠"动态信息不写进 system prompt"+ marker 位置不变。

把这四件事同时做到，普通一轮的 cache 命中率才有可能稳定在 95%+。 **前三件是 cache 几何，第四件是设计纪律。**

---

## 决策 2：永不变的 system prompt

**OpenClacky 的 system prompt 在 session 启动时一次性构建，之后字节冻结。** 任何"想往 system prompt 里塞动态信息"的需求，必须重定向到别的位置。

这条纪律是 cache 命中率的第一道地基——system prompt 一变，后面所有 cache 全废，没有任何"局部修补"能挽回。

但日常跑下来，至少有四类信息"天然想插入到 system prompt"：

1. **当前时间、当前工作目录、操作系统** ——模型需要这些来生成正确的命令和路径。
2. **当前模型 ID** ——模型知道自己是谁有助于自适应行为。
3. **用户装了新 skill** ——模型需要看到新的 skill 名称和描述才能调用。
4. **用户更新了 [USER.md](http://user.md/) / [SOUL.md](http://soul.md/)** ——agent 的人格和用户偏好发生了变化。

这四类信息都是"session 中途可能变"的。如果写进 system prompt ，任何一次变更都意味着全量 cache 失效。

### \[session context\] 块

我们的做法是把这些信息写进 message 流，而非 system prompt 。每当环境发生模型需要感知的变化时（跨天、切模型、切工作目录），agent 在 history 里追加一条 `user` 角色的消息：

```
[Session context: Today is 2026-05-13, Tuesday. Current model: claude-sonnet-4-6.
OS: macOS. Working directory: /Users/.../project]
```

这条消息被标记为 `system_injected: true` 。它不会被 cache marker 选中（决策 1 已经讲过），不会被算作真实用户轮数，压缩时也不会被原样搬进新历史。

注入是按日期 gate 的：同一天内只注入一条。跨天了，插一条新的。切了模型，插一条新的。大多数 session 里你只会看到一条 session context 块。

### 这个设计踩过的坑

第一版 `inject_session_context` 是在 agent 构造期就急切注入的。结果 `@history.empty?` 返回 false ， `run()` 误以为是后续轮，跳过了 system prompt 的构建—— **第一次请求带着一条"today is Tuesday"但没有 system prompt 就发出去了** 。agent 的行为诡异了大约一天才定位到。

修复只有一行：等 system prompt 构建完毕之后再注入。代码里有一段注释记录了这个约束：

```ruby
# IMPORTANT: Skip injection when the system prompt hasn't been built yet.
# Otherwise, appending a user message to an empty history makes
# @history.empty? false, which causes run() to skip building the
# system prompt entirely.
```

教训是： **前缀的组装顺序比前缀的内容更要紧。** 你可以花大力气设计每一段的内容，但只要组装顺序错一步，整个 cache 策略就是废的。

### Skill 列表怎么处理

Skill 列表是最容易跟"永不变的 system prompt"冲突的需求。用户可以随时装新 skill ，模型需要看到 skill 名和描述才能通过 `invoke_skill` 去调用它。

我们的取舍： **skill 列表在 session 启动时渲染进 system prompt ，之后冻结。** session 中途装的新 skill ，模型在当前 session 里看不到——它会看到一条 `[session context]` 通告说"skill 列表已更新，新 skill 从下一个 session 可用"。

这意味着用户装完 skill 想立刻用会发现用不了，要开新 session 。我们接受这个摩擦，因为替代方案是重渲染 system prompt 导致全量 cache 失效——这个代价打到所有用户的所有 session 的每一轮上。装 skill 是低频操作，cache 命中是每轮都在享受的收益，取舍方向很清楚。

[USER.md](http://user.md/) / [SOUL.md](http://soul.md/) 的更新也是同样的处理：session 启动时读取，session 内不再变。

但是，在用户体验上，我们虽然降低了一些 Skill 发现的概率，但一旦用户主动提起新的 skill 时，我们系统仍能及时发现新 Skill 。没有任何缓存，每次都会重建 Skill 列表。

## 决策 3：invoke\_skill 的妙用

`invoke_skill` 是 OpenClacky 的 16 个工具之一，它是整个 OpenClacky 最核心的设计，花费的时间也最多，它提供 Skill 热加载能力，子 Agent 架构支持，记忆召回能力、Skill 进化能力，但它只占 system prompt 不超过 200 个 Token 。

1. **启动子 agent** 。
2. **子 agent 用的工具集跟主 agent 完全相同** （ 16 个）。它不是一个"精简版"，它能做主 agent 能做的一切事情。
3. **子 agent 执行完后，把结果文本返回给主 agent** ，主 agent 的 history 里只看到"invoke\_skill → 结果"这一对消息。

这个设计一口气解决了好几个问题：

### 子 agent = 状态隔离

做代码审查的 skill 可能需要读几十个文件、跑 grep 、输出长篇分析。如果这些中间步骤都在主 agent 的 history 里，history 会膨胀得很快——cache 命中率没变，但上下文总量上去了，压缩触发得更早，成本更高。

子 agent 把这些中间过程隔离在自己的 session 里。主 agent 只看到最终结论。 **主 agent 的 history 没有被污染。**

### 动态加载 Skill ，不改 system prompt

装新 skill 的流程就是把一个 [SKILL.md](http://skill.md/) 放到 `~/.clacky/skills/<name>/` 或 `.clacky/skills/<name>/` 下。skill 列表渲染进 system prompt 的时间点是 session 启动，决策 2 已经讲过。

但 `invoke_skill` 这个工具本身是 **始终存在** 的——它不需要 system prompt 里列出所有 skill 才能调用。模型可以通过 `[session context]` 通告知道新 skill 的名称，然后直接 `invoke_skill(skill_name: "xxx")` 。Skill 的 [SKILL.md](http://skill.md/) 是在调用那一刻才读取的，不是预编译进 system prompt 的。

所以"动态加载 skill"这个能力，实际上是 `invoke_skill` 的运行时读取 + `[session context]` 的通告组合出来的。 **不需要改 system prompt ，不需要改工具列表，不需要重启 session 。**

### Skill 注入与路径处理

每个 skill 的 [SKILL.md](http://skill.md/) 可以引用相对路径的资源文件（模板、配置等）。 `invoke_skill` 在启动子 agent 之前会把 skill 的目录作为上下文路径注入，子 agent 能用 `file_reader` 、 `glob` 直接读到 skill 附带的资源。

这让 skill 可以做到"自包含"——一个 skill zip 包里既有指令又有模板，装上就能用。

### 加密 Skill 与选择性落盘

部分 skill 包含商业敏感内容（客户的 prompt 策略、内部流程等）。OpenClacky 支持对 [SKILL.md](http://skill.md/) 做加密存储，运行时解密到内存、用完不落盘。同时 session 的落盘也是选择性的——对于涉及加密 skill 的 session ，可以配置为不持久化到磁盘，只在内存中存在。

这不是 cache 工程的范畴，但它是 `invoke_skill` 架构的延伸：因为子 agent 的状态是隔离的，选择性不落盘可以精确到某次 skill 调用，而不需要把整个 session 的落盘关掉。

---

## 决策 4：控制稳定可靠的工具集 16 个

工具 schema 紧贴 system prompt 之后，在 cache 前缀里。schema 一变，后面全失效。这意味着： **每多加一个工具，你不只是多了一份 schema 的 token 成本，你还多了一份"下次改工具时全量 cache 失效"的风险面。**

另一面，工具太少也有代价：模型本来一步能做完的事，现在要分两三步（先调一个通用工具获取信息，再调另一个来操作），轮次上去了，每轮都要付 cache 和 output 的钱。

所以这不是一个"越少越好"的问题，而是一个 **经验平衡点** 。我们的答案是 **16 个** 。

### 这 16 个分别是什么

| 类别 | 工具 | 说明 |
| --- | --- | --- |
| 文件读写 | `file_reader`, `write`, `edit` | 读、写、搜索替换 |
| 代码搜索 | `glob`, `grep` | 文件查找 + 内容搜索 |
| 执行 | `terminal` | shell 命令 |
| 浏览器 | `browser` | 接管 Chrome/Edge |
| 网络 | `web_search`, `web_fetch` | 搜索 + 抓取网页内容 |
| 任务管理 | `todo_manager`, `list_tasks`, `undo_task`, `redo_task` | 规划、撤销、重做 |
| 交互 | `request_user_feedback` | 需要用户输入时 |
| 扩展 | `invoke_skill` | 调用 skill （决策 3 ） |
| 安全 | `trash_manager` | 安全删除（ rm → trash ） |

### 设计原则

**简化参数。** 每个工具的参数尽量少、语义尽量明确。比如 `glob` 只要 `pattern` 和 `base_path` ，不需要模型去组合 `--include` / `--exclude` / `--type` 这些 flag 。参数越多，模型出错的概率越高，出错就要重试，重试就是成本。

**够用但不冗余。** `glob` 和 `grep` 是两个工具而不是一个： `glob` 负责"哪些文件匹配"， `grep` 负责"文件里哪些行匹配"。合成一个会让参数变复杂，模型调错的概率上升。但也没有继续拆成 `find_files` / `list_dir` / `tree` 三个—— `glob` 一个就能覆盖这三个场景。

**为每个工具写丰富的测试用例。** 工具是 agent 跟外部世界的接口，一个工具出 bug 的代价远高于普通代码出 bug——它会让模型产生错误的观察，进而做出错误的决策，进而需要更多轮次来纠正。我们一共有 1600+ 的用例去覆盖各种场景的处理。最近有 V 友还给我们提交了子项目扫描慢（对，OpenClacky 支持子项目处理）的一个相关优化 issue 。

### 为什么不是 10 个，也不是 25 个

10 个做不到。 `undo_task` / `redo_task` / `list_tasks` 这些看起来"可以不要"的工具，拿掉之后模型就只能用 `terminal` 跑 `git` 来处理代码回滚——成功率远低于专用工具，而且 `git` 操作的副作用很难控制。很多工具设计了一个 code\_run ，我们并不推荐，实测反而导致任务变慢（需要写长代码），轮次变多（多次尝试）。

不需要 40+，只需要 16 个。

| 省掉的能力 | 替代方式 | 工具数节省 |
| --- | --- | --- |
| 代码库分析专用工具 | code-explorer Skill | ~5 个 |
| 记忆读写专用工具 | recall-memory Skill | ~3 个 |
| 浏览器自动化（多动作拆分为多工具） | 单一 browser 工具统一覆盖 | ~8 个 |
| Sub-agent 编排工具 | invoke\_skill 统一入口 | ~6 个 |
| 定时任务管理工具 | cron-task-creator Skill | ~4 个 |

如果以后需要第 17 个，我们会加。4 个月了，还没加。

---

## 决策 5：压缩——不换模型、空闲时做、压到底

上下文窗口是有限的。不管 200K 还是 1M ，长任务跑下来总会填满。填满之前必须压缩，否则要么截断丢信息，要么溢出直接报错。

压缩是 cache 命中率最大的单点威胁：老的消息被替换成一段摘要，前缀从那一刻起就跟之前不一样了—— **必然 cache miss** 。但压缩不可避免，所以问题不是"要不要压"，而是"怎么把压缩的破坏降到最低"。

### 结论一：不要换模型压缩

很多 agent 的压缩流程是开一个 **独立的 LLM call** ，用一个便宜/快速的小模型来做摘要。

问题：

- 独立 call 的 system prompt 跟主 session 不一样（通常是"你是一个摘要助手"）， **跟主 session 的 cache 没有任何共享前缀** ，压缩本身就是一次 100% cache miss 。
- 压缩完之后，主 session 的 history 被替换了（老消息变成了摘要），主 session 的 cache 也跟着失效——接下来 4–5 轮跑在 cold 费率上。

等于你为每次压缩付了 **两笔钱** ：一笔给压缩 call 本身的 cache miss ，一笔给主 session 压缩后的 cold-warm 阶段。

我们的做法： **压缩不开独立 call ，而是把压缩指令作为一条消息插进当前对话的末尾** （ Insert-then-Compress ）。

这条指令被打上 `system_injected: true` ，走正常请求路径。效果：

- **压缩 call 命中现有 cache** ：同样的 system prompt 、同样的 tools 、同样的 history 前缀。只有尾部的压缩指令是 cold 的，几百 token 。
- **压缩完成后，重建 history** ： `[system_prompt, summary, last_N_messages]` 。这一刻 cache 确实会 miss 一次——但只 miss 一轮，从第二轮开始双标记重新接管。

对比（一次 50K-token 会话的压缩事件）：

|  | 独立 call 方案 | Insert-then-Compress |
| --- | --- | --- |
| 压缩 call 的 cache hit | 0% | ~95% |
| 压缩期间 cold token | ~50,000 | ~500 |
| 主 session cold-warm 轮数 | 4–5 | 1 |

### 结论二：20–30 万 token 是压缩的甜区

太早压：浪费了上下文里还有价值的细节，摘要丢信息。 太晚压：上下文太长导致模型注意力分散、推理变慢、输出质量下降。

我们测过多个阈值。20–30 万 token 是效果和成本的甜区——模型还能有效利用上下文，但离溢出还有足够余量来完成压缩本身。

压缩后 **无论如何会压到 1 万 token 以内** 。这不是省钱，这是控制后续每一轮的 baseline 成本——history 越短，每轮 input 越少，cache miss 时的惩罚也越小。

### 结论三：空闲第 3 分钟启动压缩

这是跟 cache TTL 的博弈。大模型厂商的 prompt cache 普遍有 TTL—— **cache 在一段时间无请求后会过期** 。过期之后下一轮的 input 是全量 cold ，直接翻到 10× 成本。而且后续每轮都在叠加成本，直到 cache 重新 warm 起来。

所以我们跑了一个空闲计时器（ `idle_compression_timer.rb` ）：

- 用户停止输入 **90 秒** 后开始检查。
- 如果 history 已经接近压缩阈值 → **立刻触发压缩** 。此时 cache 还是热的，压缩代价很低。
- 压缩完之后，新的短 history 在 TTL 过期前就建立了新的 cache 断点。

效果是：用户思考了几分钟回来，看到的是一个 **已经压缩好、cache 已经 warm** 的 session 。相比之下，如果不做空闲压缩，用户回来时面对的是一个 cache 过期的长 history——那一轮的 input 可能是 30 万 token 全量付费。 **单这一个行为，在长思考间隔的场景下就能省 10× 的钱。**

空闲计时器跑在后台线程里。记得加锁！

### 百万上下文的真相

"百万 token 上下文"听起来很性感，但做 agent 有两个现实：

1. **过长的上下文对模型效果并不总是正面的。** 模型在超长上下文里的注意力分散问题是已知的——关键信息被淹没在大量历史里，输出质量反而下降。
2. **你真不一定用得起。** 记住，模型每轮都要把上一轮所有的上下文全部带上。100 万 token 的 input ，即使全部 cache hit （ 0.1× 费率），一轮也要付 10 万 token 等价的钱。如果 cache miss 了一次，那就是 100 万 token 全价。

真实世界用户停下来思考太过于常见，Cache Missing 太容易发生，Agent 开发者必须想办法帮用户减少开销。

所以我们的策略不是"尽量用满上下文"，而是" **积极压缩，保持 history 短小** "。1 万 token 的压缩后 history + 95% cache hit ，比 100 万 token 的未压缩 history + 99% cache hit 便宜得多，效果也更可控。

如何确保压缩后仍然保证足够好的效果，这是另一个话题，我们后面展开。

---

## 决策 6：自进化的工具能力

PDF 、Excel 、Word 、PPT 的阅读和解析是 Agent 经常遇到的需求。处理这类文件通常有两种路径：

1. **内置一个 tool** ：比如 `read_pdf` 、 `read_excel` 。好处是开箱即用，坏处是每个格式一个工具，工具列表膨胀（违背决策 4 ），而且解析库的依赖链往往需要 C 扩展，装起来就不"零痛"了。
2. **做成 skill 让用户装** ：对用户来说不友好——遇到一个 PDF 还得先去装 skill ，体验断裂。而且 skill 描述怎么写、什么时候触发，AI 效果不可控。

我们选了第三种路径： **首次安装时把预设的文档处理脚本 copy 到用户目录，之后允许 AI 自行更新维护这些脚本。**

具体做法：

- 首装 OpenClacky 时， `onboard` skill 会把一组 Python 脚本（ PDF 解析、Excel 读取、OCR 等） copy 到 `~/.clacky/scripts/` 。
- 这些脚本不是 Ruby ，而是 **Python 3** 。原因很实际：Python 的文档处理生态（ `pdfplumber` 、 `openpyxl` 、 `python-docx` 、 `python-pptx` ）是当前最成熟的，OCR 方面 `pytesseract` / `paddleocr` 也远比 Ruby 生态完善。
- 当 agent 需要读一个 PDF 时，它不调一个专用 tool——它用 `terminal` 工具跑 `python3 ~/.clacky/scripts/read_pdf.py <file>` 。 **工具列表没有增加。**
- 如果脚本跑不过去（缺依赖、格式变了），agent 可以直接 `write` 修改脚本、 `terminal` 跑 `pip install` 装依赖。下次再遇到同类文件就不会出问题了。

这就是"自进化"的含义： **处理文档的能力不是写死在 gem 里的，它活在用户目录的脚本里，agent 自己可以维护。** 第一次可能需要装个 `pdfplumber` ，装完之后就是永久能力。

这个设计把"文档处理"从工具层面拉到了脚本层面，避免了工具列表膨胀，也避免了硬编码 C 扩展依赖。trade-off 是用户机器上需要有 Python 3——但 macOS 和大多数 Linux 发行版默认自带，这个前提在实际用户群里几乎都满足。

---

## 决策 7：内置浏览器工具，No Headless

浏览器自动化是 Agent 越来越重要的能力——验证前端改动、抓取文档、自动化测试流程。

市面上主流的做法有两种：

1. **Headless 浏览器** （ Puppeteer / Playwright ）：agent 启一个无头浏览器实例，完全在后台跑。
2. **外接 MCP** ：通过 MCP 协议连接一个外部浏览器服务，agent 发 JSON-RPC 指令。

我们两种都不用，或者说—— **我们自己内置了一个 MCP Client ，去接管用户已经在跑的 Chrome / Edge** 。

### 为什么不用 Headless

Headless 浏览器的问题是"看不见"。agent 操作的页面用户看不到、不知道 agent 在干什么、出了问题也无法判断。对于 Agent 的使用场景——用户在旁边盯着 agent 干活——"看不见"是很大的信任问题。

另外，Headless 经常遇到反爬检测：登录态拿不到、Cloudflare challenge 过不去、需要手动验证。用户自己的浏览器里已经登录好了、cookie 都在，为什么不直接用？

### 我们怎么做的

`lib/clacky/tools/browser.rb` （ 610 行）+ `lib/clacky/server/browser_manager.rb` 是整套实现。架构是：

1. 用户的 Chrome / Edge 开启 Remote Debugging 端口（一次性配置， `browser-setup` skill 引导完成）。
2. OpenClacky 内置一个 **MCP Client** ，通过 stdio JSON-RPC 2.0 连接 `chrome-devtools-mcp` 这个 daemon 。
3. daemon 进程首次调用时启动，后续跨多次 tool call 保持存活。
4. `browser` 工具对外暴露的是高层语义动作： `snapshot` 、 `click` 、 `type` 、 `navigate` 、 `screenshot` 等——不是底层 CDP 指令。

**对模型来说，"浏览器"就是 16 个工具里的 1 个，schema 跟其他工具一样稳定，不会因为浏览器的状态变化而改 schema 。** 这符合决策 4 的原则。

### 为什么不把浏览器做成外部 MCP

我们可以不内置浏览器、让用户自己配一个 Browser MCP 服务。但这样做的问题是：

- **用户体验差** ：装 agent 之外还要装 MCP 服务、配端口、配认证。
- **稳定性不可控** ：外部 MCP 的版本、协议兼容性、超时行为都不在我们手里。
- **工具 schema 不可控** ：外部 MCP 可能暴露几十个细粒度工具（ `page.click` 、 `page.evaluate` 、 `page.waitForSelector` ……），直接打进主 agent 的 tool list 就违背了决策 4 。

内置一层封装的代价是我们要自己维护 MCP Client 和 daemon 的生命周期管理—— `browser_manager.rb` 里处理了 daemon 启动、心跳检测、超时、crash recovery 。但这个代价是一次性的工程投入，换来的是用户零配置（只要 Chrome 在跑）和工具列表的稳定。

---

## 最后，选择 Ruby 的理由

这不是一个显而易见的选择。LLM agent 生态里 Python 和 TypeScript 是主流，Ruby 几乎没有前例。但我们选 Ruby ，而且选对了。

### 动态语言 + 元编程

Ruby 的元编程能力是我们实现 Skill 自进化、动态加载、工具注册等能力的基础。 `method_missing` 、 `define_method` 、 `class_eval` 这些能力让运行时的行为修改非常自然。Python 也有类似能力，但 Ruby 在这一层的表达力明显更高。

对于一个"agent 自己可能改自己的辅助脚本"的系统来说，动态语言比静态语言更合适——你不需要重编译、不需要重启，改了就生效。

### 极致的分发能力

`gem install openclacky` 一行搞定。RubyGems 的分发链路非常成熟：版本管理、依赖解析、全局可执行文件注册（ `clacky` 命令）都是开箱即用的。用户不需要 `clone` 仓库、不需要 `npm install` 、不需要 `pip` 虚拟环境。

对比 Python 的分发—— `pip install` + 虚拟环境 + 可能的 C 扩展编译——Ruby gem 的安装体验明显更丝滑。

### 零 C 库依赖

这是我们做了大量工程投入才做到的。看 `openclacky.gemspec` 的依赖列表：

```
faraday, thor, tty-prompt, tty-spinner, diffy, pastel,
tty-screen, tty-markdown, base64, logger, websocket,
webrick, artii, rubyzip, rouge, chunky_png
```

全部是纯 Ruby gem ， **没有一个需要编译 C 扩展** 。

这意味着在 macOS / Linux 上，只要有 Ruby （ 2.6+）， `gem install openclacky` 就能装上、立刻能跑。不需要 `brew install libxml2` ，不需要 `apt-get install libffi-dev` ，不需要 Xcode Command Line Tools 。

为了做到这一点，我们做了一些反常规的选择：

- **WebSocket** ：没有用 `websocket-driver` （需要 C 扩展做 UTF-8 校验），而是用了纯 Ruby 的 `websocket` gem 。性能差一点点，但对 agent 场景来说完全够用，换来的是安装零阻力。
- **LLM 接口调用** ：完全零依赖，没有用任何第三方 LLM SDK （ `anthropic-rb` 、 `ruby-openai` 等都没用）。直接用 `faraday` 做 HTTP ，自己处理 streaming 、tool\_use 协议、cache\_control 注入。这样我们对请求格式有完全的控制权——决策 1 的双标记就是在 `client.rb` 里直接操作 cache\_control 字段实现的。
- **TUI** ：没有用 `curses` （ C 扩展），直接用 `tty-screen` + ANSI escape code "画"出整个终端界面。

### 这一切是 AI Coding 的产物

说实话，"从零重写 WebSocket 客户端"、"从零实现 LLM streaming 协议"、"用 ANSI escape code 手画 TUI"——这些事情如果纯手写，工程量很大，这在以往完全不现实。

但 OpenClacky 本身就是一个 AI coding agent 。这些"为了极致安装体验而大胆从零重写依赖"的决策， **是用 OpenClacky 自己来完成的** 。一个能写代码的 agent 让"零依赖"从不切实际变成了可执行。这是一个自举的过程——产品帮助自己变得更好。

---

## 结语

回头看这 7 个决策，它们背后其实只有一句话： **把工程预算花在 harness 上，把智能预算留给模型。**

不做 RAG ，不做多 Agent 编排，不做工具堆叠——不是因为这些东西没用，而是因为模型在快速变好。半年前需要 4 个 agent 协作才能勉强通过的任务，今天一个 agent + 一个好的 harness 就能做得更快更便宜。

我们选择把精力放在那些 **不会随模型进步而过时** 的事情上：cache 命中率、工具稳定性、安装体验、压缩策略。这些是 harness 层面的基础设施，不管模型换到哪一代都用得上。

如果这篇对你有用，请帮我们点赞，欢迎 PR 。欢迎转发和分享。

---

OpenClacky 完全开源，MIT 协议： [github.com/clacky-ai/openclacky](https://github.com/clacky-ai/openclacky)

`gem install openclacky` 一行装完即用，不需要 Docker 、不需要 clone 仓库。如果你也在做 Agent ，欢迎试试，遇到问题直接开 issue 聊。

4 家 Agent 横评的完整数据、产物对比、录像回放： [openclacky.com/benchmark](https://www.openclacky.com/benchmark)

本文引用的核心代码： [Cache 标记](https://github.com/clacky-ai/openclacky/blob/main/lib/clacky/client.rb) · [Insert-then-Compress](https://github.com/clacky-ai/openclacky/blob/main/lib/clacky/agent/message_compressor.rb) · [Session context 注入](https://github.com/clacky-ai/openclacky/blob/main/lib/clacky/agent.rb) · [空闲压缩](https://github.com/clacky-ai/openclacky/blob/main/lib/clacky/idle_compression_timer.rb) · [浏览器工具](https://github.com/clacky-ai/openclacky/blob/main/lib/clacky/tools/browser.rb)

29 条回复 **•** 2026-05-15 18:49:44 +08:00

|  |  | 1  **[yesha](https://www.v2ex.com/member/yesha)** 20 小时 53 分钟前  牛逼. |
| --- | --- | --- |

|  |  | 2  **[yafeilee](https://www.v2ex.com/member/yafeilee)**  OP  PRO  20 小时 51 分钟前 1  @ [yesha](https://www.v2ex.com/member/yesha) 帮点点赞:) 欢迎 issue 和 PR~ |
| --- | --- | --- |

|  |  | 3  **[Puteulanus](https://www.v2ex.com/member/Puteulanus)** 19 小时 48 分钟前  压缩那个确实，跑本地模型遇到的时候特别明显，时间超久的全量，还没看完，但感觉全是硬核干货，先点赞了 |
| --- | --- | --- |

|  |  | 4  **[yusf](https://www.v2ex.com/member/yusf)** 19 小时 38 分钟前  确实是干货 |
| --- | --- | --- |

|  |  | 5  **[v2yllhwa](https://www.v2ex.com/member/v2yllhwa)** 11 小时 26 分钟前 via Android  question 1:   ＞ \[session context\] 通告说"skill 列表已更新，新 skill 从下一个 session 可用"。这意味着用户装完 skill 想立刻用会发现用不了，要开新 session 。      ＞ 但 invoke\_skill 这个工具本身是始终存在的——它不需要 system prompt 里列出所有 skill 才能调用。模型可以通过 \[session context\] 通告知道新 skill 的名称，然后直接 invoke\_skill(skill\_name: "xxx")。      question 2:   ＞ 不要做工作流编排。 多 Agent 在结构上就是 cache 灾难。人类的分工不对 AI 有任何价值。AI 是万能的。      ＞ 这个设计一口气解决了好几个问题：   子 agent = 状态隔离   做代码审查的 skill 可能需要读几十个文件、跑 grep 、输出长篇分析。如果这些中间步骤都在主 agent 的 history 里，history 会膨胀得很快——cache 命中率没变，但上下文总量上去了，压缩触发得更早，成本更高。   子 agent 把这些中间过程隔离在自己的 session 里。主 agent 只看到最终结论。主 agent 的 history 没有被污染。      这两个地方是不是有一些前文和后文逻辑上的一些摩擦，是不是存在一些 trade off 和独特的设计？期待 op 分享一下。      比如第一个点我理解既然有 invoke\_skill 工具是不是没必要在 session context 里说下一回合可用？直接说列表更新就 OK ？      第二个点咱们这个 agent 和传统的 sub agent 按角色的拆分是不是有一些设计上的不同？ |
| --- | --- | --- |

|  |  | 6  **[phoenine](https://www.v2ex.com/member/phoenine)** 10 小时 37 分钟前  先顶后看，赞赞赞 |
| --- | --- | --- |

|  |  | 7  **[skuuhui](https://www.v2ex.com/member/skuuhui)** 10 小时 2 分钟前  我认真看了一下你上篇帖子的评论区，并确认几乎没人好奇你们的工程实践 |
| --- | --- | --- |

|  |  | 8  **[FakerLeung](https://www.v2ex.com/member/FakerLeung)** 9 小时 49 分钟前  一打开 [openclacky.com/benchmark](http://openclacky.com/benchmark) 整个 EDGE 就卡死了 |
| --- | --- | --- |

|  |  | 9  **[yafeilee](https://www.v2ex.com/member/yafeilee)**  OP  PRO  9 小时 49 分钟前  @ [v2yllhwa](https://www.v2ex.com/member/v2yllhwa) 主要是模型并不知道有新的更新，在处理新任务时它可能会忘记使用技能。session context 起到一个通知的作用，但也要谨慎使用它。另外，这种想法虽然不难想到，但实践时可能会导致 LLM 层要重构消息体，所以也有不少工程难度。有兴趣可以研读下代码。 |
| --- | --- | --- |

|  |  | 10  **[ytyyt](https://www.v2ex.com/member/ytyyt)** 9 小时 25 分钟前  追过来了，先赞后看 |
| --- | --- | --- |

|  |  | 11  **[yafeilee](https://www.v2ex.com/member/yafeilee)**  OP  PRO  9 小时 14 分钟前  @ [skuuhui](https://www.v2ex.com/member/skuuhui) 辛苦啦，我可能记得不是超级清楚，可能在其他渠道表达的，我微信就有不下 5 位直接私问，还有要上门学习的。      原话一：   \[握手\]亚飞总你好，我们最近也在做 AI native 和 Harness 工程。想沉淀下一些 AI 友好的内容、物料、文档、测试用例、可执行计划。   看到亚飞总你写的文章这么硬核全是干货，有时间我们交流交流。      原话二：   看了亚飞哥现在的产品 OpenClacky ，真的太硬核了。奇绩创坛。\[强\]   约个时间我线下拜访一下亚飞总可以吗，我们互相交流一下技术方案。       这篇文章是回应大家的需求。      ps: 我也想跟 V 站的伙伴说，咱们中国开源也有技术硬核的东西，希望大家多多支持，平替 ClaudeCode （它实在太“贱”了，甚至面向一般用户更友好的 Agent 。 |
| --- | --- | --- |

|  |  | 12  **[yafeilee](https://www.v2ex.com/member/yafeilee)**  OP  PRO  9 小时 11 分钟前  @ [FakerLeung](https://www.v2ex.com/member/FakerLeung) 里面渲染了几个 html 产物，可能旧的版本会有点卡。之前已经优化过一轮。升升版看下~ |
| --- | --- | --- |

|  |  | 13  **[Java4ever](https://www.v2ex.com/member/Java4ever)** 8 小时 43 分钟前  好像并不支持第三方的中转 API |
| --- | --- | --- |

|  |  | 14  **[yafeilee](https://www.v2ex.com/member/yafeilee)**  OP  PRO  8 小时 32 分钟前  @ [Java4ever](https://www.v2ex.com/member/Java4ever) 支持~ 自定义里面选 openai 兼容接口就可以了。遇到哪些配不上的中转可以提个 issue 我们排查下。已知有些中转站支兼容了 anthropic 格式的话可能会有问题。 |
| --- | --- | --- |

|  |  | 15  **[erwin985211](https://www.v2ex.com/member/erwin985211)** 7 小时 50 分钟前 1  希望论坛多一点这样干货分享而不是，”我老婆这样算出轨吗“的故事汇 |
| --- | --- | --- |

|  |  | 16  **[jerrrrr](https://www.v2ex.com/member/jerrrrr)** 7 小时 43 分钟前  感谢分享，省 token 的路线确实做得不错。我有几个问题想请教下：   1\. \`加密 skill\`这个设计是不是很难做到安全哈，特别是源码都开源了，那对于你的产品化会有什么影响呢？   2\. 您觉得\`省 token\`和\`实际任务完成度\`，怎么去权衡呢？至少站在我的角度，我更偏向于实际任务完成度，所以如果是任务完成度很重要的话，那么有能力和像 codex 、cc 那样模型厂商亲自下手扳扳手腕吗？   3\. 非模型厂商去做这种 coding agent ，您觉得有什么可以产品化的优势吗？我担心这些其实只是个人玩具，然后被淹没。 |
| --- | --- | --- |

|  |  | 17  **[yafeilee](https://www.v2ex.com/member/yafeilee)**  OP  PRO  7 小时 29 分钟前  @ [jerrrrr](https://www.v2ex.com/member/jerrrrr) 加密 skill 是另一个话题，后面会逐步展开。   第二个问题：省 token 跟任务完成是两个维度，并不矛盾。目前我们对具体任务展开评测，包括我们实际使用来说，模型一致的情况下，codex, cc 的表现我们完全是同量级的。甚至有些还更优秀。这个本质是是 harness 中最重要的部分：context 管理。这个后面我们再补充更多测试场景出来。我敢说的话，harness 工程是否优秀直接决定了最终任务情况，包括是否省钱，完成速度。   第三个问题，非模型厂就是做极致的开源好用，用 BYOK ，用社区力量，让用户拿到性价比最好的产品，不被模型锁定。商业化思考我们在价格页也有列出，大家也不用担心，这个事情可以做很长很长时间。 |
| --- | --- | --- |

|  |  | 18  **[garyalen](https://www.v2ex.com/member/garyalen)** 6 小时 41 分钟前  是强者的发型 |
| --- | --- | --- |

|  |  | 19  **[longxinglink](https://www.v2ex.com/member/longxinglink)** 5 小时 59 分钟前  支持，我有几十个 openai free 账户，想要用完之后自动切换下一个账户，我应该怎么在 openclacky 里面设置？   我想要类似 codex 的 在思考对话时跟进提示词引导对话 这个如何设置？ |
| --- | --- | --- |

|  |  | 20  **[longxinglink](https://www.v2ex.com/member/longxinglink)** 5 小时 54 分钟前  openclacky 确实地解决了我这边切换账户后，codex 本地对话记录无法读取的问题，很好用，为你点赞 |
| --- | --- | --- |

|  |  | 21  **[longxinglink](https://www.v2ex.com/member/longxinglink)** 5 小时 37 分钟前  几分钟能解决的短任务可能会选择 openclacky ，几个小时的长任务持续目前我只能用 Codex 或者 Claude Code ，没远程连接，账户切换，引导跟进，长任务只能看着 AI 犯错或者宕机 |
| --- | --- | --- |

|  |  | 22  **[yafeilee](https://www.v2ex.com/member/yafeilee)**  OP  PRO  5 小时 27 分钟前  @ [garyalen](https://www.v2ex.com/member/garyalen) 😄 |
| --- | --- | --- |

|  |  | 23  **[yafeilee](https://www.v2ex.com/member/yafeilee)**  OP  PRO  5 小时 21 分钟前  @ [longxinglink](https://www.v2ex.com/member/longxinglink) 你觉得哪些长任务特性是特别关键的 我学习下~ |
| --- | --- | --- |

|  |  | 24  **[longxinglink](https://www.v2ex.com/member/longxinglink)** 4 小时 56 分钟前  @ [yafeilee](https://www.v2ex.com/member/yafeilee) 成本低而且稳定性强，codex 有个特性，即使剩下 1%额度，也会在任务完成后才停止，也就是所谓的超用，可以用一个 free 账户跑一晚上，早上起来出结果，依赖这个特性我每个月可以节省几百美元的订阅费用      如果用 codex ，早上切换账户可能会把之前的本地聊天记录抹掉或者各种原因无法加载，openclacky 这点很好，没这个问题      还有可以随时干预，比如 gpt-5.5 模型很多时候不知道自己的问题在哪里，在错误的文稿上继续迭代，这个时候引入外部模型比如 Claude 检查每个任务完成情况很有必要，Claude sonnet 4.6 和 opus 4.7 自检能力比 GPT 强大太多， [todo.md](http://todo.md/) 完成小结时候检查一下，过分偏移任务目标直接停止任务，或者 agent 宕机陷入无限 loop ，这个时候外部的修正并且新开进程很重要，openclacky 这点完全做不到（ Claude code 给定时任务算的是额外免费额度，不占用对话额度）      比如在外面跑步或者工作，收到机器部署的发卡网站数据库漏洞被爆破中的消息，这个时候还需要坐在家庭电脑桌前用 openclacky 去停下机器完成自检追踪溯源只能说完全不可能 |
| --- | --- | --- |

|  |  | 25  **[caiweilong00](https://www.v2ex.com/member/caiweilong00)** 3 小时 53 分钟前  对于第一点，不太明白。      我只要保证每次历史对话都是追加的，不就能保证缓存的有效性吗。      这里 marker 的意义是什么？没有 marker 会有什么影响 |
| --- | --- | --- |

|  |  | 26  **[yafeilee](https://www.v2ex.com/member/yafeilee)**  OP  PRO  2 小时 49 分钟前  @ [caiweilong00](https://www.v2ex.com/member/caiweilong00) 很多 LLM 需要显式声明。还有容错需要（处理图片、文件、插入新消息等），甚至 Thinking 。所以提升缓存命中要考虑的因素很多。 |
| --- | --- | --- |

|  |  | 27  **[gongjs](https://www.v2ex.com/member/gongjs)** 1 小时 12 分钟前  @ [caiweilong00](https://www.v2ex.com/member/caiweilong00) 这里也没看懂，理论上来说，只要不改历史记录，那历史记录那块的缓存就应该一直有效的才对，最多只是新增的那块没有缓存，为啥要加 2 个 marker 的？ |
| --- | --- | --- |

|  |  | 28  **[yafeilee](https://www.v2ex.com/member/yafeilee)**  OP  PRO  57 分钟前  @ [gongjs](https://www.v2ex.com/member/gongjs) 文中有提，需要预防失败回滚一次。 |
| --- | --- | --- |

|  |  | 29  **[f11st1](https://www.v2ex.com/member/f11st1)** 54 分钟前  写的挺好，不过现在国内中转 api 与搭建 VPN 一个罪名，我看 lz 整了 OpenClacky AI Keys ，还是注意下比较好😂 |
| --- | --- | --- |

**[关于](https://www.v2ex.com/about) · [帮助文档](https://www.v2ex.com/help) · [自助推广系统](https://www.v2ex.com/pro/about) · [博客](https://blog.v2ex.com/) · [API](https://www.v2ex.com/help/api) · [FAQ](https://www.v2ex.com/faq) · [Solana](https://www.v2ex.com/solana) · 2900 人在线** 最高记录 6679 · [Select Language](https://www.v2ex.com/select/language) 创意工作者们的社区 World is powered by solitude VERSION: 3.9.8.5 · 107ms · [UTC 11:44](https://www.v2ex.com/worldclock#utc) · [PVG 19:44](https://www.v2ex.com/worldclock#pvg) · [LAX 04:44](https://www.v2ex.com/worldclock#lax) · [JFK 07:44](https://www.v2ex.com/worldclock#jfk)  
♥ Do have faith in what you're doing.

Explain