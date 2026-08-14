# QKNorm Topology-Aware AllReduce Selection

## 问题现象

QKNorm 在 Tensor Parallel 下需要跨 rank 聚合 Q/K 方差。相同 world size 在不同 Real DLC Hardware LYP 连线下可能对应不同 topology、root metadata 和可用 AllReduce 实现，因此 PyTorch DLC Backend 或 DLC Custom Kernel 不能仅凭 4/8 rank 选择算法。

本案例记录 2026-08-13 一次跨 DLC_CL、PyTorch DLC Backend 和 DLC_Custom_Kernel Repository 的历史实现与 review 闭环。它保存可复用的 selector ownership、fallback 状态机、ABI 和验证经验，不把 MiniMax 专用阈值或 strategy 编号提升为全局规则。

Evidence status: `historical_report_derived_observation`。原始个人路径当前为 `external_unavailable`，仓内没有足以重建 exact source/artifact manifest 的材料，因此 exact identity 为 `identity_unavailable`。以下“事实”是历史报告内的事实分类，不是当前 checkout 的 `direct_repository_evidence`；不得猜填缺失 SHA/digest。

不可继承：本案例不能自动成为 approved Stack Policy、current topology qualification、current model acceptance 或 current performance baseline。

## 历史方案边界

- **事实 / Fact**：DLC_CL 依据已初始化 communicator 的实际 LYP topology 和 QKNorm variance payload 复用正式 4-device/8-device lookup table，向 PyTorch DLC Backend 返回稳定 strategy、root 和 Ring order。
- **事实 / Fact**：PyTorch DLC Backend 计算 QKNorm variance payload，以 payload 为 cache key 查询 config，并将 strategy 与必要 metadata 写入 KernelDesc Descriptor ABI。
- **事实 / Fact**：DLC Custom Kernel 只按 strategy dispatch helper，并保留 strategy/rank 与 unknown strategy 的 ABI 防御。
- **事实 / Fact**：历史 source regression、DLC_CL shared-library build/symbol export、DLC Custom Kernel object build 均通过；早期 exact artifact 曾在 TP4 专用路径和 TP8 generic Ring 上以两个小 payload 完成 CPU Reference 与 V passthrough 检查。
- **未验证 / Not verified**：review 后全部专用 topology、fallback 边界及最终 exact artifact 尚未在各目标 LYP topology 上完成 Real DLC Hardware 数值复验。旧 runtime evidence 不自动转移给后续 amended source、binary 或 image。

## 跨仓 Ownership

| 层级 | Owner | Contract |
|---|---|---|
| topology 与 payload selection | DLC_CL / DLCCL | 读取 communicator topology、正式 lookup table、payload 和 root/Ring metadata；只返回已验证 config |
| framework adapter | PyTorch DLC Backend | 计算真实 payload、按 selection inputs 缓存、转换 rank domain、组装 KernelDesc Descriptor ABI |
| execution dispatch | DLC_Custom_Kernel Repository | 消费稳定 strategy 和 metadata，调用对应 helper，防御 descriptor ABI 损坏 |

world size 只能验证 strategy/rank 是否矛盾，不能成为 topology selector。framework 不复制 lookup table，DLC Custom Kernel 不从 `tp_world` 猜算法，调用者提供的 rank list 不替代 communicator 的 Ring order。

## Verified Fallback 状态机

历史 review 将“专用实现不可用”和“ABI 损坏”分开处理：

```text
formal selector chooses preferred implementation
  -> payload/alignment/root prerequisites pass
     -> commit preferred strategy

  -> formal selector defines another supported specialized implementation
     -> validate and commit that implementation

  -> generic fallback is allowed
     -> reset config to Unknown
     -> validate fallback graph/channel/order/rank metadata
        -> pass: commit fallback strategy
        -> fail: remain Unknown and fail closed
```

payload 超出专用 helper capacity、alignment 不满足、primary root 缺失或越界，都是 launch 前 capability selection 问题，不应进入 kernel 后 Assert。secondary root 不可用或与 primary root 重复时，只有正式 selector 已定义的 one-root 降级可以直接成为下一个候选；否则仍须走完整 fallback validation。

