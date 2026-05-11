---
date: 2026-05-11
tags: [ai, software-engineering]
type: concept
status: active
---
# Vibe Coding
一种 AI 编程方式：开发者用自然语言描述需求，AI 生成代码，开发者不阅读也不理解生成的代码。与 specs-to-code 本质相同——都在放弃对代码的所有权。

## Details
Matt Pocock 的诊断：vibe coding 的代码质量逐次递减——第一次还行，第二次变差，第三次更差，越跑越烂。根本原因是不理解代码就无法有效迭代。

软件熵在 AI 时代的扩散速度比人类写代码时快得多：差的代码库 + AI = 更快地变成无法维护的泥潭。

替代方案：保持对代码的所有权，使用 [[tdd]]、[[deep-module]] 等工程实践让 AI 在结构化的约束下工作。

## See Also
- [[software-fundamentals-in-ai-era]]
- [[matt-pocock]]
- [[tdd]]

## Counter-Arguments and Gaps
- 对原型验证和一次性脚本，vibe coding 可能有其合理性
