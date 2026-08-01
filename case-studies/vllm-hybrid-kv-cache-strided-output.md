# Case Study: vLLM Hybrid KV Cache 非连续输出适配热点

## 问题现象

一份 Qwen3.5 hybrid Attention + Mamba 运行记录显示，Attention 时间线中出现大量 `custom_copy_stride_general`、slice 和 slice writeback 类事件。定位目标不是从底层事件名猜测 DLC Custom Kernel 缺陷，而是确认这些事件属于哪个 parent op，以及开销位于 source、destination 还是跨层执行所有权。

原记录最终把最小边界收敛到一次 `reshape_and_cache` 调用内部的非连续 destination cache view 与 output layout adaptation。该边界不同于 [vLLM Attention 重复 KV Cache Update 性能热点](vllm-attention-duplicate-kv-cache-update.md)：旧案例定位的是同一语义 update 被两个 owner 重复执行；本案例关注调用次数正确后，单次调用内部的 materialization/writeback 成本。两种假设必须分别验证。

## 证据与身份边界

来源文档保存了 shape、stride、调用链摘录和机制分析，但没有保存可从当前工作区复核的：

- 模型完整名称、revision 或资产 digest。
- vLLM、vllm-dlc、PyTorch DLC Backend、DLCSynapse、DLC Runtime 和 DLC Custom Kernel 的 exact revision/package identity。
- image digest、dirty state、设备拓扑、TP/PP/EP、prefill/decode workload 和 Graph/eager identity。
- 原始 trace、日志、代码 diff、profile 配置和重复样本路径。
- 无插桩 baseline、候选修复结果和正式 benchmark。

因此本文使用以下证据等级：

- **Reported runtime observation**：来源文档记录的 trace、shape、stride、storage pointer 和对照实验。
- **Reported source observation**：来源文档摘录的函数、调用链和代码片段，绑定未恢复的源码版本。
- **Mathematically derived fact**：由已给 shape/stride 直接计算的关系。
- **High-confidence inference**：多项观察共同支持，但缺少底层 adapter 实现或完整 artifact 闭环。
- **Proposed design**：尚未实现和验证的候选优化。

这些内容不能描述为当前 checkout、所有 Qwen3.5、所有 hybrid model 或所有 DLC Attention Backend 的普遍事实。

## Reported Workload

- 模型族：Qwen3.5，具体变体和 revision 未保存。
- 模型结构：hybrid Attention + Mamba。
- 关注路径：DLC Attention Backend 的 KV cache update 和 `reshape_and_cache`。
- source 示例：

```text
key/value shape  = [1, 2, 256]
key/value stride = [512, 256, 1]
key/value is_contiguous = True
```

- destination 示例：

```text
key_cache/value_cache shape  = [1014, 2, 512, 256]
key_cache/value_cache stride = [524288, 131072, 256, 1]
key_cache/value_cache is_contiguous = False
```

size 为 1 的维度允许非标准 stride 而不破坏 PyTorch contiguous 判定，因此 source 示例中的 shape/stride 与 `is_contiguous=True` 不矛盾。

## 定位路径

### 1. 先确认底层事件的 Parent Boundary

来源文档记录的调用链：

```text
vLLM Attention Backend
-> vllm_dlc._custom_ops.reshape_and_cache()
-> torch.ops._C_dlc_cache_ops.reshape_and_cache()
-> vllm-dlc/csrc/cache.cpp::reshape_and_cache()
-> KernelDesc output path
-> custom_reshape_and_cache_bf16
```

同一调用边界内观察到：

```text
custom_copy_stride_general
custom_slice_tensor
custom_slice_backward
copy_by_slice
copy_by_slice_back
```

这里的 `custom_slice_backward` 是 runtime/kernel event 名，不能仅凭名称解释为模型 autograd backward。Reported runtime observation 支持这些事件嵌套在 `reshape_and_cache` parent 内；event 名本身不能证明具体 materialization 实现。

### 2. 同时检查 Source 和 Destination

来源记录在调用前同时检查 source `key/value` 与 destination `key_cache/value_cache` 的 shape、stride 和 contiguity。结果是 source contiguous、destination non-contiguous。

对 source 额外调用 `.contiguous()` 后，来源记录称 slice 数量和耗时没有明显变化。该单变量对照支持：在已观察样本中，source contiguity 不是主要解释，应继续检查 destination view 和 output adaptation。由于没有样本数和测量定义，不能外推为 source-side materialization 在所有配置中都无成本。

对 cache、in-place 和 out 路径，仅记录 shape/stride 不足。能够观测时还应保存：

