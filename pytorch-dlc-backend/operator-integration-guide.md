# PyTorch DLC Backend 算子接入指南

## 适用场景

- 为 PyTorch DLC Backend 接入新的 DLC Custom Kernel。
- 理解 ATen op dispatch 到 DLC Custom Kernel 的完整路径。
- 排查算子接入失败问题。

## 核心结论

PyTorch DLC Backend 算子接入的本质是：将 PyTorch ATen 算子通过 `native_functions.yaml` 的 DLC dispatch key 路由到 `aten/src/ATen/native/dlc/` 中的 C++ 实现，用 `KernelDesc` 打包 tensor/scalar 参数，调用 `KernelDesc::launch("custom_<name>")` 发射 DLC Custom Kernel，最后用 `DLC_CHECK_RESULT` 验证结果。

## 前提条件

接入前确保：
- DLC Custom Kernel 已在 kernel 层实现并通过 syntest。
- kernel name 已在注册元数据中启用。
- PyTorch schema、dispatch、C++ 签名与 generated header 对齐。

## 操作流程

### 步骤 1：修改 native_functions.yaml

在 `aten/src/ATen/native/native_functions.yaml` 中找到目标算子 schema，确认或添加 DLC dispatch key。

三种常见模式：

**非 inplace 算子**：
```yaml
- func: op_name(Tensor self, ...) -> Tensor
  dispatch:
    DLC: dlc_op_name
```

**inplace 算子**：
```yaml
- func: op_name_(Tensor(a!) self, ...) -> Tensor(a!)
  dispatch:
    DLC: dlc_op_name_
```

**out 变体**：
```yaml
- func: op_name.out(Tensor self, ...) -> Tensor
  dispatch:
    DLC: dlc_op_name_out
```

**autogen / structured_delegate**：
```yaml
- func: op_name(Tensor self, ...) -> Tensor
  structured_delegate: op_name.out
  dispatch:
    DLC: dlc_op_name
```

### 步骤 2：参考已有 dispatch

查找同一目录下已有 CPU/CUDA/MPS 实现作为模板。通常位于：
- `aten/src/ATen/native/cpu/<op>.cpp`
- `aten/src/ATen/native/cuda/<op>.cu`

### 步骤 3：编写 DLC 后端实现

在 `aten/src/ATen/native/dlc/` 中编写实现。当前真实目录混用 `.cxx`、`.cpp`、`.cc`，新增文件应参考相邻算子的命名和 CMake 收集方式，而不是假设只能使用 `.cxx`。

**基本结构**：

```cpp
#include <ATen/native/dlc/DLCOps.h>
#include <ATen/native/dlc/KernelDesc.h>

namespace at { namespace native {

Tensor dlc_op_name(const Tensor& self, ...) {
    // 创建输出 tensor
    auto out = at::empty_like_or_like(self);

    // 构建 KernelDesc
    KernelDesc desc("custom_<kernel>");
    desc.input(self);
    desc.output(out);
    desc.scalar_as<int64_t>("dim", dim);
    // ... 更多参数

    // 发射 kernel
    desc.launch("custom_<kernel>");

    return out;
}

}} // namespace at::native
```

### 步骤 4：使用 DLC_CHECK_RESULT 验证

```cpp
DLC_CHECK_RESULT(custom_addmm, dlc_out);
```

**关键**：`DLC_CHECK_RESULT` 的第一个参数（lambda name）对应 `enabled_kernels.hpp` 中的 dispatch 常量，与 launch kernel name 可能不同。

### 步骤 5：处理 dim 参数

PyTorch shape 与 SynShape/kernel 维度顺序相反，dim 参数需要转换：

```cpp
auto dim = self.dim() - 1 - original_dim;
```

### 步骤 6：编译

```bash
cd /work/pytorch
USE_CUDA=0 DEBUG=1 MAX_JOBS=32 python3 setup.py develop
```

`/work/pytorch/build.sh` 当前也使用上述 develop 构建命令。如需构建带发布版本号的 wheel，再按发布流程设置 `PYTORCH_BUILD_VERSION` / `PYTORCH_BUILD_NUMBER`。

或增量编译：
```bash
cd /work/pytorch
ninja -j100
```

## 常见检查点

1. `native_functions.yaml` schema 是否已有，dispatch 是否添加 DLC。
2. structured op 是否需要 meta function 和 names propagation。
3. PyTorch shape 与 SynShape/kernel 维度顺序是否相反。
4. `dim` 是否需要负数规范化和反向转换。
5. C++ schema 里的 `float`、`Int` 是否对应 `double`、`int64_t` 等真实类型。
6. bool、bf16、int64 是否按 kernel 预期传入。
7. `DLC_CHECK_RESULT` 的 DLC output tuple 与 CPU reference tuple 数量和形状是否一致。
8. scalar 是 full 还是 lite 是否匹配。
9. dtype 是否符合 kernel 预期。
10. shape/stride/layout 是否符合 kernel 预期。

## 算子接入失败排查顺序

1. kernel 层 syntest 是否通过。
2. kernel name 是否在注册文件中启用。
3. PyTorch schema 是否正确。
4. C++ 实现签名是否与 generated native header 一致。
5. `KernelDesc` input/output/scalar 顺序是否与 kernel main 参数一致。
6. scalar 是 full 还是 lite 是否匹配。
7. dtype 是否符合 kernel 预期。
8. shape/stride/layout 是否符合 kernel 预期。
9. dim 是否需要反向转换。
10. output tuple 和 CPU reference tuple 是否形状/数量一致。
11. 是否需要 CPU fallback 或明确报错。
12. 是否被 DispatchType 配置影响。

## 常见坑

1. **schema 签名不匹配**：C++ 函数签名必须与 `native_functions.yaml` 中生成的 header 完全一致。
2. **dispatch key 缺失**：如果 `native_functions.yaml` 没有 DLC dispatch，需要添加。
3. **SynShape 反向**：`SynShape([96, 96, 768, 1])` 对应 PyTorch shape `(1, 768, 96, 96)`。
4. **KernelDesc 参数顺序**：input/output/scalar 顺序必须与 kernel `main()` 参数顺序一致。
5. **编译不生效**：`python setup.py develop` 可能不检测新文件，尝试 `ninja -j100` 或 clean build。

## 验证方法

1. 运行 PyTorch DLC 原生测试：`python test/dlc_ops/test_dlc_ops.py`
2. 运行 pytorch_test Framework 对应测例：`python -m pytest pytorch_test/test_kernel/<module>/test_<op>.py`
3. 用 `DLC_SYN_DEBUG=1` 确认 kernel launch 路径正确。

## 相关资料

- [testing/pytorch-test-framework-guide.md](../testing/pytorch-test-framework-guide.md)
- [testing/dlc-kernel-test-framework-guide.md](../testing/dlc-kernel-test-framework-guide.md)
- [operator-dispatch/enabled-kernels-dispatch.md](../operator-dispatch/enabled-kernels-dispatch.md)
- [foundation/dlc-vs-cuda-comparison.md](../foundation/dlc-vs-cuda-comparison.md)

## 来源

- `/work/plan/dlc基础/pytorch算子插入.md`
- `/work/plan/dlc基础/DLC基础知识手册.md`
