# vLLM-DLC Prefill/Decode Separation

## 适用场景

本文用于理解和验证 DLC Platform 上基于 `MooncakeDLCConnector` 的 Prefill/Decode Separation（PD 分离）：单机 TCP、单机 `lyp_full`、exact checkout 支持的 `dlccl_direct`、跨机器 TCP，以及 Prefill、Decode、Proxy、KV Cache transfer 和故障边界。术语以 [CONTEXT.md](../CONTEXT.md) 为准；可执行顺序由 `pd-separation` skill 维护。

## 核心结论

- **事实 / Fact**：Prefill Worker 使用 `kv_producer` 角色生成并保留请求对应的 KV Cache；Decode Worker 使用 `kv_consumer` 角色拉取 KV 后继续 token decode；Proxy 协调同一请求的两个 role。
- **事实 / Fact**：当前资料中的 DLC connector 为 `MooncakeDLCConnector`，模块为 `mooncake.mooncake_connector_dlc_v1`。它沿用 vLLM KV Connector V1 的 Scheduler/Worker 分层和 `remote_request_id` 映射。
- **事实 / Fact**：文档化的默认 transport 是 `tcp`，其数据路径经过 CPU pinned staging；`lyp_full` 是可选的 DLCCL/LYP P2P 路径，不是第二个默认值。
- **事实 / Fact**：部分 mooncake-dlc feature branch 实现了 torch-free `dlccl_direct`，通过 package-local native extension 直接调用 DLCCL；它是 exact-checkout capability，不是 main 或任意 wheel 的通用能力。
- **建议 / Recommendation**：跨机器部署使用 TCP；`lyp_full` 仅用于已经单独验证的同机拓扑。旧 `lyp` 只用于诊断，不作为生产推荐。
- **建议 / Recommendation**：加载两个模型实例前先执行 Transport Qualification Gate，要求双端并发初始化、非空 payload、send/receive completion 和内容校验。
- **事实 / Fact**：`/health`、connector handshake、HTTP 200、非空输出和 benchmark 完成都不能单独证明 KV transfer 已完成。
- **未验证 / Not verified**：当前资料未绑定 mooncake-dlc、vLLM、vllm-dlc 和 PyTorch DLC Backend 的 exact full SHA，也未给出 TransferEngine 全部 RPC/handshake 端口，因此具体 CLI、metadata、端口和 layout 必须以实际 checkout 与 `--help` 为准。

## 术语与组件边界

- **Prefill Worker**：执行 prompt prefill，生成并保留可供远端 decode 使用的 KV Cache 和 metadata。
- **Decode Worker**：消费与 request、model 和 cache contract 匹配的 KV Cache，执行后续 token decode。
- **Proxy**：将一个 OpenAI-compatible 请求协调到指定 Prefill 和 Decode role。Proxy 可达不等于 KV 数据面可达。
- **KV Cache Transfer Contract**：两侧关于 request correlation、layer/component、block/token ownership、layout、dtype、stride、block size、metadata、完成与失败语义的显式 contract。
- **Control plane**：Proxy 路由、side channel、request metadata、`remote_request_id` 和 transfer completion 信号。
- **Data plane**：TCP CPU staging 或 DLCCL/LYP 上实际传输的 KV Cache 数据。
- **Transport Qualification Gate**：在模型 serving 前独立验证 exact data plane，必须检查内容，不只检查 benchmark 退出。
- **Site Recovery Contract**：Host maintenance 前封存的既有 workload 与设备状态恢复合同。

## Deployment Contract

启动前固定并保存：

| 类别 | 必须记录 |
|---|---|
| Source/package identity | vLLM、vllm-dlc、mooncake-dlc、PyTorch DLC Backend、DLC Runtime 的 exact identity |
| Model identity | weights、tokenizer/processor、可信 revision 或 recursive digest、API model alias |
| Role profile | Prefill/Decode host/container epoch、visible devices、TP/PP/EP/DCP、dtype、quantization、context、block/cache 配置 |
| Topology | `single_node_tcp`、`single_node_lyp_full`、qualified `single_node_dlccl_direct` 或 `cross_machine_tcp` |
| Endpoint matrix | bind address、advertised routable address、API、side-channel rank range、store rank range、TransferEngine ports |
| Connector | class/module、producer/consumer role、实际 `--kv-transfer-config` |
| Evidence | deterministic request、期望输出、request correlation 字段、日志和 cleanup baseline |
| Authorization | network/install/build、device execution、privileged Host integration、Host maintenance 分别授权 |
| Recovery | 会受 Host maintenance 影响的 workload、设备/HBM/频率、端口、完整启动和功能验证命令 |

