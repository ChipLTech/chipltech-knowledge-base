# Chipltech-Family Accelerator 性能 Profiling

## 适用场景

- 分析模型在 DLC Platform 上的端到端性能瓶颈。
- 定位 vLLM 推理中的调度、算子、数据搬运、硬件资源利用问题。
- 做优化前后的性能对比。

## 当前事实、目标架构与未验证范围

当前静态源码和已有 artifact 可直接证明的是：DLCSynapse 能生成 Chrome Trace Event JSON，记录 complete event 的名称、逻辑 track、时间戳、持续时间和字符串参数。该格式可由 Perfetto 等 Chrome trace 工具读取，但这不等于当前产品已经闭合原生 Perfetto protobuf、request/rank/device 语义或硬件 counter。

统一展示 vLLM、PyTorch、DLC Runtime、dlc-thunk、DLC Custom Kernel 和硬件 counter 是目标架构。未绑定 exact producer 与 artifact 的 request、phase、rank、device 和 counter scope 一律为 `not_verified`。

Chrome trace 中的 `pid`/`tid` 在当前 producer 中可能编码 stream、thread 或 category track。validator 只称其为 trace track，不得据此推断 OS PID/TID、rank 或 device。

### Trace Artifact Validator 最小合同

- 只读消费实际存在的 trace，不调用模型或修改 trace。
- 分开报告 JSON syntax、exact byte SHA-256 和各 localization scope。
- `dlcProfilerStart/Stop` success 不是 trace 产生证据；必须有实际文件和匹配 digest。
- 缺 request/rank/device producer 不破坏 trace syntax，只使相应 scope 为 `not_verified`。
- diagnostic profile 不得升级为 formal benchmark。

## 工具定位

DLC Profile 不是单一算子的 microbenchmark 工具，而是**端到端性能观测工具**。它把模型请求、框架调用、runtime 调度、driver 调用、kernel 执行窗口和硬件 counter 放在同一条时间线上，帮助定位真实业务负载中的性能瓶颈。

在 LLM 推理场景中，Profile 可围绕以下关键阶段展开：
- vLLM worker
- PyTorch forward
- prefill / decode
- decoder layer
- attention / MoE / FFN
- runtime queue
- kernel launch
- 同步等待
- 互联 counter

## 目标技术栈分层

目标技术栈采用"框架 profile + runtime 内部打点 + dlc-thunk/kernel trace + 硬件 counter track + Perfetto 可视化/离线分析"；当前 checkout 未证明每层均已接通：

```
vLLM / PyTorch Profile  →  框架侧语义（request、batch、layer、operator）
Runtime Profile          →  schedule、queue、stream、sync、memory copy
Driver / Kernel 打点     →  driver 调用、kernel launch、执行窗口、completion
Hardware Counter Track   →  带宽、利用率、队列深度等硬件指标
Perfetto 分析层          →  UI timeline、Trace Processor、版本对比
```

## 工作流

1. **选择 workload**：指定模型、输入输出长度、batch size、并发、精度、版本和硬件配置。
2. **开启 profile 配置**：在 runtime 内启用各层打点。
3. **运行模型或 benchmark**：仅在 owning workflow 与授权允许时采集；否则消费已有 trace artifact。
4. **校验 trace**：先闭合 syntax、byte digest 与实际具备的 localization scope。
5. **可视化或离线分析**：只解释已闭合的 trace-track/semantic scope；缺失 counter 保持 `not_verified`。

## 分层热点定位方法

Profile 工具给出时间线，热点定位还需要一套从端到端向下收敛的决策流程。不要从某个 DLC Custom Kernel 名称开始猜根因。

### 1. 固定 workload 与无插桩 baseline

先封存：

- image、source、package、模型资产和 dirty state identity。
- Real DLC Hardware、物理/逻辑设备映射、TP/PP/EP、dtype、quantization。
- server/client 完整命令和最终 worker 环境。
- prompt/input/output token policy、batch、并发、request rate/count、sampling 和 seed。
- warm-up、formal attempts、timeout、server epoch 和 correctness/liveness assertion。
- TTFT、TPOT、ITL、request latency、token/request throughput 中实际可测的指标及测量工具。

普通非流式请求通常只能直接给出总请求时间，不能据此生成 TTFT、TPOT 或 ITL。需要这些指标时使用带时间戳的 streaming client、vLLM benchmark client、server metrics 或其他可审计测量入口。

### 2. 从端到端逐层缩小

```text
request / scheduler
-> model forward
-> block / layer
-> stage（Attention、MLP、MoE、cache、communication 等）
-> framework wrapper / unified operation
-> DLC Attention Backend、PyTorch DLC Backend 或 vLLM-CL Custom Op
-> DLC Runtime / DLCSynapse launch、copy、queue、sync
-> DLC Custom Kernel
```

