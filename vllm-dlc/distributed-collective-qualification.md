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

未实现 route 必须在模型启动前返回 `blocked_collective_unimplemented`。实现存在但 exact identity/fixture/Real DLC Hardware qualification 未闭合时返回 `blocked_collective_not_qualified`。不得返回 shape 正确但内容未初始化的 tensor，也不得以 CPU fallback 形成 DLCCL production 或性能证据。

## Runner 边界

bounded runner 需要外部 watchdog、每 primitive timeout、rank exit-code 汇总、任务进程树终止、cleanup 和 post-failure health snapshot。受控 fixture 只验证 contract、timeout、聚合和 cleanup；没有可信 authorization、硬件 observation、固定 harness identity 与内容 correctness schema 时，真实 qualification 必须 fail-closed。

Real DLC Hardware 不可用时，硬件 scope 保持 `blocked_missing_hardware` 或 `not_verified`。SMI Observation Envelope 只支持设备、进程、HBM 和 cleanup 观察，不证明 collective 内容正确。

Claim Boundary: 本文定义 distributed safety 与证据边界，不证明当前 source/package/native binary/image/model/workload/topology 已配对，不证明任何 Real DLC Hardware collective、模型、benchmark 或 image acceptance。
