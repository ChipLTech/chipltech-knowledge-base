# Hy3 GPTQ-Int4 TP8 与 Graph 生命周期适配

## 案例范围

本案例从一份 2026-08-21 提交的 Hy3 GPTQ-Int4 工程复盘中提炼两类可跨模型复用的经验：量化 checkpoint 在 Tensor Parallel 分片下的 metadata ownership，以及 vLLM-CL Graph 模式异步故障的生命周期收敛方法。

Evidence status: `contributor_report_derived_historical_evidence`。贡献材料 SHA-256 为 `2e2c0021432b655c251a2c1672183eac76d1c1fa89eb29b8d0ffcaa1feb08d1f`，提交时 Host locator 为 `/home/xuansun/report/hy3-gptq-int4-tp8-graph-adaptation-case-study.md`。该 locator 不是仓内或跨环境稳定地址；digest 只绑定本次 review input。贡献材料记录了 source revision、wheel digest、Real DLC Hardware artifact 路径和正式 workload，但这些外部 artifact 未随知识库提交，维护者未在本次知识贡献中重新执行模型或复核全部原始日志。因此，文中运行结果是带 identity 的 contributor assertion，不是当前环境 runtime observation，也不能继承为新 revision、新 topology 或新模型的 acceptance。

## 报告中的历史闭环

贡献材料报告 Hy3 GPTQ-Int4 在 8 张 TYD Chip 上完成普通 TP8 和 `FULL_DECODE_ONLY`：

- checkpoint 的 GPTQ group128 被无损表示为 execution group64，35/35 shards 完成加载；
- Prefill、Decode、KV Cache 和 TP collective 路径完成，中文、英文和算术语义请求通过；
- Graph profiling、capture、服务 ready 以及单并发 `512 input / 512 output` 10/10 无插桩请求通过；
- 报告记录该 workload 的 TTFT 513.18 ms、TPOT 59.18 ms/token、ITL 59.30 ms、输出吞吐 16.65 tok/s。

这些数字只绑定贡献材料声明的 exact source、PyTorch 2.5.0 wheel、8 张 TYD Chip、普通 TP8、`FULL_DECODE_ONLY`、单并发 `512/512` 和对应 init/topology epoch。它们不是硬件上限、稳定公共 baseline，也不覆盖 Chunked Prefill、Prefix Cache、其他 Graph mode、并发或 TP2 x EP4。

贡献材料声明的核心 identity：vLLM-CL base `bf6770c339d7983b62d9698479d0cd98d4f742e2`，PyTorch DLC Backend `c467792406db04bc609c7f002a851d73f25a60d3`，upstream vLLM `852b0ddb6`，PyTorch wheel SHA-256 `4a1474bf6425dac196c48a12c4865dce16451d82a234f0c9ef60b0c82bd553e9`。upstream identity 在贡献材料中不是 full SHA，因此不足以独立恢复 Tested Revision；外部 artifacts 也仍为 `external_unavailable`。这些字段用于限制历史 claim，不升级其证据等级。

## 量化分片：先证明 ownership，再处理 alignment

### 问题边界

案例中的 routed MoE intermediate width 为 1536，checkpoint GPTQ group size 为 128，TP8 的 rank-local logical width 为：

```text
1536 / 8 = 192
192 % 128 = 64
```

rank 边界落在 checkpoint group128 内部。直接切 qweight、scale 和 qzero 会让一个量化组跨 rank，权重 ownership 与 metadata ownership 不一致。

把 logical width 192 padding 到 DLC Custom Kernel 支持的 physical width 256 只能满足 shape alignment，不能修复 metadata ownership。量化适配必须分成两个独立 contract：

```text
quantization semantics: logical channels 与 scale/qzero ownership
kernel execution: physical width、alignment 与 zero-contribution padding
```

### 无损 group refinement

该 checkpoint adapter 把 group ownership 表示为 packed axis 上的连续 channel groups；贡献材料未单独记录 `g_idx`、`desc_act` 或其他 activation-order/permutation identity。因此，下述无损细化只对“group lookup 与连续 packed-axis ownership 一致”的 adapter class 成立，不能由本案例外推到带显式重排的 GPTQ checkpoint。