- source/destination contiguity。
- raw storage identity 和 storage size。
- storage offset。
- logical view/alias relationship。
- layout 更新前后的 descriptor。

### 3. 区分 Shared Raw Storage 和 Logical View

Reported source observation 显示，`_allocate_kv_cache_tensors()` 为 `KVCacheTensor` 分配 raw Tensor，再按 `shared_by` 将同一 raw Tensor 绑定到多个 layer。Reported runtime observation 还记录一个 `shared_by` 组包含多个 `MambaSpec` layer 和一个 `FullAttentionSpec` layer，并观察到相同 `raw_storage_ptr`。

这只能证明底层 storage 共享，不能证明逻辑 Tensor 相同：

```text
shared raw storage
-> MambaSpec logical state view
-> MambaSpec logical state view
-> FullAttentionSpec logical KV cache view
```

不同 view 可以具有不同 shape、stride、dtype interpretation 和 storage offset。仅比较 pointer 会丢失 layout 和语义 ownership。

### 4. 定位 Hybrid Attention/Mamba Stride 调整

来源文档记录 `_reshape_kv_cache_tensors()` 先通过 `view(dtype)`、`view(kv_cache_shape)` 和 `permute(...)` 创建 Attention view；hybrid 场景随后调用 `_update_hybrid_attention_mamba_layout()`，并对 Attention view 使用 `as_strided_()`：

```python
hidden_size = kv_cache.shape[2:].numel()
kv_cache.as_strided_(
    size=kv_cache.shape,
    stride=(hidden_size, 2 * hidden_size, *kv_cache.stride()[2:]),
)
```

`as_strided_()` 本身建立 view，不复制数据。来源分析认为该调整用于让 Attention 与 Mamba 共享 logical block/page ownership。由于 exact source contract、注释和对应测试未恢复，这个设计目的保持为 high-confidence inference，而不是当前源码事实。

不能为了减少 trace event 直接删除这类 layout 调整。任何性能修复都必须先保留 block ID、token/page ownership、cache reuse/swap/free 和后续 Attention 语义。

### 5. 用 Shape/Stride 解释当前 View

对于：

```text
shape = [1014, 2, 512, 256]
```

普通 contiguous stride 为：

```text
[262144, 131072, 256, 1]
```

来源记录的实际 stride 为：

```text
[524288, 131072, 256, 1]
```

一个 K 或 V block 包含：

```text
2 * 512 * 256 = 262144 elements
```

因此第 0 维实际 stride 是单个 K 或 V block 的两倍。这与 K/V logical views 在 shared raw storage 上按 block/page 粒度交错映射的解释一致：

```text
[K block 0][V block 0][K block 1][V block 1]...
```

这里描述的是 block/page 粒度，不是 scalar 粒度。它是当前 descriptor 支持的 high-confidence layout explanation；仍需 K/V 两个 view 的 base pointer、storage offset、storage span 和 exact source contract 才能完全闭合物理映射。

对 destination view 调用 `.contiguous()` 会创建副本，不能原地改变既有 cache storage contract，也不能自动把后续 Attention/Mamba 切换到该副本。

### 6. 收敛 Output Adaptation 边界

Reported source observation 显示当前 `cache.cpp` 路径把 `key_cache/value_cache` 注册为 `KernelDesc` output destination：

```cpp
k.input(key);
k.input(value);
k.input_format(slot_mapping);
k.output(key_cache);
k.output(value_cache);
k.launch("custom_reshape_and_cache_bf16");
```

Reported runtime observation 同时记录 non-contiguous destination 和 parent 内的 stride copy/slice writeback 事件。两者共同支持以下 high-confidence inference：

```text
contiguous or materialized temporary output
-> stride/layout adaptation
-> writeback into non-contiguous destination cache view
```

在没有 `KernelDesc` output adapter exact implementation、event correlation artifact 和对照 trace 前，不能把临时 output 的具体生成机制提升为完全确认事实，也不能泛化为所有 `KernelDesc::output()` 都会 materialize。

## 当前最小边界

### 已有有界证据

1. Reported trace 把目标 copy/slice events 关联到 `reshape_and_cache` parent。
2. 当前样本 source contiguous，destination non-contiguous。
3. Source `.contiguous()` 对照没有报告明显改善。
4. 当前 destination 第 0 维 stride 是 contiguous 单 K/V block stride 的两倍。
5. Reported storage identity 支持多个 Attention/Mamba logical views 共享 raw storage。

### 高置信推断

