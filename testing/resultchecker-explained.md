# ResultChecker 工作原理

## 适用场景

- 理解 pytorch_test Framework 中 ResultChecker 的误差计算和判定逻辑。
- 解读测试结果中的误差指标含义。
- 判断 FAIL 结果是否属于可接受范围。

## 核心结论

ResultChecker 使用 RPD（Relative Percentage Difference）作为核心误差度量（值域 [0, 2]），结合中值滤波和统计推断来判断测试 PASS/FAIL。

## 误差计算方式

### 相对误差公式

```
rel_error = 2 * |A - B| / (|A| + |B|)
```

值域：[0, 2]。

### 特殊值处理

- 两个值都是 NaN（不分正负）：误差为 0。
- 两个值都是 Inf（分正负）：误差为 0。
- 两个值都等于 0.0：误差为 0。

## 判定流程

### 1. 异常值检查

如果**任何一个**元素的相对误差为 NaN 或 Inf，直接 FAIL。

### 2. 全部通过阈值检查

如果所有元素的相对误差都 ≤ 阈值，直接 PASS。

### 3. 平均值检查

计算相对误差的平均值。如果平均值 > 阈值，直接 FAIL。

### 4. 中值滤波 + 95% 置信区间推断

- 使用 kernel=3 的中值滤波器过滤相对误差数据。
- 分别计算过滤前和过滤后数据的平均值和标准差。
- 假设相对误差符合正态分布，用过滤后数据的 `mean + 2*std` 估计 95% 误差分布范围。
- 如果该范围 ≤ 阈值：PASS。
- 如果过滤前标准差 ≥ 过滤后标准差 × 1.75：提出警告（认为过滤过程过滤掉了有效错误值）。

这个步骤的目的是：检测有规则的连续错误但平均误差未超标的情况。注意：二维数据中纵向分布的连续错误可能无法被检测。

### 5. 输出误差波动位置

如果第 4 步 FAIL，会基于过滤后数据中两两相邻值的差异是否达到阈值 × 2 来判断误差波动位置，输出所有符合条件的下标。

## 常见 tolerance 配置

| 配置 | 含义 |
|------|------|
| `("pytorch", 1.6e-2, 1e-5)` | PyTorch 默认 tolerance |
| `("rpd-filter2sigma-p95", 1e-4, 1e-4)` | matmul/attention 类常用标准 |
| `atol=1e-6` | 过严标准，通常被 owner 拒绝 |

rpd-filter2sigma-p95 的含义：
- `rpd`：使用 RPD 误差计算
- `filter2sigma`：中值滤波 + 2-sigma 置信区间
- `p95`：至少 95% 元素通过

## 相关资料

- [precision-debugging/precision-standards.md](../precision-debugging/precision-standards.md) — 完整精度标准
- [testing/dlc-kernel-test-framework-guide.md](../testing/dlc-kernel-test-framework-guide.md) — 测试结果解读

## 来源

- `/work/plan/newraw/ResultChecker 做了什么.docx`（已转换为 Markdown）
