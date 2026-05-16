---
title: "Agent 系统正在重新走一遍 OS 和 Cloud Runtime 的老路"
source: "https://zhuanlan.zhihu.com/p/2037479092090622773"
author:
  - "[[acodespaceHumans steer. Agents execute.]]"
published:
created: 2026-05-13
description: "Agent 系统正在重新走一遍 OS 和 Cloud Runtime 的老路 2026-05-11 · 读后感，非翻译非摘要写在前面读完 Anthropic 的《Scaling Managed Agents: Decoupling the brain from the hands》之后，想从系统工程层面把…"
tags:
  - "clippings"
---
[收录于 · agent 系统工程观-随笔](https://www.zhihu.com/column/c_2037482586189074960)

null、锦恢 等 471 人赞同了该文章

目录

收起

Agent 系统正在重新走一遍 OS 和 Cloud Runtime 的老路

写在前面

一、从 prompt 到 runtime：问题的形状在变

二、harness 会腐化：过期的正确性最危险

三、context engineering 正在变成 runtime engineering

四、append-only session log：很像 event sourcing

五、brain 与 hands：本质是 control plane / data plane

六、tools 与 sandbox：正在变成 disposable runtime

七、interface 比 implementation 更重要

八、真正难的：long-running 与观测、评价都还没被吃透

九、mental model 的转变：从模型出发，到从 runtime 出发

十、还没有标准答案，但趋势已经露出来

**2026-05-11** · 读后感，非翻译非摘要

---

### 写在前面

读完 [Anthropic](https://zhida.zhihu.com/search?content_id=274686270&content_type=Article&match_order=1&q=Anthropic&zhida_source=entity) 的《Scaling Managed Agents: Decoupling the brain from the hands》之后，想从系统工程层面把想法摊开写一写：更像是一边读一边对照自己过去对 agent infra 的判断，而不是逐段复述原文。

---

### 一、从 prompt 到 runtime：问题的形状在变

我一开始对 agent 的理解挺朴素：早期总觉得核心在 prompt——怎么让模型更会拆任务、更稳地调工具、别一上来就胡写代码、遇到错误还能往前走。

做得多了会发现，prompt 当然重要，但真正麻烦的是 **workflow** 。复杂任务不是一轮问答；它有文件读写、命令执行、外部 API、测试反馈、中间状态和失败重试。于是大家写 harness、tool loop、memory、planner、evaluator；那时候我觉得 agent engineering 的核心更像是 **[workflow orchestration](https://zhida.zhihu.com/search?content_id=274686270&content_type=Article&match_order=1&q=workflow+orchestration&zhida_source=entity)** 。

再往后又觉得，workflow 也只是表象。更底层的问题是：workflow 跑在哪里？状态放哪里？失败怎么恢复？工具和模型的边界在哪？凭证属于谁？上下文满了以后什么能丢、什么绝不能丢？任务一跑几小时甚至几天，模型、sandbox、网络、工具、权限任一环节出问题，系统要怎么续上？

到这里，事情就不像 chatbot 了——它开始像 **runtime** ，甚至更准确地说，像 operating system / distributed system / cloud runtime 那一类东西。不是因为名词高级，而是 **问题的形状** 已经变了。

![](https://pic1.zhimg.com/v2-d3913bbea028ae6542c762f8479f8a06_1440w.jpg)

---

### 二、harness 会腐化：过期的正确性最危险

我以前对 harness 比较乐观：不就是 agent loop 吗——调模型、处理 tool call、维护状态、必要时压上下文、再下一轮。

现在看，它非常容易 **腐化** 。这里说的腐化不是代码写烂，而是会慢慢堆满某一代模型、某一代工具、某一种部署环境的假设：模型快顶满窗口就提前收尾，你加 context reset；模型不太会从工具错误恢复，你加 retry prompt；sandbox 偶尔卡死，你在 harness 里堆 watchdog；工具返回格式飘，你在 loop 里塞解析补丁。补丁刚加时都合理——线上要救火、任务要跑完，不能空谈架构洁癖。

麻烦在于 **模型进步太快** ：今天像缺陷的行为，三个月可能就不是了；今天好用的 prompt trick，下一代可能反而被干扰；今天能救任务的 context reset，未来可能只是打断长程推理的噪声。于是 harness 里最危险的不一定是 bug，而是 **过期的正确性** ——它曾经对过、救过线上，所以没人敢删；等底层模型变了，它就变成看不见的阻力。这和 kernel workaround、某代 NIC 行为、某一版 GPU scheduler、某个云厂商限制很像：先是补丁，再变成 ABI 的一部分，最后谁都不敢动。agent harness 也会这样。

所以我越来越觉得： **harness 不该被当成地基** ，它更像策略层、甚至某个可替换的 control loop；真正该稳定下来的是更低层的 **[runtime abstraction](https://zhida.zhihu.com/search?content_id=274686270&content_type=Article&match_order=1&q=runtime+abstraction&zhida_source=entity)** 。

---

### 三、context engineering 正在变成 runtime engineering

过去谈 context engineering，多半是在说怎么把信息塞进窗口：检索哪些文件、摘要哪些历史、保留哪些 tool output、怎么裁长日志、怎么别让模型忘掉硬约束。这些都对。

可一旦任务长时间跑起来，context engineering 就不只是「怎么喂模型」，而是 **状态管理** 。上下文窗口只是当前调用能看到的一小块工作集—— **它不是 runtime，不是 session，更不是 source of truth** 。我特别喜欢这句话： **context window is not runtime** 。模型上下文更像 CPU cache、或进程当前映射进来的页：影响很大，但不能承担持久状态。

若 agent 跑很久，中间几百次工具调用、改文件、测挂、重试、权限、用户插话，都不能只活在上下文里。上下文一定会被裁，摘要一定有损，注意力一定会漂；若系统把 context 当 session， **恢复能力基本是假的** ——你恢复的往往是被压缩、被转述、可能已经偏掉的故事版本。

更稳的做法，是把完整 session 当成 **append-only log** ：发生过什么就记下来，模型当前需要什么，再从这条 log **构造 view** 。这时 context engineering 就是 runtime engineering 的一部分：如何从事件日志 materialize 上下文、如何做 compaction、recall、权限过滤、审计、replay。这已经不只是 prompt 技巧，而是 **系统状态模型** 。

---

### 四、append-only session log：很像 event sourcing

append-only session log 越看越不像 agent 圈子的新发明，它很像分布式系统里的 **event sourcing** ：不只存当前状态，而存导致状态的事件序列；状态可重放，不同视图可从同一份 log materialize；恢复、审计、debug、回放都挂在这条 log 上。

agent 任务也适合：用户说了什么、模型想了什么、调了什么工具、返回了什么、改了哪些文件、测挂在哪里、后来为何换路线——全是事件。若只留最后摘要，很多问题根本查不了：为什么改这个文件？是否被某段 tool output 带偏？是否忽略用户后来的约束？是否在某次 compaction 后丢了关键事实？没有原始事件流，只能靠猜。

有 append-only log 就不同：可 replay、可 diff，不同版本 harness 可对同一条 session 做对比；evaluator 也可直接消费这条 log，而不只看最终结果。还有一个隐含好处： **把「记忆」从模型里拿出来** ——runtime 存事实，模型在某个 view 上推理；边界很像数据库与查询引擎：数据别塞进查询计划，计划可以变，数据不能丢。

---

### 五、brain 与 hands：本质是 control plane / data plane

「把大脑和手拆开」很直观，我更愿意看成 **control plane 与 data plane 的分离** 。brain（模型 + harness）像 control plane：决策、编排、选工具、解释反馈、定下一步。hands（sandbox、tool、MCP、浏览器、客户 VPC 里的执行）像 data plane：真执行、跑命令、访问资源、搬数据、产生副作用。

早期把两层塞进同一个容器，就是 control plane 与 data plane 混在一起——开头简单，后面一定痛，因为 **生命周期不同** ：control plane 要可恢复、可升级、可替换；data plane 要隔离、可丢、可按需起；session state 要比二者更持久；credentials 又应在另一道边界里。全塞一个盒子里，短期省事，长期全是耦合。云基础设施早就走过这条路：K8s 不会让每个 pod 自己定全局调度；网络不会把控制面与高速转发糊在一起；数据库也不会把 WAL、buffer pool、executor 搅成一团。agent 系统也在进入这个阶段——模型越强、工具越多、任务越长、客户环境越复杂，这种分离就越像 **必然** ，而不是架构品味。

---

### 六、tools 与 sandbox：正在变成 disposable runtime

以前我把 sandbox 理解成「agent 的工作目录」；现在更倾向 **disposable runtime** ：不是家、不是记忆、不是长期状态，而是某次执行的工位——需要时 provision，坏了就丢，污了就换，资源不够再分配。repo、依赖、临时文件、测产物可以放在里面，但它们不该是系统的 **source of truth** 。这和云上 cattle 思路一致：sandbox 挂了还要工程师 ssh 进去救，多半说明边界画错了。现实里当然要有调试入口，不能假装不要 ssh、日志、dump；但设计目标应是 **靠 runtime 恢复任务，而不是抢救某个容器** 。

这会落到很多细节：工具输出事件化、文件改动可追踪、创建过程可重复、依赖安装可有 cache 但 cache 不能是唯一状态、凭证不直接进 sandbox、网络按能力授权。一旦 tools/sandbox 都是 disposable，orchestration 自然更像 control plane——不关心某个 sandbox 的「感受」，而关心任务需要什么执行能力、有没有健康实例、失败后如何重试、副作用是否已发生、状态能否恢复。这就远不止是「模型调工具」了。

![](https://pic1.zhimg.com/v2-91b492970bf01db6becdd2ee24934a88_1440w.jpg)

---

### 七、interface 比 implementation 更重要

文章里反复强调的是 **interface** ，不是某段 harness 怎么写、某个 sandbox 怎么实现、某个 reset 策略多聪明，而是 session、execute、wake、provision 这类边界——很像 OS 的 syscall：应用调 `read` ，不必知道底下是机械盘、SSD、NFS 还是未来某种存储；harness 调 `execute` ，也不必知道背后是容器、VM、MCP、浏览器、客户 VPC 还是未来设备。 **接口稳定，implementation 才能乱换。**

关键不是抽象癖，而是 **未来真的会乱换** ：模型、harness、工具协议、sandbox 类型、部署形态都会换，agent 形态也会从 coding agent 扩到 data、infra、安全、浏览器、机器人等。平台若押注具体实现，很快会被拖住；若押注稳定接口，下层可以演进。这也是我觉得 future agents 更像 **runtime systems** 而不是 chatbot 的原因：chatbot 的核心是对话；agent runtime 的核心是 **长期任务的可靠执行** ——不在一个层级。

---

### 八、真正难的：long-running 与观测、评价都还没被吃透

很多 demo 很顺：给任务、拆解、调工具、改代码、跑测、出结果。但 **demo 与 long-running system 之间差得很远** ；真正难的是 **long-running reliability** ——中间模型失败、工具超时、sandbox 崩、网络抖、权限过期、用户插话、需求变、上下文压缩、评价标准变，都会发生。系统不能次次从头来，也不能指望模型「记得」；需要 orchestration、state recovery、execution coordination。

例如：动作有没有副作用？工具失败是执行前还是执行后？重试会不会双提交？改文件与调外部 API 之间有没有一致性？一个 agent 能否把一段活交给另一个？两人同时改同一 repo 谁仲裁？这些问题听起来老套，因为它们本来就是 **distributed systems** 的老问题，只是入口换成了 agent。

**观测** 也棘手：传统服务至少有 logs、metrics、traces；agent 系统要看 token、tool calls、session events、中间决策、compaction 前后丢了什么、某条 harness 策略对成功率的影响？ **评价** 也难：只看最终答案不够，还要看过程——是否走危险路径、是否滥用权限、是否把偶然通过的测试当完成、是否在不确定时装确定。这些都需要 **runtime 层面的观测** ，而不是最后扔给另一个模型打个分了事。

---

### 九、mental model 的转变：从模型出发，到从 runtime 出发

读完之后，我对 agent infra 的 mental model 有了一点位移。以前习惯从 **模型能力** 出发：模型越强，agent 越强——这话没错，但不够。现在我更愿意从 **runtime** 出发：模型是 brain，但 brain 需要运行时——手、记忆、权限边界、恢复、观测、调度；更重要的是，这些不能与某一代 brain **绑死** ，否则每次模型升级都是一次痛苦迁移（旧 harness 假设、旧 context 策略、旧工具边界、旧 sandbox 生命周期一起爆）。这就是 **harness assumptions rot** ：不一定是人写错了，而是底层世界变了，上层假设还留在那里。

当 agent infra 进入系统工程阶段，核心问题会慢慢从「怎么让模型更聪明」转向更「不性感」但决定能不能长大的那一类：

1. 状态模型是什么？
2. 控制面与执行面怎么分？
3. 失败恢复靠什么？
4. 哪些策略可插拔？
5. 哪些接口要长期稳定？
6. 什么绝不能只活在上下文窗口里？
7. 什么绝不能只塞进 sandbox？
![](https://pic2.zhimg.com/v2-70ffcb074dedcffb1a5e2824a8f93b77_1440w.jpg)

---

### 十、还没有标准答案，但趋势已经露出来

我不觉得现在已经有一个确定的「Agent OS」形态——还早。MCP、sandbox runtime、session log、multi-agent orchestration 都还会变。但有些方向已经比较清楚，可以收束成下面几条（顺序不分先后）：

- **context window 不是 runtime**
- **harness 不该变成历史补丁博物馆**
- **session log 会越来越像系统事实源**
- **tools 与 sandbox 会越来越 disposable**
- **orchestration 会越来越像 control plane**
- **interface 会比 implementation 更值钱**
- **agent 未必停在「更会聊天的应用」** ，而可能变成一种 **runtime workload**

若这个判断成立，未来几年 agent infra 的争论，可能会从 prompt、memory、tool use，慢慢滑回更老派的一组词：调度、隔离、一致性、恢复、观测、权限、评估。听起来一点都不新——系统工程常常就是这样：新 workload 先兴奋于能力，能力进生产后，老问题全回来；只是这次，执行主体从人写的程序，换成了会推理、会调工具、也会犯奇怪错误的模型。

我读原文时最强的感受是： **Agent infra 已经进入系统工程阶段** 。不是因为文章用了多少大词，而是因为它开始认真碰那些逃不掉的东西：长期运行、状态恢复、执行隔离、接口稳定、失败语义、安全边界、可观测性。这些一旦出现，游戏就不只是 [prompt engineering](https://zhida.zhihu.com/search?content_id=274686270&content_type=Article&match_order=1&q=prompt+engineering&zhida_source=entity) 了。后面会很有意思，也会很麻烦。

我个人更相信： **真正值钱的往往不是某一个 agent，而是承载 agent 的 runtime** ——就像单个进程重要，但 OS 更重要；单个服务重要，但 cloud runtime 更重要；单个模型当然重要，但当模型够多、任务够长、工具够复杂时，谁能定义那层 **稳定的运行时抽象** ，谁才可能在搭下一代基础设施的地基。

编辑于 2026-05-12 10:44・浙江