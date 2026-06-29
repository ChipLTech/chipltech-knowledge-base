# DLC Custom Kernel 与 CUDA 编程模型对比

## 适用场景

- 从 CUDA 背景转入 DLC-Family Accelerator 开发的新人。
- 需要在 DLC Platform 上设计 custom kernel launch 路径的开发者。
- 需要判断 DLC Runtime Capability Boundary 的场景。

## 核心结论

DLC-Family Accelerator 的编程模型是 **host-side custom kernel launch**，不是 CUDA 的完整 device execution model。DLC 编程的本质是"填写发射表单"：把 tensor、scalar、layout、stride、padding 等元数据打包成 KernelDesc 参数包，然后按 kernel name 发射到 DLCsim 或 Real DLC Hardware 执行。

CUDA 编程则是"直接操控机器"：按 grid/block/thread 层级组织并行，在 kernel 中直接访问 global memory，操作 ABI 参数和指针。

## 三层次心智模型

### 第一层：框架侧算子 API

PyTorch 侧调用 `torch.nn.functional.linear(...)` 或 `a @ b`，触发 ATen dispatch 到 PyTorch DLC Backend。

### 第二层：DLC Custom Kernel Launch 层

框架集成代码用 `KernelDesc` 打包参数，然后调用 `KernelDesc::launch("custom_<kernel_name>")`。
DLC tensor 在此时变成形式化协议对象（DLCTensor: address, dtype, layout, memory_format, stride, padded dim0）。

### 第三层：底层 kernel 执行层

kernel 在 DLCsim 或 Real DLC Hardware 上执行，内部按 explicit DMA dataflow 管理 HBM <-> VMEM，并通过 XYS/PGX/NWS 完成计算。这里描述的是底层执行事实，不表示 DLC Platform 暴露了 CUDA 式完整 device execution model。

DLC Runtime Capability Boundary 的核心判断：host-side custom kernel launch 支持不等于 device-side persistent runtime 支持。像 MPK（micro-batched pipeline kernel）这种需要 device-side worker queue、scheduler、shared-memory paging 的设计，不能在 DLC Platform 上直接假设 CUDA 式 device execution model。

## DLC 硬件架构基础

### 核心计算单元

| 单元 | 功能 |
|------|------|
| XYS (2 per chip) | 标量/向量 ALU，主可编程核心，DLC Vector Type (`float8_128`, `int8_128`, `bool8_128`) |
| PGX | 矩阵乘法，push/pop pipeline，12 级流水 |
| NWS | reduce/permute/transpose |

### 内存层次

```
HBM (大容量片外)
  <- Explicit DMA ->
    VMEM (向量 SRAM，XYS 使用)
    SMEM (标量 SRAM)
    IMEM (指令内存)
    CMEM/FXC (片上 scratchpad)
```

### kernel 编程模型要点

1. kernel entry 是 `main()` 函数，参数通过 KernelDesc 协议传入。
2. 所有数据搬运是显式 DMA，无自动 cache。
3. 计算以 DLC Vector Type 为核心粒度。
4. PGX matmul 走 push/pop pipeline。
5. 多 XYS core 通过 device/core id 分工。

## 算子从 kernel 到 PyTorch 的路径

### dispatch 机制

```text
native_functions.yaml (schema + dispatch key)
  -> aten/src/ATen/native/dlc/<op>.cxx/.cpp/.cc (DLC 后端实现)
  -> KernelDesc input/output/scalar packing
  -> KernelDesc::launch("custom_<kernel>")
  -> DLC_CHECK_RESULT(<lambda_name>, dlc_out)
     // lambda_name != launch kernel name
     // 修改 dispatch 时必须使用 DLC_CHECK_RESULT 第一个参数
```

### DLC_CHECK_RESULT 工作机制

```cpp
DLC_CHECK_RESULT(custom_addmm_rhsT_ROW, dlc_out);
// custom_addmm_rhsT_ROW is the dispatch lambda constant
// Actual launch name may be custom_addmm_rhsT_ROW_BIAS or other
```

DispatchType 控制：
- `DispatchType::DLC` — DLC 路径
- `DispatchType::CPU` — CPU fallback
- `BOTH` — CPU 和 DLC 对比

### KernelDesc 关键方法

| 方法 | 用途 |
|------|------|
| `input(tensor)` | 添加输入 tensor |
| `output(tensor)` | 添加输出 tensor |
| `scalar_as<T>(name, value)` | 添加 full scalar（标量参数） |
| `scalar_lite_as<T>(name, value)` | 添加 lite scalar |
| `launch("custom_<name>")` | 发射 kernel |

### DLC tensor 协议

DLC tensor 是形式化协议对象，包含：
- address（HBM 地址）
- dtype
- layout（含 stride、padded dim0）
- memory_format
- shape（SynShape，维度顺序与 PyTorch 相反）

## Attention Backend 差异

### Backend 选择

Attention backend 由平台决定，不由模型决定：

| 平台 | Backend |
|------|---------|
| DLC Platform | Torch SDPA (`attn_dlc.TorchAttentionBackend`) |
| CUDA (A100/H100) | FlashAttention |
| CUDA (Blackwell) | FlashInfer |
| CUDA (older) | FlexAttention |

强制用 Torch SDPA：`VLLM_ATTENTION_BACKEND=TORCH_SDPA`

### KV Cache 差异

| 平台 | Layout |
|------|--------|
| DLC | Merged KV Cache（K/V 交错，尤其 bf16/head-size-128） |
| CUDA | Separate KV Cache |

### 精度差异原因

不同 attention backend 的实现路径不同：
- Torch SDPA：标准 matmul + softmax
- FlashAttention：tiled online softmax
- 浮点非结合律导致结果不同

详见 [precision-debugging/precision-debugging-overview.md](../precision-debugging/precision-debugging-overview.md)。

## 常见易混淆点

### dispatch lambda name ≠ launch kernel name

```cpp
// enabled_kernels.hpp 中的常量：
const DispatchType custom_matmul_t_pingpong = DispatchType::DLC;

// 但 DLC_CHECK_RESULT 中使用：
DLC_CHECK_RESULT(custom_matmul_t_pingpong, dlc_out);

// 实际 launch：
// KernelDesc::launch("custom_matmul_t_bf16_pingpong")
```

修改 dispatch 时必须使用 `DLC_CHECK_RESULT(custom_matmul_t_pingpong, ...)` 的第一个参数，即 `custom_matmul_t_pingpong`，而不是 launch kernel name `custom_matmul_t_bf16_pingpong`。

### Variant ≠ runtime launch kernel name

pytorch_test Framework 的 Variant 是全局唯一的测试选择器，不一定等于实际 launch 的 kernel name。

### CPU fallback 是定位手段，不是生产修复

用 DispatchType::CPU 做 fallback 来隔离问题是标准定位方法，但不应作为最终的生产修复方案。

## 相关资料

- [CONTEXT.md](../CONTEXT.md) — 完整术语
- [operator-dispatch/enabled-kernels-dispatch.md](../operator-dispatch/enabled-kernels-dispatch.md) — dispatch 操作指南
- [pytorch-dlc-backend/operator-integration-guide.md](../pytorch-dlc-backend/operator-integration-guide.md) — 算子接入指南

## 来源

- `/work/plan/dlc基础/DLC算子实现与CUDA对比报告.md`
- `/work/plan/dlc基础/cuda和dlc生态分析.md`
- `/work/plan/dlc基础/后端不同.md`
