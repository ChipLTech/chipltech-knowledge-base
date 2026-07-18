---
prompt_schema: vllm-dlc-reusable-prompt/v1
skill_identity: model-adaptation
shared_contract: vllm-dlc-contract/v1
required_inputs:
  - model_id
  - model_revision
  - tokenizer_revision
  - processor_revision
  - target_vllm_full_sha
  - candidate_vllm_dlc_full_sha
  - deployment_profile
  - model_assets
  - hardware_requirement
  - available_device_count
  - manifest_dependency_identity
  - artifact_destination
missing_input_status: blocked
missing_input_reason: blocked_missing_asset
hardware_evidence: not_verified
---

# vLLM-DLC Model Adaptation 可复用 Prompt

## 用途

用于一个明确模型在 DLC Platform 上的兼容性分析和只读报告。它选择 stable `model-adaptation` skill，但不复制 shared runner 的命令或断言。

## 使用方法

替换所有尖括号占位符。nullable 字段也必须明确填写；缺失输入应在任何执行前报告。

## 可复制 Prompt

```markdown
选择 `model-adaptation`，只处理以下特定模型兼容性任务。

模型身份：
- model ID: <MODEL_ID>
- approved model revision: <MODEL_REVISION>
- tokenizer revision: <TOKENIZER_REVISION>
- processor revision（nullable）: <PROCESSOR_REVISION_OR_NULL>

仓库身份：
- target upstream full SHA: <TARGET_VLLM_FULL_SHA>
- candidate vllm-dlc full SHA: <CANDIDATE_VLLM_DLC_FULL_SHA>

deployment profile：
- TP / PP: <TP> / <PP>
- dtype / quantization: <DTYPE> / <QUANTIZATION>
- context limit / batching limit: <CONTEXT_LIMIT> / <BATCHING_LIMIT>
- Chunked Prefill selection: <CHUNKED_PREFILL_SELECTION>
- served model name: <SERVED_MODEL_NAME>
- real-weight requirement: <REAL_WEIGHT_REQUIREMENT>

资源与交接：
- model path/assets and approval: <MODEL_ASSETS>
- hardware requirement / available device count: <HARDWARE_REQUIREMENT> / <AVAILABLE_DEVICE_COUNT>
- manifest/dependency input identity: <MANIFEST_DEPENDENCY_IDENTITY>
- artifact destination outside vllm-dlc: <ARTIFACT_DESTINATION>
- optional parent run / assignment identities: <PARENT_AND_ASSIGNMENT_OR_NULL>

先列出缺失输入并停止；资产类缺失使用 `blocked_missing_asset`，分配给 mandatory run 的硬件不足使用 `blocked_missing_hardware`。通过 `shared_contract: vllm-dlc-contract/v1` 的 skills-owned public seam 消费确定性检查，不自行实现 runner 行为。Dummy 仅可在合格 real-weight failure 后用于 diagnostic-only，不得提升 acceptance。保持 vllm-dlc 源码、manifest、alignment、metadata、branch、index 和 generated files 只读；不得更新、finalize 或声称 Verified vLLM Alignment。不要把 Ticket 06 v12 operational evidence 继承到此新 target；未执行的 real weights、Real DLC Hardware、Chunked Prefill runtime 和 DLC Runtime dispatch 均报告 `not_verified`，并将报告写到声明的外部 artifact destination。
```

## 停止语义与 Evidence

- 缺少声明输入时，在执行前返回 frontmatter 定义的 blocker。
- 本 prompt 没有执行模型或硬件；`not_verified` 不等于 `not_applicable`。
- HTTP success、静态检查、fake-server、Dummy 或 DLCsim 不得提升 Real DLC Hardware 或 DLC Runtime acceptance。
- Ticket 06 exact v12 profiles 只证明 `authoritativeness: operational_only`、`acceptance_eligible: false`、alignment unchanged 和 finalization `none` 的 bounded operational state，不证明新模型 target。

## 相关资料

- [模型适配与 Main-to-Main 决策记录](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)
