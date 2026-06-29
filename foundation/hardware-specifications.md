# 硬件规格参考

## 适用场景

- 需要快速查阅 DLC-Family Accelerator 各代芯片的硬件参数。
- 理解芯片代际之间的差异和提升点。
- 了解硬件模块代号和理论性能数据。

## 核心结论

DLC-Family Accelerator 三代芯片（DLC Chip / TYD Chip / HHP Chip）共享相近的设计脉络，但具体存储模块、性能指标和可用能力以当前代际资料为准；表中为空或标记为“无”的项目不应按前代推断。

## 硬件模块命名

| 代号 | 内部名称 | 中文代号 | 功能 |
|------|---------|---------|------|
| DLC | 大脸猫（大脸cat） | 刹那 | 第一代芯片 |
| TYD | 田园狗（田园dog） | — | 第二代芯片 |
| HHP | 花花熊猫（花花panda） | — | 第三代芯片（设计中） |
| FXC | 伏羲琴（Fuxi Cord） | — | Scratchpad memory / SRAM |
| XYS | 轩辕剑（Xuanyuan Sword） | — | Scalar + Vector ALU |
| NWS | 女娲石（Nuwa Stone） | — | Permute / Transpose / Reduce |
| PGX | 盘古斧（Pangu Axe） | — | Matmul 单元 |
| LYP | 炼妖壶（Lianyao Hu） | — | 片间互联 |
| DHB | 东皇钟（Donghuang Bell） | — | 稀疏操作（4B 颗粒度） |
| KLM | 昆仑镜（Kunlun Mirror） | — | 片上网络 |
| KTS | 崆峒印（Kongtong Seal） | — | Power-on sequence |
| HTT | 昊天塔（Haotian Tower） | — | Host interface |

## 芯片代际参数对比

| 参数 | DLC Chip | TYD Chip | HHP Chip |
|------|----------|----------|----------|
| HBM | 64 GB | 64 GB | — |
| CMEM | 32 MB | 32 MB | 无 |
| VMEM | 16 MB | 16 MB | — |
| SMEM | 1 MB | 1 MB | — |
| IMEM | 16k 条指令 | 16k 条指令 | — |
| XYS 数量 | 2 | 2 | — |
| 每 XYS 中 PGX 数 | 2 | 2 | — |
| 每 XYS 中 NWS 数 | 2 | 2 | — |
| HBM→VMEM 带宽 | 800+ GB/s | 预计优于 DLC | — |
| fp32/bf16 TFLOPS | 204T (实测 fp32:174T, bf16:184T) | 204T (实测 195T) | — |
| int8 TOPS | 396T (实测 368T) | 396T (实测 391T) | — |
| VREG 大小 | 8×128×32B | 8×128×32B | — |
| SREG 大小 | 32B | 32B | — |
| 多卡通讯 | 64 GB/s 单线双向 / 256 GB/s 单卡 / 1 TB/s 8卡 | 128 GB/s 单线双向 / 512 GB/s 单卡 / 2 TB/s 8卡 | — |

## TYD Chip 相对 DLC Chip 的提升

1. **matmul 性能提升**：修复了 matmul hold issue bug。
2. **稳定性提升**：修复了 DMA bit 翻转（与数据和 HBM 频率相关）、片内网络问题（与同时在跑的 DMA 条数相关）。
3. **DMA 带宽优化**：overhead 减少 100 cycle。
4. **多卡通讯优化**：RDMA 速度理论翻倍，LYP 连线质量提升，**不需要打 garbage**。
5. **unary 指令精度优化**：部分 unary 指令精度提升。

## DLC-Family Accelerator 精度特点

- **PGX bf16 转换**：PGX 支持 fp32 输入，但所有乘法输入要求 bf16，硬件会做 `fp32→bf16 (round-tie-to-even)` 转换。这导致很多算子的精度与 CPU/GPU reference 不一致。
- **unary 指令误差**：约 2^-12 的相对误差。
- **subnormal 处理**：subnormal（指数位全 0）当作 0 处理。

## 编译器内存管理

- VMEM 中存在 vmem stack（算子开的数组和寄存器溢栈），从大往小使用。
- 实际 VMEM 可用大小以 `mem->vmem_size` 为准（已减去编译器占用部分）。

## 相关资料

- [foundation/dlc-ecosystem-overview.md](dlc-ecosystem-overview.md) — DLC Ecosystem 概述
- [foundation/dlc-vs-cuda-comparison.md](dlc-vs-cuda-comparison.md) — DLC 与 CUDA 对比
- [precision-debugging/precision-standards.md](../precision-debugging/precision-standards.md) — 精度标准

## 来源

- `/work/plan/newraw/TPU基础架构-2.docx`（已转换为 Markdown）
- `/work/plan/dlc基础/TPU基础架构.md`
