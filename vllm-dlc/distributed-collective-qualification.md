# vLLM-DLC Distributed Collective Qualification

## 适用场景

- 一个模型或 deployment profile 会激活 TP/EP/PP/DP/DCP 或 MoE dispatch/combine。
- 需要在模型启动前判断 exact collective route 是否已实现并具备匹配 identity 的资格证据。

## 分层合同

以下层级必须分别记录，不能互相替代：

1. DLC_CL/DLCCL native symbol、ordered descriptor 和 exact loaded binary。
2. PyTorch ProcessGroup DLCCL public API、dtype/shape/count、completion boundary 和 fallback。
3. vLLM DLC communicator 实际 route；native symbol 存在不证明该 route 调用了 collective。
4. 模型 workload 激活的 primitive、rank/world size、rank order 和 topology。
5. active MoE dispatch/combine、Public Operator Schema、KernelDesc Descriptor ABI 与 DLC Custom Kernel Entry ABI/binary。
6. topology/payload-aware route 的 selector owner、输入、稳定 strategy ABI、metadata domain、cache key 和 fallback validation。

未实现 route 必须在模型启动前返回 `blocked_collective_unimplemented`。实现存在但 exact identity/fixture/Real DLC Hardware qualification 未闭合时返回 `blocked_collective_not_qualified`。不得返回 shape 正确但内容未初始化的 tensor，也不得以 CPU fallback 形成 DLCCL production 或性能证据。

## Topology/Payload-Aware Selection

当一个 collective 有多种 topology-specific 实现时，qualification 还必须验证以下 ownership 和状态机：

机器合同并行保留 frozen `vllm-dlc-distributed-collective-qualification/v1` 与 exact `v2`；validator 不把 v1 route 静默解释为 v2。v2 只允许 communicator route 持有 `selection`，并使用独立版本的 closed-world `vllm-dlc-collective-selection/v1` 子合同，显式绑定 selector source/binary identity、subject hardware topology digest、payload bytes、dtype/layout constraints 与 actual layout、local rank、primary/secondary root 角色、rank domain 和 mapping、actual rank-order permutation、selected strategy ID、exact native consumer route ID/metadata ABI digest、完整 selection-input cache-key digest，以及 fallback preferred/candidate 的逐项 validation/commit state。rank order 可以是任意完整、唯一、in-range permutation，不要求自然顺序。

1. 已初始化 communicator 是实际 LYP topology、正式 lookup table、rank/root metadata 和 rank order 的唯一 selector owner；world size 只描述参与者数量，不证明 topology。
2. framework 计算并传入真实 payload 及必要的 dtype/layout 信息，按所有 selection inputs 缓存结果，并只把稳定 strategy 与 descriptor metadata 传给 DLC Custom Kernel。payload-aware config 不得按 communicator 单键缓存。
3. DLC Custom Kernel 按 strategy dispatch。payload capacity、alignment、root 可用性等正常 capability 边界在 launch 前处理；kernel assert 保留给非法 strategy 或 strategy/rank 等 descriptor ABI 矛盾。
4. 首选实现失败后先形成 fallback candidate；未 commit 时 selected strategy 仍是 preferred strategy。再从 `Unknown` 状态验证候选自身的 graph、channel、完整 rank order、rank range、无重复 rank、primary/secondary root 和所需 metadata。验证成功后才能提交并替换 selected strategy；无候选通过时 fail closed。
5. root domain 转换必须显式记录 communicator-local、logical rank 与 physical rank 的映射。多 root 实现缺 secondary root 时，只能使用 formal selector 已定义且可验证的较弱专用实现，否则进入完整 fallback validation。

现有 machine tests 覆盖 threshold 边界和越界、alignment 满足/不满足、root 缺失/越界/重复、未知 mapping、rank order 不完整/重复/越界、fallback partial validation、同一 communicator payload 变化后的 cache-key mismatch、layout contradiction 与 strategy/rank descriptor contradiction。controlled base fixture 仍只产生 non-authoritative fixture evidence。source/selector regression 证明决策结构；Real DLC Hardware content correctness 仍需绑定 exact source、binary、topology 和 payload，并检查所有 rank 输出与 task-owned 资源清理。

相关 historical supporting evidence 见 [QKNorm Topology-Aware AllReduce Selection](../case-studies/qknorm-topology-aware-allreduce-selection.md)。该案例不属于 qualification fixture，其中的具体 strategy ID、阈值和 helper 不是通用默认值。

## Runner 边界

bounded runner 需要外部 watchdog、每 primitive timeout、rank exit-code 汇总、任务进程树终止、cleanup 和 post-failure health snapshot。受控 fixture 只验证 contract、timeout、聚合和 cleanup；没有可信 authorization、硬件 observation、固定 harness identity 与内容 correctness schema 时，真实 qualification 必须 fail-closed。

Real DLC Hardware 不可用时，硬件 scope 保持 `blocked_missing_hardware` 或 `not_verified`。SMI Observation Envelope 只支持设备、进程、HBM 和 cleanup 观察，不证明 collective 内容正确。

Provider freshness 的 library consumer 可以接收受控的 UTC clock context，使一次 collection epoch 及其稳定性复核共享同一时刻；production CLI 不提供时间回拨参数，只使用真实 UTC。该 seam 仅用于确定性验证 future、valid window、exact expiry 和 stale generation，不证明 Host system clock 可信，也不证明 provider seal 已认证。

Claim Boundary: 本文定义 distributed safety 与证据边界，不证明当前 source/package/native binary/image/model/workload/topology 已配对，不证明任何 Real DLC Hardware collective、模型、benchmark 或 image acceptance。
