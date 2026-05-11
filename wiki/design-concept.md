---
date: 2026-05-11
tags: [software-engineering, design]
type: concept
status: active
---
# Design Concept (设计概念)
来自 Frederick P. Brooks《设计的设计》。当多人协同设计时，他们之间漂浮着一个无形的共同理解——对将要构建之物的构想。

## Details
在 AI 编程中，你和 AI 之间缺的就是设计概念。AI 没做你想要的事，根本原因往往是双方没有建立共同的构想。

Matt Pocock 的解法是 grill-me 技能：AI 以无情审问的方式持续提问（40-60 个问题），逐一追究设计树的每个分支，直到达成共识。这比 Claude Code 默认的 plan 模式更有效，因为 plan 模式急于创建计划并开始干活，而 grill-me 先明确设计概念。

## See Also
- [[software-fundamentals-in-ai-era]]
- [[matt-pocock]]
- [[ubiquitous-language]]

## Counter-Arguments and Gaps
- 设计概念作为隐性知识，其可传递性本身存在争议
