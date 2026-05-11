---
date: 2026-05-11
tags: [software-engineering, ai]
type: concept
status: active
---
# TDD (Test-Driven Development)
测试驱动开发：先写一个失败的测试，然后写最少的代码让测试通过，最后重构。红-绿-重构循环。

## Details
在 AI 编程中，TDD 的核心价值是防止 AI 一次产出太多代码。AI 默认行为是先写完所有实现再考虑测试，这导致《程序员修炼之道》所说的"跑在前灯前面"——开车太快，灯光照不到前面的路。

反馈速度是速度上限。没有类型检查、测试和静态分析，AI 就是在黑暗中编程。

Matt Pocock 的 tdd 技能强制 AI 走红-绿-重构循环，AI 很难作弊——它必须在代码存在之前把测试写好。

## See Also
- [[software-fundamentals-in-ai-era]]
- [[matt-pocock]]
- [[deep-module]]
- [[vibe-coding]]

## Counter-Arguments and Gaps
- TDD 在探索性/原型阶段可能过度约束，需平衡
