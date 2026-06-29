# Case Study: RSThinker custom_softmax 诊断

## 问题现象

RSThinker 多模态模型在 DLC Platform 上 multi-token decode 时，SC_AID-0 样本从 token 2 开始产生 `::` 重复。

## 背景与环境

- **模型**：RSThinker (GLM-4.1V-9B-Base)
- **问题**：multi-token greedy decode 产生重复字符
- **对比**：CPU oracle vs DLC

## 定位路径

### 从输出倒推

1. `topk` 操作 — 不是问题
2. `lm_head` Linear — 不是问题
3. 后续 decoder Linear/RMSNorm — 不是问题
4. 进入 attention 核心 → 拆解 attention 内部

### attention 内部拆解

在 attention 内部逐步骤对比：
- query/key/value 输入: exact
- score matmul: exact
- softmax weights: drift ← first drift
- value matmul: drift (downstream)
- attention output: drift

### 确认 kernel 路径

- Python API: `F.softmax(..., dtype=torch.float32)`
- dtype=float32 导致 DLC dispatch 到 `custom_softmax` (FP32 路径，不是 bf16)
- Runtime kernel: `custom_softmax_last_dim_640` (shape 从 548 被 padding 到 640)
- DLC_CHECK_RESULT lambda: `custom_softmax`

### 构建反馈环

用 `scripts/dump_transformers_multimodal_precision.py` 的 attention probe 做 1-token dump，验证每次 dispatch 修改后的效果。

### pytorch_test 复现

编写 `test_rsthinker_custom_softmax_repro.py`：
- shape `[1, 32, 1, 548]` (padding 到 640)
- `F.softmax(x, dim=-1, dtype=torch.float32)`
- 结果：`224/17536` errors at `atol=1e-6`

## 根因或当前结论

DLC `custom_softmax` FP32 路径在 last-dim padded shape (从 548 padding 到 640) 下产生不正确结果。

## 验证方式

- pytorch_test `test_rsthinker_custom_softmax_repro.py` 可复现。
- Runtime kernel `custom_softmax_last_dim_640` 确认。

## 可复用经验

1. **`dtype=` 参数改变 dispatch 目标**：`F.softmax(x)` 默认走 bf16，`F.softmax(x, dtype=torch.float32)` 走 FP32 kernel。
2. **用 verbose trace 找 runtime kernel name**：`DLC_SYN_DEBUG=1 DLC_SYN_VERBOSE=1 --perf --skip-check` 显示 `custom_softmax_last_dim_640`（padding 后 shape）。
3. **反馈环比设计更重要**：1-token dump 是 cheap 的精准反馈环，比 50-token smoke 快得多。
4. **从输出往前拆**：`topk → lm_head → decoder → attention` 逐段排除。

## 后续动作

交付算子团队修复 `custom_softmax` FP32 路径在 padded shape 下的行为。

## 来源

- `/work/RSThinker/docs/rsthinker_custom_softmax_diagnosis_experience.md`
- `/work/RSThinker/steps/rsthinker-dlc-platform-adaptation/18-narrow-multitoken-decode-to-custom-softmax.md`