两侧通常必须具有相同或明确兼容的 weights/tokenizer、dtype、quantization、KV layout/cache dtype、block size 和 context 语义。资料中的 TP=4、TP=8、端口和模型参数只是环境示例，不是默认 contract。

## MooncakeDLCConnector 适配

### Scheduler 与 Worker

- Scheduler 处理 matched-token accounting、allocation state、connector metadata 和 request-finished 生命周期。
- Worker 处理 KV Cache 注册、发送、接收，以及 TransferEngine 或 LYP transport 初始化。
- `do_remote_decode=True` 表示 Prefill 完成后保留 blocks 等待 Decode 拉取。
- `do_remote_prefill=True` 表示 Decode 通过 metadata 请求远端 Prefill KV。
- `remote_request_id` 连接 Decode-local request ID 与 Prefill-side request ID。

### Import-before-spawn

**事实 / Fact**：`pd_launcher` 必须在 import vLLM 和 worker spawn 前设置设备可见性、Mooncake protocol、side-channel 和 store 配置。资料记录的关键映射包括：

- `DLC_VISIBLE_DEVICES`，并镜像到 `CHIPLTECH_VISIBLE_DEVICES`。
- `VLLM_MOONCAKE_PROTOCOL`。
- `VLLM_MOONCAKE_SIDE_CHANNEL_PORT`。
- `MOONCAKE_LYP_STORE_HOST` 与 `MOONCAKE_LYP_STORE_PORT`。

Launcher 使用 `--` 分隔自身参数与转发给 vLLM server 的参数。具体参数名以实际 parser 为准。

## KV Cache Transfer Contract

### TCP 与 CPU pinned staging

文档化流程：

1. 两侧为 KV Cache tensor 分配同 shape、同 dtype 的 CPU pinned staging buffer，并向 TransferEngine 注册 CPU memory。
2. Decode 通过 metadata 发布本端 staging address、request identity 和 block IDs。
3. Prefill 从 DLC KV Cache 选取对应 blocks，复制到 Prefill staging。
4. TransferEngine 将对应范围写到 Decode staging，并返回 transfer completion。
5. Decode 把收到的 blocks 写回 DLC KV Cache，并通过 `torch.dlc` 同步。

必须验证 allocation lifetime、注册长度、block offset、DLC-side 最终同步、pinned-memory/memlock 容量。checksum、absmax、mean、first values 和 NaN/Inf 只能作为 fingerprint diagnostics，不能证明完整 tensor 相等。

TCP 路径引入 DLC-to-CPU 和 CPU-to-DLC 拷贝与 pinned-memory 占用。任何“性能差异不大”的结论都必须绑定模型、拓扑、并发、输入输出长度和测量结果。

**已观察故障 / Observed failure**：部分 TransferEngine build 即使收到 protocol `tcp`，仍会 auto-discover HCA 并安装 RDMA transport。日志中的 `Auto-discovering topology`、`installTransport, type=rdma`、completion queue allocation failure 或 `No available RNIC` 表明 exact build 不具备 TCP-only 初始化语义。Python import、单个 engine 初始化或 protocol 字符串不能替代双 role Transport Qualification Gate。

### `lyp_full`

`lyp_full` 绕过 TransferEngine memory registration，以 DLCCommunicator/DLCCL + LYP 传输每个 layer/component gather 后的连续 tensor。需要满足：

- Prefill 先启动并建立 store master。
- 两侧 TP、store host/base、每个 `base + tp_rank` 端口、shape、dtype、component 顺序和 tag 策略一致。
- 正确选择 `torch.dlc` device 并完成同步。
- 评估 gather tensor 带来的设备内存峰值和并发 DLCCL operation 稳定性。

**经验 / Experience**：`LYPFullTransportManager` 可能在 API ready 前阻塞于 TCPStore。正确顺序是启动 Prefill、观察 store listener、立即启动 Decode，然后再等待两个 role readiness。等待 Prefill `/health` 后才启动 Decode，可能把 store timeout 耗尽并形成交替超时。

