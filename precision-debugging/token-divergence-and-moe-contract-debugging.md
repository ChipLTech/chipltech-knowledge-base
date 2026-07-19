# vLLM 生成 token 分叉与 MoE 语义契约定位

## 适用场景

- DLC Platform 与 CUDA 后端使用相同模型权重、tokenizer、prompt 和 greedy 参数时，生成 token 不一致。
- 多卡 MoE、AWQ/W4A16 或 W8A16 路径出现低质量重复输出、首个分叉 token 或 logits top-1 不一致。
- 怀疑问题位于 routed MoE、expert dispatch、activation、quantization 或 DLC Custom Op 到 DLC Custom Kernel 的参数契约。

## 核心结论

生成分叉定位不要直接从完整长 decode 开始。优先把问题压成严格等价的单 token 问题，再按 `endpoint -> prompt -> logits/sampler -> layer -> branch -> EP rank -> expert -> activation/kernel contract` 收敛。

可复用经验：算子数值爆炸不一定是 GEMM 或权重错误，也可能是模型配置中的 activation type、clamp limit、quantization metadata 或 processor/tokenizer 语义没有传到底层 DLC Custom Op / DLC Custom Kernel。

## 标准定位流程

### 1. 建立严格等价 API 对照

先保证两边请求语义完全一致：

- 相同 endpoint：不要混用 `/v1/chat/completions` 和 `/v1/completions` 直接比较。
- 相同 prompt 或同一套 chat template 后的 prompt token。
- 相同 tokenizer、模型权重、served model name、TP/EP、dtype、quantization。
- 相同 `temperature=0`、`max_tokens`，需要时打开相同 `logprobs`。
- 相同 prompt token 数和输入 token IDs。

chat API 会应用 chat template；如果另一边使用 completions API，两者不是等价输入。

### 2. 把多步 decode 改写为单步 prefill

如果第 N 个生成 token 开始分叉，不要一开始调试第 N 个 decode step。将分叉前的 token 直接拼回 prompt，只生成 1 个 token：

```text
original prompt + agreed generated prefix -> max_tokens=1
```

如果仍然复现，问题可视为完整 prompt 首个生成位置的问题，优先排除多步 decode 累积、KV cache 更新、scheduler 历史回灌和 speculative decoding 的干扰。

### 3. 先排除 sampler

比较 sampler 前 raw logits 的 `argmax` 与最终 sampled token：

- 如果 raw logits `argmax` 与 sampled token 一致，sampler 通常不是首个错误边界。
- 关闭 `logprobs` 后仍分叉，说明 logprobs 不是必要触发条件。
- 如果 logits 在 sampler 前已经不同，继续向 model forward / LM head 前收敛。

### 4. 逐层、逐分支、逐 rank 收敛

推荐下钻顺序：

```text
final hidden / logits
  -> layer index
  -> attention vs MoE/MLP branch
  -> EP rank / TP rank
  -> global/local expert
  -> W13 / activation / W2
  -> DLC Custom Op 参数契约 / DLC Custom Kernel 实现
```

记录每层最后一个 token 的 `norm`、`std`、`absmax`，先找第一个数量级突然变化的层，再拆分该层 attention、post-attention norm、MoE 输出和 residual。

### 5. 核对模型到算子的语义契约

MoE 路径必须核对：

- selected global expert IDs 和 routing weights 是否一致。
- EP rank 上本地 expert 映射是否一致。
- activation type 是否来自模型配置并传到底层，例如 `SwiGLUStep` vs 普通 SiLU。
- clamp limit 是否传递，例如 `limit=7.0`。
- quantization metadata 是否完整传递，例如 `quant_method`、`bits`、`group_size`、`zero_point`。
- W4A16、W8A16、AWQ、AWQ-Marlin、compressed-tensors 是否走到了预期 kernel 路由。

如果模型配置要求 `SwiGLUStep(limit=7.0)`，但 DLC fused-MoE 路径固定执行普通 SiLU，输出可能被局部 expert 显著放大并翻转 token。这类问题的最小边界是 activation contract mismatch，而不是 sampler 或 LM head。

## 硬件重复一致性干扰项

跨后端精度定位前，先在每张卡上对目标 DLC Custom Op 做固定输入多次 replay：

- 比较 exact equality、`abs_max`、top-k expert IDs。
- 单卡重复结果不一致时，先隔离 Real DLC Hardware 或 DLC Runtime 问题。
- 不要把单卡不稳定归因到模型 activation 修复，也不要让它污染确定性语义契约判断。

## 常见坑

1. 混用 chat/completions 与 completions 结果直接比较。
2. 在多步 decode 上直接加重 hook，导致 KV cache、scheduler 和 instrumentation 变量混在一起。
3. 看到 sampled token 不同就怀疑 sampler，没有先比较 raw logits top-1。
4. 只比较整体 hidden state，不按 layer/branch/rank/expert 下钻。
5. 把 MoE 输出爆炸默认归因于 GEMM，而忽略 activation type、clamp limit 和 quantization metadata。
6. 单卡重复一致性未确认时就做跨后端精度结论。

## 相关资料

- [precision-debugging-overview.md](precision-debugging-overview.md)
- [low-disturbance-generation-debugging.md](low-disturbance-generation-debugging.md)
- [model-site-dump-to-repro.md](model-site-dump-to-repro.md)

## 来源

- `/work/test/同事文档/vllm-moe-swiglustep-activation-mismatch.md`
