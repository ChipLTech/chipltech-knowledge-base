# vLLM DLC Custom Op 接入和测试

## 适用场景

- 为 vLLM 添加新的 DLC Custom Op。
- 理解 vLLM 侧从 Python API 到 DLC Custom Kernel 的完整接入链路。
- 排查 vLLM DLC 编译或运行问题。

## 核心结论

vLLM DLC Custom Op 通过 PyTorch extension 机制注册，接入链路为：Python wrapper (`vllm/_custom_ops.py`) → C++ binding (`torch_bindings.cpp`) → C++ 实现 (`ops.h` + `<op>.cpp`) → KernelDesc::launch("custom_<kernel>") → DLC_Custom_Kernel Repository。

本链路中的具体文件位置属于 source-version evidence，不是永久布局。开始修改前必须固定 vLLM、vLLM-DLC、PyTorch DLC Backend 和 DLC_Custom_Kernel Repository 的实际 full SHA、packaging mode、import origin 与 native binary identity，再确定当前 owner 是 vLLM in-tree extension、独立 vLLM-DLC plugin 还是 PyTorch DLC Backend。

## 接入链路

下面路径描述历史 in-tree DLC extension 布局。当前 checkout 使用独立 vLLM-DLC plugin 或其他 packaging mode 时，应以实际 registration、build metadata 和 import path 为准，不要创建平行 binding 或强行套用旧目录。

```
Python API: vllm/_custom_ops.py
  -> torch.ops._C.<op_name>(...)

C++ binding: vllm/csrc/dlc/torch_bindings.cpp
  -> TORCH_LIBRARY_IMPL / m.def("op_name", ...)

C++ declaration: vllm/csrc/ops.h
  -> 函数声明

C++ implementation: vllm/csrc/dlc/<op_name>.cpp
  -> KernelDesc::launch("custom_<kernel_name>")

DLC_Custom_Kernel Repository: /work/DLC_Custom_Kernel/dlc_kernels/
  -> 实际 kernel 实现
```

## 操作步骤

### 步骤 1：找到对应的 DLC Custom Kernel

在 `DLC_Custom_Kernel/dlc_kernels/` 中找到 kernel 源码和 syntest。确认 kernel 已通过测试。

同时记录 exact DLC Custom Kernel source revision、kernel name、entry ABI 和 binary digest。源码中存在 kernel 不能单独证明当前运行环境加载了该 binary。

### 步骤 2：参考已有实现

以 `awq_gemm` 为参考模式：

1. `torch_bindings.cpp` — 注册 schema
2. `ops.h` — 函数声明
3. `<op>.cpp` — C++ 实现，用 KernelDesc 打包参数并 launch

### 步骤 3：添加 Python wrapper

在 `vllm/_custom_ops.py` 中添加 Python 接口：

```python
def my_dlc_op(input, ...):
    return torch.ops._C.my_dlc_op(input, ...)
```

### 步骤 4：注册 C++ binding

在 `vllm/csrc/dlc/torch_bindings.cpp` 中注册：

```cpp
m.def("my_dlc_op(Tensor input, ...) -> Tensor");
```

### 步骤 5：实现 C++ 逻辑

在 `vllm/csrc/dlc/<op>.cpp` 中实现：

```cpp
#include "ops.h"
#include <KernelDesc.h>

Tensor my_dlc_op(const Tensor& input, ...) {
    auto out = at::empty_like_or_like(input);
    KernelDesc desc("my_custom_kernel");
    desc.input(input);
    desc.output(out);
    // ... other params
    desc.launch("custom_my_kernel");
    return out;
}
```

### 步骤 6：编译 vLLM

先读取当前 source tree 的 build metadata，确认是 vLLM in-tree device extension 还是独立 vLLM-DLC plugin。只有当前构建系统明确支持 `VLLM_TARGET_DEVICE=dlc` 时才使用：

```bash
VLLM_TARGET_DEVICE=dlc pip install -e .
```

若 core vLLM 使用 `empty` platform 并由 vLLM-DLC plugin 提供 DLC Platform 能力，应按该 plugin 的版本化构建方式安装，不得为复用本命令改变 packaging mode。

### 步骤 7：编写测试

创建测试脚本，包含 CPU reference 实现：

```python
import torch

def cpu_reference(input, ...):
    # CPU 实现
    return result

def test_my_dlc_op():
    x = torch.randn(shape, dtype=torch.bfloat16)
    cpu_out = cpu_reference(x)
    dlc_out = torch.ops._C.my_dlc_op(x.to("dlc"), ...)
    assert torch.allclose(cpu_out, dlc_out.to("cpu"), atol=1e-4)
```

### 步骤 8：运行测试

```bash
python test_dlc/run.py my_custom_kernel
```

## 常见坑

1. **schema 返回值不一致**：`torch_bindings.cpp` 中的 schema 返回值必须和 C++ 函数真实返回一致。
2. **只实现 Python wrapper 不够**：要 C++ declaration、binding、implementation 和 kernel launch 都对齐。
3. **vLLM attention 不能直接用 NVIDIA FlashAttention**：DLC Platform 必须使用 DLC Attention Backend。
4. **DLC Attention Backend 和 CUDA backend 精度对比**：要考虑后端实现差异，不是简单的 equivalency。
5. **把 schema 当成 Kernel ABI**：Public Operator Schema、KernelDesc Descriptor ABI 和 DLC Custom Kernel Entry ABI 的正式定义见 [CONTEXT.md](../CONTEXT.md)；optional 参数为 `None` 不表示 host adapter 没有发送对应 slot。
6. **为冻结 DLC Custom Kernel 收窄通用 schema**：优先保持 caller contract，在 host adapter 中做 exact ABI adaptation，并对未证明能力 fail closed。
7. **混用 source 与 binary identity**：repository HEAD、installed package、native extension 和 DLC Custom Kernel binary 必须分别记录。

## 验证方法

1. 确认 kernel syntest 已通过。
2. 在 Python 侧调用 `torch.ops._C.<op_name>()` 验证 binding 正确。
3. 用 CPU reference 对比 DLC 输出。
4. 用 `DLC_SYN_DEBUG=1` 确认 kernel launch 路径正确。
5. 对照 exact DLC Custom Kernel Entry ABI 检查 descriptor slot 顺序、kind 和 optional metadata 行为。
6. 对 public schema、dispatch、descriptor 和 kernel binary 分别记录 identity；任何执行相关 binary 变化后重跑对应 C1b、collective 和模型 Gate。

## 相关资料

- [CONTEXT.md](../CONTEXT.md) — vLLM DLC Custom Op、DLC Attention Backend 定义
- [pytorch-dlc-backend/operator-integration-guide.md](../pytorch-dlc-backend/operator-integration-guide.md)
- [model-adaptation-and-main-to-main-decisions.md](model-adaptation-and-main-to-main-decisions.md)

## 来源

- `/work/plan/dlc基础/vllm DLC算子添加及测试方法.md`
- `/work/plan/dlc基础/DLC基础知识手册.md` vLLM DLC Custom Op 接入速记部分
