# 精度定位方法论

## 适用场景

- DLC-Family Accelerator 模型推理出现 CPU/DLC 输出不一致。
- 需要从端到端模型输出差异收敛到单个算子边界。
- 需要区分 DLC Precision Difference（预期差异）和真正的算子 bug。

## 核心结论

精度定位的核心原则：
1. **CPU 是主要精度 oracle**，不是 CUDA。CUDA 路径对图像输入、前处理、numerics 设置敏感，可能产生不同于 CPU/DLC 的输出。
2. **先判断是否属于 DLC Precision Difference**。PGX bf16 转换、硬件 exp/rsqrt、tiling、累加顺序造成的差异是预期行为。
3. **从端到端收敛到算子级**：用 Model-Site Dump 捕捉 failure 现场，逐步缩小范围。
4. **单变量定位**：一次只改动一个算子的 dispatch，确认边界后再继续。
5. **生成分叉先压成单 token 问题**：多步 decode 的 token 分叉优先改写成“分叉前 token 拼回 prompt，`max_tokens=1`”的 prefill 问题，再排除 sampler、KV cache 和 scheduler 干扰。

## 精度差异的来源

### 预期差异（DLC Precision Difference）

- PGX 乘法输入可能先转 bf16（即使输入是 fp32）。
- 硬件 exp/rsqrt 是近似实现（约 2^-12 相对误差）。
- matmul tiling 和累加顺序与 CPU 不同。
- Attention 后端算法不同（DLC Attention Backend vs FlashAttention）。

### 真正的算子 bug

- NaN/Inf 在不应该出现的位置出现。
- 非连续 tensor（Channels-Last Strided Tensor）处理错误。
- KernelDesc 参数协议不一致（scalar lite/full、shape/stride 错误等）。
- optimizer foreach/multi-tensor 路径对完整参数组依赖不正确。

## 定位流程

### 从模型问题到算子问题的标准流程

1. **定位第一个非有限张量**：不要只看最终 loss/输出，要从 input、forward、loss、backward、grad sync、optimizer.step 顺序找最早的 NaN/Inf。
2. **保存现场**：记录 input、output、中间变量、shape、stride、dtype、device、layout。
3. **识别高风险路径**：优先看 non-contiguous、channels-last、broadcast、foreach tensorlist。
4. **Model-Site Dump**：用 dump 脚本捕捉 failure 点的 tensor 状态。
5. **收敛到 pytorch_test**：把大模型问题压缩成 pytorch_test 可复现 case。
6. **单变量定位**：用 enabled_kernels.hpp dispatch fallback 确认单个算子边界。
7. **重复**：修复一个边界后，跑 1-token dump 确认下一个边界。

### 对 optimizer/foreach 问题的特殊注意

- 保存完整参数组和 tensor list 顺序。
- 避免过重 dump 改变异步调度，必要时使用 Lazy Dump。
- 同时记录 workaround，它能帮助缩小问题范围。

### 判断是 DLC Precision Difference 还是 bug

| 问题类型 | 判断方式 |
|----------|---------|
| matmul/attention 类 | 优先考虑 Hardware-Aware Reference |
| PGX 相关 | 检查是否需要 bf16 转换 |
| 非有限值位置差异 | 优先看 Finite Mask Mismatch |
| 小 scale 差异被放大 | 看 RMSNorm、SiLU 等非线性/非均匀放大路径 |
| MoE token 分叉 | 核对 expert routing、EP rank、activation type、clamp limit 和 quantization metadata |

### 生成 token 分叉的特殊注意

- 先建立严格等价 API 对照：相同 endpoint、prompt/token IDs、tokenizer、模型权重、TP/EP、`temperature=0` 和 `max_tokens`。
- 不要混用 `/v1/chat/completions` 与 `/v1/completions` 结果直接比较；chat API 会应用 chat template。
- 若第 N 个生成 token 分叉，将分叉前 token 拼入 prompt，仅生成 1 个 token，先判断问题是否可变成单步 prefill。
- 比较 raw logits `argmax` 和 sampled token；两者一致时不要继续优先怀疑 sampler。
- MoE 路径按 `layer -> branch -> EP rank -> expert -> W13/activation/W2` 收敛，并核对模型配置到 DLC Custom Op / DLC Custom Kernel 的语义契约。

## CPU Reference 策略

### 为什么 CPU 是主要 oracle

- CPU 路径固定、可复现、不受 GPU numerics 影响。
- CUDA 路径在图像 resize、patch embedding、位置编码插值、attention implementation 上可能与 CPU 有细微差异。
- 多模态模型对图像输入极其敏感，前处理差异会被视觉 encoder 放大。

### 使用方式

1. 在同一套 processor inputs 下运行 CPU 和 DLC。
2. 用 compare report 对比每个 module 的 tensor 输出。
3. 找到第一个 divergent module（abs_diff_max 不为 0）。
4. 对该 module 展开做算子级 replay。

### 何时用 CUDA

- 作为补充参照，帮助判断差异是否 DLC 特有。
- 必须在确认 CUDA 和 CPU/DLC 的 processor inputs 完全一致后使用。

## 常见坑

1. **把预期差异当 bug**：matmul bf16 转换造成的差异可能是预期的。
2. **看最终 loss 而不是第一个异常张量**：后面的差异可能是前面的差异放大。
3. **CUDA 输入不一致**：多模态模型的图像读取路径在不同设备上可能不同。
4. **过重 dump 遮盖 bug**：同步/拷贝可能改变异步执行顺序。
5. **tolerance 设置过严**：kernel owner 通常接受 `("rpd-filter2sigma-p95", 1e-4, 1e-4)`，过严的 `atol=1e-6` 可能被拒绝。

## 相关资料

- [precision-debugging/model-site-dump-to-repro.md](model-site-dump-to-repro.md)
- [precision-debugging/token-divergence-and-moe-contract-debugging.md](token-divergence-and-moe-contract-debugging.md)
- [operator-dispatch/enabled-kernels-dispatch.md](../operator-dispatch/enabled-kernels-dispatch.md)
- [testing/dlc-kernel-test-framework-guide.md](../testing/dlc-kernel-test-framework-guide.md)
- [case-studies/](../case-studies/)

## 来源

- `/work/plan/dlc基础/DLC基础知识手册.md` 精度速记部分
- `/work/plans/rsthinker_post_conv_embeddings_drift说明.md`
- `/tmp/rsthinker_dlc_precision_handoff_20260623_cpu_oracle_update.md`
