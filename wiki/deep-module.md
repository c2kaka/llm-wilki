---
date: 2026-05-11
tags: [software-engineering, architecture]
type: concept
status: active
---
# Deep Module (深模块)
来自 John Ousterhout《软件设计哲学》。深模块接口简单、内部封装大量功能；浅模块接口复杂、功能少、大量小文件相互依赖。

## Details
深模块对 AI 编程尤为重要：
- AI 只需理解接口，不需钻进每个细节
- 更容易测试：在模块外部建测试边界，从接口层面验证行为
- 代码库可理解性高，AI 不易在依赖图中迷失

Matt Pocock 的实践：将浏览器端视频编辑器的前后端完整流程包进一个大模块，从外部统一测试，AI 表现"翻天覆地的改变"。

对比：
- **浅模块**：接口复杂，功能少，大量小文件依赖 → AI 迷失
- **深模块**：接口简单，封装丰富 → AI 只需理解接口

建议的设计原则：设计接口，委托实现（Design the interface, delegate the implementation）。

## See Also
- [[software-fundamentals-in-ai-era]]
- [[matt-pocock]]
- [[tdd]]

## Counter-Arguments and Gaps
- 深模块与浅模块的边界判断依赖经验，缺乏客观度量标准
