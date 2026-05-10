---
date: 2026-05-10
tags: [mathematics, abstract-algebra]
type: concept
status: active
---
# Group Theory（群论）

研究"群"这一代数结构的数学分支。群是配备一个满足结合律的二元运算、有单位元、每个元素有逆元的集合。如果运算还满足交换律，则称为阿贝尔群（如整数加法）；不满足则为非阿贝尔群。

## Details

群论与魔方的关系：魔方的每次转动是一个置换操作，所有合法转动序列构成一个非阿贝尔群（Rubik's Group）。这意味着 A B 和 B A 的效果不同——先 R 后 U 和先 U 后 R 产生不同的魔方状态。

理解群论能帮助理解为什么交换子（Commutator）A B A⁻¹ B⁻¹ 能实现"只改变目标块而不影响环境"的效果，这是不背公式复原魔方的数学基础。

群论在其他领域的应用包括：
- 对称性研究（物理、化学中的分子对称性）
- 密码学（有限域上的椭圆曲线群）
- 伽罗瓦理论（多项式方程的可解性）

## See Also
- [[commutator]]
- [[rubiks-cube]]
- [[solve-rubiks-cube-without-formulas]]

## Counter-Arguments and Gaps
- 未深入讨论群的公理化定义（封闭性、结合律、单位元、逆元）的严格证明
- 未涉及置换群的循环表示和轨道-稳定化子定理