```text
hybrid shared raw storage
-> Attention/Mamba logical views
-> non-contiguous Attention destination view
-> KernelDesc output materialization/layout adaptation
-> copy/slice writeback events
```

### 尚未证明

- 所有目标事件都来自同一个 output adapter 分支。
- 当前 checkout 仍存在完全相同的调用链和 layout。
- 该路径是正式无插桩 workload 的主要端到端热点。
- 新 kernel 是唯一或最佳修复。

## Proposed Design: Strided-Output Cache Kernel

来源文档提出一个候选设计，不是已实现结论：

- 保留 shared raw Tensor pool 和 hybrid Attention/Mamba layout contract。
- 接收 source、slot mapping 和 destination stride/storage metadata。
- 直接写入 non-contiguous destination cache view，避免普通 output materialization/writeback。
- 不支持的 dtype、head size、block size、quantization、alignment 或 layout 明确回退旧路径。

候选调用链：

```text
key/value + slot_mapping + destination descriptor
-> strided-output reshape-and-cache path
-> direct write into destination cache view
```

在实现前必须核验 DLC Custom Kernel ABI、storage offset、overlap、alignment、DMA/vectorization、Graph capture 和 fallback contract。不得仅为消除 profiler slice event 修改上层 page/layout 语义。

## 验证矩阵

### Correctness 和 Cache Semantics

- 新旧路径的 key/value cache 内容。
- 非目标 cache 区域保持不变。
- 后续 Attention output 和生成 token。
- `slot_mapping == -1` 不发生语义写入。
- 正常、重复、非顺序和非法 slot。
- block 内边界和跨 block boundary。
- prefill、decode、batch、多 rank 和 cache reuse/swap/free。

### Layout 和 Dispatch

- contiguous destination。
- 第 0 维有 gap 的 non-contiguous view。
- K/V block-interleaved logical views。
- 非零 storage offset 和共享 raw storage 的多个 view。
- source contiguous/non-contiguous。
- 支持与不支持的 dtype、head size、block size、quantization、alignment 和 Graph/capture 组合。
- Unsupported 组合明确走旧路径，不能 silent wrong cache。

### Ownership 和 Trace

- 每个目标 layer/step 恰好执行所需次数的 cache update，不是零次或重复执行。
- 新路径和旧路径不同时执行。
- Diagnostic profile 中目标 materialization/writeback events 是否减少。
- Event 消失只证明 trace 行为变化，不自动证明 correctness 或端到端收益。

### 正式性能

1. 先封存无插桩 baseline。
2. Diagnostic epoch 使用 verbose trace 或同步闭合定位边界。
3. 删除 verbose、临时同步、print 和 wrapper。
4. 使用相同 source/package/model/device/workload 重跑无插桩 benchmark。
5. 报告重复 attempts、离散度、correctness 和 server liveness。

## 可复用经验

1. 看到底层 slice/copy event，先确认 parent op，不从 event 名直接猜模型热点。
2. 调用次数正确后，再检查一次调用内部的 materialization 和 layout adaptation；反之也要先排除重复执行。
3. 对 layout/view 候选同时保存 source 和 destination 的 shape、dtype、stride、contiguity、storage identity、storage offset 和 view relationship。
4. Shared storage 不等于 shared logical Tensor，pointer 相同不能替代完整 tensor descriptor。
5. `.contiguous()` 对照只能回答被复制一侧是否相关，不能自动修复另一侧的 destination storage contract。
6. 不得为了减少 trace event 破坏 cache/state、block/page ownership 或 fallback 语义。
7. 修复必须同时闭合 correctness、ownership/call contract 和无插桩性能。

## Claim Boundary

本文保存一份缺少 exact source/image/artifact identity 的历史工程经验。它支持“在当前记录中，应优先检查 hybrid logical cache view 与 `reshape_and_cache` output adaptation”这一有界诊断方向，但不证明所有 Qwen3.5、所有 hybrid Attention/Mamba、所有 non-contiguous output 或所有 `KernelDesc` output 都具有相同根因。Strided-output kernel 仍是 proposed design；其正确性、支持矩阵、DLC Runtime dispatch、Real DLC Hardware 无插桩性能收益和稳定性均为 `not_verified`。

## 来源

- `/work/inn/non-contiguous-kv-cache-reshape-and-cache.md`
- [vLLM Attention 重复 KV Cache Update 性能热点](vllm-attention-duplicate-kv-cache-update.md)
- [Chipltech-Family Accelerator 性能 Profiling](../runtime-debugging/performance-profiling.md)
