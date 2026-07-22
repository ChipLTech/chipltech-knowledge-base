# 模型运行资格与 DLC/TYD 镜像交付 Contract

## 适用场景

本文是模型从本地资产和可选 ModelZoo reference 进入 DLC Chip/TYD Chip image 交付的唯一规范状态机。术语遵循 [CONTEXT.md](../CONTEXT.md)。执行细节由 [Host Daily Image Runbook](../prompt-examples/host-daily-image-to-model-validation.md) 提供；prompt 示例不得复制或改写本文状态语义。

本文不证明任何具体模型、image 或硬件已通过。当前运行事实必须来自本次 evidence。

## 核心状态机

```text
A. 输入与授权
B. 本地模型资产解析
C. 可选 ModelZoo reference 解析
D. Ordinary Daily Base 资格
E. Runtime Qualification Contract
F. Task-owned Validation Environment
G. C1a Package/Import
H. C1b DLC Runtime Execution
I. Real-weight Functional Qualification
J. Declared Benchmark Workload
K. Sealed Delivery Record
L. DLC Build -> Exact-image Validation -> Export
M. TYD Base Resolution -> Build/Packaging -> Static/Exact-image Validation -> Export
N. Cleanup Closure
O. DLC/TYD 独立 final status
```

Gate 规则：

- C1a 失败：不执行 C1b。
- C1b 失败：不加载模型。
- Functional 失败：不执行 benchmark，不构建正式 image。
- Benchmark workload 失败：不构建正式 image。
- 在 J 前构建的 image 只能是 `prequalification_only`。
- DLC 与 TYD target 独立报告；TYD blocked 不覆盖 DLC 结果。
- Cleanup 未闭合时，受影响 target 为 `blocked_cleanup_incomplete`。

## 输入与授权

最小输入：

```text
model_name
absolute_local_model_path
mode: qualification_only | qualification_and_image_delivery
```

可选输入：

```text
framework_selector
modelzoo_root
benchmark_workload
requested_targets: dlc | tyd | both
artifact_root
```

授权按动作分类，不从“自动执行”推导：image pull、network dependency access、clone/fetch、package install、build、device execution、tar export、registry push、Host maintenance。缺少继续所需授权时返回 `blocked_missing_authorization`，并列出最小授权。

## 四类记录

### 1. Resolved Model Manifest

`modelzoo-dlc-tyd-resolved-manifest/v1` 只包含稳定解析信息：

- 输入 model name/path/framework。
- 本地 config、weight shards、tokenizer/processor applicability 与 digest。
- 可选 ModelZoo Git/source identity、exact candidates、metafile/README hashes 和 claims。
- adapter decision、missing fields、conflicts、reference statuses。

不包含 PID、port、HBM、occupancy、时间戳或执行结果。`resolution_id` 仅覆盖 canonical stable payload。

### 2. Runtime Qualification Contract

固定：ordinary daily base、source refs、dependency/extension identities、model asset、device mapping、functional assertions、benchmark workload、artifact root、authorization 与 cleanup baseline policy。

### 3. Runtime Action Record

记录本次 Host observations、container/process identity、命令、server/failure epochs、C1a/C1b/function/benchmark evidence 和 cleanup observations。每个 retry 新建 epoch，每次只改变一个变量。

### 4. Sealed Delivery Record

绑定前三类记录的 digest、DLC/TYD fixed tags、build context、export scope、exact-image validation level、tar identity 和 cleanup closure。`sealed` 表示本 workflow 内 content-addressed 且关闭修改，不表示外部签名、可信时间或独立认证。

## 本地模型资产

本地资产优先于目录名和 README：

- 必须有 `config.json` 和非空权重。
- tokenizer/processor 按架构判定 required、not applicable 或 not verified。
- 记录架构、dtype、quantization、shard inventory、大小和 digest，不输出模型内容。
- selected local path 不存在或缺必需资产：`blocked_missing_asset`。
- README 中历史 weight path 不存在只记录 `historical_weight_path_not_present`，不覆盖完整的 selected local path。

## ModelZoo Reference

