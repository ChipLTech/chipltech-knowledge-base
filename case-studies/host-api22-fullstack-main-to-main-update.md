# Host API 22 全栈主线更新与 TP4/EP 验证

## 问题现象

一次持久化容器内完成 Host API 21 → 22 的运行时栈升级、DLC_Custom_Kernel Repository 从 `develop` 切到 `main`、PyTorch DLC Backend 与 vLLM-CL 在保留 MiniMax PR 修改的前提下合入各自主线，并完成 TP4/EP4 模型服务验证。过程中暴露了几类与“版本配对 / 身份核对”相关的独立问题，失败表象与真实 owner 常常不一致。

## 背景与环境

Evidence status: `historical_report_derived_observation`。原始过程资产当前为 `external_unavailable`；本文虽记录若干 SHA 和运行结论，但缺少仓内 manifest/digest 集合来复核完整 exact identity，未闭合部分标为 `identity_unavailable`，不得补猜。

不可继承：本案例不能自动成为 approved Stack Policy、current topology qualification、current model acceptance 或 current performance baseline。

- 执行节点 `node-8-45`，持久化容器 `minimax-m2.7-api21-persistent`（`--privileged`、`--pid=host`、`--ipc=host`、`--net=host`）。
- 目标模型 `MiniMax-M2.7-Mix`（`/mnt/jfs/models/MiniMax-M2.7-Mix`），物理卡 `0,1,2,3`，TP4 + Expert Parallel、AWQ Marlin、BF16、eager。
- Host Driver `v22.0.0` / Host API `Api_Version_22`。
- 历史证据身份：本文是过程资产，所有结论绑定本文记录的源码 SHA、binary hash、模型资产、容器、Host API、设备拓扑和本次执行结果。

## 定位路径

### 1. 先核对实际 binary identity，再改表面算子

LLVM source 已指向最新 main（`1907051bc985135b6c776b23ef917f418d827c17`），源码里已存在 `dlc-enable-auto-loopunroll`，但 `clang` 实际 binary 自报构建来源是旧 SHA（`a657f149e86a87e9fd0e1200bc9242e92f6d78a5`），导致 Custom Kernel 构建报 `Unknown command line argument '--dlc-enable-auto-loopunroll=true'`。

结论：**只检查 Git HEAD 不足以证明 toolchain identity，必须读取实际 compiler binary 自报 SHA 或其他可审计 binary identity。** clean rebuild 后 clang 自报 SHA 与 source HEAD 一致，构建恢复。

### 2. 更新运行仓库前先确认 editable source

容器内同时存在 `/work/*` 与 `/work/minimax-src/*` 多套 checkout。安装元数据确认实际执行链是 `/work/minimax-src/{pytorch,vllm,vllm-cl}`，而非通用 `/work/{pytorch,vllm,vllm-cl}`。判定运行源码不能只看 editable metadata 的 version 字符串（来自安装时生成），还要同时核对 `direct_url.json`、`module.__file__` 和进程命令，最后落到 source Git SHA。

### 3. 主机只读 mirror + Git bundle 中转，避免把私钥复制进容器

容器内 remote 需要 SSH，但容器内没有可用 identity，直接 fetch 报 `Permission denied (publickey)`。宿主机具备已批准私有仓库读取能力。为不复制或输出私钥，采用宿主机只读 mirror → 从 mirror 生成只含目标 refs 的 Git bundle → 容器从挂载的 bundle fetch。该路径没有把 SSH 私钥复制进容器，也没有在日志或命令参数中暴露凭据。

### 4. PR 更新不能机械重放旧 patch

PyTorch 主线已吸收旧 PR 的多个功能并包含后续修复（fused MoE expanded ABI、BMM VMEM、int64 gather、top-k 2048、rotary、AWQ/W8、routing/activation schema 扩展）。正确做法是：

```text
merge-base 审计
逐 conflict 追溯提交意图
主线替代实现优先
只保留仍缺失且 workload 必需的 PR 差异
```

对 QKNorm，主线冲突把 Public Operator Schema 覆盖掉，随后定位到正式 PR commit 并采用正式 hardened 实现，而不是保留临时弱 wrapper。用 merge 而非 rebase，不重写旧 PR 历史。

### 5. 三层 ABI 必须分别核对

Public Operator Schema、KernelDesc Descriptor ABI 与 DLC Custom Kernel Entry ABI 是三个独立 contract；optional 参数为 `None` 不代表 descriptor slot 自动消失。本次对 QKNorm 与 AWQ fused-MoE 分别核对了调用链（vLLM-CL patch → torch op → CommOps.cpp → KernelDesc → DLC Custom Kernel entry）的有序参数匹配。

### 6. 后台 wrapper 停止不保证容器内子进程退出

停止后台 wrapper 后，容器内 `vllm serve`、EngineCore 和 TP/EP workers 曾继续运行。清理必须同时检查 HTTP endpoint、`docker top`、EngineCore、TP/EP worker 与端口监听。

### 7. 验证 checkout 与发布 candidate 是两种 identity

