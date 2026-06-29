# Case Study: Paraformer QKV Linear 精度问题到 pytorch_test 复现

## 问题现象

Paraformer 语音识别模型在 DLC Platform 上推理时，DLC 输出与 CPU 参考输出存在 token 级差异：重复、替换。最终确认第一个显著差异在 `encoder.encoders0.0.self_attn.linear_q_k_v` (Linear, shape [1, 3542, 1536], float32)。

## 背景与环境

- **模型**：Paraformer (FunASR)
- **对比方式**：CPU oracle vs DLC
- **初始对比结果**：997/997 tensors 对齐，无 shape mismatch

## 定位路径

### 阶段 1：模型级 dump 定位第一个 divergent module

用 dump 工具捕捉模型全部 tensor 输出，生成 compare report。第一个显著差异：
| Module | Class | abs_mean | rtol_mean | cosine_sim |
|--------|-------|----------|-----------|------------|
| encoder.encoders0.0.self_attn.linear_q_k_v | Linear | ~1e-5 | ~1e-5 | 0.9999 |

### 阶段 2：确认输入不是问题

捕捉 Linear 的 input、weight、bias。确认：
- input tensor CPU vs DLC exact
- weight 参数 exact
- 问题在 `F.linear` 本身

### 阶段 3：单算子 replay

最小 replay：
```python
F.linear(input, weight, bias)  # CPU vs DLC
```
结果：可复现。

### 阶段 4：确认真实 kernel 路径

用 verbose trace 确认 DLC trace：
```
aten: custom_addmm_rhsT_ROW_BIAS
launch custom_addmm_rhsT_ROW_BIAS
```
DLC_CHECK_RESULT lambda: `custom_addmm_rhsT_ROW`

### 阶段 5：移植到 pytorch_test

在 `test_addmm.py` 中增加对应的 Static Shape Test，覆盖真实 shape `[1, 3542, 1536]`。

### 阶段 6：收敛到最小边界

缩小 shape：
- `[3542, 768]` x `[1536, 768]` → 可复现
- `[128, 768]` x `[1536, 768]` → 可复现
- `[896, 768]` x `[1536, 768]` → 可复现

### 阶段 7：code-only repro

最终的 code-only repro 不需要任何外部文件：
```python
# 固定 seed，随机生成 input 和 weight
# 调用 F.linear
# CPU vs DLC 比较
```

## 根因或当前结论

问题在 `custom_addmm_rhsT_ROW_BIAS` kernel（DLC matmul 路径），特定 shape 和 tiling 下产生 DLC Precision Difference。

## 验证方式

- code-only repro 可复现
- 交付算子团队的最小 shape 可复现
- 收敛到不需要 .pt dump 文件的自包含测试

## 可复用经验

1. **模型级 dump 是关键**：997/997 tensors 对齐看起来很健康，但细微的差异会在后续层中被放大。
2. **先确认输入是否一致**：input/weight 不一致时 blame 模型路径，一致时 blame 算子。
3. **收敛到 code-only**：最终交付给算子团队的必须是自包含测试，不依赖真实模型 dump 文件。
4. **pytorch_test 是正确交付形式**：不是 Python 脚本，不是模型路径，而是 pytorch_test Framework 中的测例。
5. **synShape 反向**：pytorch_test 中要注意维度顺序。
6. **不要假设所有差异都是 bug**：DLC Precision Difference 是预期行为，需要判断是否在合理范围内。
7. **记录 kernel 路径**：`custom_addmm_rhsT_ROW_BIAS` 的 DLC_CHECK_RESULT lambda 是 `custom_addmm_rhsT_ROW`。

## 后续动作

交付算子团队后跟踪修复进展。

## 来源

- `/work/plan/dlc基础/Paraformer_DLC_CPU精度差异定位报告.md`
- `/work/plan/dlc基础/Paraformer_QKV_Linear精度问题到pytorch_test复现经验报告.md`
