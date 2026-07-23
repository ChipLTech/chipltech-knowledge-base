---
prompt_schema: vllm-dlc-prefill-decode-separation/v2
skill_identity: pd-separation
required_user_inputs:
  - model_name
  - absolute_local_model_path
derived_or_proposed_contract:
  - model_and_source_identities
  - topology_and_role_profiles
  - kv_transfer_and_port_contract
  - artifact_destination
  - monolithic_comparison_contract
  - site_recovery_contract
discovery_policy: local_query_only_then_contract_proposal
missing_input_status: blocked
missing_input_reason: blocked_missing_contract
hardware_evidence: not_verified
---

# vLLM-DLC Prefill/Decode Separation Prompt

## 用途

这是 `pd-separation` skill 的薄入口，用于部署或诊断单机 TCP、qualified `lyp_full` / `dlccl_direct` 或跨机器 TCP 的 Prefill/Decode/Proxy，并要求 transport qualification、request-correlated KV transfer 与 site recovery evidence。稳定知识以 [Prefill/Decode Separation](../vllm-dlc/prefill-decode-separation.md) 为准。

## 最少填写

```text
模型名称：<MODEL_NAME>
模型目录：<ABSOLUTE_LOCAL_MODEL_PATH>
```

其余 deployment contract 全部由 agent 从当前 Host、容器、source/package、模型 config、设备、端口、历史过程资产和实际 `--help` 自动发现或提出。默认先评估 `single_node_tcp`；若 exact checkout、native extension 和 group-scoped transport evidence 支持更可靠的 `dlccl_direct`/`lyp_full`，可基于证据选择。只有 cross-machine 地址、业务强制 topology 或不可观测 workload recovery 等确实无法推导的字段才作为后续最小 blocker。

## 可直接复制 Prompt

