# Case Study: Jarvis LayerNorm NaN 到 pytorch_test 复现

## 问题现象

Jarvis spatial-parallel 6 卡训练能初始化和通信，但默认训练 loss 出现 NaN。NaN 最早出现在 forward 中的 `Downsample.norm_layer.normalized`。

## 背景与环境

- **模型**：Jarvis
- **配置**：spatial-parallel 6-card DLC
- **初始表现**：训练能启动、能通信，但 loss NaN

## 定位路径

### 阶段 1：添加 finiteness 检查

在训练的各个阶段添加检查点：
```
input: finite
forward -> LayerNorm -> sub: NaN/Inf  ← 第一个异常
loss: NaN
backward: NaN
grad sync: NaN
optimizer.step: NaN
after_optimizer: NaN
```

### 阶段 2：dump 确认

强制 dump LayerNorm 的输入和中间变量：
- `input - mean` 产生 NaN/Inf
- input 是 Channels-Last Strided Tensor
- CPU 重新计算 `input - mean`：finite

### 阶段 3：算子级描述

```
finite channels-last strided input 与 finite broadcast mean
做 sub 时，DLC Custom_sub_tensor_f32 输出 NaN/Inf。
```

### 阶段 4：移植到 pytorch_test

在 `test_sub_tensor.py` 中添加测试：
- 使用真实 dump 的 channels-last strided tensor 作为输入
- `Custom_sub_tensor_f32` Variant
- 确认在 pytorch_test Framework 中可复现

### 阶段 5：workaround

`JARVIS_SAFE_LAYER_NORM=1` 绕过 DLC LayerNorm sub 路径，继续排查下一个问题（AdamW）。

## 根因或当前结论

DLC `Custom_sub_tensor_f32` kernel 在处理 channels-last strided input + broadcast mean 做 sub 时，输出 NaN/Inf。

## 验证方式

- pytorch_test Framework 中的 `test_sub_tensor.py` 可复现。
- real model dump 和 pytorch_test 复现结果一致。

## 可复用经验

1. **第一个非有限张量是定位的关键**：不要从 loss 找，要从 forward 输入开始。
2. **Channels-Last Strided Tensor 是高风险路径**：非连续 layout 是 DLC Custom Kernel 的常见雷区。
3. **CPU 验证是关键**：同样输入在 CPU 上 finite，确认问题在 DLC 算子而非数据。
4. **workaround 不是修复，是定位工具**：用 workaround 绕过问题后继续排查下一个。
5. **dump 要包含 input、output、shape、stride、dtype**：完整的状态信息才能复现。

## 后续动作

- 交付算子团队 `test_sub_tensor.py` 和最小复现 case。
- 并行排查 AdamW foreach 问题（见 [jarvis-adamw-operator-error.md](jarvis-adamw-operator-error.md)）。

## 来源

- `/work/plan/dlc基础/Jarvis_DLC问题定位到pytorch_test复现流程报告.md`
