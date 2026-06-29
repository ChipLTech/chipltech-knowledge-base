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

## 相关资料

- [runtime-debugging/runtime-troubleshooting.md](../runtime-debugging/runtime-troubleshooting.md)
- [debugging-workflows/common-debug-commands.md](../debugging-workflows/common-debug-commands.md)

## 来源

- `/work/plan/newraw/TPU Profile 工具简介.docx`（已转换为 Markdown）
