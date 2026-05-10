---
date: 2026-05-10
tags: [tools, ai]
type: concept
status: active
---
# Claude Code

Anthropic 的命令行 AI 工具，可直接在终端中执行文件读写、搜索、bash 命令和多步骤工作流。通过插件系统（如 llm-wiki）扩展功能，可在 Obsidian 知识库中作为 AI 协作者运行。

## Details

Claude Code 支持通过 `claude plugin install` 安装社区插件，插件可定义：
- **Skills**：可被调用的操作序列（如 wiki init、compile、query）
- **Hooks**：在特定事件（如会话启动）时自动执行的脚本
- **Commands**：用户可触发的命令

在 Obsidian 中的集成通过 Claudian 插件实现，使 Claude Code 的 working directory 指向 vault 目录。

## See Also
- [[llm-wiki]]
- [[obsidian]]

## Counter-Arguments and Gaps
- 未涉及 Claude Code 的具体 API 限制和成本模型
