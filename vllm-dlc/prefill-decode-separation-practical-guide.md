# vLLM-DLC PD 分离通用实操配置指南

## 1. 用途与结论边界

本文是一份可复制、可填写、可审计的 PD 分离执行模板，适用于：

- `X` 个 Prefill 实例组成 P 池；
- `Y` 个 Decode 实例组成 D 池；
- 单机或跨机器部署；
- `MooncakeDLCConnector` 的 `tcp` 路径；
- 已经独立完成资格验证的 `lyp_full` 或 exact-checkout `dlccl_direct`；
- 普通功能验证，以及只采集指定请求的 Prefill/Decode profiling。

本文不提供一组“到处都能直接运行”的固定 IP、端口或 TP 值。所有
`<PLACEHOLDER>` 都必须在执行前替换，并把替换后的版本作为本次运行合同保存。
底层原理、transport 差异和 claim boundary 见
[Prefill/Decode Separation](prefill-decode-separation.md)。

**事实 / Fact**：通常一个请求由 Proxy 选中一个 P 实例和一个 D 实例完成。
`X P + Y D` 表示实例池规模和调度能力，不表示一个请求自动由全部 `X+Y`
实例共同计算。只有 exact implementation 明确提供跨实例并行语义时才能作不同解释。

**验收边界**：以下状态必须分开报告：

```text
static_configuration
transport_qualification
service_readiness
request_routing
kv_transfer
functional_equivalence
profiling_isolation
performance_workload
lifecycle_cleanup
```

`/health=200`、Proxy 返回 HTTP 200、connector handshake 或非空输出，都不能单独
证明 KV Cache 数据面完成。

## 2. 先填写运行合同

### 2.1 全局信息

| 字段 | 本次填写值 |
|---|---|
| Run ID | `<RUN_ID>` |
| 操作人/时间 | `<OWNER_AND_TIME>` |
| 拓扑 | `<single_node_tcp / cross_machine_tcp / qualified_lyp_full / qualified_dlccl_direct>` |
| P 实例数 | `<X>` |
| D 实例数 | `<Y>` |
| Proxy 地址 | `<PROXY_HOST>:<PROXY_PORT>` |
| Artifact 根目录 | `<ARTIFACT_ROOT>` |
| 模型绝对路径 | `<MODEL_PATH>` |
| API model alias | `<SERVED_MODEL_NAME>` |
| tokenizer/processor identity | `<TOKENIZER_IDENTITY>` |
| weights revision/digest | `<MODEL_REVISION_OR_DIGEST>` |
| vLLM full SHA / package | `<VLLM_IDENTITY>` |
| vllm-dlc full SHA / package | `<VLLM_DLC_IDENTITY>` |
| mooncake-dlc full SHA / package | `<MOONCAKE_DLC_IDENTITY>` |
| PyTorch DLC Backend | `<PYTORCH_DLC_IDENTITY>` |
| DLC Runtime/driver/image | `<RUNTIME_DRIVER_IMAGE_IDENTITY>` |
| Transport | `<tcp / lyp_full / dlccl_direct>` |
| 功能断言 | `<EXPECTED_OUTPUT_OR_COMPARISON_CONTRACT>` |

不要只记录 branch 名、容器名或 `pip show` 版本。editable install、容器内 source 和
实际 worker import path 都要能够对应到 exact identity。

### 2.2 P/D 统一模型合同

下表的 P、D 值必须相同，或有明确且经过验证的兼容关系：

| 参数 | P | D |
|---|---|---|
| dtype | `<P_DTYPE>` | `<D_DTYPE>` |
| quantization | `<P_QUANT>` | `<D_QUANT>` |
| TP / PP / EP / DCP | `<P_PARALLELISM>` | `<D_PARALLELISM>` |
| block size | `<P_BLOCK_SIZE>` | `<D_BLOCK_SIZE>` |
| max model length | `<P_MAX_MODEL_LEN>` | `<D_MAX_MODEL_LEN>` |
| KV cache dtype/layout | `<P_KV_CONTRACT>` | `<D_KV_CONTRACT>` |
| Attention/MLA 模式 | `<P_ATTN_CONTRACT>` | `<D_ATTN_CONTRACT>` |
| model overrides | `<P_MODEL_OVERRIDES>` | `<D_MODEL_OVERRIDES>` |
| speculative decode | `<P_SPEC_CONFIG>` | `<D_SPEC_CONFIG>` |
| MoE/EP 参数 | `<P_MOE_EP_CONFIG>` | `<D_MOE_EP_CONFIG>` |

