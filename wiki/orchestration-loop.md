---
date: 2026-05-11
tags: [ai, agents]
type: concept
status: active
---
# Orchestration Loop (编排循环)
Agent Harness 的核心组件，实现"思考-行动-观察"(Thought-Action-Observation, TAO) 循环，也称为 ReAct 循环。

## Details
循环流程：整合提示词 → 调用 LLM → 解析输出 → 执行工具调用 → 反馈结果 → 重复，直到任务完成。

技术实现通常是一个 while 循环，但复杂之处在于循环内要处理的状态和逻辑。Anthropic 将其描述为"笨循环"——所有智慧在模型中，Harness 只管理回合切换。

七步完整循环（Anthropic 模型）：
1. 提示词组装
2. 模型推理
3. 输出分类（是否有工具调用）
4. 工具执行（沙箱中）
5. 结果打包
6. 上下文更新（必要时压缩）
7. 返回第一步

## See Also
- [[agent-harness]]
- [[anatomy-of-agent-harness]]
- [[context-engineering]]

## Counter-Arguments and Gaps
- "笨循环"假设可能过于简化，实际 Harness 中的智慧分布更复杂
