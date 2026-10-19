# CSA 单 SPMD 配套 PTOAS（2026-10-19）

本分支 `dsv4-csa-spmd-repro-20261019` 固定 CSA 当前实测编译器对应的源码。
版本基础是 PTOAS 0.66；实际身份由 Git 提交号决定，不能以 CLI 版本字符串
代替源码记录。完整安装说明、三仓版本清单及验证记录见
[CSA 复现目录](https://github.com/nalinaly/vllm-ascend/tree/dsv4-flash-pto-v0.25.1rc1/tests/pypto_test/csa_toolchain_repro_20261019)。

## 源码组合

- 基于 `b7b4a82c9623f54c2e782b929c2adbcc82fc4fec`。
- 保留 Acc 连续列窗口、INT8 分形 L1 窗口和 TEXTRACT 读范围分析。
- `lib/PTO/Transforms/InsertSync/MemoryDependentAnalyzer.cpp` 恢复为
  `a5e0a766^` 的版本，与原实测 `toolchain_v3` 的单文件覆盖一致。
  未采用的 GM 列分片分析和 F32 条件变体不包含在复现源码中。
- `materializeControlFlowOperands` 在边赋值前物化 BlockArgument，
  保护链式传递、交换、循环轮换的并行赋值语义，对应上游 issue #1622。
- 当 `PTO_ENABLE_PYTHON_BINDING=OFF` 时提供 native CLI，
  直接调用同一个 `mlir::pto::runPTOAS` 驱动和 lowering 流水。
  没有新增另一套编译器实现。

这些修改仅发布到用户 fork 的复现分支。相关缺陷仍通过上游 issue 跟踪，
本次不创建上游 PR，也不修改未采用试验所在的原工作区。

## 从源码构建

使用 LLVM/MLIR 19；本次配套源码来自
`https://github.com/vpto-dev/llvm-project.git`，
提交 `fa2fd1f75d742fec91ecf40d98c9c7961647ec42`。
LLVM 构建使用 `BUILD_SHARED_LIBS=ON`、`LLVM_ENABLE_ASSERTIONS=ON`、
`MLIR_ENABLE_BINDINGS_PYTHON=OFF`、Release，Host 编译器为 GCC 15.2。

```bash
export LLVM_BUILD_DIR=/your/llvm-build
cmake -G Ninja -S . -B build -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_DIR="$LLVM_BUILD_DIR/lib/cmake/llvm" \
  -DMLIR_DIR="$LLVM_BUILD_DIR/lib/cmake/mlir" \
  -DPTO_ENABLE_PYTHON_BINDING=OFF
cmake --build build --target ptoas-native --parallel 2
export PTOAS_ROOT="$PWD/build"
export LD_LIBRARY_PATH="$LLVM_BUILD_DIR/lib:${LD_LIBRARY_PATH:-}"
export PATH="$PTOAS_ROOT/bin:$PATH"
ptoas --version
```

CANN cmake 使用 `master-055`，对应提交
`a7ef9a7f04ea9efc5ad388cdb8cce0b0c1759c70`。
GitCode 无法访问时，可预先准备该源码，在 CMake 中传入
`-DCANN_3RD_LIB_PATH=/your/dependency-parent`，使其下有
`cann-cmake/function/prepare.cmake`。

本分支使用 `build/bin/ptoas`；不需要安装到系统目录。共享库仍来自指定
LLVM 构建，因此运行时需保留上述 `LD_LIBRARY_PATH`。
复现应从源码构建，不依赖原机器的混合目标文件归档。