首次打通 PD 时建议关闭 speculative decode，并保持最简单、已通过 monolithic
baseline 的 MoE/EP 配置。PD 基线通过后，每轮只增加一个变量，并创建新的 Run ID；
否则 MTP、MoE routing、KV transfer 和 transport failure 会混在同一个现象里。

### 2.3 X P / Y D 实例清单

复制行直到覆盖全部实例：

| Role | 名称 | Host/容器 | 可见设备 | API | side-channel base | profile 目录 |
|---|---|---|---|---|---|---|
| P | `<P0_NAME>` | `<P0_HOST>` / `<P0_CONTAINER>` | `<P0_DEVICES>` | `<P0_API_HOST>:<P0_API_PORT>` | `<P0_SIDE_BASE>` | `<P0_PROFILE_DIR>` |
| P | `<P1_NAME>` | `<P1_HOST>` / `<P1_CONTAINER>` | `<P1_DEVICES>` | `<P1_API_HOST>:<P1_API_PORT>` | `<P1_SIDE_BASE>` | `<P1_PROFILE_DIR>` |
| D | `<D0_NAME>` | `<D0_HOST>` / `<D0_CONTAINER>` | `<D0_DEVICES>` | `<D0_API_HOST>:<D0_API_PORT>` | `<D0_SIDE_BASE>` | `<D0_PROFILE_DIR>` |
| D | `<D1_NAME>` | `<D1_HOST>` / `<D1_CONTAINER>` | `<D1_DEVICES>` | `<D1_API_HOST>:<D1_API_PORT>` | `<D1_SIDE_BASE>` | `<D1_PROFILE_DIR>` |

同一物理 Host 上的实例不得使用重叠设备、API 端口、side-channel rank 端口或
store 端口。容器内看到的 device 0 不自动等于 Host physical device 0，必须同时记录
physical/local mapping。

## 3. 端口矩阵

执行前完成下面的矩阵，并从实际 P、D namespace 双向验证：

| 用途 | 监听方 | 端口/范围 | 对端必须可达 | 验证结果 |
|---|---|---|---|---|
| Proxy API | Proxy | `<PROXY_PORT>` | Client | `<PASS/FAIL>` |
| P API | 每个 P | `<P_i_API_PORT>` | Proxy | `<PASS/FAIL>` |
| D API | 每个 D | `<D_j_API_PORT>` | Proxy | `<PASS/FAIL>` |
| P side channel | 每个 P rank | `<P_i_SIDE_BASE> ... <P_i_SIDE_BASE+TP-1>` | D/Proxy，按实现 | `<PASS/FAIL>` |
| D side channel | 每个 D rank | `<D_j_SIDE_BASE> ... <D_j_SIDE_BASE+TP-1>` | P/Proxy，按实现 | `<PASS/FAIL>` |
| `lyp_full` store | 每个 rank | `<STORE_BASE> ... <STORE_BASE+TP-1>` | P 与 D | `<PASS/FAIL>` |
| TransferEngine RPC/handshake | actual TransferEngine | `<DISCOVERED_PORTS>` | 对端 role | `<PASS/FAIL>` |

`0.0.0.0` 只能作为 bind address，不能发布成 peer destination。跨机器不能发布
loopback、container-private IP 或仅在本机路由表中存在的地址。

**重要**：普通 TCP 路径除固定 side-channel 外，还可能由 TransferEngine 动态选择
RPC/handshake 端口。必须从 exact code、启动日志和 `ss -lntp` 发现实际端口，不能只
放通 API 和 `side_base + rank`。若 metadata 公布的端口在对端没有 listener，服务仍可能
显示 `/health=200`，但第一个真实 KV 请求会失败。

