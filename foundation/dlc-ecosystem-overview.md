# DLC Ecosystem 概述

## 适用场景

初次接触 DLC-Family Accelerator 的开发者、需要建立 DLC Ecosystem 整体心智模型的新人、或需要快速恢复项目上下文的情况。

## 核心结论

DLC-Family Accelerator 的开发主路径不是 CUDA 那种完整暴露 GPU device execution model 的编程平台，而更像 **host-side custom kernel launch 生态**：上层框架把 tensor、scalar、layout、stride、padding 等元数据打包成 `KernelDesc` 参数包，再经 DLCSynapse/DLC Runtime 按 kernel name 发射到 DLCsim 或 Real DLC Hardware。

## 核心链路

```
vLLM / SGLang / PyTorch
  -> PyTorch DLC Backend / vLLM DLC Custom Op
  -> KernelDesc argument packing
  -> DLCSynapse / DLC Runtime launch
  -> DLC Custom Kernel binary
  -> DLCsim / Real DLC Hardware
```

## 芯片代际

| 代际 | 名称 | 备注 |
|------|------|------|
| 第一代 | DLC Chip | 内部代号大脸猫/刹那，主要硬件目标 |
| 第二代 | TYD Chip | 架构延续，修复 errata，matmul/DMA/RDMA/unary 精度提升 |
| 第三代 | HHP Chip | 设计阶段 |

## 硬件模块

### 计算单元

| 模块 | 角色 | 备注 |
|------|------|------|
| **XYS** | 标量/向量 ALU，主计算核心 | 每芯片通常 2 个，kernel 内通过 device/core id 分工 |
| **PGX** | 矩阵乘法单元 | 即使 fp32 输入，乘法输入可能先转 bf16 |
| **NWS** | reduce、permute、transpose | |

### 存储层次

| 模块 | 角色 |
|------|------|
| **HBM** | 大容量片外高带宽内存 |
| **FXC / CMEM** | 片上 scratchpad / SRAM |
| **VMEM** | 向量 SRAM，XYS 使用 |
| **SMEM** | 标量 SRAM |
| **IMEM** | 指令内存 |

### 互联

| 模块 | 角色 |
|------|------|
| **LYP** | 片间互联，多卡/RDMA 路径 |

### 典型数据流

```
HBM --DMA--> VMEM --XYS/PGX/NWS compute--> VMEM --DMA--> HBM
```

### 关键差异 vs CUDA/GPU

- DLC-Family Accelerator kernel 需要显式管理 HBM 与 VMEM 之间的数据搬运（Explicit DMA Dataflow）。
- 计算以 `float8_128`、`int8_128`、`bool8_128` 等 packed vector type 为核心粒度。
- matmul、exp、reduce 常见 push/pop pipeline。
- VMEM 实际可用量要考虑 compiler stack、数组和 spill 占用。

## 软件栈

### 推荐理解顺序

```
LLVM
  -> dlc-thunk
  -> DLCsim
  -> DLCSynapse
  -> DLC_Custom_Kernel Repository
  -> DLC_CL
  -> PyTorch DLC Backend
  -> vLLM / SGLang DLC integration
```

### 模块职责

| 组件 | 角色 |
|------|------|
| **PyTorch DLC Backend** | ATen 算子 dispatch 到 DLC 实现 |
| **vLLM DLC Custom Op** | vLLM 侧通过 torch extension 暴露 DLC op |
| **DLC_Custom_Kernel Repository** | kernel 源码、syntests、注册元数据和二进制产物 |
| **DLCSynapse** | kernel 编译/执行框架组件 |
| **DLC Runtime** | launch、stream/event、异步错误等执行 API 表面 |
| **DLCsim** | 模拟器执行目标 |
| **dlc-thunk** | 用户态到 kernel driver 的桥接层 |
| **LLVM** | kernel 编译器基础设施 |
| **DLCCL** | 集合通信库（类 NCCL），AllReduce/Broadcast/Reduce/AllGather/ReduceScatter |
| **DLC_CL** | PyTorch 等组件的前置支持库 |

### 当前真实仓库快照

以下为 `/work` 下当前代码仓库的结构事实，适合做新 session 的路径入口：

