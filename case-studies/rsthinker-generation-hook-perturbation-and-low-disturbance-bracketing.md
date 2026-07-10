# Case Study: RSThinker generation hook 扰动与低扰动 bracketing

## 问题现象

RSThinker 在 DLC Platform 上的 `generate()` 路径出现 CPU vs DLC 精度差异时，最初看起来像是一个普通的 decoder late-path 漂移问题：`prefill_logits.pt`、`first_decode_logits.pt`、偶发 `generated_ids.pt` 不一致。

进一步诊断发现，**generation hook 自己会改变现象是否暴露**。同一输入合同下，仅仅改变 hook window、hook body 或是否做 metadata / dump，就可能改变 `prefill_logits.pt` 和 `first_decode_logits.pt` 的 drift 结果。

## 背景与环境

- **模型**：RSThinker（基于 `GLM-4.1V-9B-Base`）
- **对比方式**：CPU Reference vs DLC
- **主 reality track**：`suite-native input`
- **固定合同**：相同 prompt、相同 image、`max_tokens=1`、相同 compare contract
- **关键脚本**：`scripts/dump_transformers_multimodal_precision.py`

## 定位路径

### 阶段 1：先确认 no-hook 真实现象存在

在 broad hook 之前，先跑 no-hook baseline，只对比顶层输出：

- `processor inputs`
- `prefill_logits.pt`
- `first_decode_logits.pt`
- `generated_ids.pt`

关键结论：**真实 no-hook drift 存在**，不是 hook 幻觉。

这一步非常重要，因为如果不先确认 no-hook 现象，就无法区分：

- 真正的模型执行漂移
- 还是 instrumentation 自己引入的扰动

### 阶段 2：确认 high-perturbation hook 会改变结果

在 generation path 上使用 metadata / dump hook 后，观察到：

- `record["input"] = tensor_metadata(input_tensor)` 会扰动 `first_decode`
- `record["output"] = tensor_metadata(input_tensor)` 也会扰动 `first_decode`
- `first_tensor(...)`
- `tensor_metadata(...)`
- `tensor.detach().to("cpu")`
- `torch.save(...)`

这些动作都不能再被视为“透明观测”。

### 阶段 3：切换到低扰动 count-only hook

为了隔离单纯 hook registration / call timing 的影响，引入 `count-only` generation hook：

- 仍注册同样的 hook
- hook body 只记录 count / module name
- 不做 tensor traversal
- 不读 metadata
- 不做 D2H copy
- 不写 `.pt` 文件

这是在线 generation probe 中的当前最低扰动路径。

### 阶段 4：做 late-window bracketing

在 `suite-native input` 下，得到的关键结果如下：

| Hook window | 结果 |
|---|---|
| `lm_head only + count-only` | `prefill` drift + `first_decode` drift |
| `model.language_model.norm + lm_head + count-only` | `prefill` drift，`first_decode` exact |
| `layer30 + lm_head + count-only` | `prefill` exact，`first_decode` exact |
| `layer29 + lm_head + count-only` | `prefill` exact，`first_decode` exact |
| `layer28 + lm_head + count-only` | `prefill` exact，`first_decode` exact |

这一步给出两个高价值结论：

1. **decode drift 对极低扰动的 hook registration / call timing 仍然敏感**。
2. **这种敏感性对 hook window 非连续**，不是简单的“越靠后越真实”。

### 阶段 5：停止继续在线扩窗口，转向 offline replay

因为 online probe 已经证明自己会改变量测结果，所以不能再把它直接当“真实 boundary 探针”。

后续切换到 offline replay：

- no-hook 真实 `lm_head` input 已经 drift
- 但对同一个 CPU hidden state 做 `lm_head` real-input replay 时，`F.linear` CPU vs DLC exact

因此可以排除：

- `lm_head` 本身不是 `first_decode` drift 的独立新增来源

更合理的下一步是继续上移到 `model.language_model.norm` 及其上游 hidden state。

## 当前结论

1. **Generation hook 不是透明观测工具**。
2. **Broad dumping hook 不能直接拿来判断真实 first divergent boundary**。
3. **低扰动 count-only hook 比 metadata/dump hook 更可信，但仍可能改变现象**。
4. **当 online probe 已经证明自己会扰动结果时，应转向 offline minimal replay**。
5. **`lm_head` same-input replay exact，说明 online `lm_head only` 观测到的 drift 来自上游，而不是 `lm_head` 自身新增漂移。**

## 可复用经验

1. **先做 no-hook baseline**：没有 no-hook reality check，不要急着扩 hook。
2. **先看顶层输出，再下钻模块**：`prefill_logits.pt` / `first_decode_logits.pt` / `generated_ids.pt` 的变化足以先判断现象是否真实。
3. **把 generation instrumentation 按扰动等级排序**：no hook < count-only < metadata-only < dumping hook。
4. **不要把 hook 观察到的 earliest drift 直接当真实边界**：特别是 generation path。
5. **在线 probe 会扰动时，尽快改成 offline replay**：先回答“当前算子 same-input 是否 still drifts”，再决定要不要继续上游。

## 相关资料

- [precision-debugging/low-disturbance-generation-debugging.md](../precision-debugging/low-disturbance-generation-debugging.md)
- [precision-debugging/model-site-dump-to-repro.md](../precision-debugging/model-site-dump-to-repro.md)
- [testing/pytorch-test-replay-from-synapse-log.md](../testing/pytorch-test-replay-from-synapse-log.md)

## 来源

- `/work/RSThinker/AGENTS.md`
- `/work/RSThinker/steps/dlc-precision-test-analysis/02-remaining-operator-root-cause-plan.md`
- `/work/RSThinker/docs/rsthinker_precision_dump_runbook.md`
- `/tmp/kilo/rsthinker_nohook_suite_native_20260710`
- `/tmp/kilo/rsthinker_suite_native_countonly_20260710`