基础检查：

```bash
# 每台机器分别执行
ss -lntp

# 从 Proxy namespace 检查各 P/D API
curl -fsS http://<P_i_API_HOST>:<P_i_API_PORT>/health
curl -fsS http://<D_j_API_HOST>:<D_j_API_PORT>/health

# 从实际数据发送方检查已经发现的 peer port
nc -vz -w 3 <PEER_HOST> <DISCOVERED_TRANSFER_PORT>
```

## 4. 启动前门禁

### 4.1 Monolithic baseline

使用相同模型、tokenization、dtype、decoding 参数和 deterministic prompt，先在单体
serving 中完成：

- API 请求成功；
- 输出满足 `<EXPECTED_OUTPUT_OR_COMPARISON_CONTRACT>`；
- 至少生成一个 token；
- 服务在请求后仍健康；
- 保存 server/client 命令和日志。

Monolithic 失败时不要进入 PD。此时最早失败点还在模型、DLC operator、并行配置或
基本 serving，而不是 KV transfer。

### 4.2 Read-only preflight

每个 role 至少保存：

```bash
date
hostname
ps -ef
ss -lntp
ulimit -l
python3 -m pip show vllm vllm-dlc mooncake-transfer-engine 2>/dev/null
```

同时记录 DLC 设备、进程、HBM、频率和 link 状态。确认：

- role 所需设备空闲或归本任务所有；
- P/D device set 不重叠；
- 模型目录和 connector module 在容器内可读；
- pinned-memory/memlock 能满足 TCP CPU staging；
- 没有旧 epoch 遗留的 API、side-channel、store 或 dynamic transfer listener。

### 4.3 Transport Qualification Gate

在加载大模型前，用目标容器、目标可见设备和相同 transport 做双端并发初始化，传输
非空 payload，并要求：

1. sender 和 receiver 都完成；
2. payload 长度一致；
3. 内容逐字节或按声明精度合同一致；
4. 两端进程无 hang、无隐式 RDMA 初始化错误；
5. 测试后 listener 和进程可清理。

只有 import 成功、单端 engine 初始化或 benchmark 正常退出，不算 transport gate 通过。
若 `tcp` 日志仍显示 `installTransport, type=rdma`，必须按 exact TransferEngine capability
处理，不能写成 TCP-only 已通过。

## 5. 生成角色启动命令

以下是命令骨架。先在 staging 文档中替换全部占位符，再执行替换后的版本；不得直接把
尖括号模板粘贴进 shell。

### 5.1 Prefill role：对每个 P 实例执行

```bash
export PROFILE_RUN_DIR=<P_i_PROFILE_DIR>
export MODEL=<MODEL_PATH>
export PYTHONPATH=<MOONCAKE_AND_VLLM_SOURCE_PATHS>

KV_TRANSFER_CONFIG='{"kv_connector":"MooncakeDLCConnector","kv_role":"kv_producer","kv_connector_module_path":"mooncake.mooncake_connector_dlc_v1"}'

python3 -m mooncake.pd_launcher \
  --visible-devices <P_i_VISIBLE_DEVICES> \
  --mooncake-protocol <TRANSPORT> \
  --side-channel-port <P_i_SIDE_BASE> \
  -- \
  --host 0.0.0.0 \
  --port <P_i_API_PORT> \
  --model "$MODEL" \
  --served-model-name <SERVED_MODEL_NAME> \
  --dtype <DTYPE> \
  --tensor-parallel-size <P_TP> \
  --block-size <BLOCK_SIZE> \
  --max-model-len <MAX_MODEL_LEN> \
  --max-num-batched-tokens <MAX_BATCHED_TOKENS> \
  --gpu-memory-utilization <MEMORY_UTILIZATION> \
  --trust-remote-code \
  --enable-prefix-caching \
  --kv-transfer-config "$KV_TRANSFER_CONFIG" \
  <OTHER_VALIDATED_P_ARGS>
```