```md
请使用 `pd-separation` skill 直接完成 vLLM-DLC Prefill/Decode Separation，并持续到 `pd_validated` 或一个不可继续的 terminal blocker；若存在历史失败或过程资产，从最早未闭合阶段恢复。不要要求我预填可自动发现的字段。

模型名称：<MODEL_NAME>
模型目录：<ABSOLUTE_LOCAL_MODEL_PATH>

先自动发现 Knowledge base/Skills roots、model/tokenizer digest、vLLM/vllm-dlc/mooncake/PyTorch identities、可用 container/image、空闲设备、LYP/transport 能力、role profile、KV layout/cache contract、API/side-channel/store ports、monolithic baseline、deterministic assertion、artifact root、既有 workload 与 Site Recovery Contract。优先复用同机已验证过程资产和仍匹配当前 identity 的成功配置；不照抄过期 PID 或端口。

自动生成 resolved deployment proposal 和 authorization status；在 `qualification_execution` 与 artifact root 闭合前只放在响应中，不写文件。授权后再保存。发现和 proposal 不授权 network、clone/fetch、install、build、qualification execution、device execution、privileged Host integration、task-owned KILL 或 Host maintenance；优先消费已验证 standing authorization，仍缺失时只请求下一步所需的最小 scope，不要求我手工填写整份矩阵。KILL 必须完整满足 Contract 的 `task_owned_kill` 条件。

开始前完整读取：
- <KNOWLEDGE_BASE_ROOT>/CONTEXT.md
- <KNOWLEDGE_BASE_ROOT>/vllm-dlc/prefill-decode-separation.md
- <KNOWLEDGE_BASE_ROOT>/vllm-dlc/model-adaptation-and-main-to-main-decisions.md
- <KNOWLEDGE_BASE_ROOT>/runtime-debugging/performance-profiling.md
- <KNOWLEDGE_BASE_ROOT>/runtime-debugging/chipltech-smi-observability.md
- <KNOWLEDGE_BASE_ROOT>/testing/arsenal-ci-and-blackbox-testing.md
- <SKILLS_ROOT>/skills/engineering/pd-separation/SKILL.md 及其 conditional references
- 实际 checkout 中的 pd_launcher、proxy、connector、transport 源码和各自 `--help`

执行要求：
1. 先确认当前 session 可读取已安装的 `pd-separation` skill，或自动发现可读的 skill/references；两者均不可用时返回 `blocked_missing_contract`。完成 query-only discovery 后自动冻结 deployment、cleanup 和 Site Recovery Contract。只有模型名称或绝对路径缺失/无效时才视为初始 input 缺失；其他字段先发现和推导，仍不唯一时返回精确 blocker。
2. 先建立或消费同一模型、tokenization、prompt 和 decoding parameters 的 monolithic serving baseline。环境修复路由到 `dlc-env-setup`；模型级 Attention/MLA/MoE/quantization/MTP/KV-layout/TP/DCP 问题路由到 `model-adaptation`。
3. 通过 `dlc-hardware-observability` 只读检查 device/process/memory、device-set overlap、group-scoped LYP/link/operator smoke、model path、physical/local visible-device mapping、TP 和所有 API/side-channel/store/TransferEngine ports；为两个 role PGID 保存四阶段 raw/normalized SMI evidence。保留 pass criteria 和 raw output；observer 缺失返回 `blocked_missing_observability`，不声明硬件失败。
4. 默认选择 TCP；仅在明确同机且 group-scoped LYP 已验证时使用 `lyp_full`。仅当 exact checkout/parser/connector 包含 protocol、native extension identity 已固定时使用 `dlccl_direct`。旧 `lyp` 只用于 diagnostics；不得把 `rdma_direct` 当 workaround。
5. 加载两个模型前执行 Transport Qualification Gate：按目标进程和设备可见性让双端并发初始化，传输非空 payload，要求 send/recv completion 和内容校验。TCP 日志若仍安装 RDMA，不得宣称 TCP-only 通过。失败返回 `blocked_transport_unqualified`。
6. 在 import vLLM 和 worker spawn 前设置设备、protocol、side-channel 和 store。普通 TCP 按 Prefill -> Decode；`lyp_full` 在 Prefill store listener 出现后立即启动 Decode，不等待 Prefill HTTP ready；两个 role probe 通过后启动 Proxy。readiness 以 actual routes/capability 为准，Proxy 不存在 `/health` 时使用 listener + real request。
7. 通过 Proxy 发送一个 deterministic request。关联 client、Proxy、Prefill、Decode identity，保存 Prefill block retention/registration、Decode `remote_request_id`、local/remote blocks、transport completion、DLC-side receive/synchronization、response 和 role health-after。
8. 只有 request-correlated evidence 同时证明 routing 与 KV consumption，且 PD response 按 comparison contract 与 monolithic baseline 等价，才报告 `pd_validated`。HTTP 200、非空输出、readiness probes 或 connector handshake 不能替代 KV evidence。
9. 基础请求通过后，每轮只改变一个变量，再做 lifecycle、concurrency、long-context 和 benchmark。分别报告 TTFT、TPOT、ITL、throughput、queueing 和 KV-transfer cost。
10. 若失败，按 identity/config -> device-group qualification -> transport gate -> startup -> control plane -> request correlation -> KV layout/block mapping -> Decode cache -> functional -> performance -> cleanup 分类最早失败阶段。request/block 已对齐但 native completion hang 时先回 transport gate 和 LYP status。
11. 按 Proxy -> Decode -> Prefill graceful stop，只清理 task-owned PGID。停止 `docker exec` client 不等于容器内 process 退出；需要 KILL 时必须满足 sealed identity、graceful timeout、impact record 和独立 `task_owned_kill`。比较 process/ports/device memory/HBM-frequency/link 和既有 workload；维护后按 Site Recovery Contract 恢复并执行真实请求。无法恢复返回 `blocked_cleanup_incomplete`。
12. Device execution 不授权终止非 task-owned process、`--privileged`、Host PID/system mounts、driver reload、power cycle、Chip ID/HBM/Bluejay/LYP/PCIe state change 或 reboot。Host maintenance 未逐项授权时禁止执行；tool-local failure index 必须先映射 physical device。
13. 最终分别报告 static configuration、transport qualification、service readiness、request routing、KV transfer、functional equivalence、lifecycle/cleanup、site recovery、performance workload、stability baseline、artifact paths、claim boundaries 和 remaining risks。
```

## 停止语义与 Evidence

- topology、cache contract、endpoint、hardware 和 Site Recovery Contract 先按 source/config/Host evidence 自动生成安全 proposal；proposal 不等于执行授权。只有证据无法闭合或下一动作缺少授权时才返回对应 structured blocker。
- 跨机器 endpoint 从对端 namespace 不可达时返回 `blocked_network_unreachable`。
- 两侧 model/cache contract 不兼容时返回 `blocked_model_or_cache_incompatible`。
- 仅完成 static/readiness/HTTP 的结果保持 `not_verified`，不得写成 PD 已验证。

## 相关资料

- [Prefill/Decode Separation](../vllm-dlc/prefill-decode-separation.md)
- [Model Adaptation 与 Main-to-Main 决策](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)
- [性能分析](../runtime-debugging/performance-profiling.md)
- [cltech_smi 设备观测与诊断证据](../runtime-debugging/chipltech-smi-observability.md)
- [Arsenal CI 与黑盒测试](../testing/arsenal-ci-and-blackbox-testing.md)
- 外部 workflow：`skills.git` 中的 `skills/engineering/pd-separation/SKILL.md`，需预先安装到 Kilo 或通过 `<SKILLS_ROOT>` 提供可读路径。
