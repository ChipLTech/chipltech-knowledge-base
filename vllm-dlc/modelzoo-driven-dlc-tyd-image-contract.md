# ModelZoo 驱动的 DLC Chip/TYD Chip 镜像与验证 Contract

## 适用场景

本文定义一个可复用的 SOP：未来 Agent 只接收 ModelZoo model name，即可只读发现模型条目，生成确定性的 resolved manifest，并在证据和授权充分时驱动相互独立的 DLC Chip 与 TYD Chip 镜像流程。术语以 [CONTEXT.md](../CONTEXT.md) 为准。

本文是执行 contract，不是某次真实构建记录。本文中的 tag、路径和状态字段均为 schema 或占位符，不证明已经生成镜像、tar、模型 artifact，或执行 Real DLC Hardware 验证。

## 核心结论

1. ModelZoo 是只读发现来源和历史 claim 集合，不是当前 Host、源码、模型资产或硬件状态的事实来源。
2. Agent 必须先完成 ModelZoo identity、exact entry resolution、source hashing 和当前 Host preflight，再写 resolved manifest；manifest 未达到 `resolved` 时不得进入 build。
3. 当前自动执行 adapter 仅支持具有完整 contract 的 vLLM entry。其他 framework 可完成发现和 manifest，但必须返回 `blocked_unsupported_framework`，不得套用 vLLM 命令。
4. DLC Chip 与 TYD Chip 是独立 build target，必须具有不同的 fixed tag、Image ID、tar、tar SHA-256、attestation 和 validation report。
5. TYD 全栈构建要求每个实际编译进程继承 `DLC_TPU_VERSION=2`。仅向已有 DLC 镜像添加 `ENV DLC_TPU_VERSION=2` 不充分。
6. TYD vLLM target 的完整链至少包括 dlc-thunk、LLVM、DLCsim、DLCSynapse、DLC_CL、DLC_Custom_Kernel Repository、PyTorch DLC Backend 和 vLLM。缺少任何 required component 的 build provenance 时不得称为 TYD full-stack build。
7. TYD 产物严禁在 DLC Chip 上执行 device operation、C1b、DLCCL、模型加载、serving 或 benchmark；状态固定为 `intentionally_not_executed_on_dlc_gen1`。

## 只读发现与选择

### ModelZoo identity

行动前记录：

- 用户提供或只读发现的 authoritative ModelZoo root。
- `git rev-parse --show-toplevel`、remote URL、branch/tag、full HEAD 和 `git status --short`。
- root 不在预期 Git repository、存在多个 authoritative candidate，或 Git identity 无法读取时停止。
- 不修改 ModelZoo，不 checkout、fetch、pull、reset、clean、生成 index 或修复 metadata。

### Current Host observation trust boundary

Current Host observation is not established by a caller-authored JSON file, a caller-provided boolean, a SHA-256 alone, or a historical README. The resolver accepts actionable observations only when each allowlisted observation payload and its detached signature verify against the protected local observer public key. A production caller-provided model path is a discovery hint, not a qualified asset observation, until the later sealed action record binds its asset identity. Production component roots must additionally match the root-owned, non-writable per-component allowlist provisioned outside the workflow input surface. A caller cannot select either trust root. SHA-256 closes bytes after producer authentication, but does not by itself authenticate a producer. Secret-bearing path segments are redacted before manifest serialization.

Fixture-only diagnostic keys may be used for offline resolver tests. Their manifests must record `trust_class: fixture_diagnostic` and `action_eligible: false`; they are resolver behavior evidence only and cannot authorize build, export, package validation, hardware validation, device operation, model loading, serving, or benchmark. Resolver output is qualification-only even with a production observer. A production action must create its own sealed action record that binds the exact asset identity, approved component source/ref, authorization, and execution scope; it cannot inherit action authorization from a resolver preflight wrapper.

### Exact entry resolution

按 `metafile.yml` 的 `Name` 精确匹配 model name，并对候选路径做稳定字典序排序后记录。不得按目录遍历顺序、大小写近似、README 标题或顶层 index 的第一条结果猜测。

| 发现结果 | 状态 | 行为 |
|---|---|---|
| 无精确匹配 | `blocked_model_not_found` | 列出 model name 和已搜索 root，停止 |
| 一个精确匹配 | 继续解析 | 记录 framework 和 model directory |
| 多个匹配且无 framework selector | `blocked_ambiguous_model` | 按 framework/path 排序列出全部候选，停止 |
| selector 精确选中一个候选 | 继续解析 | 记录 selector 是 user override |
| selector 无匹配或仍有多个匹配 | `blocked_ambiguous_model` | 不回退到任意候选 |

