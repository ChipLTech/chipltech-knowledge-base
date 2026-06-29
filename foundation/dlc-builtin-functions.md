# DLC Custom Kernel Builtin 函数

## 适用场景

- 编写 DLC Custom Kernel 时查阅常用 builtin 函数的参数和用法。
- 理解 DMA、tensor load/store、Print 等基础操作的工作方式。

## 核心结论

DLC Custom Kernel 的 builtin 函数围绕 Explicit DMA Dataflow 设计：DMA 负责 HBM ↔ VMEM 数据搬运，tensor load/store 负责 VMEM ↔ register 的数据访问，Print 用于调试时输出中间结果。

## dlc_dma_new

DMA（Direct Memory Access）是 DLC 数据搬运的核心指令。

### 函数签名

```c
int dlc_dma(tensor, int, tensor, int, int, int, int, int, int);
```

### 参数说明

| 参数 | 含义 | 可选值 |
|------|------|--------|
| src_addr | 源地址 | — |
| src_spc | 源地址类型 | HBM, CMEM, VMEM, SMEM |
| dst_addr | 目标地址 | — |
| des_spc | 目标地址类型 | HBM, CMEM, VMEM, SMEM |
| length | 长度（单位：4 bytes，实为 512 bytes 即 128×4） | 必须是 128 的倍数 |
| src_stride | 每读完一个 unit_length 后跳的长度（单位：4 bytes） | 必须是 128 的倍数 |
| dst_stride | 每写完一个 unit_length 后跳的长度（单位：4 bytes） | 必须是 128 的倍数 |
| sync_flag0 | DMA 完成的同步 flag | 通常用其中一个 |
| sync_flag1 | DMA 完成的同步 flag | 通常用其中一个 |
| unit_len | 单元长度（单位：4 bytes） | **只能填定值，不能用变量**，必须是 128 的倍数 |
| addr_exp | 地址单位指数 | 7 表示 2^7 bytes = 128 bytes |

### 约束

- `length`、`src_stride`、`dst_stride`、`unit_len` 都必须是 128 的倍数（即 512 bytes）。

### 使用示例

```c
char SEMAPHORE_SPACE f1[1];
dlc_dma_new(hbm, HBM, vmem, VMEM, len, src_stride, 128, f1, NULL_SEMAPHORE, 128, 7);
dlc_sync_new(f1);
```

## v_f32_ld_tnsr_st_msk (Tensor Load)

### 参数说明

| 参数 | 含义 |
|------|------|
| offset | 起始 offset，单位 128 bytes |
| tensor | base 地址 |
| stride | 步长，单位 512 bytes |
| mask | 8 位二进制 (0-255)，每一位对应一个 subcore，表示是否从 VMEM load 数据 |

### 行为

每个 subcore 以 `subcore_i * stride + offset + base` 为起始地址，判断 `mask >> subcore_i` 是否为 1，若是则取 512B 数据。

## Print

Print 用于在 kernel 执行时输出调试信息。

### 打印字符串

```c
Print("message");
```

### 打印寄存器

```c
// 两个参数
Print("format:%d\%h\%f\n", sreg/vreg);

// 打印连续 vreg 区域（四个参数）
Print("format:%d\%h\%f\n", vreg, start, length);
// start：起始 offset（32bit 个数）
// length：打印的 32bit 个数
```

格式说明符：
- `%d`：十进制。
- `%h`：十六进制。
- `%f`：浮点。

### 打印 Memory

```c
// 基本用法（四个参数）
Print("format\n", tensor_name, mem_name, len);
// 第二个参数：mem 地址，单位 128 bytes
// 第三个参数：VMEM / SMEM / CMEM / HBM
// 第四个参数：长度，单位 4B

// 高级用法（七个参数）
Print("format\n", tensor_name, mem_name, len, offset, stride, unit_len, n_per_line);
// 地址 = tensor_addr（128B 单位）+ offset（4B 单位）
// stride 和 unit_len：每取 unit_len 要跳 stride，单位 4B
// n_per_line：每打印多少个值换行
```

## Unary 指令精度

Vector 指令中存在 unary 指令（如 exp、log、sin 等），这类指令：
- **硬件 unary**：约 2^(-12) 相对误差，性能较好。
- **libdevice 拟合**：用多项式拟合的基础函数，误差较小，性能较差。

详见精度标准相关文档。

## 相关资料

- [precision-debugging/precision-standards.md](../precision-debugging/precision-standards.md)
- [foundation/dlc-ecosystem-overview.md](../foundation/dlc-ecosystem-overview.md)

## 来源

- `/work/plan/newraw/常用builtin介绍.docx`（已转换为 Markdown）