**已观察故障 / Observed failure**：当 request identity、block count、shape/dtype、tag/order 均对齐，但 `ProcessGroupDLCCL.send().wait()` 与 `recv().wait()` 不完成时，应回到 Transport Qualification Gate 和 LYP group status；这比继续修改 request metadata 更接近最早失败边界。

### `dlccl_direct`

部分 feature branch 提供 `_dlccl_direct` native extension 与 `NativeDLCCLTransportManager`：

1. Connector 记录 KV tensor pointer、shape、stride、item size、block dimension 和 local device index。
2. Decode 生成 DLCCL unique ID，经 control plane 让 Prefill 启动 rank 0，再启动 Decode rank 1。
3. Prefill 将选中 block 通过 native DLCCL 发送，Decode 直接接收到声明的 local blocks。
4. 当前实现通常用 communicator lock 串行化请求，除非 exact code 另有并发证明。

使用前必须固定 branch/full SHA、extension build identity、Python ABI、linked `libdlccl` 和 payload benchmark。手工构建 extension 是本机 qualification artifact，不等于正式 wheel。`DLC_VISIBLE_DEVICES=<physical-id>` 后 native API 通常使用 process-local device 0；必须同时记录物理和 local identity。

旧 `lyp` chunked 路径在源资料中已弃用。`rdma_direct` 只有在 driver 与 connector 对 device-memory registration/GDR-equivalent capability 提供独立证据时才可重新评估；当前资料把它视为不可用。

## KV Cache Layout

源资料记录了三种 shape：

| Layout | Shape | Block dimension |
|---|---|---|
| TorchAttention merged | `(num_blocks, num_kv_heads, block_size, head_size * 2)` | `0` |
| TorchAttention non-merged | `(2, num_blocks, num_kv_heads, block_size, head_size)` | `1` |
| SparseMLA | `(num_blocks, block_size, head_size)` | `0` |

对首维为 2 的 5D tensor，资料中的 block 数来自 `shape[1]`；其他 shape 默认来自 `shape[0]`，block size 来自 `shape[-2]`。

**未验证 / Not verified**：源资料同时使用“当前为 5D non-merged”和“多数场景 merged/unified、`split_k_and_v=False`”两种描述。执行时必须从 exact DLC Attention Backend tensor 和 connector branch 记录 K/V/component 语义、shape、stride、dtype 和 block dimension，不能只按维数猜测。

## 拓扑与端口

### 单机 TCP

Prefill 和 Decode 使用不重叠的 device sets；两者均选择 `tcp`，Proxy 连接两个本地 API。数据仍经过各自 CPU staging，不等同于 direct P2P。

### 单机 `lyp_full`

device sets 同样不重叠。两侧使用相同 store host/base，先 Prefill 后 Decode，确保所有 rank-derived store ports 可用。

### 单机 `dlccl_direct`

仅在 exact checkout 明确支持且 native extension 已构建时使用。目标设备所在 LYP group 必须独立通过，随后用相同物理可见设备和 process-local index 通过 content-validating native benchmark。一个 group 通过不能证明另一个 group 或整机所有 pair。

### 跨机器 TCP

每个 role 使用本机 device numbering。Peer endpoint 必须是从对端 namespace 可路由的地址；`0.0.0.0` 只用于 bind，不能作为 peer destination；跨机器不得发布 loopback 或 container-private address。

端口计划至少覆盖：

- Prefill/Decode/Proxy API。
- Prefill 与 Decode 的 `side_channel_base + tp_rank`。
- `lyp_full` 的 `store_base + tp_rank`。
- TCP 的 TransferEngine RPC/handshake 端口。

源资料未完整枚举 TransferEngine 端口，部署前必须从 actual code/config 发现并验证双向连通性。

## 分层验证流程

1. **Monolithic baseline**：同一 model identity、tokenization、deterministic prompt 和 decoding parameters 先通过单体 serving。
2. **Static contract**：两侧 identity、role profile、connector、layout、topology、endpoint matrix 和 Site Recovery Contract 闭合。
3. **Read-only preflight**：device/process/memory、group-scoped LYP/link test、operator smoke、ports 和 peer reachability 有明确结果。
4. **Transport Qualification Gate**：使用部署同款 visibility/process boundary 传输非空 payload，双端 completion 且内容一致。
5. **Role startup**：Prefill、Decode、Proxy 各有独立 server epoch，launcher/connector/transport 日志符合 contract；readiness probe 来自 actual routes/capability，不假设 Proxy 一定有 `/health`。
6. **Single request**：通过 Proxy 发一个 deterministic request，关联 client、Proxy、Prefill 和 Decode request identity。
7. **KV transfer**：保存 Prefill block retention/registration、Decode `remote_request_id`、local/remote block mapping、transport completion 和 DLC-side receive/synchronization。
8. **Functional equivalence**：响应满足声明的 assertion，两个 role health-after 和 Proxy 实际请求通过。
9. **Lifecycle/concurrency**：逐步增加请求、并发和 context，每轮只改变一个变量。
10. **Performance**：分别报告 TTFT、TPOT、ITL、throughput、queueing 和 KV-transfer cost。
11. **Cleanup/recovery**：只停止 task-owned Proxy、Decode、Prefill；process/port/device memory、HBM/频率/link 和既有 workload 回到 sealed baseline。