`metafile.yml` 必须可安全解析为 mapping；`Name`、`Task`、`Infer`、`Train` 和 `Parameters` 保留原始值及类型。YAML parse error、非 mapping、重复关键字段导致语义不唯一，或 `Infer`/`Train` 不是 boolean 时返回 `blocked_malformed_metadata`。`Infer: false` 是 ModelZoo claim，不等同于通用不支持，但在没有额外 adapter evidence 时不能进入自动 inference workflow。

README 缺失或缺关键字段不应被猜值掩盖。Agent 应列出 `missing_fields`；若无法形成 supported adapter 所需的 weight、component、serve 或 smoke contract，返回 `blocked_missing_required_field`。

## 来源优先级与冲突

优先级从高到低如下：

1. **User override**：framework selector、批准的模型路径、固定 component refs、输出目录和明确授权。Override 是执行意图，不得覆盖实际文件不存在、Git ref 不可解析、硬件代际不符或操作未授权等事实。Model path override 与 README 权重声明不同必须同时保留两个值，并记录 `resolution_reason: user_override`。
2. **Current Host observation**：当前文件存在性和 digest、Git identity、package/import identity、image identity、硬件代际、设备占用与授权状态。只有本次采集的 observation 才能成为当前事实。
3. **Selected `metafile.yml`**：模型名称、任务、Infer/Train 和参数规模的结构化 ModelZoo claim；它负责索引，不证明当前可运行。
4. **Selected model README**：权重路径、component SHA、环境变量、命令和历史结果的非统一历史线索。
5. **Top-level or historical ModelZoo claim**：例如“支持但未充分测试”，只用于发现和 provenance，不能升级为当前功能 PASS。

冲突处理采用 fail closed：

- User override 与 README 不同：保留两者，选择 override，并记录 `resolution_reason=user_override`。
- Host observation 与任意 claim 不同：当前执行采用 observation；claim 保留在 `source_claims`，冲突写入 `conflicts`。
- `metafile.yml` 与 README 不同：不静默择一。结构化索引字段采用 metafile，运行字段保持 unresolved；关键字段冲突返回 `blocked_conflicting_source_claims`。
- README 的 component SHA 只能作为 requested ref。当前 approved remote 无法解析时返回 `blocked_unresolved_component_ref`，不得回退 movable branch、tag 或 latest。
- README 权重路径在当前 Host 不存在时返回 `blocked_missing_asset`，不得下载、重命名或猜测同名替代资产。
- README 中的 Host、IP、旧工具名、命令、benchmark 数值和 SHA 均标记为 `historical_modelzoo_claim`。原文若需保留，使用引用块并明确“历史原文，不是当前 verified fact”。

## Deterministic Resolved Manifest

Resolved manifest 必须在任何 build、设备执行或模型加载前写入 artifact 目录。推荐 YAML/JSON 使用 UTF-8、固定 key order、候选和数组按稳定字典序排列、绝对路径规范化但不解析不存在路径，并为 source 文件记录 SHA-256。时间戳和 run ID 不参与 resolution identity；可另存为 run metadata。

```yaml
schema: modelzoo-dlc-tyd-resolved-manifest/v1
resolution_status: resolved | blocked
resolution_id: sha256:<canonical-resolution-payload>
inputs:
  model_name: <exact Name>
  framework_selector: null
  model_path_override: null
modelzoo:
  root: <absolute-path>
  git_root: <absolute-path>
  remote: <url>
  branch_or_tag: <value-or-null>
  head: <full-sha>
  dirty_observation: <git-status-short>
selection:
  framework: vllm
  model_directory: <relative-path>
  candidates: []
sources:
  metafile:
    path: <relative-path>
    sha256: <sha256>
    fields: {}
  readme:
    path: <relative-path-or-null>
    sha256: <sha256-or-null>
source_claims:
  weight_paths: []
  component_refs: {}
  required_environment: {}
  serve_contract: {}
  smoke_contract: {}
current_observations:
  model_asset: {}
  component_refs: {}
  host_and_hardware: {}
resolved:
  model_path: <absolute-path>
  component_refs: {}
  required_environment: {}
  serve_contract: {}
  smoke_contract: {}
  framework_adapter: vllm/v1
missing_fields: []
conflicts: []
claim_boundary:
  historical_modelzoo_claims_are_current_facts: false
  unverified_scope: []
blocked:
  code: null
  missing_or_conflicting_fields: []
  evidence: []
```

Canonical payload 包含 `inputs`、ModelZoo Git identity、selection、source path/hash、source claims、current observations、resolved fields、missing fields、conflicts 和 blocked result；不包含 secret、credential、URL userinfo、私有模型内容、易变时间戳或绝对 Host 用户信息之外的不必要详情。相同输入与相同 source/observation 必须产生相同 `resolution_id`。

## Blocked 状态

Blocked 是安全结果，不是泛化 failure。每个结果必须给出 `code`、最小原因、缺失/冲突字段、已检查 evidence 和允许的恢复输入。

