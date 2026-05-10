---
date: 2026-05-10
tags: [puzzles, speedsolving]
type: concept
status: active
---
# Roux Method（桥式解法）

由 Gilles Roux 发明的魔方复原方法。与层先法不同，桥式解法先在左右两侧各搭建一个 1×2×3 的方块（称为"桥"），再处理顶层和剩余位置。步数少、灵活度高，且需要的公式记忆量少，因为核心逻辑就是交换子。

## Details

四步流程：
1. **搭建左桥**（1×2×3 方块）：白红棱块归位 → 蓝红棱块归位 → 前方两个红色角块归位
2. **搭建右桥**（对称操作）：将红色换为橙色，注意不破坏左桥
3. **复原顶层角块**：
   - 角块三轮换调整朝向（R U' L' U R' U' L U），让四个黄色朝上
   - J-perm 变体（R U2 R' U' R U2 L' U R' U' L）交换角块位置使侧边颜色对齐
4. **LSE（Last Six Edges）**：仅用 M 和 U 操作
   - EO（Edge Orientation）：通过 M' U M 翻转坏棱
   - 复原左右棱（红黄、橙黄）
   - 复原最后四棱（蓝、绿）：M' U2 M 三棱换

优势：观察位置固定，不需要频繁转动魔方整体；状态混乱时可用大量基础置换快速降熵。

## See Also
- [[rubiks-cube]]
- [[commutator]]
- [[cfop]]
- [[speedsolving]]
- [[solve-rubiks-cube-without-formulas]]

## Counter-Arguments and Gaps
- 未详细对比 Roux 与其他主流解法的平均步数和竞速表现
- 未讨论桥式解法的高级优化技巧（如同时搭建左右桥）
