# LLVM 19 下嵌套位运算的表达式临时量

本地复现 `a & b & (c | d) & e` 被输出成 `(a & b & c | d) & e`。
32组布尔输入中6组错误，可能绕过动态Tile的容量条件。主仓问题：
[PTOAS #1648](https://github.com/hw-native-sys/PTOAS/issues/1648)。

兼容表达式形成pass沿用三元表达式的处理方式，将包含位或、位异或、
逻辑与/或的producer保留为普通SSA临时量，避免旧C++ emitter错误展平。
临时量不使用volatile；后续C++优化器仍可正常优化。没有修改共享LLVM。

独立构建后的生成C++通过32项布尔真值表与625项i32位运算数值检查，
以及新增FileCheck。原有三项gather回归通过，设备Top-K helper的
59组输入在三次同址变输入图重放中精确通过。两个重点CSA规格随后完成
全量输出、状态、保护区与图回放对照；其性能收益属于算子候选整体，
不能单独归因于本兼容修复。

这不是所有EmitC表达式或LLVM版本的完整验证。根本解决应在配套MLIR
C++ emitter正确维护嵌套表达式precedence；本地保守隔离作为兼容措施。