| Code | 触发条件 | 禁止的回退 |
|---|---|---|
| `blocked_model_not_found` | 无 exact model name | 模糊匹配或换模型 |
| `blocked_ambiguous_model` | 同名跨 framework/path 且 selector 不唯一 | 按目录顺序选择 |
| `blocked_malformed_metadata` | YAML 或字段类型异常 | 用 README 猜 schema |
| `blocked_missing_required_field` | supported adapter contract 缺关键字段 | 隐藏缺失值或填默认值 |
| `blocked_conflicting_source_claims` | 关键 source claim 无法安全裁决 | 静默采用任一来源 |
| `blocked_missing_asset` | 权重/processor/tokenizer 等批准资产不存在 | 下载或替换模型 |
| `blocked_unresolved_component_ref` | component ref 无法从 approved source 固定 | 使用 latest/movable ref |
| `blocked_missing_hardware` | mandatory run 无匹配硬件或资源 | 在错误代际执行 |
| `blocked_missing_authorization` | build/export/device/Host action 缺明确授权 | 把自动发现视作授权 |
| `blocked_unsupported_framework` | 没有已定义 adapter contract | 套用 vLLM workflow |
| `blocked_cleanup_incomplete` | task-owned process/resource 未回到 sealed cleanup contract | 隐藏残留或删除非本任务资源 |

## Framework Adapter 边界

| Framework | Discovery/manifest | Build/validation adapter | 结论 |
|---|---|---|---|
| vLLM | Supported | `vllm/v1`，输入完整且安全门通过时可直接执行 | Supported, bounded |
| SGLang、Transformers、PyTorch、NeMo、Diffusers、KTransformers、其他 | Supported | 本 contract 未定义 | `blocked_unsupported_framework` |

vLLM adapter 必须解析真实模型路径、component refs、环境、serve command 和最小 smoke request，并复用 [Host 每日镜像到模型验证](../prompt-examples/host-daily-image-to-model-validation.md) 的 C1a、C1b、模型功能、安全进程 identity、HBM baseline 和 cleanup contract。不得把该长模板复制进新 prompt，也不得假设历史 skill script flags 存在；执行前读取当前可用模板、skill 和脚本能力。

## 独立镜像 Contract

### DLC Chip target

构建前固定 base image ID/digest、每个 source full SHA、dirty-tree policy、模型不进入 image 的边界和 fixed tag。构建和导出获授权后，交付至少包含：

- DLC Chip fixed tag，不以 `latest` 作为交付身份。
- DLC Chip Image ID、image configuration 和 provenance labels。
- DLC Chip tar 绝对路径、size 和 SHA-256。
- DLC Chip build attestation 与 validation report。
- 独立状态：C1a、C1b、model functional、benchmark、not verified、blocked。

DLC Chip 上先执行 C1a。只有目标硬件、占用、授权和 fresh-process gate 满足时才执行 C1b；只有 C1b 通过后才允许最小模型 functional smoke。Benchmark 不是默认功能验收，必须另有 workload contract、资源和授权。

### TYD Chip target

TYD fixed tag、Image ID、tar、hash 和 attestation 必须与 DLC target 独立。TYD attestation 至少逐组件记录 source full SHA、build command identity、build epoch、artifact hash，以及实际 build process 的 `DLC_TPU_VERSION=2` evidence。

完整 vLLM 构建链至少覆盖：

```text
dlc-thunk -> LLVM -> DLCsim -> DLCSynapse -> DLC_CL
          -> DLC_Custom_Kernel Repository -> PyTorch DLC Backend -> vLLM
```

镜像 config 中存在 `ENV DLC_TPU_VERSION=2` 只证明运行环境声明，不证明上述二进制由继承该变量的进程编译。任一 required component 缺少实际 process-environment evidence 时返回 `blocked_missing_required_field`，并在 `missing_or_conflicting_fields` 列出缺失 provenance；不能使用正式 TYD tag，也不能声称 full-stack build PASS。历史 partial image 只能作为 `historical_modelzoo_claim` 或外部历史 evidence，不是本 contract 的 validation state。

在 DLC Chip Host 上，TYD target 只允许不触发设备执行的 static/package/import、artifact hash、image label 和 attestation consistency checks。所有 TYD device scope 必须报告 `intentionally_not_executed_on_dlc_gen1`。只有 Agent 已证明位于 TYD Chip Host、目标资源可用且另有执行授权时，才可按独立 contract 执行 TYD functional smoke。

## Validation 与 Claim 状态