ModelZoo 是可选、只读、历史 reference channel，不是 runtime gate。

Reference statuses：

```text
modelzoo_reference_resolved
modelzoo_reference_unavailable
modelzoo_reference_incomplete
modelzoo_reference_ambiguous
modelzoo_reference_malformed
```

解析规则：

- 按 `metafile.yml` 的 `Name` 精确匹配并稳定排序。
- framework selector 只消除 reference ambiguity，不覆盖本地 adapter evidence。
- ModelZoo root 缺失、非 Git、无 exact entry、README 缺失、metadata malformed、`Infer: false` 均不自动阻断完整本地模型。
- `Infer: false` 记录为 `historical_modelzoo_negative_claim`。只有本地 config、当前 runner、package/import identity 和 CLI capability 能独立解析 adapter 时才可执行。
- README 命令、component refs、environment 和 benchmark 是历史线索；与当前 CLI/Host 不符时记录差异。
- 不修改、checkout、fetch、pull、reset、clean 或生成 ModelZoo 文件。

只有某个执行必需字段没有任何其他当前可信来源时，才使用相应 workflow blocker，例如 `blocked_unresolved_runtime_contract`，不能把 optional reference 状态升级为全局 blocker。

## Ordinary Daily Base Eligibility

ordinary daily base 必须完成以下检查：

- immutable Image ID；repo digest 在可用时记录，缺失时为 null，不猜测。
- source/tag 仅作 provenance，不以 tag 证明不可变性。
- 不包含目标模型权重、tokenizer/processor。
- 不继承模型专用 registry alias、patch、plugin 配置、model server、cache、HBM state 或历史 acceptance claim。
- 记录原始 package inventory/import paths。
- task-owned offline dependency overlay、clean source archive 和 extension 允许使用，但必须有 provenance/hash，并能绑定到后续 image build context。
- validation container 为 task-owned，container Image ID 必须与 qualified base 相等。
- pre-launch process、port、device occupancy 和 HBM baseline 已记录。

模型专用、historical golden、candidate 或来源无法解释的 image 为 `blocked_unqualified_daily_base`。

## Runtime Qualification

### C1a

`c1a_package_import_pass` 至少证明：

- fresh process、源码树外运行。
- Python executable、package metadata、actual import paths。
- PyTorch DLC Backend、DLC Platform、vLLM 和适用 plugin/extension identity。
- NumPy bridge 和 backend availability。

C1a 不证明 device execution。

### C1b

`c1b_runtime_execution_pass` 必须 fresh-process 分层完成：

```text
device enumeration/properties
allocation
H2D
nontrivial device operation
synchronize
D2H
correctness
```

多设备 deployment 对每张 logical device 分别运行，再做同时 probe；使用 collective communication 的 TP/PP/EP profile 还需对应 DLCCL correctness。

### Model Functional

`model_functional_pass` 至少要求：

- exact model asset 与 serving profile。
- real-weight load、health-before、model alias。
- 至少两个适合该模型/endpoint 的正交 deterministic assertions。
- raw requests/responses、HTTP status、request completion、non-empty output、semantic/assertion result、finish reason/token count（可观测时）。
- 明显 repetition/corruption、NaN/Inf、异常截断或 server error 检查。
- health-after 与 exact server epoch/process identity。

HTTP 200、weight load、health 或非空输出均不能单独构成功能 PASS。CPU Reference 可用于诊断，不替代 DLC Chip 功能 evidence。

### Benchmark

Benchmark contract 固定 model/alias、endpoint、functional/benchmark profile diff、dataset/corpus digest、input/output token policy、request count/rate/concurrency、seed、sampling、timeout、warm-up 和 formal attempt count。

状态分离：

- `benchmark_workload_pass`：声明的 warm-up 和至少一个 formal attempt 完成；满足 contract 的请求成功阈值、client 正常退出、server health-after、raw client/server logs 和 structured result。若 delivery contract 明确接受单次 observation，它足以解锁 image build。
- `benchmark_stability_baseline_pass`：声明的重复 attempts 完成并报告 median、min/max 或离散度。仅稳定性能、回归或 baseline claim 需要此状态。