### 5.2 Decode role：对每个 D 实例执行

```bash
export PROFILE_RUN_DIR=<D_j_PROFILE_DIR>
export MODEL=<MODEL_PATH>
export PYTHONPATH=<MOONCAKE_AND_VLLM_SOURCE_PATHS>

KV_TRANSFER_CONFIG='{"kv_connector":"MooncakeDLCConnector","kv_role":"kv_consumer","kv_connector_module_path":"mooncake.mooncake_connector_dlc_v1"}'

python3 -m mooncake.pd_launcher \
  --visible-devices <D_j_VISIBLE_DEVICES> \
  --mooncake-protocol <TRANSPORT> \
  --side-channel-port <D_j_SIDE_BASE> \
  -- \
  --host 0.0.0.0 \
  --port <D_j_API_PORT> \
  --model "$MODEL" \
  --served-model-name <SERVED_MODEL_NAME> \
  --dtype <DTYPE> \
  --tensor-parallel-size <D_TP> \
  --block-size <BLOCK_SIZE> \
  --max-model-len <MAX_MODEL_LEN> \
  --max-num-batched-tokens <MAX_BATCHED_TOKENS> \
  --gpu-memory-utilization <MEMORY_UTILIZATION> \
  --trust-remote-code \
  --enable-prefix-caching \
  --kv-transfer-config "$KV_TRANSFER_CONFIG" \
  <OTHER_VALIDATED_D_ARGS>
```

`pd_launcher` 的参数必须出现在第一个 `--` 前，vLLM server 参数放在 `--` 后。
设备可见性、protocol、side-channel 和 store 环境必须在 import vLLM 和 worker spawn 前
完成，不能等 worker 启动后再修改。

### 5.3 Proxy：注册 X 个 P 和 Y 个 D

```bash
python3 -m mooncake.vllm_v1_proxy_server \
  --host 0.0.0.0 \
  --port <PROXY_PORT> \
  --prefiller-hosts <P0_HOST> <P1_HOST> <...> \
  --prefiller-ports <P0_API_PORT> <P1_API_PORT> <...> \
  --decoder-hosts <D0_HOST> <D1_HOST> <...> \
  --decoder-ports <D0_API_PORT> <D1_API_PORT> <...> \
  --prefiller-side-channel-hosts <P0_HOST> <P1_HOST> <...> \
  --prefiller-side-channel-ports <P0_SIDE_BASE> <P1_SIDE_BASE> <...>
```

host 列表与 port 列表必须按索引一一对应。某个实例不可用时应从本次 Proxy 配置中
移除，而不是保留死节点并依赖重试。修改池成员后重启 Proxy，保存新的 server epoch。

## 6. 启动顺序

### 6.1 普通 TCP

```text
确认无旧进程/端口
-> 启动全部 P
-> 启动全部 D
-> 两类 role 都完成模型和 KV cache 初始化
-> 检查 API 与实际 transport listener
-> 启动 Proxy
-> 发送最小确定性请求
```

### 6.2 `lyp_full`

Prefill 可能在 API ready 前等待 TCPStore peer，顺序应为：

```text
启动 P
-> 观察 store listener 已建立
-> 立即启动对应 D，不等待 P /health
-> P/D rendezvous 完成
-> 再等待两侧 API ready
-> 启动 Proxy
```

`dlccl_direct` 只用于 exact checkout/parser/extension 已固定且 native payload gate 已通过的
配置，不要把文档里的参数名推断到其他 branch。

## 7. 分层验证

### 7.1 Readiness

```bash
curl -fsS http://<P_i_HOST>:<P_i_API_PORT>/health
curl -fsS http://<D_j_HOST>:<D_j_API_PORT>/health
curl -fsS http://<PROXY_HOST>:<PROXY_PORT>/<ACTUAL_PROXY_HEALTH_ROUTE>
```

同时检查 `v1/models`、进程、listener 和 worker 日志。Proxy 不实现固定 health route 时，
使用 listener 加真实请求，不要假定一定存在 `/health`。

### 7.2 最小确定性请求

