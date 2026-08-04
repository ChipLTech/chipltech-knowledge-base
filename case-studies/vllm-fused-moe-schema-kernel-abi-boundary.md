# vLLM Fused-MoE Schema 与 Kernel ABI 边界

## 问题现象

一次 MiniMax-M2.7-Mix main 适配中，上层 main 已扩展 generic fused-MoE public schema，而运行环境继续使用冻结 baseline DLC Custom Kernel。模型可能在算子注册、worker 初始化、日志格式化或异步 completion 边界失败，表面症状不足以直接指出真实 owner。

## 证据与身份边界

- **事实 / Fact**：调查覆盖 DLC_Custom_Kernel Repository、PyTorch DLC Backend、vLLM 和 vLLM-DLC；最终交付只有后三个仓库的 patch。DLC_Custom_Kernel Repository 无 patch 是冻结 baseline 方案的设计结果。
- **事实 / Fact**：历史 P8 安装 artifact 曾完成 QKNorm/RoPE、TP4 DLCCL、模型加载、服务和两类功能请求。
- **事实 / Fact**：之后重建的正式 PyTorch wheel 完成 source test、torchgen、全量编译、wheel integrity、隔离安装和 schema/dispatch smoke。
- **未验证 / Not verified**：上述 P8 Real DLC Hardware evidence 绑定旧安装 artifact，不能转移给新正式 wheel 或候选 image；交付时新 artifact 的硬件复验仍待完成。
- **历史证据 / Historical evidence**：本案例基于 2026-08-04 交付材料。它不描述当前 Host、container、device、process、HBM 或服务状态。
- **历史证据身份 / Historical evidence identity**：交付目录的 `artifact-manifest.sha256` digest 为 `sha256:d262e16123a3bad2f1c240e1cb450198cd02e5ea7e273eed3add340ed087d102`；正式 wheel digest 为 `sha256:956cea3eba4480de4282f150affb9b6d6d6ff75272bb9386ac09ea0208a066c9`，候选 image ID 为 `sha256:f768f006d56ee8675582d46343bba4964bc797aeb13b81b1b5213be466106552`。这些 identity 用于区分历史 evidence，不建立当前可访问性或验收状态。

## 定位路径

### 1. 先隔离 completion path

baseline 与 candidate 若都在同步或 D2H 边界超时，应先用 fresh process 执行普通 device operation 和分层 C1b。普通 operation 同样失败时，最小边界位于 PyTorch DLC Backend、DLCSynapse、DLC Runtime 或 Host completion path，不能先归因模型算子。

历史调查还发现，使用一套 LLVM 构建的 binary 与另一套 baseline native stack 混用，可以表现为算子 completion timeout。固定配对 toolchain/native stack 重编后，相关 Gate 恢复而 RoPE source 无需修改。这支持“先核验 actual binary identity，再修改表面算子”的顺序，但不证明某个固定 LLVM 版本适用于未来任务。

### 2. 分开 public schema 和 launch ABI

main public schema 新增 optional routing metadata 后，host descriptor 曾追加相应 slots；冻结 baseline DLC Custom Kernel 仍按旧 entry ABI 读取。参数值为 `None` 不会自动删除已经构造的 descriptor slot，因此后续位置参数可能错位。

最终边界是：

```text
main public schema
  -> exact capability check
  -> baseline KernelDesc descriptor adapter
  -> frozen baseline DLC Custom Kernel entry ABI
```

Public Operator Schema 保持 caller source compatibility；baseline adapter 不发送冻结 baseline DLC Custom Kernel 不认识的 metadata slots。hash routing、未证明 scoring mode、symmetric W4、不完整 qzeros 组合和未证明 group size 在 launch 前 fail closed。三层 ABI 正式定义见 [CONTEXT.md](../CONTEXT.md)。

### 3. 收敛跨 rank lifecycle

多个 rank 进入不同 runner construction branch 后，如果后续 model initialization 依赖所有 rank 到达同一 lifecycle stage，barrier 必须放在分支汇合后的共同控制流。只在一个分支内部同步不能建立跨 rank contract。

该结论不是“所有 runner 都应无条件增加 barrier”。使用前必须证明 lifecycle ordering 是最小边界，并核验 barrier group、参与 rank 和死锁风险。

### 4. 移除日志中的 device tensor operation

expert-map logging 在字符串格式化前对 Chipltech-Family Accelerator tensor 执行比较和 advanced indexing，导致日志路径本身触发 device operation。修复先执行一次有界 `detach().cpu().tolist()`，随后只处理 Python values。