单次成功不得称为稳定 baseline，但不被全局禁止作为明确声明的 delivery qualification workload。

## Image Delivery

### Pre-build 与 Exact-image Evidence

pre-build validation environment 与 produced image 是不同身份。报告必须独立记录：

```text
runtime_qualification_pass
exact_image_c1a_pass
exact_image_c1b_pass | not_executed
exact_image_model_functional_pass | not_executed
exact_image_benchmark_pass | not_executed
```

若 exact image 未重复 C1b/function/benchmark，允许的 bounded delivery 状态是：

```text
delivered_runtime_qualified_by_equivalent_environment
```

等价性 record 必须绑定 base identity、source SHAs/dirty state、package/import paths、extension/dependency hashes、environment、build context identity、validation environment 与 image 差异，以及 exact-image C1a。

### DLC Target

交付完成条件：fixed non-`latest` tag、Image ID、base/source/model identity、模型资产未进入 image、exact-image C1a、build attestation、validation report、tar absolute path/size/SHA-256、export exit status 和 cleanup evidence。Tar re-import 未执行时记录 `not_verified`。

### TYD Target

优先使用 qualified full-stack TYD base。Eligibility 至少包含 immutable Image ID、TYD generation/provenance mode、upstream attestation identity、full-stack 非 ENV-only 证明、package/static consistency，以及所选 DLC Platform/plugin identity。

基于 qualified base 的模型 packaging 使用：

```text
provenance_mode: inherited_full_stack_tyd_base
```

无 qualified base 时默认：

```text
blocked_missing_qualified_tyd_base
```

从 DLC base 重编 dlc-thunk、LLVM、DLCsim、DLCSynapse、DLC_CL、DLC_Custom_Kernel Repository、PyTorch DLC Backend、vLLM 和适用 vLLM-DLC extension 是独立的 `create_tyd_full_stack_base` contract，必须另获构建授权；不能成为模型 prompt 的隐式 fallback。`ENV DLC_TPU_VERSION=2` 不证明重编。

DLC Chip Host 上 TYD target 仅允许 static/package/import、hash、label、attestation 和 export。以下状态固定为：

```text
intentionally_not_executed_on_dlc_gen1
```

适用于 TYD device operation、C1b、DLCCL、model load、serving、benchmark。

## 独立 Target Status

每个 target 独立记录：

```text
delivered_runtime_qualified
delivered_runtime_qualified_by_equivalent_environment
delivered_static_package_only
prequalification_only
failed_validation
blocked_missing_qualified_base
blocked_missing_hardware
blocked_missing_authorization
blocked_cleanup_incomplete
```

最终使用 matrix，不把 DLC/TYD 压缩成单一 PASS/FAIL。

## Cleanup

只处理 task-owned resources。完成条件：task APIServer/EngineCore/workers/clients 退出，task ports 释放，task HBM delta 回到 sealed baseline/tolerance，builder/static containers 和未完成 staging/export 清理，正式 tags/tars/logs/failed epochs 保留。共享 Host 不要求全机 HBM 为 0，不触碰预存在资源。

## Claim Matrix

| Evidence | 可以证明 | 不能证明 |
|---|---|---|
| ModelZoo resolved | reference 已解析 | 当前 runtime/image PASS |
| C1a | package/import ready | device execution |
| C1b | bounded DLC Runtime execution | model correctness/benchmark |
| Functional PASS | exact profile 语义正确 | 性能/其他 profile |
| Benchmark workload PASS | exact workload 完成 | 稳定 baseline/其他模型 |
| Static package PASS | import/hash/label consistency | C1b/功能 |
| Docker build/export | image/tar 产生 | hardware/model PASS |

## 相关资料

- [ModelZoo 到 DLC/TYD Prompt](../prompt-examples/modelzoo-model-to-dlc-tyd-images.md)
- [Host Daily Image Runbook](../prompt-examples/host-daily-image-to-model-validation.md)
- [新模型 Quickstart](../prompt-examples/new-model-validation-quickstart.md)
- [安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md)
