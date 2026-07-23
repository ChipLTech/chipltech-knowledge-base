# Chipltech-Family Accelerator 术语和规则

本文档是全仓库的领域词汇入口。所有专题文档应引用本文档的术语，不重复维护术语定义。

## 核心术语

### 硬件家族

**Chipltech-Family Accelerator**：公司 AI 加速器产品线，包括 DLC Chip、TYD Chip 和 HHP Chip。用于指代整个硬件家族。
禁止使用：`TPU`、`国产 TPU`。

**DLC Chip**：第一代 Chipltech-Family Accelerator，内部代号大脸猫 / 刹那。裸 `DLC` 仅在本上下文明确指第一代芯片时使用。
禁止使用：用 `DLC` 泛指平台、生态、后端或运行时。

**TYD Chip**：第二代 Chipltech-Family Accelerator，架构基本延续 DLC Chip，修复 errata 并提升 matmul、DMA、RDMA 和 unary 精度。

**HHP Chip**：第三代 Chipltech-Family Accelerator，设计中。

**Real DLC Hardware**：物理 Chipltech-Family Accelerator 硬件，与 DLCsim 相对。

### 硬件单元

**XYS**：标量/向量计算核心，主可编程计算单元，Chipltech-Family Accelerator 通常每芯片含 2 个 XYS core。
禁止使用：`CUDA core`、`GPU core`。

**PGX**：矩阵乘法单元，用于 matmul 和 attention/GEMM。即使输入为 fp32，乘法输入可能先转换为 bf16，是 DLC Precision Difference 的主要来源。
禁止使用：`Tensor Core`（除非对比 CUDA）。

**NWS**：reduce、permute、transpose 类操作单元。

**FXC / CMEM**：片上 scratchpad / SRAM，大小因芯片代际和资料版本而异。

**LYP**：片间互联，用于多卡和 RDMA 路径。
禁止使用：`network`（指 DLC 片间互联时）。

**HBM**：大容量片外高带宽内存。

**VMEM**：向量 SRAM，XYS 计算使用，需从 HBM 显式搬运数据。

**SMEM**：标量 SRAM，用于标量侧数据和运行时/编译器支持。

**IMEM**：指令内存，存储 Chipltech-Family Accelerator 可执行指令流。

**DLC Vector Type**：packed vector 计算类型，如 `float8_128`、`int8_128`、`bool8_128`。

**Explicit DMA Dataflow**：Chipltech-Family Accelerator 的显式数据搬运模型，HBM <-> VMEM。
禁止使用：`直接访问全局内存`。

**Push/Pop Pipeline**：硬件执行模型，数据推入 PGX 或 reduction/elementwise pipeline，结果从硬件队列弹出。

### 平台与运行时

**DLC Ecosystem**：Chipltech-Family Accelerator 的完整硬件和软件环境，包括 PyTorch DLC Backend、vLLM DLC Custom Op、DLC_Custom_Kernel Repository、DLCSynapse、DLC Runtime、DLCsim、dlc-thunk、DLCCL、DLC_CL 和 Real DLC Hardware。
禁止使用：裸 `DLC` 指代生态。

**DLC Platform**：框架可见的执行目标，PyTorch、vLLM、SGLang 等在 Chipltech-Family Accelerator 上运行 tensor 或 DLC Custom Op 时选择的平台。

**PyTorch DLC Backend**：将 ATen 算子 dispatch 到 DLC 实现的 PyTorch 后端，通常使用 KernelDesc + DLCSynapse 发射 DLC Custom Kernel。

**vLLM DLC Custom Op**：vLLM 框架中通过 PyTorch extension 机制注册的 DLC tensor 操作，实现打包参数并发射 DLC Custom Kernel。
禁止使用：`custom kernel` 指框架可见的 op。

**DLC Custom Op**：框架可见（PyTorch / vLLM / SGLang）的 DLC tensor 操作，不等于 DLC Custom Kernel。

