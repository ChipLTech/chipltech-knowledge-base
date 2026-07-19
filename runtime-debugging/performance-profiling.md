# DLC-Family Accelerator 性能 Profiling

## 适用场景

- 分析模型在 DLC Platform 上的端到端性能瓶颈。
- 定位 vLLM 推理中的调度、算子、数据搬运、硬件资源利用问题。
- 做优化前后的性能对比。

## 核心结论

DLC Profile 工具链以 Perfetto 为核心，在 runtime 内对 vLLM、PyTorch、runtime、driver 与 kernel 执行事件进行统一打点，并将硬件 counter 导出到 Perfetto counter track。用于将模型请求、框架调用、runtime 调度、kernel 执行和硬件 counter 放在同一条时间线上分析。

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

## 技术栈分层

当前技术栈采用"框架 profile + runtime 内部打点 + driver/kernel trace + 硬件 counter track + Perfetto 可视化/离线分析"：

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
3. **运行模型或 benchmark**：生成统一 Perfetto trace。
4. **在 Perfetto UI 中查看**：观察调用栈、queue 等待、kernel 空泡、同步点、数据搬运、硬件 counter。
5. **通过 Trace Processor 离线分析**：统计热点算子、慢请求路径、counter 峰值区间。

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

## 来源

- `/work/plan/newraw/TPU Profile 工具简介.docx`（已转换为 Markdown）
- `/work/test/同事文档/DLC FlashAttention _ BLASST 在 vLLM Llama3.1-8B 64k Chunked Prefill 场景中的可行性评估.md`
- `/work/arsenal/vllm_benchmark_serving_script/`
