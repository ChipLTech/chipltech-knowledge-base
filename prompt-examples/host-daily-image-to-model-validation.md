---
prompt_schema: host-daily-image-model-validation/v3
role: operational_runbook
required_user_inputs:
  - model_name
  - absolute_local_model_path
discovery_policy: local_query_only_then_contract_proposal
---

# Host Daily Image 到模型验证 Runbook

## 用途

本文只拥有 Host/container preparation、C1a/C1b、real-weight functional、benchmark 和 cleanup 的执行 mechanics。状态、claim 和 image delivery 语义由 [规范 Contract](../vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md) 定义。

## 用户接口

```text
model_name
absolute_local_model_path
```

只有这两个字段由用户填写。Runbook 自动发现或提出 ordinary daily base、artifact root、device scope、functional assertions、benchmark workload、driver/container profile、SMI observer 和 authorization status，并写入 Runtime Qualification Contract。不得在 discovery 前向用户索要这些可派生字段。

自动发现只建立事实和 proposal，不自动授予 image pull、network、clone/fetch、package install、build、qualification execution、task-owned KILL、device execution、privileged Host integration、tar export、registry push 或 Host maintenance。优先消费可验证且仍在有效 scope 内的 standing authorization；没有时只在该动作成为继续条件时请求最小授权和精确 profile diff。

## 1. Host 与 Ordinary Daily Base

1. Query-only 发现 OS/kernel、Docker client/server、存储、设备节点、hardware generation、occupancy、HBM、已有 containers/ports，并先在响应内生成 baseline proposal。只有 `qualification_execution` 和 artifact root 已闭合后才持久化 baseline/evidence。对实际执行 Real DLC Hardware 的 run，按 [cltech_smi 设备观测与诊断证据](../runtime-debugging/chipltech-smi-observability.md) 生成并持久化 SMI Observation Envelope；只读/static run 可记录 `not_applicable` 及依据。Complete when：目标设备可唯一映射、所需授权闭合后 baseline 已保存且 observer 有 terminal state；busy 则 `blocked_missing_hardware`，工具缺失/不可解析则 `blocked_missing_observability`，不得改写为硬件故障。
2. 只读检查 local images 并自动选择 ordinary daily base candidate；获 pull 授权时才访问 remote registry 或 pull。记录 Image ID，repo digest 在本地可用时记录。Complete when：base 通过 contract 的 ordinary eligibility；否则 `blocked_unqualified_daily_base`。
3. 自动提出源码树外的 task-owned `src/build/wheels/artifacts/logs` 和 cache 路径，验证绝对路径、owner 和剩余容量；只有 `qualification_execution` 已授权才创建和写入，模型只读挂载。Complete when：artifact root 位于源码树外且运行契约已脱敏保存。
4. 只有 `qualification_execution` 已授权才创建 task-owned persistent container。只有 sealed C1b record 的 canonical fingerprint 与当前 kernel、hardware generation、Host driver API/version、container runtime/config、base Image ID、device nodes/mapping 和 profile digest 完全相等，且 profile 全部 mount/privilege 已通过本次 `privileged Host integration` 明确授权，首次才直接使用该 driver-compatible profile；否则使用最小 profile 探测。不得把 device execution 授权解释为 `--privileged` 或 Host system mounts 授权。C1b 不得在 pre-handoff deployment profile 已绑定 runtime qualification contract 前运行；之后只有 C1a 通过且 device execution 已获授权，才按该 profile 运行完整 fresh C1b。只有该 C1b 将失败定位为 container profile mismatch，且精确 profile diff 已获授权时，才重建 task container 并从 fresh C1a/C1b 开始。Complete when：container Image ID 等于 qualified base、模型只读可见、设备映射一致且 C1b profile 已闭合。

## 2. DLC Ecosystem 环境

5. 发现实际 source roots、full SHAs、dirty status、Python/pip/CMake/compiler、offline wheels/dependency bundles。先使用本地 remote config、object 和 worktree 检查；`git ls-remote` 仅在 network/remote-access 已授权时运行。仅在明确 push 任务中检查写权限。Complete when：所有 active sources/imports 可唯一解释，dirty conflict 已阻断或显式纳入 policy。
6. 优先使用现有离线 wheel/source。Network、clone/fetch、package install 各自需要授权。安装脚本执行前读取内容；名称含 preflight 不代表只读。对每个 task-owned repo 和 build-time submodule 验证 fixed commit object、detached checkout、worktree entrypoint 和 Git ownership；builder UID 与 source owner 不同时，只把 canonical task paths 写入 task-local Git global config 的 `safe.directory`，cleanup 时删除该 config。Complete when：每项 mutation 的来源、命令、输出和 artifact 已记录。
7. 记录 actual Python executable、package metadata、module paths、DLC Platform/plugin/extension identity。若 source archive 不含编译 extension，单独记录 binary overlay 的来源/hash。验证 approved CMake 在 interactive shell 和实际 Python/setuptools subprocess 中同源可用；`ctest`、`cpack` 仅在实际调用时记录来源和版本。Complete when：C1a fresh process 输出 `c1a_package_import_pass`；C1a 不允许加载模型。

## 3. C1b DLC Runtime Execution

8. 消费与 runtime qualification contract 绑定的 pre-handoff deployment profile 后，对每张 requested logical device 分别运行 fresh process layered probe：enumeration/properties、allocation、H2D、nontrivial device op、synchronize、D2H、correctness。Complete when：所有 requested devices 独立通过。
9. 多设备 profile 做 simultaneous probe；使用 collective communication 的 TP/PP/EP profile 做 DLCCL correctness。Complete when：deployment 所需通信范围通过，否则不加载模型。

