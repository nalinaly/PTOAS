# L0C完整列窗口的同步范围分析

InsertSync可以对普通FP32/INT32 L0C父Tile的部分列窗口使用精确物理范围。
仅当父shape静态、覆盖全部行、列起点和宽度按16对齐且通过边界检查时启用。
布局须为ColMajor/RowMajor、fractal=1024、CompactMode=Null；其字节起点为
`colOffset * parentRows * 4`，长度为`colSize * parentRows * 4`。

这允许同一父池的互不重叠窗口分别供FIX写回和Cube计算；真实重叠仍需同步。
部分行、动态offset、compact及其它布局保持原保守父范围。不能把NoneBox
的单列长度直接套给完整boxed列窗口，否则会漏掉后续列上的真实冲突。

该改进不新增ODS、CLI或Python接口，不修改算子数学和核间信号。
回归用例位于`test/samples/Sync/test_acc_column_subview_sync.pto`，覆盖
不相交、相同/部分重叠、不同alloc同地址、嵌套窗口及动态offset回退。
当前设备验证针对A3动态CSA；CPU通过本身不等于真机精度或性能通过。