## Evidence 与状态边界

建议独立报告：

```text
static_configuration
service_readiness
request_routing
kv_transfer
functional_equivalence
lifecycle_cleanup
performance_workload
stability_baseline
```

- Static consistency 不证明进程执行。
- 两侧 ready 不证明 Proxy routing。
- Proxy 返回 response 不证明指定 Prefill 提供了 KV。
- Connector handshake 不证明 data plane 已完成。
- Benchmark workload pass 不等于 stable performance baseline。
- PD 结果不建立 Verified vLLM Alignment，也不自动提升为通用 Real DLC Hardware acceptance。

## 常见失败分类

| 最早失败阶段 | 优先检查 |
|---|---|
| Role startup | visible devices、TP、model path、package/source identity、link/operator preflight、rank-derived ports |
| Control plane | Proxy targets、server epoch、side channel、local/remote request ID、advertised host |
| KV metadata/layout | block IDs、block dimension、shape/stride/dtype、component ownership、两侧 connector generation |
| TCP transfer | pinned allocation/memlock、registration length、offset、RPC/firewall/container network、最终 `torch.dlc` sync |
| `lyp_full` hang | Prefill-first、store/rank ports、TP、tag/order、DLCCL/LYP health、peak memory |
| `dlccl_direct` failure | exact protocol support、extension/libdlccl ABI、physical/local device identity、communicator init、native send/recv completion |
| Transport gate hang | group-scoped LYP status、payload completion/content、stale task PGID；先于模型/connector语义修改 |
| LYP init/repair | timestamped log、reset script execution、tool-local to physical index mapping、group-scoped status、maintenance authorization |
| Functional divergence | monolithic baseline、prompt tokens、decoding params、first divergent token/logits、model-specific Attention/MLA/MoE/MTP |
| Performance regression | queueing、Prefill compute、copy/transfer、Decode compute、warm-up、workload equivalence |
| Cleanup leakage | task process group、ports、device allocation；Host maintenance 单独授权 |

## Host 维护边界

Chip ID 写入、kernel module reload、HBM repair、Bluejay/firmware 初始化、会改变状态的 LYP init、power cycle、PCIe generation reconfiguration、reboot、`--privileged` + Host PID/system mounts，以及终止非 task-owned process 都是 Host-maintenance action。Device execution 授权不包含这些操作。

执行前必须建立 Site Recovery Contract。即使工具使用 `-t` 或 `--lyp-limit-*`，也不能假设它不会全局清理 DLC process、重装 driver 或重建容器。维护后重新封存 device numbering、HBM/frequency/link、container、ports 和既有 workload；恢复服务必须验证 model identity 和实际 completion，不只 `/health`。

停止 `docker exec` client 不证明 Host-PID container 内的 process 已退出。cleanup 必须检查 task-owned PID/PGID、listener 和 device ownership。若无法安全恢复，记录 `blocked_cleanup_incomplete`。

## 相关资料

- [Model Adaptation 与 Main-to-Main 决策](model-adaptation-and-main-to-main-decisions.md)
- [性能分析](../runtime-debugging/performance-profiling.md)
- [Arsenal CI 与黑盒测试](../testing/arsenal-ci-and-blackbox-testing.md)
- [PD 分离可复用 Prompt](../prompt-examples/vllm-dlc-prefill-decode-separation.md)
- [node-8-45 PD transport 与 LYP 恢复案例](../case-studies/vllm-dlc-pd-transport-and-lyp-recovery.md)
- 外部 workflow：`skills.git` 中的 `skills/engineering/pd-separation/SKILL.md`，需安装到 Kilo 或从 skills checkout 读取。