这不是所有调用都会完整经过的固定直线。host metadata、layout conversion、Python/framework scheduling 和 collective wait 可能在 DLC Custom Kernel 之外形成热点。每轮只深入当前已确认最慢的边界。

### 3. 闭合异步计时区间

host wall-clock timer 不自动等于设备执行时间。诊断时可使用明确的 device synchronization 或 profile event 闭合区间，但必须记录：

- 同步位置和计时区间。
- inclusive 或 exclusive time。
- parent/child 是否使用相同同步策略。
- layer、rank、prefill/decode、shape 和时间单位。
- warm-up 与 steady-state 样本。

强制同步、blocking、debug print 和 verbose trace 会改变调度与 overlap。它们产生的是 **instrumented localization evidence**，不能直接称为生产性能 baseline。

### 4. 对齐相邻抽象层

同时比较 parent total、children sum 和未解释 residual：

```text
residual = parent inclusive time - covered child intervals
```

稳定 residual 说明相邻边界的工作集合或等待归属不一致。候选解释包括：

- 同一语义操作重复执行。
- wrapper 的预处理、后处理或 output materialization。
- layout conversion、copy、quantize/dequantize。
- 前一个异步操作在当前同步点等待。
- queue、collective 或 host scheduling wait。
- parent/child 计时区间定义不一致。

residual 与某个子操作耗时接近只是强线索；在调用次数和来源确认前，不得直接写成重复执行根因。

### 5. 用调用身份、次数和 shape 完成归因

对候选热点至少保存：

- request、phase、layer/module、parent call、TP rank 和设备。
- 调用次数、累计耗时、平均耗时和分布。
- input/output shape、dtype、stride/layout。
- 对 layout、view、alias 或 materialization 候选，在可观测时同时保存 source/destination contiguity、storage identity、storage offset 和 logical view relationship。相同 storage pointer 不等于相同 logical Tensor。
- framework op、DLC Custom Op 和 DLC Custom Kernel 的准确身份。
- 是否包含同步、copy、materialization 或 collective wait。

同名 DLC Custom Kernel 可服务多个 layer、shape 和模型路径。kernel name 或聚合耗时排名只能提供候选，不能单独证明模型热点归属。

### 6. 审计跨层执行所有权

相邻 wrapper/backend 对以下工作必须有唯一且与实际实现一致的 owner：

- KV cache update。
- output allocation/materialization。
- layout conversion。
- quantize/dequantize。
- collective。
- normalization/fusion。
- recurrent/state update。

声明的 capability/flag 与实际调用行为不一致时，一个正常 DLC Custom Kernel 也可能因重复执行成为端到端热点。修复目标是恢复执行 contract，而不是默认修改某个固定 flag。

### 7. 修复后的三重验证

每个性能修复必须同时验证：

1. **Correctness**：输出 token、cache/state 和适用语义断言保持正确。
2. **Call contract**：调用次数、parent/child ownership 和 runtime state 符合预期。
3. **Performance**：移除临时同步、print、wrapper 和 debug 配置后，用原封存 workload 重跑端到端指标。

## 性能证据与 Claim Boundary

| 证据 | 可以证明 | 不能自动证明 |
|---|---|---|
| Instrumented localization | 慢边界、调用顺序、次数和候选 residual | 生产 TTFT/TPOT/throughput 收益 |
| Perfetto / DLCSynapse trace | timeline、queue、copy、sync、kernel 关联 | 单变量因果关系 |
| Microbenchmark | 固定输入下算子/kernel 成本 | 模型端到端收益 |
| Uninstrumented benchmark | exact workload 下端到端性能 | 其他模型、长度、并发或稳定性 |
| Repeated stability benchmark | 声明 workload 下的分布和离散度 | 未执行 profile 或其他硬件配置 |

诊断 profile 与正式 benchmark profile 必须作为独立 epoch，保存精确 profile diff。单次请求或单次 profile 可以定位线索；稳定性能、回归或 baseline claim 需要声明并完成重复 attempts 和离散度报告。

## 常见误判

1. 从 kernel 名或聚合排名直接猜模型根因。
2. 将 inclusive parent/child 时间直接相加。
3. 用 host timer 测异步执行却不记录 completion 边界。
4. 看到稳定 residual 就直接断言重复执行。
5. 把强制同步和 debug print 下的数字称为生产 latency。
6. 只看单层平均耗时，不乘以请求中的调用次数和层数。
7. 只验证局部 latency，不验证 correctness、throughput 和端到端指标。
8. 用一个短请求声明稳定性能收益。

## 可观测能力

