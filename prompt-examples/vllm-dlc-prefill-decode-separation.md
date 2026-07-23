---
prompt_schema: vllm-dlc-prefill-decode-separation/v1
skill_identity: pd-separation
required_inputs:
  - model_identity_and_assets
  - source_and_package_identities
  - topology
  - prefill_and_decode_profiles
  - kv_transfer_contract
  - endpoint_and_port_matrix
  - artifact_destination
  - monolithic_baseline
  - deterministic_comparison_contract
  - authorization_matrix
  - site_recovery_contract_or_not_applicable_evidence
missing_input_status: blocked
missing_input_reason: blocked_missing_contract
hardware_evidence: not_verified
---

# vLLM-DLC Prefill/Decode Separation Prompt

## 用途

这是 `pd-separation` skill 的薄入口，用于部署或诊断单机 TCP、qualified `lyp_full` / `dlccl_direct` 或跨机器 TCP 的 Prefill/Decode/Proxy，并要求 transport qualification、request-correlated KV transfer 与 site recovery evidence。稳定知识以 [Prefill/Decode Separation](../vllm-dlc/prefill-decode-separation.md) 为准。

## 最少填写

```text
模型名称与绝对目录：<MODEL_NAME> <ABSOLUTE_MODEL_PATH>
Source/package identities：<vLLM/vllm-dlc/mooncake-dlc/PyTorch DLC Backend identities>
拓扑：<single_node_tcp | single_node_lyp_full | single_node_dlccl_direct | cross_machine_tcp>
Prefill profile：<host/address/devices/TP/API/side-channel>
Decode profile：<host/address/devices/TP/API/side-channel>
Proxy profile：<host/address/API>
KV contract 与端口：<connector/layout/dtype/block size/store/TransferEngine ports>
Monolithic baseline 与 comparison contract：<artifact；默认 deterministic output token IDs exact match>
Artifact root：<ABSOLUTE_ARTIFACT_PATH>
既有 workload recovery：<process/container/device/port/model/full launch/health/functional request>
授权：<network/install-build/device execution/privileged Host integration/Host maintenance，逐项 yes | no>
```

## 可直接复制 Prompt