大 tensor 默认仍应 metadata-only；value logging 需要受大小和诊断目的约束。带同步、D2H 或额外 device operation 的诊断 epoch 不能贡献正式性能 evidence。

### 5. 明确 collective owner

模型 patch 与 fused-MoE layer 同时执行 TP all-reduce 会形成重复 reduction。每个 distributed output contract 必须有唯一 collective owner；TP=1 不能覆盖该类问题，至少需要目标 TP 下的 content correctness。

## 三仓归属

| 变更 | Owner | 理由 |
|---|---|---|
| QKNorm public schema、DLC dispatch、host validation、baseline descriptor adapter | PyTorch DLC Backend | 属于 public op 和 KernelDesc launch contract |
| expert-map accelerator-safe logging | vLLM | 是跨后端通用 framework 行为 |
| runner lifecycle barrier、模型 patch 重复 reduction | vLLM-DLC | 属于 DLC integration 和 model patch |
| frozen baseline DLC Custom Kernel | DLC_Custom_Kernel Repository 无本轮 patch | 当前路径通过 adapter 使用其已证明能力子集 |

## 验证矩阵

| 层级 | 本案例建立的历史 evidence | Claim Boundary |
|---|---|---|
| Source/AST | schema、校验顺序、descriptor slot、barrier 位置、禁止重复 reduction、logging 路径 | 不证明 build 或 runtime 行为 |
| Build/artifact | PyTorch wheel build、ZIP integrity、embedded native library digest | 不证明设备执行 |
| C1a | isolated install、import、schema/dispatch smoke | 不证明 C1b |
| 历史 Real DLC Hardware | 旧安装 artifact 的算子、collective、模型功能 | 不转移给新 wheel/image |
| 新正式 wheel/image | build/import/schema-only | Real DLC Hardware revalidation pending |

## 可复用经验

1. exact HEAD 静态审计可以确认 source 缺失；要把该缺失认定为当前 workload 的必要失败边界，还需要 exact artifact 运行必要性 evidence。
2. 调查仓库范围可以大于 patch 范围；无 patch 也必须有明确结论。
3. Public Operator Schema、KernelDesc Descriptor ABI 和 DLC Custom Kernel Entry ABI 分别固定 identity。
4. capability 绑定 exact DLC Custom Kernel revision；未知组合 fail closed，不按模型名特判。
5. toolchain、DLC Custom Kernel binary、DLCSynapse、DLC Runtime 和上层 wheel 构成配对 artifact graph。
6. source/AST tests 锁定结构不变量，但不能替代 C1b、多 rank collective、模型功能和 exact-image evidence。
7. wheel、native extension、DLC Custom Kernel binary、launch adapter 或 Image ID 变化后，旧 runtime acceptance 不可继承。
8. 修复完成、构建完成、image smoke、Real DLC Hardware qualification 和 PR 合入是不同状态。

## 不应泛化

- MiniMax 的具体 QKNorm kernel name、44 层计数、请求内容、TP4 数值和模型 alias。
- 冻结 baseline DLC Custom Kernel 对 routing、scoring、W4 mode 和 group size 的具体支持集合。
- 本次 toolchain commit、wheel digest、image ID、container、端口和设备占用状态。
- 将本案例 barrier、CPU value logging 或 collective 删除机械应用到其他执行路径。

## Claim Boundary

本案例建立的是历史代码事实、问题机制和工程决策。它不建立当前 artifact、DLC Runtime、Real DLC Hardware、模型功能或性能状态；新 target、wheel、DLC Custom Kernel binary、native stack、deployment profile 和 image 都需要自己的 sealed evidence。

## 来源

- `/home/xuansun/inn/MiniMax-M2.7-main交付材料-20260804/00-成果总览与当前状态.md`
- `/home/xuansun/inn/MiniMax-M2.7-main交付材料-20260804/01-问题根因与修复明细.md`
- `/home/xuansun/inn/MiniMax-M2.7-main交付材料-20260804/02-最终合入范围与逐仓Diff说明.md`
- `/home/xuansun/inn/MiniMax-M2.7-main交付材料-20260804/03-测试构建与Artifact证据.md`
- `/home/xuansun/inn/MiniMax-M2.7-main交付材料-20260804/04-其他模型兼容性分析.md`
- `/home/xuansun/inn/MiniMax-M2.7-main交付材料-20260804/05-后续镜像与硬件复验步骤.md`
- `/home/xuansun/inn/MiniMax-M2.7-main交付材料-20260804/06-主线确认缺失项问题单.md`

以上绝对路径是历史 provenance，其他 clone 可能不可访问；可审计性以记录的 manifest、wheel 和 image identity 为界，不推测缺失 artifact 的内容。