一个关键 review 修复是将“strategy 已标成 Ring”改为“Ring candidate”。进入 fallback 路径先重置 `Unknown/-1`，只有 graph ID/pattern、channel、完整 rank order、rank range 和唯一性全部通过后才提交 Ring。这样验证中途失败不会泄漏一个未验证的成功状态。

## ABI 与 Cache 经验

1. strategy int 是跨仓 ABI，应有单一映射 owner；重复的合法 strategy 列表会漂移，unknown 检查应与实际 dispatch `switch/default` 共址。
2. descriptor 防御覆盖每个 strategy 的 rank-count 前提，但不承担正常能力选择。
3. root 的 communicator-local、logical-rank 和 physical-rank domain 必须显式转换；把不同 domain 的整数直接透传会形成静默错路由。
4. selector 输入包含 payload 时，cache key 必须包含 payload 及其他影响 selection 的输入。每 communicator 单 config 会把第一次 shape 的策略错误复用于后续调用。
5. 多个 legacy helper header 同编译单元出现 file-scope 符号冲突时，可在 wrapper include 边界隔离符号，但不应借机复制或修改各 helper 的 RDMA/CMEM 协议。

## Review 与验证矩阵

Review 应从“每个候选为什么能被提交”反向检查，而不只枚举 happy-path strategy：

| 维度 | 必测边界 |
|---|---|
| payload lookup | 每个正式阈值的下侧、边界和上侧 |
| alignment | 专用路径满足与不满足；后者只在 fallback 完整通过后成功 |
| root metadata | primary/secondary 缺失、越界、重复及 rank-domain 转换 |
| Ring | graph/pattern/channel/order 缺失，rank 越界或重复 |
| mapping | 当前 adapter 未识别的 future algorithm mapping |
| cache | 同一 communicator 连续使用不同 payload |
| ABI | 每个 strategy 的 rank-count guard 和唯一 dispatch default |
| runtime | 所有 rank 的 content correctness、CPU Reference、passthrough 与 cleanup |

边界 payload 应由公式和正式 lookup table 生成，而不是长期复制一张 token 常量表。source/AST regression 可锁定 selector ownership、cache key 和 dispatch 结构；build/symbol 检查证明 artifact 可形成；只有绑定 exact source、loaded binary、实际 LYP topology 和 payload 的多 rank 运行才能建立 Real DLC Hardware collective correctness。

## 可复用经验

1. topology 是 communicator runtime state，不是 world-size heuristic。
2. capability failure 向上游 selector 收敛；kernel 只处理 execution 和 ABI corruption。
3. fallback 是重新 qualification 一个候选，不是无条件替换枚举值。
4. config 应以 `Unknown` 为初态，采用 validate-then-commit，避免 partial validation 泄漏成功状态。
5. lookup table、strategy mapping 和 dispatch 各自只有一个 owner；跨层传递稳定 ABI，不复制决策数据。
6. threshold、alignment、metadata 和 unknown mapping 都需要 negative tests；只跑基础 decode/prefill shape 无法证明 selector 完整。
7. source regression、build、历史 hardware smoke、目标 topology qualification 和 review approval 是不同状态，不能互相替代。

## 不应泛化

- 历史 QKNorm payload 公式、strategy ID、4-device/8-device lookup 阈值和 helper 名称。
- 某个 topology 必然对应某种算法，或同 world size 的机器拥有同一 LYP topology。
- include-time macro isolation 适用于所有 helper；新实现应优先消除 header 的 file-scope 符号泄漏。
- 算子同事一次“暂时没看到问题”的 review 反馈等于 runtime qualification 或未来 revision acceptance。

## Claim Boundary

本案例建立历史 source 事实、review 闭环和可复用的 Collective Selection Contract。它不证明当前 checkout、loaded DLCCL/DLC Custom Kernel binary、Host、LYP topology、模型、Real DLC Hardware correctness、性能或 image acceptance。任何新 target 都需要重新绑定 exact identity，并在实际 topology 上运行对应 boundary payload。

## 来源

- 个人过程资产：`external_unavailable`（历史位置已脱敏）
- 个人验证手册：`external_unavailable`（历史位置已脱敏）

以上绝对路径记录历史 provenance，其他 clone 可能不可访问；本文只保存从该内部工程材料独立提炼的 DLC-native 结论。
