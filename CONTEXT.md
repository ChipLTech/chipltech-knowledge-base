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

**DLC Ecosystem**：Chipltech-Family Accelerator 的完整硬件和软件环境，包括 PyTorch DLC Backend、vLLM-CL Custom Op、DLC_Custom_Kernel Repository、DLCSynapse、DLC Runtime、DLCsim、dlc-thunk、DLCCL、DLC_CL 和 Real DLC Hardware。
禁止使用：裸 `DLC` 指代生态。

**DLC Platform**：框架可见的执行目标，PyTorch、vLLM、SGLang 等在 Chipltech-Family Accelerator 上运行 tensor 或 DLC Custom Op 时选择的平台。

**PyTorch DLC Backend**：将 ATen 算子 dispatch 到 DLC 实现的 PyTorch 后端，通常使用 KernelDesc + DLCSynapse 发射 DLC Custom Kernel。

**vLLM-CL Custom Op**：vLLM 框架中通过 PyTorch extension 机制注册的 DLC tensor 操作，实现打包参数并发射 DLC Custom Kernel。
禁止使用：`custom kernel` 指框架可见的 op。

**vLLM-CL Repository**：Chipltech 维护的 vLLM hardware plugin 仓库，正式 Git 仓库名和 Python Distribution 均为 `vllm-cl`，正式 Git URL 为 `git@github.com:ChipLTech/vllm-cl.git`，Python import package 为 `vllm_cl`，native extension 为 `vllm_cl.vllm_cl_C`。新命令、路径、环境变量、artifact identity 和文档必须使用这些名称。改名前的 `vllm-dlc` / `vllm_dlc` 只允许出现在明确标注为 historical/legacy 的原始证据或兼容说明中，不代表另一个组件。

**DLC Custom Op**：框架可见（PyTorch / vLLM / SGLang）的 DLC tensor 操作，不等于 DLC Custom Kernel。

**DLC Custom Kernel**：编译为 Chipltech-Family Accelerator 执行的命名底层 kernel，通过 kernel name 和 packed metadata 经运行时发射。
禁止使用：`custom op`、`repository`。

**DLC_Custom_Kernel Repository**：包含 DLC Custom Kernel 源码、kernel 测试、注册元数据和编译产物的仓库/库。
禁止使用：`DLC Custom Kernel` 指仓库。

**KernelDesc**：host 侧参数打包对象，框架集成用它编码 tensor、output、scalar、format 和操作元数据，然后按名称发射 DLC Custom Kernel。

**Public Operator Schema**：caller 可见的 DLC Custom Op 参数、返回值、mutation/alias 和 dispatch contract。它不等于 host 侧实际发送的 descriptor，也不证明底层 DLC Custom Kernel 支持 schema 可表达的全部组合。

**KernelDesc Descriptor ABI**：KernelDesc host adapter 实际发送给 DLC Custom Kernel 的有序 tensor、output、scalar 和 metadata slots。optional public 参数为 `None` 不自动证明对应 descriptor slot 不存在。

**DLC Custom Kernel Entry ABI**：exact DLC Custom Kernel binary 按顺序读取的 entry 参数 contract。它必须与 KernelDesc Descriptor ABI 和 binary identity 配对验证。

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

**Collective Selection Contract**：由已初始化 communicator 的实际 LYP topology、payload、dtype/layout、rank/root metadata 和候选实现能力共同决定 collective strategy 的 launch 前 contract。framework 和 DLC Custom Kernel 消费其稳定 strategy/metadata，不按 world size 重建 topology 或 lookup table。

**Verified Collective Fallback**：首选 collective 实现不满足 payload、alignment、root 或其他能力前提时，对候选 fallback 的 graph、channel、rank order、rank range、唯一性和 metadata 重新完成验证后才能下发的降级路径。候选标记不等于验证通过；无可验证实现时 fail closed。

**Evidence Ledger**：将报告输入按运行已证明、源码已存在、参考观察、推断和未知分类，并把每个对外结论绑定到来源、身份和 Claim Boundary 的事实账本。

**Technical Delivery Summary**：面向 review、日报、Sprint 或跨团队同步的一句话交付能力总结，以交付状态、对象、新增行为、关键依据或条件和可感知结果构成能力主干；不把跨仓 plumbing 或更弱证据压缩升级为更强交付状态。

**Decision Summary**：面向指定读者和决策问题的独立结论产品，只保留主判断、最少支撑事实和边界；完整身份、调用链、失败证据和验证计划下沉到 Technical Attachment。

**Technical Attachment**：供技术人员复核 Decision Summary 的详细材料，保存 exact identity、模型结构、量化参数、调用条件、失败边界、原始 evidence、性能口径和后续验证项。

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