**DLC Custom Kernel**：编译为 Chipltech-Family Accelerator 执行的命名底层 kernel，通过 kernel name 和 packed metadata 经运行时发射。
禁止使用：`custom op`、`repository`。

**DLC_Custom_Kernel Repository**：包含 DLC Custom Kernel 源码、kernel 测试、注册元数据和编译产物的仓库/库。
禁止使用：`DLC Custom Kernel` 指仓库。

**KernelDesc**：host 侧参数打包对象，框架集成用它编码 tensor、output、scalar、format 和操作元数据，然后按名称发射 DLC Custom Kernel。

**DLC Kernel Launch Protocol**：平台定义的 host 侧协议，打包 tensor/scalar 元数据并通过运行时发射命名 kernel，不暴露 CUDA 式 device kernel ABI。

**DLCSynapse**：核心 DLC 框架组件，编译和/或执行 DLC Custom Kernel，将发射请求路由到 DLCsim 或 Real DLC Hardware。
禁止使用：裸 `Synapse`。

**DLC Runtime**：框架集成和 DLCSynapse 使用的执行 API/运行时表面，负责 kernel launch、stream/event 和异步错误报告。
禁止使用：未限定的 `runtime`。

**DLCsim**：Chipltech-Family Accelerator 模拟器，在不使用真实硬件时执行和建模 kernel 行为。

**dlc-thunk**：用户态到 DLC kernel driver 的桥接层。
禁止使用：`driver` 指用户态桥接层时。

**DLCCL**：类 NCCL 的 DLC 集合通信库，用于 AllReduce、Broadcast、Reduce、AllGather、ReduceScatter 等多卡操作。
禁止使用：`NCCL` 指 DLC 集合通信时。

**DLC_CL**：PyTorch 等 DLC Ecosystem 组件的支持库。
禁止使用：`OpenCL`（除非明确讨论 OpenCL）。

### 测试与正确性

**CPU Reference**：CPU 计算的期望结果，用于 DLC 测试或 DLC_CHECK_RESULT 验证 DLC 输出。

**Hardware-Aware Reference**：有意模拟 Chipltech-Family Accelerator 硬件行为的参考计算，例如 fp32-to-bf16 转换、近似 unary 指令或已知累加顺序差异。
禁止使用：`精确 CPU reference` 当硬件行为有意不同时。

**DLC Precision Difference**：Chipltech-Family Accelerator 与 CPU/CUDA 之间由于 bf16 转换、硬件 exp/rsqrt/matmul、tiling、累加顺序等原因造成的预期数值差异。
禁止使用：`precision bug` 指预期差异时。

**DLC_CHECK_RESULT**：PyTorch DLC 验证宏，可运行 CPU 和 DLC 路径、比较输出，调试时可选择 fallback 或用 CPU 结果覆盖。

**DispatchType**：控制 DLC 算子是 CPU-only、DLC-only 还是 CPU/DLC 双路径比较的 dispatch 模式。

**pytorch_test Framework**：DLC_Custom_Kernel Repository 中运行 CPU 和 DLC 执行并比较结果的 PyTorch 测试框架。
禁止使用：`PyTorch tests` 特指此框架时。

**Variant**：pytorch_test Framework 中全局唯一的测试选择器，将测例绑定到 DLC Custom Kernel 路径。Variant 不一定等同于 runtime launch kernel name。

**Static Shape Test**：pytorch_test Framework 中的固定 shape 回归测试。

**Dynamic Fuzz Test**：pytorch_test Framework 中随机输入或 shape 的测试，用于发现边界、layout 和 dtype 问题。

**SynShape**：kernel/测试侧 shape 表示，维度顺序通常与 PyTorch shape 相反。

**Model-Site Dump**：从真实模型 failure 中保存的 tensor snapshot、元数据和执行状态 `.pt` 文件，用于后续算子级复现。

**Lazy Dump**：低扰动的 Model-Site Dump 策略，仅在检测到 failure 后写完整状态，降低同步/拷贝遮盖异步 bug 的可能性。