```md
请使用 `pd-separation` skill 直接部署并验证 vLLM-DLC Prefill/Decode Separation；若当前已有失败，则从最早失败阶段开始诊断。不要只给计划。

输入：
- Knowledge base root：<KNOWLEDGE_BASE_ROOT>
- Skills root：<SKILLS_ROOT>
- 模型名称：<MODEL_NAME>
- 模型目录：<ABSOLUTE_MODEL_PATH>
- 模型/tokenizer revision 或 recursive digest：<IDENTITY>
- vLLM full SHA：<FULL_SHA>
- vllm-dlc full SHA：<FULL_SHA>
- mooncake-dlc full SHA：<FULL_SHA>
- PyTorch DLC Backend/package identity：<IDENTITY>
- 拓扑：<single_node_tcp | single_node_lyp_full | single_node_dlccl_direct | cross_machine_tcp>
- Prefill profile：<host/container epoch、routable address、devices、TP/PP/EP/DCP、API port、side-channel base>
- Decode profile：<host/container epoch、routable address、devices、TP/PP/EP/DCP、API port、side-channel base>
- Proxy profile：<host/container epoch、client address、API port>
- KV contract：<connector/module、dtype、quantization、KV layout/cache dtype、block size、context、common/role-specific args>
- Store/TransferEngine ports：<rank ranges and discovered RPC/handshake ports>
- Monolithic baseline：<artifact path | execute first>
- Deterministic comparison contract：<prompt、request model alias、decoding params、baseline output token IDs；默认 exact token match，或声明指标/容差/理由>
- Artifact root：<ABSOLUTE_ARTIFACT_PATH>
- 既有 workload 与 Site Recovery Contract：<process/container/device/HBM-frequency/ports/model/full launch/health/functional request>
- 授权：
  - network access：<yes | no>
  - install/build：<yes | no>
  - device execution：<yes | no>
  - privileged Host integration：<yes | no；附批准 profile>
  - Host maintenance：<yes | no；附批准 action 和 maintenance window>

开始前完整读取：
- <KNOWLEDGE_BASE_ROOT>/CONTEXT.md
- <KNOWLEDGE_BASE_ROOT>/vllm-dlc/prefill-decode-separation.md
- <KNOWLEDGE_BASE_ROOT>/vllm-dlc/model-adaptation-and-main-to-main-decisions.md
- <KNOWLEDGE_BASE_ROOT>/runtime-debugging/performance-profiling.md
- <KNOWLEDGE_BASE_ROOT>/testing/arsenal-ci-and-blackbox-testing.md
- <SKILLS_ROOT>/skills/engineering/pd-separation/SKILL.md 及其 conditional references
- 实际 checkout 中的 pd_launcher、proxy、connector、transport 源码和各自 `--help`

执行要求：
1. 先确认当前 session 可读取已安装的 `pd-separation` skill，或可读取 `<SKILLS_ROOT>/skills/engineering/pd-separation/SKILL.md` 及 references；两者均不可用时返回 `blocked_missing_contract`。冻结 deployment、cleanup 和 Site Recovery Contract，固定 source/package/model identity、两侧 role profile、connector、transport、endpoint/port matrix、actual readiness routes、deterministic comparison contract 和 evidence 字段。无既有 workload 且不执行 Host maintenance 时，Site Recovery Contract 仍填写 `not_applicable` 及观测依据；任何 required input 缺失都返回 `blocked_missing_contract`。
2. 先建立或消费同一模型、tokenization、prompt 和 decoding parameters 的 monolithic serving baseline。环境修复路由到 `dlc-env-setup`；模型级 Attention/MLA/MoE/quantization/MTP/KV-layout/TP/DCP 问题路由到 `model-adaptation`。
3. 只读检查 device/process/memory、device-set overlap、group-scoped LYP/link/operator smoke、model path、physical/local visible-device mapping、TP 和所有 API/side-channel/store/TransferEngine ports。保留 pass criteria 和 raw output。
4. 默认选择 TCP；仅在明确同机且 group-scoped LYP 已验证时使用 `lyp_full`。仅当 exact checkout/parser/connector 包含 protocol、native extension identity 已固定时使用 `dlccl_direct`。旧 `lyp` 只用于 diagnostics；不得把 `rdma_direct` 当 workaround。
5. 加载两个模型前执行 Transport Qualification Gate：按目标进程和设备可见性让双端并发初始化，传输非空 payload，要求 send/recv completion 和内容校验。TCP 日志若仍安装 RDMA，不得宣称 TCP-only 通过。失败返回 `blocked_transport_unqualified`。
6. 在 import vLLM 和 worker spawn 前设置设备、protocol、side-channel 和 store。普通 TCP 按 Prefill -> Decode；`lyp_full` 在 Prefill store listener 出现后立即启动 Decode，不等待 Prefill HTTP ready；两个 role probe 通过后启动 Proxy。readiness 以 actual routes/capability 为准，Proxy 不存在 `/health` 时使用 listener + real request。
7. 通过 Proxy 发送一个 deterministic request。关联 client、Proxy、Prefill、Decode identity，保存 Prefill block retention/registration、Decode `remote_request_id`、local/remote blocks、transport completion、DLC-side receive/synchronization、response 和 role health-after。
8. 只有 request-correlated evidence 同时证明 routing 与 KV consumption，且 PD response 按 comparison contract 与 monolithic baseline 等价，才报告 `pd_validated`。HTTP 200、非空输出、readiness probes 或 connector handshake 不能替代 KV evidence。
9. 基础请求通过后，每轮只改变一个变量，再做 lifecycle、concurrency、long-context 和 benchmark。分别报告 TTFT、TPOT、ITL、throughput、queueing 和 KV-transfer cost。
10. 若失败，按 identity/config -> device-group qualification -> transport gate -> startup -> control plane -> request correlation -> KV layout/block mapping -> Decode cache -> functional -> performance -> cleanup 分类最早失败阶段。request/block 已对齐但 native completion hang 时先回 transport gate 和 LYP status。
11. 按 Proxy -> Decode -> Prefill 停止，只清理 task-owned PGID。停止 `docker exec` client 不等于容器内 process 退出。比较 process/ports/device memory/HBM-frequency/link 和既有 workload；维护后按 Site Recovery Contract 恢复并执行真实请求。无法恢复返回 `blocked_cleanup_incomplete`。
12. Device execution 不授权终止非 task-owned process、`--privileged`、Host PID/system mounts、driver reload、power cycle、Chip ID/HBM/Bluejay/LYP/PCIe state change 或 reboot。Host maintenance 未逐项授权时禁止执行；tool-local failure index 必须先映射 physical device。
13. 最终分别报告 static configuration、transport qualification、service readiness、request routing、KV transfer、functional equivalence、lifecycle/cleanup、site recovery、performance workload、stability baseline、artifact paths、claim boundaries 和 remaining risks。
```

## 停止语义与 Evidence

- 缺少 topology、cache contract、routable endpoint、hardware、授权或 Site Recovery Contract/not-applicable evidence 时返回对应 structured blocker，不猜测默认值。
- 跨机器 endpoint 从对端 namespace 不可达时返回 `blocked_network_unreachable`。
- 两侧 model/cache contract 不兼容时返回 `blocked_model_or_cache_incompatible`。
- 仅完成 static/readiness/HTTP 的结果保持 `not_verified`，不得写成 PD 已验证。

## 相关资料

- [Prefill/Decode Separation](../vllm-dlc/prefill-decode-separation.md)
- [Model Adaptation 与 Main-to-Main 决策](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)
- [性能分析](../runtime-debugging/performance-profiling.md)
- [Arsenal CI 与黑盒测试](../testing/arsenal-ci-and-blackbox-testing.md)
- 外部 workflow：`skills.git` 中的 `skills/engineering/pd-separation/SKILL.md`，需预先安装到 Kilo 或通过 `<SKILLS_ROOT>` 提供可读路径。
