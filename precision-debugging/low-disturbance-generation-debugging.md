# generation / decode 精度定位的低扰动策略

## 适用场景

- 多模态模型或 decoder-only 模型在 `generate()` 路径上出现 CPU vs DLC 精度差异。
- 需要对 `prefill_logits`、`first_decode_logits`、`generated_ids` 做逐步定位。
- 怀疑 instrumentation 本身会改变 `generate()` 行为。

## 核心结论

generation / decode 精度定位时，不要默认 forward hook 是透明观测。对这类问题，推荐的默认顺序是：

1. no-hook baseline
2. top-level outputs compare only
3. count-only / lower-perturbation hook
4. metadata-only hook
5. dumping hook
6. 如果 online probe 已经证明自己扰动现象，转 offline replay

## 为什么 generation path 特别敏感

相比普通 prefill-only forward，`generate()` 路径更容易受到以下因素影响：

- hook registration / call timing
- hook body 中的 tensor traversal
- metadata 读取
- D2H copy
- `torch.save(...)`
- use-cache / decode step 的细粒度执行顺序变化

因此“为了定位而加的 hook”本身，可能改变：

- `prefill_logits.pt`
- `first_decode_logits.pt`
- `generated_ids.pt`

## 推荐诊断顺序

### 阶段 1：no-hook baseline

先确认在当前输入合同下，真实现象是否存在。最少对比：

- `processor inputs`
- `prefill_logits.pt`
- `first_decode_logits.pt`
- `generated_ids.pt`

如果 no-hook 本身 intermittent：

- 先量化复现率
- 不要急着扩 hook window

### 阶段 2：只看顶层输出

当目标只是回答“现象是否真实”，先不要直接进模块 dump。优先用顶层 compare report 先回答：

- 输入是否 exact
- drift 是 prefill、first decode，还是 token 级分叉

### 阶段 3：count-only 或更低扰动 hook

如果必须下钻模块，先用最低扰动 hook：

- 注册同样的 hook
- 只记录模块被调用
- 不读取 tensor
- 不做 D2H
- 不写 `.pt`

这一步适合回答：

- mere hook registration / call timing 是否已经足以改变结果
- hook window 是否本身就是强变量

### 阶段 4：metadata-only hook

只有当 count-only 不能回答问题时，才升级到 metadata-only：

- 只读必要 metadata
- 不做不必要的 tensor traversal
- 不保存 hook tensor 文件

### 阶段 5：dumping hook

以下动作都应视为 high-perturbation instrumentation：

- `first_tensor(...)`
- `tensor_metadata(...)`
- `tensor.detach().to("cpu")`
- `torch.save(...)`

使用 dumping hook 时，必须明确：

- 它是诊断工具
- 不是 boundary truth oracle

## 什么时候停止在线 probe

满足以下任一条件，就应停止继续在线扩 hook window：

1. hook variant 改变了 `prefill_logits` / `first_decode_logits` 是否 drift
2. count-only 不同 window 的结果不连续
3. metadata / dumping hook 会改变是否出现 token-level drift
4. 你已经无法区分“真实边界”和“probe 自己造成的边界”

此时更好的下一步是：

- offline same-input replay
- target-module real-input replay
- dispatch fallback 验证

## offline replay 的进入条件

当在线 probe 已经不可信时，改问更小、更稳的问题：

1. 当前模块输入在真实模型路径里是否已经 drift
2. 若已 drift，给定同一输入，该模块 CPU vs DLC 是否 still drifts
3. 若 replay exact，说明该模块不是新增 drift 源
4. 若 replay 不 exact，再继续下钻该模块内部 operator

## 常见坑

1. **先上 broad hook 再做 reality check**：很容易把 hook-induced perturbation 当真边界。
2. **把 earliest captured drift 当真实 first divergent module**：generation path 尤其危险。
3. **在 intermittent no-hook 现象上继续扩 hook**：常常会把问题越搞越假。
4. **count-only exact 就等于“没有问题”**：count-only 只说明该 probe 配置下现象被 suppress，不代表真实 drift 不存在。
5. **忽略顶层输出 compare**：很多问题其实先看 `prefill_logits` / `first_decode_logits` 就足够判断方向。

## 相关资料

- [precision-debugging/precision-debugging-overview.md](precision-debugging-overview.md)
- [precision-debugging/model-site-dump-to-repro.md](model-site-dump-to-repro.md)
- [case-studies/rsthinker-generation-hook-perturbation-and-low-disturbance-bracketing.md](../case-studies/rsthinker-generation-hook-perturbation-and-low-disturbance-bracketing.md)

## 来源

- RSThinker generation 精度定位经验
- `/work/RSThinker/AGENTS.md`
- `/work/RSThinker/docs/rsthinker_precision_dump_runbook.md`
