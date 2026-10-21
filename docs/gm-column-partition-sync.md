# GM列分片同步分析

二维ND、相同基指针与元素类型、相同静态2次幂正行距、单位内stride，且两个
64B对齐的固定列窗口在行内不相交时，不因GM根相同而增加barrier。
行起点可以动态；未知列/stride/指针、真实重叠、非ND或嵌套视图保留原分析。
实现位于MemoryDependentAnalyzer.cpp，回归位于
`test/samples/Sync/test_gm_column_partition_sync.pto`。这是优化保守依赖，
没有修改数学或公共IR/CLI接口，ODS及Python API无需变更。

完整CSA的非同步C++逐行不变，FIX/MTE3各减少4/2个静态barrier位置；
不能把静态数量等同于执行次数或性能收益，设备验证另行记录。