一个 group128 内的 channel 共用相同 scale 和 zero-point。把它细化为两个连续 group64 时，qweight 不变，只复制 metadata：

```text
[C0 ... C127] -> S0, Z0

[C0  ... C63 ] -> S0, Z0
[C64 ... C127] -> S0, Z0
```

TP8 每个 rank 的 192 channels 因而拥有 3 个完整 group64。该变换没有重新量化；它保留反量化语义的前提是 refined group 必须完整划分 checkpoint group，并且 group-index/permutation identity、packed-axis ownership、metadata lookup equivalence、slice ownership 和 qweight invariance 都有测试证明。若存在 `g_idx`、`desc_act` 或其他 permutation，必须先证明其转换与 refined groups 等价，不能只复制 scale/qzero。

本案例采用的边界是：

```text
checkpoint group size: 128
effective execution group size: 64
logical rank width: 192
physical kernel width: 256
```

稳定实现原则：

1. 在 TP slicing 前 refinement scale/qzero，保持 qweight byte/element identity。
2. 闭合 `g_idx`、`desc_act`、packed-axis permutation 或证明它们不适用，再从 checkpoint group、logical partition 和 kernel capability 推导 effective group；不按模型名硬编码。
3. 将 logical data 写入 zero-initialized physical parameter，显式断言 padding tail 为零贡献。
4. 复制每层 quant config，避免 effective group mutation 污染共享配置。
5. 对本来可按 checkpoint group 完整分片的 profile 保持原行为。
6. effective group 无法完整覆盖 logical partition，或低于 exact DLC Custom Kernel capability 时，launch 前 fail closed。

PyTorch DLC Backend 在案例中保持 Public Operator Schema 和 DLC Custom Kernel Entry ABI 不变，只在 host launch 前验证 expert 维、logical dimensions、group metadata coverage、expert map 和成对 optional inputs。这说明 shape adaptation 与 descriptor validation 是互补门禁：前者构造合法表示，后者阻止 malformed representation 进入异步 DLC Runtime。

## Graph 故障：沿生命周期找首个未完成边界

### 传播错误不是根因

Graph 模式最初在 513/527 token Prefill 出现 worker 不返回，随后报告 shared-memory broadcast timeout、RPC timeout、XYS index 越界和 `synErrorLaunchFailure`。这些 observation point 可能晚于真正的异步错误。

案例使用以下顺序收敛，而不是从最后一条 kernel 日志猜测：

1. 用 `513 input -> 1 output` 建立低成本 red loop。
2. 在相同 model、weights 和 TP8 下做 Eager/Graph differential。
3. 在每个 decoder layer 的 parent boundary 做 capture-safe diagnostic synchronization。
4. 80 层、8 rank 全部完成后，把边界缩到 final norm 之后的 logits/sampling。
5. 分别在 LM Head local matmul 后和 TP logits aggregation 后同步。
6. local matmul 全部完成而 aggregation 不返回后，只声明“该模型 Graph 生命周期组合下的 logits collective boundary 不稳定”。
7. 用独立 8-rank direct harness 验证同名 AllGather 在 Eager、Graph 和 replay 下可完成，拒绝外推为通用 DLCCL/DLC Custom Kernel 缺陷。

这套方法的关键不是“增加同步”，而是构造父子生命周期 bracket。diagnostic epoch 必须记录是否到达原目标边界；若更早停止，它只建立一个新 failure boundary。定位完成后删除 instrumentation，再运行无插桩 acceptance workload。

### 受限 collective adapter

案例把 rank-local vocabulary logits 从 AllGather 改为 disjoint-range AllReduce：每个 rank 在完整 vocabulary tensor 的自身区间写 local logits，其他区间写零，然后求和。因为各区间只有一个非零 owner，结果与按 rank 拼接等价。

该 adapter 的可复用决策条件是：

- output ranges 完整、互斥，并与 rank-local vocabulary ownership 一致；
- 非 owner ranges 严格为零；
- dtype、shape、rank order 和 collective route 已闭合；
- helper semantics、multi-rank equivalence 和真实模型路径分别验证；
- adapter 受 exact workload predicate 限制，不替换所有模型的默认 collective。

