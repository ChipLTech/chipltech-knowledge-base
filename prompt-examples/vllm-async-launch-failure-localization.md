# vLLM 异步 launch failure 定位 Prompt

## 用途

用于 vLLM-DLC serving 已完成部分请求后出现 `synErrorLaunchFailure`、worker abort、EngineCore dead 或 HTTP 500 的场景。目标是找到第一 fatal 和最小执行边界，再用单变量实验捕获首个失败 DLC Custom Kernel。

## 可复制 Prompt

```markdown
使用 `/diagnosing-bugs` 处理这个 vLLM-DLC serving failure；涉及模型级 Attention、MLA、MoE、量化、MTP 或 distributed compatibility 时同时遵守 `model-adaptation` 的 claim boundary。

【Host/容器】<HOST_AND_CONTAINER>
【模型绝对路径】<ABSOLUTE_MODEL_PATH>
【完整启动命令】<FULL_SERVE_COMMAND>
【固定失败请求】<REQUEST_OR_CURL>
【服务日志】<SERVER_LOG_PATH>
【客户端输出】<CLIENT_OUTPUT_PATH>
【已有实验目录】<ARTIFACT_DIR_OR_NONE>

先读取当前知识库的：
- `CONTEXT.md`
- `runtime-debugging/runtime-troubleshooting.md`
- `debugging-workflows/synapse-log-and-kernel-summary-workflow.md`
- `runtime-debugging/chipltech-smi-observability.md`
- `case-studies/vllm-async-speculative-decode-launch-failure.md`

按以下顺序执行：

1. 建立 red-capable feedback loop。固定短 prompt、`temperature=0`、`top_p=1.0`、小 `max_tokens`，断言用户的精确症状；模型加载很慢时如实记录 loop latency，不用附近的初始化 failure 替代目标 failure。
2. 按 package/import、distributed init、checkpoint load、weight placement、KV cache、Graph capture、API ready、request admission、prefill、decode、worker exit、Engine death 切分生命周期，列出每段的通过证据和第一失败证据。
3. 找第一 fatal。把 cancellation、EngineDead、HTTP 500 和 cleanup warning 列为传播结果。异步 DLC Runtime 错误的报告 API 只记为 observation point，不直接当失败 kernel。
4. 保存 fatal scheduler state：computed/output/scheduled token count、speculative token count/placeholder、position、seq_len、slot mapping、accepted-token metadata、实际 eager/Graph dispatch 和 Graph size。placeholder 必须按当前安装源码解释。
5. 保存 image digest、PyTorch/vLLM/vllm-dlc/source identity、完整启动命令、预期环境和每个最终 worker 的 `/proc/<pid>/environ`。外层 shell 或 launcher 环境不作为 worker 环境的替代证据。
6. 形成 3-5 个可证伪假设。优先做 Graph vs eager，然后依次拆 async scheduling、speculative token count、IndexCache/Sparse MLA、量化 MoE/Attention，最后才进入逐设备或 DLCCL/LYP probe；每轮只改一个变量。
7. 每个 server epoch 独立保存 before_launch、after_ready、during_request、after_cleanup 的 SMI Observation Envelope、进程/端口/HBM/handles、请求响应和日志。SMI 只作为观测，不作为模型正确性证据。
8. 若 blocking/debug 使服务在初始化或权重加载阶段 stall，登记为新的失败边界并恢复最后一个能到达目标 failure 的 profile；不要将其写成原 decode failure 已前移。
9. 只有诊断变量确实到达最终 worker、run 到达原失败阶段并产生逐 rank `syn_*.ansi`/kernel summary 时，才报告最后成功和首个失败 DLC Custom Kernel。异步摘要末行只能作为候选。
10. 清理优先 TERM。KILL 只针对已授权且 PID、PGID、starttime、cmdline、namespace/cgroup 身份仍匹配的任务进程；清理后对照 baseline 验证进程、端口、handles 和 HBM。

输出三个文件：
- 短摘要：问题点位、直接错误、影响、已排除项、尚未确认项。
- 全面技术报告：完整证据、源码语义、假设、每个 epoch 和 claim boundary。
- 过程资产：实际排查时间线、无效路径、安全清理、可复用方法和下一接手动作。

所有结论标记 `confirmed`、`high-confidence` 或 `not_verified`。没有 blocking completion evidence 时，不命名具体失败 kernel，不把 worker rank 写成物理设备故障，不把新初始化 stall 合并为原 decode 根因。未经独立授权不执行 reset、LYP repair、驱动重载、重启或清理非任务进程。
```

## 交付判据

- 一条已经实际运行并捕获目标症状的固定请求命令，或明确记录无法建立反馈回路的 blocker。
- 一个精确到生命周期、scheduler 和 eager/Graph mode 的最小失败边界。
- 每个实验都说明是否到达目标边界以及只改变了什么。
- 首个失败 kernel 有逐 rank blocking/同步证据，或明确保持 `not_verified`。
- cleanup 与 sealed baseline 闭环。

## 相关资料

- [Runtime 排障指南](../runtime-debugging/runtime-troubleshooting.md)
- [Synapse log 与 kernel 摘要工作流](../debugging-workflows/synapse-log-and-kernel-summary-workflow.md)
- [vLLM async speculative decode launch failure](../case-studies/vllm-async-speculative-decode-launch-failure.md)