技术 probe 细节引用 [安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md)，不要在其他 prompt 重写。

完成 C1a 和所有 pre-handoff deployment profile 所需的 C1b/device/collective probes后，由本 Runbook 在 `artifact_root` 写入 `environment_handoff/v1`，并在 Runtime Action Record 中引用其绝对路径和 digest。它只能索引本次 evidence，不创建新的 PASS、状态或 claim；字段和 consumer 规则以 Contract 的 `Environment Handoff` 为准。写入前必须确认 `daily_base_image_id`、`task_container_identity`、fingerprint、requested logical-device mapping、source/package/plugin/extension identities、每个 required C1a/C1b/collective artifact path、SMI observer status/artifact、cleanup baseline、unresolved blockers 和 explicitly unverified scope 均已绑定本次 runtime qualification contract。任何 required C1a/C1b/collective 未明确 pass 或存在 unresolved blocker 时，不生成可供 model load、functional、benchmark 或 delivery 消费的 handoff；保留 Runtime Action Record 与 blocker/failure epoch。

## 4. Real-weight Functional

10. 由 config、weight bytes、dtype/quantization、HBM 和 deployment 需求导出 TP/PP/EP、max lengths、batching、block size、Chunked Prefill、prefix caching 和 port。启动前读取实际 server `--help`，固定 CLI 参数；使用显式 absolute `--model` 路径和离线 Hub guard，禁止模型位置参数回退为默认远端模型。Complete when：profile 六类依据可审计。
11. 每个 server epoch 记录 container ID、Host/container PID、NSpid、namespace、starttime、cmdline、PPID、cgroup、port 和 HBM baseline。按同一 run ID 保存 `before_launch` 与 `after_ready` raw/normalized SMI evidence；tracked wrapper 退出不等于 container child 退出。
12. readiness 后保存 health-before 和 model-list/alias。执行 runtime contract 中至少两个正交 assertions，分别记录 HTTP、completion、non-empty、semantic result、finish reason/token count 和 raw JSON。Complete when：全部 assertions 和 health-after 通过，状态为 `model_functional_pass`。
13. 空文本、错误答案、重复/损坏文本、异常截断、NaN/Inf、timeout 或 server death 均停止提升。CPU Reference 可诊断但不能提升 DLC functional 状态。

## 5. Benchmark

14. 保存当前 `vllm bench serve --help`，固定 workload 与 functional/benchmark profile diff。新 profile 使用新 server epoch，并重跑 readiness、alias 和 strict smoke。
15. 运行同形小规模 warm-up，确认成功且服务健康；warm-up 不计入 formal result。
16. 执行声明 formal attempts，保存 commands、environment allowlist、raw client/server logs、JSON、actual token distributions、throughput、TTFT/TPOT/ITL、Peak concurrent requests、`during_request` SMI evidence 和 health-after。
17. Complete when：达到 contract 的请求成功阈值、client 正常退出、server 测后健康。一个成功 attempt 可为 `benchmark_workload_pass`；只有 contract 要求并完成重复 attempts 时才为 `benchmark_stability_baseline_pass`。

## 6. Failure Epoch

18. 保存第一失败边界、最小输入、raw response/log、PID/device snapshot 和对应 SMI sample。按 device ownership -> C1b -> group-scoped communication -> model/runtime 的层次选择下一 probe；每次 retry 新建 epoch且只改变一个变量。没有中间 fresh probe 时只报告 `recovered_after_action_sequence`。
19. OOM 后降低 profile、新 endpoint、不同 prompt 或不同 source 是新验证对象，不覆盖原失败。
20. Host maintenance、service restart、reset/reboot、driver/HBM/LYP 变更需独立授权；工具缺失不自动转换成硬件故障。

## 7. Cleanup

21. 只停止 task-owned APIServer/EngineCore/workers/clients。TERM 前重新核对 identity；KILL 必须完整满足 Contract 的 `task_owned_kill`：sealed PID/PGID/container identity 仍精确匹配、graceful stop 已超时、影响范围已记录且独立授权有效。
22. 确认 task ports、handles 和 HBM delta 回到 sealed baseline/tolerance，并保存 `after_cleanup` raw/normalized SMI evidence。共享 Host 不要求全机 HBM 为 0，不 kill 其他任务，不 prune。
23. Complete when：task runtime resources 和 observation envelope 闭合，persistent validation container 按 contract 保留或停止，evidence 和失败 epochs 持久化。

## Checkpoints

| Checkpoint | Completion |
|---|---|
| C0 Base Ready | ordinary daily base + task container identity 闭合 |
| C1a | package/import/origin/plugin identity 通过 |
| C1b | fresh device operation/sync/D2H correctness 通过 |
| Environment Handoff | Contract-defined `environment_handoff/v1` 已写入 artifact root，并绑定 required C1a/C1b/collective evidence |
| C2 Functional | real-weight assertions + health before/after 通过 |
| C3 Benchmark Workload | declared workload 和测后 health 通过 |
| C4 Stability Baseline | declared repeated attempts 与摘要通过 |
| C5 Cleanup | task process/port/HBM delta 与 `after_cleanup` observation 闭合 |

## 结论边界

HTTP 200、package import、allocation、weight load、non-empty output、单次 benchmark、static checks 或 Docker build 都不能升级为未执行的更高层状态。最终报告按 `modelzoo_claims`、`local_observations`、`inferences`、`execution_evidence`、`unverified_scope` 分栏。