通过 Proxy 发送一个短 prompt、`temperature=0`、固定 token policy 的请求，并保存：

- client request ID 和完整响应；
- Proxy 选择的 P/D endpoint；
- P local request ID；
- D local request ID 与 `remote_request_id`；
- P block retention/registration；
- D local/remote block mapping；
- transfer completion 与 DLC-side receive/synchronization；
- 请求后 P、D、Proxy 的健康状态。

只有这些证据能用同一 request identity 串起来，才进入长上下文或 profiling。

### 7.3 功能等价

用与 monolithic baseline 相同的 prompt、sampling 和输出长度比较：

| 项目 | 结果 |
|---|---|
| 请求成功且响应完整 | `<PASS/FAIL>` |
| token/文本满足 comparison contract | `<PASS/FAIL>` |
| KV transfer completion | `<PASS/FAIL>` |
| P/D 请求后健康 | `<PASS/FAIL>` |
| 无新增 EngineCore/transport error | `<PASS/FAIL>` |

HTTP response headers 已发送但 body 中断，例如客户端 `IncompleteRead`，属于请求失败；
不能因为 Proxy 日志出现一次 `200` 就算功能通过。

## 8. 只采集第二条请求的 P/D profiling

### 8.1 填写 workload

| 字段 | 值 |
|---|---|
| 总 prompt tokens | `<TOTAL_TOKENS>` |
| 公共前缀 tokens | `<SHARED_TOKENS>` |
| 变化后缀 tokens | `<CHANGED_TOKENS = TOTAL-SHARED>` |
| 目标 block 命中率 | `<SHARED/TOTAL>` |
| warm-up 输出 tokens | `<WARMUP_OUTPUT_TOKENS>` |
| measured 输出 tokens | `<MEASURED_OUTPUT_TOKENS>` |
| block size | `<BLOCK_SIZE>` |
| selected P/D | `<SELECTED_P>` / `<SELECTED_D>` |

要求：

```text
TOTAL_TOKENS % BLOCK_SIZE == 0
SHARED_TOKENS % BLOCK_SIZE == 0
TOTAL_TOKENS + MEASURED_OUTPUT_TOKENS <= MAX_MODEL_LEN
available_kv_blocks >= ceil((TOTAL_TOKENS + MEASURED_OUTPUT_TOKENS) / BLOCK_SIZE) + safety_margin
```

为了可审计，直接提交 token IDs 或保存 tokenizer 后的 exact token sequence。两条请求构造为：

```text
warmup  = COMMON_PREFIX + SUFFIX_A
measured = COMMON_PREFIX + SUFFIX_B
```

两条总长度相同，仅后缀不同。不要用“字符数大约相同”的文本代替 token 合同。

### 8.2 固定路由

算子采集时推荐创建只包含一个 P 和一个 D 的临时 Proxy，使 warm-up 和 measured 都落到
同一 pair。若必须使用 X/Y pool，则需要：

- profile 所有候选 P/D；
- 保存 Proxy 的实际选择；
- 通过 request ID 和 ANSI active ranks 确认 selected pair；
- 不把未选中实例的空 profile 当成错误；
- 不把不同 replica 的 cycle 相加。

### 8.3 执行顺序

```text
1. 重启 selected P/D，清空 prefix cache、计数器和旧 profiler epoch
2. 确认 request counters=0、KV usage=0、无其他 completion 流量
3. 启动临时固定路由 Proxy
4. 发送 warm-up，请求完整返回；此时 profiler 未启动
5. 启动 selected P/D 的 framework profiler
6. 在每个 selected P/D 上等待 ANSI writer 安静并记录 start byte offsets
7. 发送唯一一条 measured 请求
8. 请求完整返回后记录 end byte offsets并切出 ANSI byte ranges
9. 停止 profiler并刷新 trace
10. 转换 selected P/D ANSI 为 cycle TXT
11. 保存 metrics delta、Proxy request correlation、服务日志和 checksum
```

必须先记录 ANSI end，再停止 profiler，避免把 profiler stop/flush 阶段混入算子切片。

