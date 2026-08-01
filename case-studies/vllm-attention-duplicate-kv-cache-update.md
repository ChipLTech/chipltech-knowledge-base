# Case Study: vLLM Attention 重复 KV Cache Update 性能热点

## 问题现象

一份 Qwen3.5-27B 历史运行报告显示：模型在 DLC Chip、TP=2、vLLM-DLC eager serving 中单 token decode 延迟偏高。目标是从端到端请求逐层定位热点，而不是预设某个 Attention kernel 有缺陷。

## 证据与身份边界

原始过程文档记录了 Real DLC Hardware 请求日志、profile JSON、DLCSynapse `syn_*.ansi` log、kernel summary 和带 `torch.dlc.synchronize()` 的插桩结果，但没有保存：

- vLLM、vllm-dlc、PyTorch DLC Backend、DLCSynapse 的 exact revision。
- image digest、dirty state 和修改文件路径。
- profile 工具/version/config identity。
- 可由当前工作区读取的原始 artifact 路径。

当前 `/work/vllm` 和 `/work/vllm-dlc` 源码搜索不到报告中的 `forward_includes_kv_cache_update` 或 `unified_kv_cache_update()`。因此本文把具体字段、调用链和修复记为 **historical reported observation**，不描述为当前源码事实。通用方法和 claim boundary 不依赖该字段存在。

## 历史 workload

- 模型：Qwen3.5-27B。
- 硬件：2 张 DLC Chip。
- 并行：TP=2。
- dtype：bfloat16。
- 执行：`--enforce-eager`。
- 请求：固定 prompt、`max_tokens=5`、`temperature=0`、`top_p=1`。
- 环境：报告记录 `DLC_SYN_COPY_ASYNC=O2`，但最终 worker 环境证据未保存。

`max_tokens=5` 不自动证明每个样本都是稳定单 token decode，也不能从普通非流式请求直接得出 TTFT、TPOT 或 ITL。实际 token 数、finish reason、warm-up、sample count 和 server epoch 没有完整恢复。

## 定位路径

### 1. 从 decoder layer 找到优先边界

带同步插桩的历史观察：

```text
linear-attention layer：约 5-6 ms
full-attention layer：约 33-34 ms
```

报告称该差异在已观察的 full-attention layers 重复出现。由于 layer index、样本数和 rank 分布没有保留，本文不将其提升为“所有层”的完整覆盖结论。

### 2. 拆 full attention stage

历史报告继续拆分：

```text
qkv projection      约 0.2 ms
split normalization 约 1.6-2.0 ms
rotary              约 3.1-3.8 ms
attention wrapper   约 25 ms
output gate         约 0.1-0.2 ms
output projection   约 0.4-0.6 ms
```

这些数字包含诊断同步，只用于排序下钻边界，不是正式 benchmark。

### 3. 对齐 wrapper 与 DLC Attention Backend

报告记录的相邻边界：

```text
模型 self.attn / vLLM Attention wrapper 约 25-26 ms
DLC Attention Backend interval         约 15-16 ms
```

DLC Attention Backend 内部报告：

```text
reshape_and_cache interval 约 10.2 ms
attention-compute interval 约 4.9-5.3 ms
```

这里的 `attention-compute` 是历史 stage label，不声明为 NVIDIA FlashAttention，也不代表一个唯一 DLC Custom Kernel。

### 4. 用稳定 residual 形成假设

```text
25.5 ms - 15.2 ms ≈ 10.3 ms
```

residual 与一次 cache update interval 接近，因此首先提出“相邻 owner 重复执行 KV cache update”的假设。单靠该数值仍不能证明重复调用；它也可能来自 layout conversion、同步等待或区间定义差异。

### 5. 用调用来源和次数确认历史行为

原报告称插桩同时观察到：

```text
vLLM wrapper / unified cache-update path
  -> reshape_and_cache

DLC Attention Backend forward
  -> reshape_and_cache
  -> attention compute
```

并记录历史 backend contract 值为：

```text
forward_includes_kv_cache_update=False
```

该值与 backend 实际执行 cache update 的行为不一致，导致同一 layer 的 KV cache update 执行两次。由于 exact source revision 未恢复，该结论绑定到历史日志所代表的代码和 profile，不能外推到当前 checkout。

### 6. 历史修复观察

原报告称将历史 contract 调整为：

```text
forward_includes_kv_cache_update=True
```

之后：

- 每个已观察 full-attention layer 只剩一次 cache update。
- wrapper、Attention layer 和 DLC Attention Backend 时间收敛。
- 带同步插桩的 full-attention layer 从约 25-26 ms 降至约 15-16 ms。

这证明历史诊断 run 中重复调用被移除。它不证明当前源码应设置同名 flag，也不证明无插桩生产性能收益。

## 根因或当前结论

历史证据支持的根因是：framework wrapper 与 DLC Attention Backend 对 KV cache update 的执行所有权声明和实际行为不一致，导致重复调用。`reshape_and_cache` 自身是否需要 kernel 优化没有被该案例证明。

若调用次数已经正确，但单次 `reshape_and_cache` 内仍出现 layout conversion 或 slice writeback，应转到 [vLLM Hybrid KV Cache 非连续输出适配热点](vllm-hybrid-kv-cache-strided-output.md)，检查 source/destination descriptor、shared storage、logical view 和 output adaptation。两个案例是不同根因边界，不能互相替代。

通用结论是审计执行所有权，而不是复制一个固定 flag：

```text
declared ownership
== actual invocation ownership
== one required semantic update per layer/step
```

## 验证缺口

以下仍为 `not_verified`：

- exact source/image/package identity。
- 当前 vLLM/vllm-dlc 是否仍存在相同 contract。
- 去除 print、强制同步和临时 wrapper 后的 TTFT、TPOT、ITL、request latency 和 throughput 收益。
- 多 server epoch 的性能分布和稳定性。
- 长上下文、prefill、并发、Graph、speculative decoding 和其他 KV cache layout。
- 多 token、多请求和 cache slot/block mapping 的完整 correctness coverage。

## 后续若重做该验证

1. 固定 exact source/package/image/model/device/workload identity。
2. 先运行无插桩 correctness 与性能 baseline。
3. 用 request、layer、stage、wrapper、DLC Attention Backend、DLC Runtime/DLCSynapse 的非重叠边界下钻。
4. 保存调用次数、parent、shape、dtype、stride/layout、rank 和 prefill/decode identity。
5. 修改唯一 ownership contract 后验证每个目标 layer 恰好一次 cache update，而不是零次或两次。
6. 移除诊断同步和日志，再使用同一 workload 运行正式 benchmark。

## 可复用经验

1. 从端到端逐层缩小，不从 kernel 名猜热点。
2. 相邻边界稳定 residual 是工作集合不一致的线索，不是单独根因证据。
3. 调用次数、来源、shape 和 parent identity 才能确认重复执行。
4. wrapper/backend 的执行所有权错误可能比单 kernel 优化更重要。
5. 插桩定位收益与无插桩生产收益必须分开报告。

## Claim Boundary

本文只保存历史报告中 Qwen3.5-27B、TP=2、DLC Chip、eager decode profile 的有界观察。它不证明所有模型或 DLC Attention Backend 都应设置 `forward_includes_kv_cache_update=True`，不证明 `reshape_and_cache` kernel 本身有性能缺陷，也不声明正式 benchmark 收益已经验证。

## 来源

- `/work/err/performance-hotspot-localization-case.md`
