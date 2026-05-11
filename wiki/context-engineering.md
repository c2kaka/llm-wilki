---
date: 2026-05-11
tags: [ai, agents]
type: concept
status: active
---
# Context Engineering (上下文工程)
管理 LLM 在什么时间点能看到什么内容的工程实践。位于提示词工程之上、Harness 工程之下。

## Details
核心问题：上下文腐烂。斯坦福"迷失在中间"研究发现，关键信息处于窗口中间位置时模型表现下降 30%+。即便百万 Token 窗口，指令遵循能力也随上下文增长而退化。

Anthropic 的目标定义：找到能最大化达成目标概率的、信号最强的最小 Token 集合。

生产环境应对策略：
- **压缩**：接近限制时总结对话历史（Claude Code 保留架构决策和未修复 Bug）
- **观察掩码**：隐藏旧的工具输出但保留调用记录
- **即时检索**：保留轻量级标识符，动态加载数据
- **子智能体委托**：深度探索后仅返回 1000-2000 Token 摘要

Matt Pocock 的选择：不压缩，要清除。每次从稳定起点开始，不在沉积上下文中越走越偏。

## See Also
- [[agent-harness]]
- [[anatomy-of-agent-harness]]
- [[orchestration-loop]]
- [[software-fundamentals-in-ai-era]]

## Counter-Arguments and Gaps
- "压缩 vs 清除"的最佳选择可能因任务类型而异，尚无统一结论
