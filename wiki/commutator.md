---
date: 2026-05-10
tags: [mathematics, abstract-algebra]
type: concept
status: active
---
# Commutator（交换子）

群论中的核心概念，写作 **A B A⁻¹ B⁻¹**，其中 A⁻¹ 和 B⁻¹ 分别是 A 和 B 的逆操作。交换子衡量两个操作的"不可交换程度"——如果 A 和 B 可交换，则交换子恒等于恒等操作（什么都没做）。

## Details

在魔方中，交换子是实现"只改变特定块而不影响其他块"的关键工具。生活化比喻——电梯：
1. **A**：人走进电梯
2. **B**：电梯上到 3 楼
3. **A⁻¹**：人走出电梯
4. **B⁻¹**：电梯回到 1 楼

结果：电梯回到原位（环境复原），但人从 1 楼换到了 3 楼（目标移动）。关键在于 B⁻¹ 执行时，人已经不在电梯里了。

魔方中最基础的原子交换动作：**R U R' U'**，即"移开—插入—归位—复位"。可以组合为更长的序列实现不同效果，如角块三轮换 R U' L' U R' U' L U。

不能用 A A⁻¹ B B⁻¹ 替代，因为每个操作直接被逆操作抵消，等于什么都没做。

## See Also
- [[group-theory]]
- [[rubiks-cube]]
- [[roux-method]]
- [[solve-rubiks-cube-without-formulas]]

## Counter-Arguments and Gaps
- 未讨论高阶交换子（嵌套交换子）及其在更复杂置换中的应用
- 未涉及交换子群（由所有交换子生成的子群）的数学性质
