---
date: 2026-05-11
tags: [ai, agents, architecture]
type: source-summary
source-url: https://x.com/dotey/status/2053601852261110201
---
# The Anatomy of an Agent Harness (AI Agent Harness 的构造)
Akshay Pachaar 的深度文章，拆解 Anthropic、OpenAI、Perplexity 和 LangChain 的 Agent 基础设施。Harness 是包裹在 LLM 之外的完整软件架构，包含编排循环、工具、记忆、上下文管理等 12 个组件。

## Key Points
- Harness 定义："如果你不是模型本身，那你就是 Harness"（Vivek Trivedy, LangChain）
- 冯·诺依曼类比：上下文窗口=内存，外部数据库=硬盘，工具=设备驱动，Harness=操作系统
- 三层工程：提示词工程 → 上下文工程 → Harness 工程
- 12 组件：编排循环、工具、记忆、上下文管理、提示词构建、输出解析、状态管理、错误处理、护栏与安全、验证循环、子智能体编排、上下文压缩
- TerminalBench 证据：仅改变 Harness 即可让排名变动 20+ 位
- 协同进化原则：模型训练时已考虑 Harness 存在，好的 Harness 在模型升级时自动获益
- Harness 应随模型能力增强而变薄，但永远不会消失

## Entities Mentioned
- [[agent-harness]]
- [[orchestration-loop]]
- [[context-engineering]]
- [[mcp]]
- [[claude-code]]
- [[deep-module]]