direct harness 能通过意味着通用 kernel 缺陷未建立；它不否定模型生命周期组合中的兼容问题。修复 owner 应落在最小模型/plugin adapter，而不是无证据修改 DLCCL 或 DLC Custom Kernel。

### Graph profiling 与真实执行必须同语义

案例还发现 `DLCRotaryEmbedding.forward_native` 在 Graph profiling/capture 时回退到 upstream tensor indexing 路径，而真实 DLC Platform 路径使用 DLC rotary 实现。最终让 native dispatch 委托 DLC rotary，使 profiling、capture 和 execution 共享平台语义。

因此 Graph adaptation 必须审计至少三条 route：profile、capture 和 replay/execute。某条 route 名为 `native` 不代表它天然适用于 DLC Platform；route equivalence 要由 exact source 和执行证据建立。

## 验证阶梯与现场止损

本案例形成的低成本到高成本阶梯是：

```text
helper semantics
-> direct op / descriptor guard
-> single layer or rank
-> fresh-process C1b per requested device
-> TP collective content correctness
-> real weights and minimal semantic requests
-> 513/1
-> 527/16
-> one 512/512
-> two consecutive 512/512
-> ten uninstrumented 512/512
```

每个 epoch 记录 Hypothesis、Prediction、one variable changed、command、actual result、first fatal boundary、artifact 和 conclusion。不能让真实 red loop 转绿的实验 patch 应回退，不作为“预防性修复”留在 production diff。

hang 后若 fresh-process C1b 也停在 DLC Runtime completion，模型实验必须暂停。HBM 配置成功、设备节点存在或 `cltech_smi` 可见不等于 LYP/TP8 可执行。Host reboot、`cltech-init`、LYP repair 和非任务进程清理属于授权操作；恢复后应重新通过 LYP terminal state 和每张请求设备的 fresh-process C1b，才能继续模型实验。

## 应写入新模型适配 Contract 的检查项

- 量化：checkpoint group、`g_idx`/`desc_act`/packed-axis permutation、effective group、logical partition、physical shape、scale/qzero lookup equivalence 和 qweight invariance 分别闭合。
- Graph：profile/capture/replay routes、scheduler state、final worker environment、last successful stage 和 first fatal boundary 分别记录。
- Collective：模型 route 的失败与同名 primitive 的普遍缺陷分开证明；fallback/adapter 必须有等价性和 bounded predicate。
- Identity：source、package、plugin entrypoint、wheel digest、worker environment、topology 和 init epoch 共同绑定结果。
- Acceptance：服务 ready、单请求语义、连续稳定性和性能 workload 分别报告；diagnostic timing 不进入正式性能。

## 不应泛化

- group128 一定可以或应该细化为 group64；effective group 必须由目标 shape 和 exact kernel capability 推导。
- 所有 TP logits AllGather 都应改成 AllReduce。
- 513 token 是通用 Graph 边界，或 `FULL_DECODE_ONLY` 是所有模型的推荐配置。
- 同名 AllGather、rotary、MoE kernel 在所有 source、topology 和 lifecycle 下具有相同结论。
- 本案例的 TPOT、吞吐或 10/10 能代表其他 workload、current main 或硬件上限。

## Claim Boundary

本案例建立 contributor report 中记录的历史适配设计和可复用诊断门禁。它不建立当前 source、package、DLC Runtime、Real DLC Hardware、LYP、模型正确性、性能、TP2 x EP4、image 或 release 状态。任何新 target 必须重新闭合 identity、capability、Graph route、collective semantics、无插桩 acceptance 和性能口径。

## 相关资料

- [模型适配与 Main-to-Main 决策记录](../vllm-cl/model-adaptation-and-main-to-main-decisions.md)
- [Model Adaptation Prompt](../prompt-examples/vllm-cl-model-adaptation.md)
- [异步 Launch Failure 定位 Prompt](../prompt-examples/vllm-async-launch-failure-localization.md)
- [DLC Runtime 排障](../runtime-debugging/runtime-troubleshooting.md)
- [性能 Profiling](../runtime-debugging/performance-profiling.md)
- [Distributed Collective Qualification](../vllm-cl/distributed-collective-qualification.md)