### 8.4 Profiling 控制模板

```bash
# 每个 selected role 的 vLLM API
curl -f -X POST http://<SELECTED_P_API>/start_profile
curl -f -X POST http://<SELECTED_D_API>/start_profile

# 分别在 P/D 文件所在 namespace 记录 ANSI 起点
python3 <ANSI_WINDOW_TOOL> start <P_PROFILE_DIR> --label <REQUEST_LABEL> --expected-ranks <P_TP>
python3 <ANSI_WINDOW_TOOL> start <D_PROFILE_DIR> --label <REQUEST_LABEL> --expected-ranks <D_TP>

# 发送唯一 measured 请求后，分别记录终点
python3 <ANSI_WINDOW_TOOL> stop <P_PROFILE_DIR> --label <REQUEST_LABEL> --expected-ranks <P_TP>
python3 <ANSI_WINDOW_TOOL> stop <D_PROFILE_DIR> --label <REQUEST_LABEL> --expected-ranks <D_TP>

# 最后停止 framework profiler
curl -f -X POST http://<SELECTED_P_API>/stop_profile
curl -f -X POST http://<SELECTED_D_API>/stop_profile
```

如果 ANSI 文件只存在容器内，上述工具也必须在同一容器 namespace 执行，或先按保持
byte identity 的方式复制到 Host。不要在 Host 上对一个不存在或不同步的目录记录 offset。

### 8.5 验收统计

Prefill 的 measured compute 不是 `TOTAL_TOKENS`，而应为：

```text
measured_prefill_compute = TOTAL_TOKENS - prefix_cache_hit_tokens
```

检查 metrics/log delta，而不是只看请求参数。至少报告：

- prompt queried tokens；
- local prefix cache hit tokens；
- actual computed Prefill tokens；
- P 侧 KV export/copy/transfer 开销；
- D 侧 KV receive/copy/sync；
- D 首 token 与后续 Decode iterations；
- selected P/D 每个 rank 的 ANSI bytes 和 cycle TXT；
- client wall time，以及可审计时的 TTFT/TPOT/ITL。

各 TP/EP rank 的 cycle 不能直接相加当端到端延迟；P 与 D 的 cycle 也不能相加当请求
wall time。端到端关键路径要结合 Proxy/client 时间线和 transfer completion。

## 9. X/Y 池扩容验证

先用 `1P+1D` 完成 PD functional validation，再逐步扩容：

```text
1P+1D functional
-> 1P+1D long-context
-> 1P+1D profiling
-> XP+1D routing/lifecycle
-> 1P+YD routing/lifecycle
-> XP+YD concurrency
-> uninstrumented performance benchmark
```

每轮只改变一个维度，保存新的 Run ID。扩容验收至少覆盖：

- 每个实例都实际被选中过；
- request ID 与 selected pair 一致；
- 单实例退出时 Proxy 行为符合声明的 failover contract；
- 没有把新请求发给旧 server epoch；
- prefix cache 的归属和命中范围按实例解释；
- 负载不均衡、queueing 和 capacity 分别报告；
- profiling 与正式性能 benchmark 使用独立 epoch。

## 10. 常见故障与止损

| 现象 | 最早可能失败层 | 立即动作 |
|---|---|---|
| P/D `/health=200`，真实请求后动态端口 refused | TransferEngine data plane / firewall / container network | 停止请求；记录 metadata 公布端口；在监听方执行 `ss -lntp`，从发送方 `nc` 验证 |
| Proxy 出现 200，但 client `IncompleteRead` | Decode 崩溃或流式 body 中断 | 判定请求失败；按 request ID 查 P/D/Proxy 日志，不进入 measured profiling |
| D 返回 `EngineCore encountered an issue` 后 API 消失 | D worker/transfer fatal error | 从池中移除该 D，停止 Proxy；保留 crash epoch，不反复发送长请求 |
| `batch_transfer_sync_write ... -1` | P 到 D staging transfer 失败 | 检查 D advertised RPC port、registration、memlock、receiver lifetime |
| P prefix hit 与预期不符 | cache 未清、路由换 P、token sequence 不同、block 未对齐 | 新 epoch 重启 selected P；固定 route；保存 exact token IDs |
| measured 落到另一 D | pool routing 未固定 | 依据 Proxy request ID 确认 selected D；不要使用错误 D 的 profile |
| ANSI 包含 warm-up | start offset 记录过早或 writer 未安静 | 丢弃该切片，用干净 epoch 重跑；不能事后仅凭文件名声称隔离成功 |
| `lyp_full` 两侧 health 一直不出现 | store rendezvous 顺序错误 | P store listener 出现后立即启动 D，不等待 P health |
| native send/recv wait hang | LYP/DLCCL data plane 未通过 | 回到 Transport Qualification Gate 和 group-scoped link 状态 |
| 增加 speculative/MoE 参数后才失败 | 多变量 contract 改变 | 回到已通过基线，每次只恢复一个参数并单独记录 |

