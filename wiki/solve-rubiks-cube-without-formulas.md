---
date: 2026-05-10
tags: [mathematics, puzzles]
type: source-summary
source-url: https://philoli.com/zh/blog/solve-rubiks-cube-without-formulas/
---
# 如何不背公式解开魔方：小学生也能看懂

Philo Li 撰写的面向零基础读者的魔方复原教程，核心思路是用群论中的交换子（Commutator）原理替代死记硬背公式。采用 Roux 桥式解法框架，通过"开门—操作—关门"的逻辑，靠观察和理解完成从零到完整复原。

## Key Points

- 魔方是一个非阿贝尔群：操作可组合、可逆、但不可交换（R U ≠ U R）
- 交换子 A B A⁻¹ B⁻¹ 是复原的核心工具，可以在不影响其他块的情况下交换特定块
- 最基础的原子动作：R U R' U'，理解"移开—插入—归位"即可举一反三
- Roux 桥式解法四步：搭建左右桥 → 复原顶层角块朝向（三轮换）→ 调整角块位置（J-perm）→ LSE 复原最后六棱
- 魔方总状态数约 4.3 × 10¹⁹，但"上帝之数"证明任何状态最多 20 步可解
- 角块三轮换公式：R U' L' U R' U' L U 及镜像 L' U R U' L U R' U'
- LSE 步骤仅用 M 和 U 操作，包括 EO（朝向调整）、复原左右棱、复原最后四棱

## Entities Mentioned
- [[rubiks-cube]]
- [[group-theory]]
- [[commutator]]
- [[roux-method]]
- [[ernoe-rubik]]
- [[philo-li]]
- [[cfop]]
- [[speedsolving]]
