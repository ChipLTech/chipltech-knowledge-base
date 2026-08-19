# vLLM 性能热点分层定位 Prompt

## 用途

用于 vLLM-CL serving 的 decode latency、TTFT、TPOT/ITL 或 throughput 回归。目标是从端到端逐层收敛到最小热点边界，并区分插桩定位证据与无插桩正式 benchmark。

## 可复制 Prompt

```markdown
使用 `/diagnosing-bugs` 定位这个 vLLM-CL 性能问题。先读取当前知识库的 `CONTEXT.md`、`runtime-debugging/performance-profiling.md`、`runtime-debugging/chipltech-smi-observability.md` 和最近的性能 case study。

【模型名称和绝对路径】<MODEL_AND_PATH>
【完整服务命令】<FULL_SERVE_COMMAND>
【固定请求或 benchmark 命令】<WORKLOAD_COMMAND>
【性能现象或可比较 baseline】<SYMPTOM_OR_BASELINE>
【artifact 目录】<ARTIFACT_DIRECTORY>
【已有 DLCSynapse log】<SYN_LOG_PATH_OR_AUTO_DISCOVER>

按以下顺序执行：

1. 封存 source/package/image/model、最终 worker 环境、Real DLC Hardware 映射、TP/PP/EP、dtype/quantization、输入输出 token policy、batch/concurrency/request rate、sampling/seed、warm-up、attempts 和 server epoch。
2. 先建立可重复的无插桩 baseline，记录 correctness、server liveness 和实际可测指标。普通非流式请求不能凭空给出 TTFT、TPOT 或 ITL；需要时使用可审计 benchmark/streaming/metrics 入口。
3. 另建 diagnostic profile，保存它与 baseline 的精确 diff。强制 device synchronization、blocking、debug print、verbose trace 和临时 wrapper 只用于定位，不作为正式性能结果。
4. 从 request/scheduler -> model forward -> block/layer -> stage -> framework wrapper -> DLC Attention Backend/PyTorch DLC Backend/vLLM-CL Custom Op -> DLC Runtime/DLCSynapse -> DLC Custom Kernel 逐层缩小。每轮只深入当前已确认最慢的边界。
5. 每个 timer 标记 inclusive/exclusive、parent、request、layer/module、TP rank、prefill/decode、shape、dtype、stride/layout、同步位置和时间单位。对 layout/view/alias/materialization 候选，在可观测时同时记录 source/destination contiguity、storage identity、storage offset 和 logical view relationship。
6. 比较 parent total、covered child intervals 和 residual。稳定 residual 先列出重复执行、预/后处理、layout conversion、copy、异步等待、queue/collective wait 和计时定义差异等候选，不提前归因。
7. 如果已有 `syn_*.ansi` 或 `.log`，使用 `diagnosing-bugs/scripts/export-dlc-kernel-csv.py` 和 `/home/xuansun/llama2-fine-tune/tool.py`，只生成 `operators.csv`。时间统一按 `tool.py` 的 1400 MHz 口径，不使用 `table.py` 的 1500 MHz 时间换算，不生成 kernel 文本、图片或 manifest。对疑似热点统计调用来源、调用次数、累计/平均耗时和输入身份；kernel 名或聚合排名只能提供线索。
8. 审计 KV cache update、output materialization、layout conversion、quantize/dequantize、collective 和 state update 的跨层执行所有权。声明必须与实际调用行为一致。
9. 每个修复只改变一个变量，并同时验证 correctness、调用/ownership contract 和局部 timing。
10. 删除 debug print、强制同步、blocking 和临时 wrapper，用原封存 workload 重跑无插桩端到端 benchmark。稳定 baseline 或回归 claim 需要声明的重复 attempts 和离散度报告。

输出：
- workload/identity manifest。
- 无插桩 baseline 与 diagnostic profile diff。
- 分层 timing 表和计时边界语义。
- 调用次数、parent、shape/layout、rank、phase 表。
- 按总 cycles 排序的 DLCSynapse `operators.csv`；需要审计时保存含源 log/tool/CSV digest 的命令 stdout，没有可解析 log 时明确 `not_verified`。
- 已确认根因或最小未决边界。
- correctness 与 ownership 验证。
- 无插桩 benchmark 结果，未执行时明确 `not_verified`。
- `confirmed`、`high-confidence`、`not_verified` 和 Claim Boundary。

不要预设 Attention、KV cache 或某个 kernel 是根因；不要把单次 profile、microbenchmark 或强制同步下的时间外推成生产 TTFT/TPOT/throughput。需要 reset、LYP repair、重启或清理非任务进程时必须先取得独立授权。
```

## 相关资料

- [性能 Profiling](../runtime-debugging/performance-profiling.md)
- [vLLM Attention 重复 KV Cache Update 案例](../case-studies/vllm-attention-duplicate-kv-cache-update.md)
- [vLLM Hybrid KV Cache 非连续输出适配案例](../case-studies/vllm-hybrid-kv-cache-strided-output.md)
- [Arsenal benchmark 与黑盒测试](../testing/arsenal-ci-and-blackbox-testing.md)