| 状态 | 最小 evidence | 明确不证明 |
|---|---|---|
| `c1a_package_import_pass` | package/import、实际 import path、metadata 和 backend availability | device execution |
| `c1b_runtime_execution_pass` | fresh process 的 allocation、H2D、device operation、synchronize、D2H、correctness | 模型功能或 benchmark |
| `static_package_pass` | 不触发 device execution 的 package/static/hash/label/attestation checks | C1b 或功能 |
| `model_functional_pass` | 已批准真实资产、指定 profile、liveness、非空输出和可观察 correctness | benchmark 或广泛 acceptance |
| `benchmark_pass` | 独立固定 workload、warm-up gate、原始结果和测后健康 | 其他 profile 或模型正确性 |
| `not_verified` | 本次未执行且不构成 blocker | PASS/FAIL/not applicable |
| `blocked_*` | 结构化 blocker evidence | 执行已尝试或部分 PASS |
| `intentionally_not_executed_on_dlc_gen1` | TYD 产物位于 DLC Chip Host 的代际禁止规则 | 普通 pending 或 TYD 功能 PASS |

最终报告必须将 `modelzoo_claims`、`current_observations`、`inferences`、`execution_evidence` 和 `unverified_scope` 分栏。历史 benchmark 数值不能成为默认 acceptance threshold。

## Evidence 与 Cleanup

每个 image target 的 attestation 和报告至少包含：

- Manifest schema/resolution ID、ModelZoo Git identity、selected source path/hash 和 override 决策。
- Base image immutable identity、source refs、actual package/import identity 和 build scope。
- Fixed tag、Image ID、tar path/size/SHA-256；registry digest 仅在实际获批 push/pull 后记录。
- 命令、cwd、environment allowlist、start/end、exit code、stdout/stderr artifact path；不写 secret。
- 每个 validation layer 的独立状态、最早失败边界、blocked/not verified/prohibited scope。
- 任务进程、端口、device handles 和 HBM 的 sealed pre-launch baseline 与 post-cleanup observation。

只停止已证明属于本任务的进程。默认不得下载权重或依赖、更新 Host driver、restart service、reset/reboot、push registry、覆盖 dirty tree、kill 无关进程或 prune 用户资源。任何一项成为继续执行的必要条件而未获授权时返回 `blocked_missing_authorization`。

构建和导出收尾还必须精确清理本任务创建且已记录 identity 的 builder/static-check containers、临时 staging tags、临时 build directories 和未完成 export artifacts。正式 fixed tag、已完成 tar、attestation、logs 和失败 epoch evidence 按交付 contract 保留。不得删除预先存在的容器、tag、目录或其他任务 artifact；cleanup 未完成时返回 `blocked_cleanup_incomplete`，不得声称交付闭环。

## 恢复策略

- ModelZoo entry 不完整：保留 manifest 和 source hashes，列出缺失字段，不进入 build。
- Ref 不可解析：保留 requested ref 与 approved remote evidence，不切换 latest。
- Asset 不存在：报告已检查路径，不下载。
- Build failure：保留首个错误、失败 epoch 和已完成组件 provenance；修复后创建新 epoch，不覆盖失败 evidence。
- C1a 失败：回到 package/build identity；不得执行 C1b。
- C1b 失败或 hang：按 [安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md) 记录最早边界；不得加载模型。
- Model smoke 失败：保留前面已通过的 bounded state，不提升 benchmark。
- Cleanup 未回到 baseline：返回 `blocked_cleanup_incomplete`；未经授权不 restart service、reset 或 kill 其他任务。

## Supported / Blocked / Prohibited 矩阵

| 类别 | Supported | Blocked | Prohibited |
|---|---|---|---|
| ModelZoo | 只读 identity、exact discovery、hash、manifest | missing、ambiguity、malformed、conflict | 修改、生成 index、猜 selector |
| Framework | 完整输入的 vLLM adapter | 无 adapter 的 framework | 伪称所有 framework supported |
| Assets/refs | 已批准且当前可验证的路径/full ref | missing asset/unresolved ref | 自动下载、使用 movable latest |
| DLC target | 独立 build/export；安全门满足时 C1a/C1b/model smoke | hardware/authorization/evidence 不足 | 抢占设备、未授权 Host 变更 |
| TYD target | 全链 `DLC_TPU_VERSION=2` build 与 static/package | 链不完整、缺 TYD Host/授权 | 在 DLC Chip 上 device execution |
| Delivery | fixed tag、Image ID、tar、SHA-256、attestation/report | identity/evidence 不完整 | 用 latest 代替固定身份、未授权 push |

## 相关资料

- [ModelZoo model 到 DLC/TYD images Prompt](../prompt-examples/modelzoo-model-to-dlc-tyd-images.md)
- [Host 每日镜像到模型验证](../prompt-examples/host-daily-image-to-model-validation.md)
- [新模型验证 quickstart](../prompt-examples/new-model-validation-quickstart.md)
- [安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md)
- [模型适配与 Main-to-Main 决策记录](model-adaptation-and-main-to-main-decisions.md)
