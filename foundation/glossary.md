# 术语速查表

本文档是 [CONTEXT.md](../CONTEXT.md) 的速查索引，不独立维护术语定义。完整定义、禁用叫法和组件关系以 [CONTEXT.md](../CONTEXT.md) 为准。

## 硬件家族

| 术语 | 定义 |
|------|------|
| DLC-Family Accelerator | 公司 AI 加速器产品线：DLC Chip、TYD Chip、HHP Chip |
| DLC Chip | 第一代芯片，大脸猫/刹那 |
| TYD Chip | 第二代芯片，修复 errata |
| HHP Chip | 第三代芯片，设计中 |
| Real DLC Hardware | 物理硬件（区别于 DLCsim） |

## 硬件单元

| 术语 | 定义 |
|------|------|
| XYS | 标量/向量计算核心，每芯片 2 个 |
| PGX | 矩阵乘法单元 |
| NWS | reduce/permute/transpose 单元 |
| FXC / CMEM | 片上 scratchpad / SRAM |
| LYP | 片间互联 |
| HBM | 片外高带宽内存 |
| VMEM | 向量 SRAM |
| SMEM | 标量 SRAM |
| IMEM | 指令内存 |
| DLC Vector Type | 如 `float8_128`、`int8_128` |
| Explicit DMA Dataflow | 显式 HBM <-> VMEM 搬运 |
| Push/Pop Pipeline | 硬件 push/pop 计算模型 |

## 平台与运行时

| 术语 | 定义 |
|------|------|
| DLC Ecosystem | 完整硬件+软件环境 |
| DLC Platform | 框架可见的执行目标 |
| PyTorch DLC Backend | ATen 算子 dispatch 到 DLC |
| vLLM DLC Custom Op | vLLM 框架的 DLC tensor 操作 |
| DLC Custom Op | 框架可见的 DLC tensor 操作 |
| DLC Custom Kernel | 编译为 DLC 执行的命名底层 kernel |
| DLC_Custom_Kernel Repository | kernel 源码+测试+注册元数据+产物的仓库 |
| KernelDesc | host 侧参数打包对象 |
| DLC Kernel Launch Protocol | 按 kernel name 发射的协议 |
| DLCSynapse | kernel 编译/执行框架组件 |
| DLC Runtime | launch/stream/event 执行 API |
| DLCsim | 模拟器 |
| dlc-thunk | 用户态到 kernel driver 桥接 |
| DLCCL | 集合通信库（类 NCCL） |
| DLC_CL | 前置支持库 |

## 测试与正确性

| 术语 | 定义 |
|------|------|
| CPU Reference | CPU 计算的期望结果 |
| Hardware-Aware Reference | 模拟 DLC 硬件行为的参考计算 |
| DLC Precision Difference | 预期的 DLC/CPU/CUDA 数值差异 |
| DLC_CHECK_RESULT | PyTorch DLC 验证宏 |
| DispatchType | CPU-only / DLC-only / BOTH |
| pytorch_test Framework | DLC_Custom_Kernel PyTorch 测试框架 |
| Variant | 全局唯一的测试选择器 |
| Static Shape Test | 固定 shape 回归测试 |
| Dynamic Fuzz Test | 随机输入/shape 测试 |
| SynShape | kernel 侧 shape（维度与 PyTorch 相反） |
| Model-Site Dump | 模型 failure tensor snapshot |
| Lazy Dump | 低扰动 Model-Site Dump |
| Finite Mask Mismatch | CPU/DLC NaN/Inf 位置不一致 |
| Channels-Last Strided Tensor | 高风险非连续 tensor layout |
| Foreach / Multi-Tensor Path | 在 tensor 列表上操作的融合路径 |

## Attention 与运行时

| 术语 | 定义 |
|------|------|
| DLC Attention Backend | DLC attention 执行路径 |
| Merged KV Cache | DLC K/V 交错存储 layout |
| Separate KV Cache | CUDA K/V 分离存储 layout |
| DLC Runtime Capability Boundary | host-side launch vs device-side runtime 能力分界 |

## 关系速记

```
DLC-Family Accelerator > {DLC Chip, TYD Chip, HHP Chip}
DLC Ecosystem > {DLC Platform, PyTorch DLC Backend, vLLM DLC Custom Op,
                 DLC_Custom_Kernel Repository, DLCSynapse, DLC Runtime,
                 DLCsim, dlc-thunk, DLCCL, DLC_CL, Real DLC Hardware}
PyTorch DLC Backend/vLLM DLC Custom Op -> KernelDesc -> DLC Kernel Launch Protocol
DLCSynapse -> DLCsim | Real DLC Hardware
Explicit DMA Dataflow: HBM -> VMEM -> XYS/PGX/NWS -> VMEM -> HBM
```
