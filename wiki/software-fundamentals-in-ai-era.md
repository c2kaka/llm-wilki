---
date: 2026-05-11
tags: [ai, software-engineering]
type: source-summary
source-url: https://zhuanlan.zhihu.com/p/2032512098626954988
---
# Software Fundamentals Matter More Than Ever (软件基本功没死)
Matt Pocock 在 AI Engineer World's Fair 2026 的演讲摘要：软件工程基本功在 AI 时代比以往更重要。好的代码库让 AI 表现指数级更好，差的代码库让 AI 越改越烂。他归纳了六个 AI 编程失败模式及对应解法，封装为 mattpocock/skills 仓库。

## Key Points
- 六个失败模式：设计概念缺失、术语不对齐、无反馈循环、代码库混乱、认知过载、大任务进入愚蠢区
- grill-me：通过无情审问建立设计共识，优于 plan 模式
- ubiquitous-language：扫描代码库提取术语表，对齐人机话语体系
- tdd：强制红-绿-重构循环，防止 AI 一次写太多代码
- improve-codebase-architecture：将浅模块合并为深模块，提升 AI 可理解性
- 完整工作流：grill-me → to-prd → to-issues → AFK 智能体实现 → 人工 QA
- 反对"specs to code"范式和 vibe coding，主张保持对代码的所有权
- 核心引用：《软件设计哲学》(深模块)、《程序员修炼之道》(曳光弹)、《设计的设计》(设计概念)、《领域驱动设计》(通用语言)

## Entities Mentioned
- [[matt-pocock]]
- [[deep-module]]
- [[design-concept]]
- [[ubiquitous-language]]
- [[tdd]]
- [[vibe-coding]]
- [[agent-harness]]
- [[claude-code]]