**Verified vLLM Alignment**：一个经过全部强制回归验证，并由可审计 evidence 证明的 vLLM commit 与 vllm-cl revision 组合。候选 commit、当前 checkout、安装版本或 README 记录只能作为恢复线索，不能称为 Verified vLLM Alignment。
禁止使用：`alignment` 指未经强制回归验证的推测组合。

**Tested Revision**：实际产生 build/runtime evidence 的 exact source revision、dirty state 和 artifact graph。验证期间使用的 merge history 或 working checkout 属于该 identity，不自动等于最终 PR commit。

**Publication Candidate**：从当前目标 main 的隔离 clean worktree 构造、仅包含批准净差异的待发布 revision。它必须显式关联 Tested Revision、base SHA、scoped diff/tree identity 和回归结果；单独的 commit count 或 patch-id 不建立 runtime acceptance。

**Patch Equivalence**：Tested Revision 与 Publication Candidate 在声明 path/scope 内净差异等价的证据。stable patch-id 可作为 supporting evidence，但不能替代 base identity、完整 scoped diff/tree、artifact rebuild 或受影响回归。

**Static Stack Compatible**：exact immutable image 与 Host stack 的只读身份匹配通过的状态，绑定 Driver/Runtime API、四文件 CRT、DLC Custom Kernel library 和 LLVM 完整身份到一个显式批准或撤销的 policy profile。它只证明 artifact 身份，不代表 Real DLC Hardware execution。
**Cold First-Compute Completion Ready**：在 fresh process 中完成 allocation、H2D、真实 device operation、synchronize、D2H 和 exact correctness 后得到的有界基础执行状态。它发现静态 policy 尚未认识的兼容性问题，但不能外推模型功能或 benchmark 稳定性。
**Stack Preflight**：不修改运行库的独立启动前资格检查，由 Static Stack Compatible 与 Cold First-Compute Completion Ready 两个不可互相替代的 checkpoint 组成；任一失败即阻止模型加载或镜像发布。

## 高效术语速查表

| 禁用叫法 | 应使用 |
|----------|--------|
| `TPU` | Chipltech-Family Accelerator |
| `国产 TPU` | Chipltech-Family Accelerator |
| `DLC`（指平台/生态/后端） | DLC Platform / DLC Ecosystem / PyTorch DLC Backend |
| `DLC Custom Kernel`（指仓库） | DLC_Custom_Kernel Repository |
| `vllm-dlc` / `vllm_dlc`（当前仓库、Distribution 或 import） | `vllm-cl` / `vllm_cl`；仅历史原始证据保留旧名 |
| `custom kernel`（指框架 op） | DLC Custom Op / vLLM-CL Custom Op |
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
- **DLC Ecosystem** 包含 **DLC Platform**、**PyTorch DLC Backend**、**vLLM-CL Custom Op**、**DLC_Custom_Kernel Repository**、**DLCSynapse**、**DLC Runtime**、**DLCsim**、**dlc-thunk**、**DLCCL**、**DLC_CL** 和 **Real DLC Hardware**。
- **PyTorch DLC Backend** 和 **vLLM-CL Custom Op** 都使用 **KernelDesc** 遵循 **DLC Kernel Launch Protocol**。
- **KernelDesc** 打包 tensor、output、scalar、format 和操作元数据后按名称发射 **DLC Custom Kernel**。
- **Public Operator Schema**、**KernelDesc Descriptor ABI** 和 **DLC Custom Kernel Entry ABI** 是三层独立 contract，共同约束 **DLC Kernel Launch Protocol**。
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
- **Collective Selection Contract** 在 communicator 侧拥有 topology/payload-aware selection；**Verified Collective Fallback** 在 launch 前消化正常能力边界，DLC Custom Kernel 只按稳定 strategy dispatch 并防御 descriptor ABI 损坏。
- **Tested Revision** 绑定实际执行证据；**Publication Candidate** 绑定最新目标 main 上的交付表示；二者只能通过声明范围内的 **Patch Equivalence** 和重跑门禁建立可审计关系。
- **Evidence Ledger** 约束事实、推断和未知不混写；**Decision Summary** 回答读者决策问题；**Technical Attachment** 保存可复核细节，三者不能把摘要可读性换成证据越界。
- **Technical Delivery Summary** 把已完成技术工作压缩为一个能力主张；source implemented、integrated、validated 和 released 必须按实际 Evidence 分开表达。

## 核心链路

```
vLLM / SGLang / PyTorch
  -> PyTorch DLC Backend / vLLM-CL Custom Op
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