| 仓库 | 当前分支 | 关键入口 |
|------|----------|----------|
| `/work/DLC_CL` | `inc_nsteps` | `README.md`、`CLAUDE.md`、`build.sh`；定位为 inter-DLC communication optimized primitives，依赖 `DLC_Custom_Kernel` 和 `DLCSynapse` |
| `/work/DLC_Custom_Kernel` | `develop` | `README.md`、`AGENT.md`、`doc/context.md`；核心目录包括 `dlc_kernels/`、`dlc_src/`、`syntests/`、`python/`、`pytorch_test/` |
| `/work/DLCSynapse` | `main` | `README.md`、`compile.sh`；核心目录包括 `synapse_core/`、`synapse_backend/`、`dlc_runtime/`、`dlc_thunk/`、`synapse_perf/`、`tests/` |
| `/work/pytorch` | `release_25` | 标准 PyTorch `README.md`、`DLC_TORCH_ENV.md`、`setup.py`；DLC 实现位于 `aten/src/ATen/native/dlc/` |

PyTorch DLC Backend 额外事实：
- `aten/src/ATen/native/dlc/` 存在，C++ 实现文件混用 `.cxx`、`.cpp`、`.cc`。
- `aten/src/ATen/native/native_functions.yaml` 中 DLC dispatch 使用 `dispatch:` 下的 `DLC: <impl>`，也存在 `CPU, CUDA, DLC: <impl>` 这类复合 key。
- `test/dlc_ops/test_dlc_ops.py` 是 PyTorch 原生 DLC 算子测试入口；`test_dlc/` 另有自定义 `dlctester` 测试体系。
- `enabled_kernels.hpp` 源文件位于 `/work/DLC_Custom_Kernel/dlc_src/enabled_kernels.hpp`，安装后常见路径为 `/usr/local/chipltech/synapse/include/enabled_kernels.hpp`。

### vLLM

PagedAttention 高吞吐 LLM 推理引擎。在 DLC Platform 上通过 vLLM DLC Custom Op 运行。

### SGLang

RadixAttention 结构化解码框架。

### 版本对齐

`undefined symbol`、PyTorch import 失败、vLLM 编译失败、peek 参数异常等问题经常来自仓库版本不匹配。详见 [environment-setup-and-update.md](../runtime-debugging/environment-setup-and-update.md)。

## DLC 与 CUDA 的核心差异

| 维度 | DLC-Family Accelerator | CUDA |
|------|----------------------|------|
| 并行模型 | 少量 XYS core 处理大向量块 | 大量 thread/block/warp |
| 内存访问 | 显式 DMA，HBM <-> VMEM | thread 直接访问 global memory |
| launch 语义 | KernelDesc 打包参数，按 kernel name launch | `kernel<<<grid, block, smem, stream>>>(args...)` |
| 参数形式 | tensor/scalar/layout/stride/padding 协议对象 | ABI 参数和指针为主 |
| matmul | PGX push/pop pipeline | Tensor Core / WMMA / MMA |
| 调试/验证 | DLC_CHECK_RESULT 可 CPU/DLC 对比 | 通常无内置 CPU 对比 |
| runtime 能力 | host-side custom launch 为主 | 完整 device execution model 更成熟 |

判断问题时不要套 CUDA thread/block 思维。很多 bug 不是数学逻辑错误，而是参数协议不一致：tensor/scalar 分组、scalar lite/full、shape/stride/layout、dtype、output tuple 数量、kernel name 注册等。

## 精度特征

### 常见差异来源

- PGX 虽支持 fp32 输入，但乘法输入可能先转换为 bf16。
- bf16 转换方式、硬件 exp/rsqrt、tiling 和累加顺序会造成预期差异。
- Attention 差异通常最大，因为后端算法、分块、KV cache layout 和硬件实现都可能不同。
- CPU/CUDA reference 如果不模拟 DLC-Family Accelerator 硬件行为，可能把预期差异误判为 bug。

### 判断建议

1. 先确认是否属于 DLC Precision Difference。
2. matmul/attention 类问题优先考虑 Hardware-Aware Reference。
3. 有 NaN/Inf 位置差异时，优先看 Finite Mask Mismatch，而不是只看 max error。

## 相关资料

- [CONTEXT.md](../CONTEXT.md) — 完整术语表
- [foundation/glossary.md](glossary.md) — 术语快速查阅
- [foundation/dlc-vs-cuda-comparison.md](dlc-vs-cuda-comparison.md) — DLC 与 CUDA 详细对比
- [foundation/hardware-specifications.md](hardware-specifications.md) — 硬件参数表，避免在本文重复维护规格细节
- [pytorch-dlc-backend/operator-integration-guide.md](../pytorch-dlc-backend/operator-integration-guide.md) — 算子接入指南
- [precision-debugging/precision-debugging-overview.md](../precision-debugging/precision-debugging-overview.md) — 精度定位方法论

## 来源

- `/work/plan/dlc基础/DLC基础知识手册.md`
- `/work/plan/dlc基础/TPU基础架构.md`
- `/work/plan/dlc基础/DLC各个概念介绍.md`
