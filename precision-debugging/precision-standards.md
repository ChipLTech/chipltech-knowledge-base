# DLC-Family Accelerator 精度标准

## 适用场景

- 评估 DLC Custom Kernel 的数值精度是否在规定范围内。
- 为不同类别算子选择合适的 tolerance 配置。
- 理解算子组使用的各类误差计算方式和判定逻辑。

## 核心结论

DLC-Family Accelerator 算子精度标准按计算特性分为四类：无计算类（bit-exact）、常规向量类（Elemwise/Binary）、拟合函数类（超越函数）、Reduce 累加类。matmul 系列（含 attention、MoE）使用 RPD-filter2sigma-p95 判定。

整数类型算子默认要求零误差。

## 误差计算方式

### ULP（Unit in the Last Place）

两个浮点数在该精度格式下相隔的可表示浮点格点数。

```
0x3f800000 和 0x3f800001 之间差 1 个 ULP
```

**适用场景**：非 matmul 算子、libdevice（基础算子）、精度要求高的场景。

### atol + rtol

```
abs(actual - expected) <= atol + rtol * abs(expected)
```

**适用场景**：非 matmul 算子，PyTorch 默认使用。

### RPD（Relative Percentage Difference）

```
rel_error = 2 * |actual - expected| / (|actual| + |expected|)
```

**扩展配置**：

**逐元素策略**：
| 策略 | 含义 |
|------|------|
| `all` | 所有元素都必须通过（默认） |
| `p95` | 至少 95% 元素通过 |

**统计策略**：
| 策略 | 含义 |
|------|------|
| `2sigma` | mean + 2 × std < threshold |
| `mean` | 平均 RPD < threshold |
| `filter2sigma` | 先做 median filter，再算 mean + 2 × std（可过滤零星大误差） |

**误差形式**：
| 策略 | 含义 |
|------|------|
| `nopattern` | 错误不能呈现重复 pattern |

**适用场景**：matmul 系列及涉及 matmul 的融合算子（attention、MoE）。

## 算子精度标准分级

### 一、无计算类 — Bit 级一致

**算子**：Reshape, Transpose, Slice, Broadcast, Copy

**场景**：纯显存搬运、步长（Stride）或形状（Shape）变换。

**验收要求**：必须保证输出与 Golden Reference 在 Bit 级别完全一致。

**排错**：出现 1 bit 误差通常意味着搬运踩内存、分批对齐处理错误或越界。

### 二、常规向量类 — Elemwise/Binary

**算子**：Add, Sub, Mul, Div, Relu 等

| 数据类型 | ULP 容忍度 | rtol | atol |
|----------|-----------|------|------|
| Float32 | 4 ULP | 5e-7 | 1e-7 |
| Bfloat16 | 2 ULP | 2 ULP | 1e-5 |

**原理**：
- FP32 的 4 ULP：机器极小值 ~1.19e-7，5e-7 约等于 4 倍极小值误差，结合性能考虑放宽到 4 ULP。
- BF16 的 2 ULP：尾数仅 7 位，纯 fp32 内部计算容易实现 0 ULP，但混合精度计算中不可能保证一致的 bf16 转换和运算步骤。

### 三、拟合函数类 — 超越函数

**算子**：Exp, Log, Sin, Gelu, SiLU 等

| 数据类型 | ULP 容忍度 | rtol | atol |
|----------|-----------|------|------|
| Float32 | 16 ULP | 2e-6 | 1e-6 |
| Bfloat16 | 4 ULP | 4 ULP | 1e-5 |

**原理**：超越函数使用多项式拟合，无法精确实现，允许更大的容差。

### 四、Reduce 累加类

**算子**：Sum, Mean, Softmax（Reduce 与累和计算）

| 数据类型 | ULP 容忍度 | rtol | atol |
|----------|-----------|------|------|
| Float32 | 16 ULP | 2e-6 | 1e-6 |
| Bfloat16 | 4 ULP | 4 ULP | 1e-5 |

**原理**：累和操作存在 "大数吃小数" 现象。`REDUCE_N(N)` 会按 `min(sqrt(N)/4, 128)` 自动缩放阈值。

### 五、matmul 系列

**算子**：matmul, attention, MoE 等涉及 PGX 的算子

**标准**：`matmul(rpd-filter2sigma-p95, rtol=1e-4, atol=1e-5)`

**原理**：matmul 的精度特殊性来自 PGX 的 bf16 转换和 tiling 行为，需要单独标准。以上标准并不是数学严格证明的结果，而是经过与对齐了 DLC 计算逻辑的 reference 进行大量测试后得到的，目标尽可能让误差阈值达到 1e-4 ~ 1e-5 级别。

**常见配置**：
| 配置 | 使用场景 |
|------|---------|
| `("rpd-filter2sigma-p95", 1e-4, 1e-4)` | kernel owner 接受的标准 |
| `("pytorch", 1.6e-2, 1e-5)` | PyTorch 较宽松标准 |

## 速查表

| 算子类别 | 典型算子 | Float32 ULP | BF16 ULP | FP32 rtol | FP32 atol |
|----------|---------|-------------|----------|-----------|-----------|
| 无计算类 | Reshape, Transpose, Copy | 0 | 0 | 0.0 | 0.0 |
| 常规向量类 | Add, Sub, Mul, Relu | 4 | 2 | 5e-7 | 1e-7 |
| 拟合函数类 | Exp, Log, Gelu, SiLU | 16 | 4 | 2e-6 | 1e-6 |
| Reduce 累加类 | Sum, Mean, Softmax | 16 | 4 | 2e-6 | 1e-6 |
| matmul 系列 | matmul, attention, MoE | — | — | 1e-4 (rpd) | 1e-5 |

## 相关资料

- [testing/resultchecker-explained.md](../testing/resultchecker-explained.md) — ResultChecker 判定逻辑
- [precision-debugging/precision-debugging-overview.md](../precision-debugging/precision-debugging-overview.md) — 精度定位方法论
- [precision-debugging/model-site-dump-to-repro.md](../precision-debugging/model-site-dump-to-repro.md) — 从 dump 到复现

## 来源

- `/work/plan/newraw/精度标准.docx`（已转换为 Markdown）
