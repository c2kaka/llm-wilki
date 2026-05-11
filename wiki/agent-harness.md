---
date: 2026-05-11
tags: [ai, agents, architecture]
type: concept
status: active
---
# Agent Harness
包裹在 LLM 之外的完整软件架构，让无状态模型变成能自主行动的智能体。包含编排循环、工具、记忆、上下文管理、状态持久化、错误处理和护栏等组件。

## Details
Harness 的定义来自 LangChain 的 Vivek Trivedy："如果你不是模型本身，那你就是 Harness。"Anthropic 在 Claude Code 文档中称 SDK 就是"驱动 Claude Code 的 Agent Harness"。

冯·诺依曼类比（Beren Millidge, 2023）：
- 上下文窗口 = 内存（快但有限）
- 外部数据库 = 硬盘（大但慢）
- 工具集成 = 设备驱动
- Harness = 操作系统

三层工程层次：
1. 提示词工程：设计模型指令
2. 上下文工程：管理模型在何时看到什么内容
3. Harness 工程：包含上述两者 + 完整应用架构

TerminalBench 2.0 证据：仅改变 Harness（模型不变），LangChain 排名从 30 名外升至第 5 名。

协同进化原则：模型训练时已考虑 Harness 存在，好的 Harness 在模型升级时自动获益。随着模型增强，Harness 应变薄但不会消失。

MCP（Model Context Protocol）被 Jensen Huang 称为"agentic computer 的开源操作系统"。

## See Also
- [[anatomy-of-agent-harness]]
- [[orchestration-loop]]
- [[context-engineering]]
- [[mcp]]
- [[claude-code]]

## Counter-Arguments and Gaps
- 12 组件框架来自综合多家实践，各家的组件划分并不统一
- "Harness 随模型增强而变薄"的假设尚未被长期验证