| 能力 | 说明 |
|------|------|
| 端到端请求路径 | vLLM → PyTorch → runtime → driver → kernel 在同一条 timeline |
| 模型层级与算子关联 | 按 request、layer、operator、attention、MoE 观察嵌套关系 |
| runtime 调度分析 | queue 等待、stream 切换、同步点、host-device copy、空泡 |
| driver/kernel 关联 | 算子与底层 kernel 事件对齐，判断瓶颈来源 |
| 硬件 counter 同屏 | 互联、利用率等 counter 与模型事件同屏 |
| 离线聚合 | Trace Processor / SQL 统计热点、慢路径、优化前后差异 |

## Trace 示例解读

典型 trace 中一个请求呈现为多层嵌套 slices：
```
vLLM worker
  → PyTorch forward
    → runtime schedule / queue
      → driver / kernel
        → attention / MoE 等模块
```

硬件 counter 以 counter track 形式在同一时间轴上，可以判断耗时来源是模型结构、runtime 调度、kernel 执行、数据搬运还是硬件资源瓶颈。

## Chunked Prefill 与 FlashAttention 优化上界

在长上下文 vLLM serving 中，不要只看单个 DLC Attention Backend / FlashAttention 类 kernel 的 microbenchmark 收益来判断端到端 TTFT 收益。真实端到端路径还包含：

- copy / to / clone / contiguous 等数据搬运和张量物化。
- Chunked Prefill 循环中的 host 调度和 DLC Runtime 调度。
- TP 通信编排和等待。
- cache 管理、cat、index_select、reshape_and_cache 等非核心 compute 路径。
- matmul、allreduce、norm、activation 等剩余热点。

同事在 `Llama3.1-8B`、64k prompt、TP=2、Chunked Prefill 场景中的评估显示：单 kernel 中 softmax+PV 是 FlashAttention 内部主要部分，但将整个 FlashAttention 近似清空后，端到端 TTFT 只改善约 9%。因此，BLASST 这类只覆盖 FlashAttention 内部子阶段的优化，在该场景下的端到端收益上界会明显低于单 kernel 收益，合理预期应以几个百分点为量级，而不是按 kernel 内部比例直接外推。

分析这类问题时建议同时保存：

- workload：模型、prompt 长度、输出长度、TP/PP、`max_model_len`、`max_num_batched_tokens`、prefix caching、并发。
- 请求指标：TTFT、latency、throughput、server liveness。
- kernel 汇总：attention、matmul、allreduce、cache、cat/index_select、norm、activation。
- trace 观察：copy/to/clone/contiguous、host scheduling、DLC Runtime queue/sync、TP 等待。
- chunk size 对比：如果 `max_num_batched_tokens` 增大后 TTFT 反而变差，优先检查单步 tensor 尺寸、数据搬运、cache 写入和 TP 通信颗粒，而不是只归因于 chunk 数量。

## Arsenal vLLM Benchmark 指标入口

`/work/arsenal/vllm_benchmark_serving_script/` 提供了一个轻量 pipeline：启动 vLLM server、运行 `benchmark_serving.py`、从 client log 中提取 CSV。适合做模型 serving 性能横向对比，重点指标包括 request throughput、token throughput、TTFT、TPOT 和 ITL 的 mean/median/P99。

使用时必须同时保存 server command、client command、模型路径、输入/输出长度、请求数量、dtype、TP/PP、Chunked Prefill、prefix caching 和日志目录。benchmark success 只证明该 workload 下服务跑完，不证明模型正确性、DLC Runtime dispatch 或 Real DLC Hardware acceptance。

## 相关资料

- [runtime-debugging/runtime-troubleshooting.md](../runtime-debugging/runtime-troubleshooting.md)
- [debugging-workflows/common-debug-commands.md](../debugging-workflows/common-debug-commands.md)
- [testing/arsenal-ci-and-blackbox-testing.md](../testing/arsenal-ci-and-blackbox-testing.md)
- [debugging-workflows/synapse-log-and-kernel-summary-workflow.md](../debugging-workflows/synapse-log-and-kernel-summary-workflow.md)
- [case-studies/vllm-attention-duplicate-kv-cache-update.md](../case-studies/vllm-attention-duplicate-kv-cache-update.md)
- [case-studies/vllm-hybrid-kv-cache-strided-output.md](../case-studies/vllm-hybrid-kv-cache-strided-output.md)
- [prompt-examples/vllm-performance-hotspot-localization.md](../prompt-examples/vllm-performance-hotspot-localization.md)

## 来源

- `/work/plan/newraw/TPU Profile 工具简介.docx`（已转换为 Markdown）
- `/work/test/同事文档/DLC FlashAttention _ BLASST 在 vLLM Llama3.1-8B 64k Chunked Prefill 场景中的可行性评估.md`
- `/work/arsenal/vllm_benchmark_serving_script/`
- `/work/err/performance-hotspot-localization-case.md`