**Finite Mask Mismatch**：CPU 和 DLC 在 NaN/Inf 位置上不一致的正确性 failure，通常表现为 `torch.isfinite(...)` 结果不匹配。

**Channels-Last Strided Tensor**：具有 channels-last 风格 stride 的非连续 tensor layout，是 DLC Custom Kernel 的高风险正确性路径。

**Foreach / Multi-Tensor Path**：在 tensor 列表上操作的融合 optimizer/算子路径，正确性可能依赖完整的 tensor list 排序和分组。

### Attention 与运行时能力

**DLC Attention Backend**：DLC tensor 选择的 vLLM/PyTorch attention 执行路径，使用 DLC 专用 SDPA 或 attention kernel，而非 NVIDIA FlashAttention kernel。

**Merged KV Cache**：DLC attention layout，K 和 V cache 条目可能在同一个 tensor 中交错存储，尤其是 bf16/head-size-128 场景。

**Separate KV Cache**：CUDA 风格 layout，K 和 V cache tensor 分别存储。

**Prefill/Decode Separation**：将 prompt prefill 与 token decode 分配到独立 serving role 的 vLLM deployment topology。不得仅因两个 role 均 ready 就声称端到端 separation 已验证。

**Prefill Worker**：执行请求 prefill，并产出供后续 decode 使用的 KV Cache 与 metadata 的 serving role。

**Decode Worker**：消费与 request identity、model identity 和 cache contract 匹配的 KV Cache，并执行后续 token decode 的 serving role。

**KV Cache Transfer Contract**：Prefill Worker 与 Decode Worker 之间关于 request correlation、layer/block/token ownership、cache layout/dtype、metadata、传输完成和失败语义的显式 contract。不得把 connector handshake 或进程存活等同于 KV transfer 已完成。

**Transport Qualification Gate**：在加载两个 serving role 前，用与目标部署相同的 transport、设备可见性和进程边界完成非空 payload 的 endpoint 初始化、send/receive completion 与内容校验。进程退出、延迟样本或单端初始化不构成通过。

**Site Recovery Contract**：Host maintenance 前固定的现场恢复 contract，包含既有 workload 的 process/container、设备、HBM/频率、端口、模型、完整启动命令、健康探针和最小功能请求。维护完成后必须逐项恢复并验证。

**SMI Observation Envelope**：由官方 `cltech_smi` raw output、工具/source identity、physical/logical device mapping，以及 `before_launch`、`after_ready`、`during_request`、`after_cleanup` 四阶段 normalized observation 组成的 query-only 运行证据。它用于定位设备、HBM、进程和 cleanup 边界，不替代 C1b、模型正确性或维护授权。

**DLC Runtime Capability Boundary**：host 侧 custom kernel launch 支持与 CUDA 式 device 侧持久运行时控制之间的能力分界。

**Verified vLLM Alignment**：一个经过全部强制回归验证，并由可审计 evidence 证明的 vLLM commit 与 vllm-dlc revision 组合。候选 commit、当前 checkout、安装版本或 README 记录只能作为恢复线索，不能称为 Verified vLLM Alignment。
禁止使用：`alignment` 指未经强制回归验证的推测组合。

## 高效术语速查表

| 禁用叫法 | 应使用 |
|----------|--------|
| `TPU` | Chipltech-Family Accelerator |
| `国产 TPU` | Chipltech-Family Accelerator |
| `DLC`（指平台/生态/后端） | DLC Platform / DLC Ecosystem / PyTorch DLC Backend |
| `DLC Custom Kernel`（指仓库） | DLC_Custom_Kernel Repository |
| `custom kernel`（指框架 op） | DLC Custom Op / vLLM DLC Custom Op |
| `Synapse` | DLCSynapse |
| `runtime` | DLC Runtime |
| `NCCL`（指 DLC 通信） | DLCCL |
| `CUDA core` | XYS |
| `Tensor Core`（除非对比） | PGX |
| `precision bug`（预期差异） | DLC Precision Difference |
| `PyTorch tests`（特指框架时） | pytorch_test Framework |
| `NHWC`（除非精确 layout） | Channels-Last Strided Tensor |
| `driver`（用户态桥接） | dlc-thunk |

