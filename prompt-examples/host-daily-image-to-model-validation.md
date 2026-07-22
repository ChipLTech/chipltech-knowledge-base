---
prompt_schema: host-daily-image-model-validation/v2
role: operational_runbook
---

# Host Daily Image 到模型验证 Runbook

## 用途

本文只拥有 Host/container preparation、C1a/C1b、real-weight functional、benchmark 和 cleanup 的执行 mechanics。状态、claim 和 image delivery 语义由 [规范 Contract](../vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md) 定义。

## 输入

```text
model_name
absolute_local_model_path
immutable_daily_base_reference_or_local_image
artifact_root
device_scope
functional_assertions
benchmark_workload
authorization_matrix
```

未提供的非敏感运行字段可自动发现并写入 contract；变更授权不能自动推导。

## 1. Host 与 Ordinary Daily Base

1. 记录 OS/kernel、Docker client/server、存储、设备节点、hardware generation、occupancy、HBM、已有 containers/ports。Complete when：目标设备可唯一映射且 baseline 已保存；busy 则 `blocked_missing_hardware`。
2. 只读检查 local images；获 pull 授权时才 pull。记录 Image ID，repo digest 在可用时记录。Complete when：base 通过 contract 的 ordinary eligibility；否则 `blocked_unqualified_daily_base`。
3. 创建 Host 持久 `src/build/wheels/artifacts/logs`。模型只读挂载，可写 cache 独立。Complete when：artifact root 位于源码树外且运行契约已脱敏保存。
4. 创建 task-owned persistent container；最小 mount/device/ipc/shm/ulimit，不默认 `--privileged`。Complete when：container Image ID 等于 qualified base、模型只读可见、设备映射一致。

## 2. DLC Ecosystem 环境

5. 发现实际 source roots、full SHAs、dirty status、Python/pip/CMake/compiler、offline wheels/dependency bundles。读权限验证用 `git ls-remote`；仅在明确 push 任务中检查写权限。Complete when：所有 active sources/imports 可唯一解释，dirty conflict 已阻断或显式纳入 policy。
6. 优先使用现有离线 wheel/source。Network、clone/fetch、package install 各自需要授权。安装脚本执行前读取内容；名称含 preflight 不代表只读。Complete when：每项 mutation 的来源、命令、输出和 artifact 已记录。
7. 记录 actual Python executable、package metadata、module paths、DLC Platform/plugin/extension identity。Complete when：C1a fresh process 输出 `c1a_package_import_pass`；C1a 不允许加载模型。

## 3. C1b DLC Runtime Execution

8. 每张 requested logical device 分别运行 fresh process layered probe：enumeration/properties、allocation、H2D、nontrivial device op、synchronize、D2H、correctness。Complete when：所有 requested devices 独立通过。
9. 多设备 profile 做 simultaneous probe；使用 collective communication 的 TP/PP/EP profile 做 DLCCL correctness。Complete when：deployment 所需通信范围通过，否则不加载模型。

技术 probe 细节引用 [安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md)，不要在其他 prompt 重写。

## 4. Real-weight Functional

10. 由 config、weight bytes、dtype/quantization、HBM 和 deployment 需求导出 TP/PP/EP、max lengths、batching、block size、Chunked Prefill、prefix caching 和 port。Complete when：profile 六类依据可审计。
11. 每个 server epoch 记录 container ID、Host/container PID、NSpid、namespace、starttime、cmdline、PPID、cgroup、port 和 HBM baseline。tracked wrapper 退出不等于 container child 退出。
12. readiness 后保存 health-before 和 model-list/alias。执行 runtime contract 中至少两个正交 assertions，分别记录 HTTP、completion、non-empty、semantic result、finish reason/token count 和 raw JSON。Complete when：全部 assertions 和 health-after 通过，状态为 `model_functional_pass`。
13. 空文本、错误答案、重复/损坏文本、异常截断、NaN/Inf、timeout 或 server death 均停止提升。CPU Reference 可诊断但不能提升 DLC functional 状态。

## 5. Benchmark

14. 保存当前 `vllm bench serve --help`，固定 workload 与 functional/benchmark profile diff。新 profile 使用新 server epoch，并重跑 readiness、alias 和 strict smoke。
15. 运行同形小规模 warm-up，确认成功且服务健康；warm-up 不计入 formal result。
16. 执行声明 formal attempts，保存 commands、environment allowlist、raw client/server logs、JSON、actual token distributions、throughput、TTFT/TPOT/ITL、Peak concurrent requests 和 health-after。
17. Complete when：达到 contract 的请求成功阈值、client 正常退出、server 测后健康。一个成功 attempt 可为 `benchmark_workload_pass`；只有 contract 要求并完成重复 attempts 时才为 `benchmark_stability_baseline_pass`。

## 6. Failure Epoch

18. 保存第一失败边界、最小输入、raw response/log、PID/device snapshot。每次 retry 新建 epoch且只改变一个变量。没有中间 fresh probe 时只报告 `recovered_after_action_sequence`。
19. OOM 后降低 profile、新 endpoint、不同 prompt 或不同 source 是新验证对象，不覆盖原失败。
20. Host maintenance、service restart、reset/reboot、driver/HBM/LYP 变更需独立授权；工具缺失不自动转换成硬件故障。

## 7. Cleanup

21. 只停止 task-owned APIServer/EngineCore/workers/clients。TERM 前重新核对 identity；KILL 仅在 identity 仍匹配且授权时使用。
22. 确认 task ports、handles 和 HBM delta 回到 sealed baseline/tolerance。共享 Host 不要求全机 HBM 为 0，不 kill 其他任务，不 prune。
23. Complete when：task runtime resources 闭合，persistent validation container 按 contract 保留或停止，evidence 和失败 epochs 持久化。

## Checkpoints

| Checkpoint | Completion |
|---|---|
| C0 Base Ready | ordinary daily base + task container identity 闭合 |
| C1a | package/import/origin/plugin identity 通过 |
| C1b | fresh device operation/sync/D2H correctness 通过 |
| C2 Functional | real-weight assertions + health before/after 通过 |
| C3 Benchmark Workload | declared workload 和测后 health 通过 |
| C4 Stability Baseline | declared repeated attempts 与摘要通过 |
| C5 Cleanup | task process/port/HBM delta 闭合 |

## 结论边界

HTTP 200、package import、allocation、weight load、non-empty output、单次 benchmark、static checks 或 Docker build 都不能升级为未执行的更高层状态。最终报告按 `modelzoo_claims`、`local_observations`、`inferences`、`execution_evidence`、`unverified_scope` 分栏。
