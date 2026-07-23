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
- SMI Observation Envelope：<artifact paths | execute through dlc-hardware-observability | not_applicable with reason>

先列出缺失输入并停止；资产类缺失使用 `blocked_missing_asset`，分配给 mandatory run 的硬件不足使用 `blocked_missing_hardware`。模型路径优先从 `/mnt/jfs/models` 选择并记录完整绝对路径；如果资产不在该根目录下，先说明来源和批准依据。通过 `shared_contract: vllm-dlc-contract/v1` 的 skills-owned public seam 消费确定性检查，不自行实现 runner 行为。Dummy 仅可在合格 real-weight failure 后用于 diagnostic-only，不得提升 acceptance。保持 vllm-dlc 源码、manifest、alignment、metadata、branch、index 和 generated files 只读；不得更新、finalize 或声称 Verified vLLM Alignment。不要把 Ticket 06 v12 operational evidence 继承到此新 target；未执行的 real weights、Real DLC Hardware、Chunked Prefill runtime 和 DLC Runtime dispatch 均报告 `not_verified`，并将报告写到声明的外部 artifact destination。

模型适配分析时额外检查：
- serving 启动参数与请求中的 `model` 名必须一致；如果启动用了 `--served-model-name`，OpenAI 请求必须使用该 alias，而不是误用模型路径。
- 先做短 prompt、`temperature=0`、`top_p=1.0`、小 `max_tokens` 的 serving smoke；通过后再增加 prompt 长度、one-shot/CoT、采样温度、Chunked Prefill 或并发，每轮只改变一个变量。
- 长 prompt 或 one-shot 退化要单独记录 prompt token 数、decode 参数、finish reason、输出内容、latency、server liveness 和日志路径；重复 `!`、空输出、超时、异常截断不能归类为环境已通过。
- 量化模型要核对 `quant_method`、`bits`、`group_size`、`zero_point` 与实际 kernel 路由，避免把 `compressed-tensors` 误判为 AWQ/AWQ-Marlin 兼容；W8A16、MoE、vision/multimodal processor 路径应单独列风险。
- 多卡/MoE/精度问题先建立严格等价对照：相同 endpoint、prompt、tokenizer、模型权重、TP/EP、`temperature=0`、`max_tokens` 和可选 `logprobs`；不要混用 chat/completions 与 completions 的结果直接比较。
- 若生成 token 与基线分叉，把多步 decode 改写为单步 prefill：将分叉前 token 拼入 prompt，仅生成 1 个 token，以隔离 KV cache、scheduler 和历史回灌变量。
- 如果怀疑算子或设备卡住，记录 DP/TP 初始化日志、shared memory broadcast 日志和服务状态；`peek_stuck.sh`、软重置、LYP repair、kill 进程或 reboot 必须有明确授权，不作为默认自动步骤。
- Real DLC Hardware serving epoch 通过 `dlc-hardware-observability` 保存 `before_launch`、`after_ready`、`during_request` 和 `after_cleanup`；SMI 正常不证明模型正确，observer 缺失不改写为硬件 failure。若只做 static/read-only compatibility analysis，填写 `not_applicable` 及依据，不新增 device execution。
```

## 停止语义与 Evidence

- 缺少声明输入时，在执行前返回 frontmatter 定义的 blocker。
- 本 prompt 没有执行模型或硬件；`not_verified` 不等于 `not_applicable`。
- HTTP success、静态检查、fake-server、Dummy 或 DLCsim 不得提升 Real DLC Hardware 或 DLC Runtime acceptance。
- Ticket 06 exact v12 profiles 只证明 `authoritativeness: operational_only`、`acceptance_eligible: false`、alignment unchanged 和 finalization `none` 的 bounded operational state，不证明新模型 target。
- 单个短 prompt serving smoke 只能证明对应启动配置的最小可用性，不证明长上下文、one-shot、Chunked Prefill、DLC Runtime dispatch 或 Real DLC Hardware acceptance 已验证。

## 相关资料

- [模型适配与 Main-to-Main 决策记录](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)
- [cltech_smi 设备观测与诊断证据](../runtime-debugging/chipltech-smi-observability.md)