## 组件关系

- **Chipltech-Family Accelerator** 包含 **DLC Chip**、**TYD Chip** 和 **HHP Chip**。
- **DLC Ecosystem** 包含 **DLC Platform**、**PyTorch DLC Backend**、**vLLM DLC Custom Op**、**DLC_Custom_Kernel Repository**、**DLCSynapse**、**DLC Runtime**、**DLCsim**、**dlc-thunk**、**DLCCL**、**DLC_CL** 和 **Real DLC Hardware**。
- **PyTorch DLC Backend** 和 **vLLM DLC Custom Op** 都使用 **KernelDesc** 遵循 **DLC Kernel Launch Protocol**。
- **KernelDesc** 打包 tensor、output、scalar、format 和操作元数据后按名称发射 **DLC Custom Kernel**。
- **DLCSynapse** 消费发射请求，将执行分发到 **DLCsim** 或 **Real DLC Hardware**。
- **DLC_Custom_Kernel Repository** 包含源码、kernel 测试、注册元数据和实现 **DLC Custom Kernel** 的产物。
- **Explicit DMA Dataflow** 从 **HBM** 搬运到 **VMEM**，经 **XYS**、**PGX** 或 **NWS** 计算后搬回 **HBM**。
- **PGX**、近似 unary 指令、bf16 转换、tiling 和累加顺序是 **DLC Precision Difference** 的主要来源。
- **DLC_CHECK_RESULT** 使用 **CPU Reference** 在 **PyTorch DLC Backend** 中验证 **DLC Custom Kernel** 结果。
- **pytorch_test Framework** 使用 **Variant**、**Static Shape Test**、**Dynamic Fuzz Test**、**SynShape** 和 **Model-Site Dump** 发现和最小化正确性问题。
- **DLC Attention Backend** 可能使用 **Merged KV Cache**，CUDA 路径使用 **Separate KV Cache**。
- **Prefill/Decode Separation** 要求 **Prefill Worker** 与 **Decode Worker** 的 model、tokenizer、cache layout、connector 和 request correlation identity 满足同一个 **KV Cache Transfer Contract**。
- **Transport Qualification Gate** 在加载双 role 模型前验证实际 data plane；**Site Recovery Contract** 约束会改变 Host 状态的 LYP/driver/firmware/reboot 操作及其收尾。
- **SMI Observation Envelope** 为模型验证、镜像交付、PD 分离、环境修复和 runtime debug 提供统一的 query-only device/process/HBM evidence seam。

## 核心链路

```
vLLM / SGLang / PyTorch
  -> PyTorch DLC Backend / vLLM DLC Custom Op
  -> KernelDesc argument packing
  -> DLCSynapse / DLC Runtime launch
  -> DLC Custom Kernel binary
  -> DLCsim / Real DLC Hardware
```

## Agent 读取规则

1. 新 session 必须先读取本文件。
2. 使用正式术语，避免禁用叫法。
3. 不要假设 CUDA thread/block/device execution model 适用于 Chipltech-Family Accelerator。
4. Chipltech-Family Accelerator 开发的主路径是 host-side custom kernel launch 生态，不是 CUDA device execution model。

## 文档写作规则

- 文件名和目录名使用英文 kebab-case。
- 正文主要使用中文，英文技术术语保留原名。
- 每篇专题文档建议包含：适用场景、核心结论、操作步骤、常见坑、相关资料。
- Case study 建议包含：问题现象、背景与环境、定位路径、最小边界、根因或当前结论、验证方式、可复用经验。
- Agent context 文档应短小精悍，适合作为新 session 入口。
- 所有文档应区分事实、经验、建议和未验证假设。
- 不要新建立以单个模型名作为一级或二级长期目录的结构。