出现 transport fatal error、HTTP body 中断或 role crash 后，不要继续发送 measured 请求。
长上下文请求会污染 P cache；下一轮必须重启相关 role 并验证计数归零。

## 11. 停止、清理与产物

推荐 graceful stop 顺序：

```text
Proxy -> Decode roles -> Prefill roles
```

只停止本任务拥有的 session/PID/PGID。停止 `docker exec` client 不证明容器内 worker 已退出，
还要检查：

```bash
ps -ef
ss -lntp
curl --max-time 3 http://<OLD_API>/health
```

每个成功 Run 至少保存：

```text
<ARTIFACT_ROOT>/<RUN_ID>/
├── README.md                 # 填写后的合同、结论和 claim boundary
├── commands/                 # P/D/Proxy/client exact commands
├── identities/               # source/package/model/image/runtime identity
├── logs/                     # P/D/Proxy/client logs
├── metrics/                  # before/after raw metrics 与 delta
├── request-correlation/      # external/local/remote request ID
├── ansi/                     # 原始或 byte-range-isolated ANSI
├── cycle-txt/                # 每 rank 和明确口径的汇总
├── trace/                    # framework/runtime trace
└── SHA256SUMS
```

失败 Run 也要保留最早失败证据，但目录名和 README 必须明确标记 `failed` 或
`not_verified`，不得与成功 profiling 混放。

## 12. 最终验收表

| 验收项 | 必须证据 | 结果 |
|---|---|---|
| Static configuration | 填写后的模型、source、role、cache、endpoint 合同 | `<PASS/FAIL>` |
| Transport qualification | 非空 payload 双端 completion + 内容一致 | `<PASS/FAIL>` |
| Service readiness | 每个 role 实际 route、listener、worker 状态 | `<PASS/FAIL>` |
| Request routing | Proxy 与 P/D request ID 关联 | `<PASS/FAIL>` |
| KV transfer | block mapping、transfer completion、DLC receive/sync | `<PASS/FAIL>` |
| Functional equivalence | 与 monolithic comparison contract 一致 | `<PASS/FAIL>` |
| Profiling isolation | warm-up 排除、offset、active ranks、唯一 measured 请求 | `<PASS/FAIL/NA>` |
| Performance workload | 固定 workload、重复 attempts、指标与离散度 | `<PASS/FAIL/NA>` |
| Cleanup | task process、端口、设备/HBM 回到 sealed baseline | `<PASS/FAIL>` |

只有 request-correlated routing、KV consumption 和 functional equivalence 同时通过，才能写
`pd_validated`。只完成配置、readiness 或 HTTP 请求时，结论保持 `not_verified`。

## 13. 相关资料

- [Prefill/Decode Separation](prefill-decode-separation.md)
- [PD transport 与 LYP 恢复案例](../case-studies/vllm-dlc-pd-transport-and-lyp-recovery.md)
- [PD 分离可复用 Prompt](../prompt-examples/vllm-dlc-prefill-decode-separation.md)
- [性能 Profiling](../runtime-debugging/performance-profiling.md)
- [DLC 设备观测](../runtime-debugging/chipltech-smi-observability.md)