模型构建和运行验证使用保留现场的 PyTorch DLC Backend 与 vLLM-CL merge checkout。领导随后要求每个 PR 相对最新 main 只保留一个 commit；这不是把验证 checkout 原地 squash，而是从各仓最新 main 创建隔离 clean worktree，只应用批准的 PR 净差异，并排除验证现场中的无关文件。

历史闭环同时记录：

- Tested Revision 的 full SHA、dirty/untracked state 和实际 build/runtime artifact。
- Publication Candidate 的 exact base SHA、单提交 SHA、限定路径净差异和 stable patch-id。
- squash/rebase 前后的 stable patch-id 相等，作为 Patch Equivalence supporting evidence。
- vLLM-CL main 并发推进后，旧 expected SHA 的 lease 阻止了推送；candidate 随后基于新 main 重建并重跑回归。
- 用户明确授权后才使用带 exact expected old SHA 的 `--force-with-lease`，推送后重新核对远端 commit count 和 PR 状态。

Patch Equivalence 只说明声明 scope 的净差异保持。base 变化可能改变组合行为，因此 patch-id 不转移完整 runtime acceptance；至少要重新执行受影响 source/contract 回归，若 artifact 或 runtime dependency graph 变化则重新 build 和运行对应门禁。

## 最小边界与根因

- **LLVM 构建失败**：根因是 source 与 binary 不配对，不是 source ref 落后；最小边界在 compiler binary identity。
- **多 checkout 混淆**：根因是同一容器存在多套 checkout 且 editable metadata 不随源码更新；最小边界在“实际 import source”确认。
- **fetch 失败**：根因是容器无 SSH identity，非网络或权限策略；通过宿主 mirror + bundle 绕过。

## 验证方式

- LLVM clean rebuild 后 clang 自报 SHA 与 source HEAD 一致，`dlc-enable-auto-loopunroll` 存在。
- DLC Custom Kernel full build/install 2341 steps，build 与 installed binary SHA-256 一致。
- PyTorch wheel build、ZIP integrity、隔离安装与 schema/dispatch smoke 通过。
- 最终 wheel 安装后 fresh cold first-compute `[3.0, 6.0, 9.0]`、TP4 QKNorm CPU Reference 4 ranks PASS。
- TP4/EP4 服务 ready，中英文回答与多步计算题通过；`finish_reason=stop` 且 `</think>` 闭合。
- 服务清理后端口 `Connection refused`、EngineCore 与 workers 退出。

## 可复用经验

1. 记录 native component 时应同时记录 source HEAD、dirty state、build command、binary hash/source identity、install target hash 与 consumer identity；不满足时以 `identity_unavailable` 记录，不从 source checkout 推断。
2. 更新运行仓库前，通过 `direct_url.json`、`module.__file__` 和进程命令确认实际 editable source，避免“源码已更新但运行仍用旧 checkout”的假象。
3. 容器无 SSH identity 时，用宿主机只读 mirror + Git bundle 中转，避免复制或输出私钥。
4. 主线已吸收旧 PR 功能时，用 merge 而非 rebase，逐 conflict 追溯意图，主线替代实现优先，只保留仍缺失且必需的差异。
5. 后台 wrapper 停止不保证容器内子进程退出；清理需同时检查 endpoint、`docker top`、EngineCore、worker 与端口。
6. 每次底层 artifact 变化后，重新执行 fresh-process runtime smoke 与模型验证，不继承旧 wheel/binary 的运行结论。
7. 保留验证现场并在隔离 worktree 构造 Publication Candidate；记录 Tested Revision、candidate base/commit、scoped diff/tree 和 Patch Equivalence，不用单提交数量替代验证身份。
8. 远端 main 或 PR branch tip 变化会使 publication lease 失效；基于新 main 重建 candidate、重跑受影响门禁，再在单独授权下使用 exact `--force-with-lease`。无 lease 的 force push 不属于安全发布路径。

## 不应泛化

- 特定 SHA、wheel digest、Image ID、容器、端口、设备占用与模型 alias。
- 本次固定 BF16 MiniMax profile 验证不能外推到任意 AWQ 配置、其他 TP/EP 拓扑、TYD/HHP Chip 或长上下文/并发/benchmark。
- reasoning parser 未把 `<think>` 分离到独立 `reasoning` 字段是当前模型/parser 行为，不作为通用框架缺陷。
- stable patch-id 相等不证明新 base 上的 build artifact、模型行为或 Real DLC Hardware acceptance 等价。

## Claim Boundary

本文建立的是 exact identity（Host API 22、容器 Image ID、各仓库 SHA、wheel hash、MiniMax 本地资产、物理卡 0-3、TP4/EP4 + AWQ Marlin + BF16 + eager）上的过程和运行证据。它不建立其他版本、镜像、模型、拓扑、长期稳定性、性能基线或更高等级 Real DLC Hardware acceptance。

## 来源

- 个人过程资产：`external_unavailable`（历史位置已脱敏）
- [vllm-fused-moe-schema-kernel-abi-boundary.md](vllm-fused-moe-schema-kernel-abi-boundary.md)
- [model-adaptation-and-main-to-main-decisions.md](../vllm-cl/model-adaptation-and-main-to-main-decisions.md)
