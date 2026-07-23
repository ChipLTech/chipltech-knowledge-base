---
prompt_schema: vllm-dlc-reusable-prompt/v2
skill_identity: model-adaptation
shared_contract: vllm-dlc-contract/v1
required_user_inputs:
  - model_name
  - absolute_local_model_path
derived_or_proposed_inputs:
  - revisions_or_null_and_asset_digest
  - source_and_package_identities
  - deployment_profile
  - hardware_and_observability
  - manifest_dependency_identity
  - artifact_destination
discovery_policy: local_query_only_then_contract_proposal
missing_input_status: blocked
missing_input_reason: blocked_missing_asset
hardware_evidence: not_verified
---

# vLLM-DLC Model Adaptation 可复用 Prompt

## 用途

用于一个明确模型在 DLC Platform 上的兼容性分析，并可在环境 handoff 和授权闭合后继续 device-backed validation。用户只填写模型名称和绝对目录；其余身份、profile、硬件和 artifact 字段自动发现或提出。

## 使用方法

只替换模型名称和模型目录两个占位符。revision 不可发现时记录 `null` 并以本地 recursive digest 闭合 identity；不得要求用户手工提供可从 checkout、package、config、设备或历史 evidence 得到的字段。

## 可复制 Prompt

```markdown
选择 `model-adaptation`，持续处理以下模型直到 static analysis 或已授权 device-backed validation 达到 terminal state。

模型名称：<MODEL_NAME>
模型目录：<ABSOLUTE_LOCAL_MODEL_PATH>

先 query-only 读取模型 config、weights、tokenizer/processor，计算 stable digest；自动发现 vLLM/vllm-dlc/source/package full SHAs、dirty state、可用设备和历史 evidence，并在源码树外提出 artifact destination。然后生成 resolved model identity、capability matrix、TP/PP/EP/DCP、dtype/quantization、context/batching、Chunked Prefill、served alias、hardware requirement、SMI status 和 dependency inventory。revision 无可信来源时写 `null`，不要阻断已由绝对路径和 digest 唯一确定的本地资产。

默认先执行只读 static/pre-handoff analysis。若当前已有匹配的 `environment_handoff/v1`、`qualification_execution` 和 device execution authorization，则继续 device-backed validation；KILL 另需 `task_owned_kill`。否则只在该步骤成为继续条件时返回 `blocked_missing_authorization` 或 handoff blocker，不要求我预填上述派生字段。

通过 `shared_contract: vllm-dlc-contract/v1` 消费 runner，不自行复制其行为。Dummy 仅可在合格 real-weight failure 后 diagnostic-only。保持 vllm-dlc source/manifest/alignment/metadata/branch/index/generated files 只读；不得声称 Verified vLLM Alignment。未执行的 real weights、Real DLC Hardware、Chunked Prefill runtime 和 DLC Runtime dispatch 均报告 `not_verified`。

模型适配分析时额外检查：
- serving 启动参数与请求中的 `model` 名必须一致；如果启动用了 `--served-model-name`，OpenAI 请求必须使用该 alias，而不是误用模型路径。
- 先做短 prompt、`temperature=0`、`top_p=1.0`、小 `max_tokens` 的 serving smoke；通过后再增加 prompt 长度、one-shot/CoT、采样温度、Chunked Prefill 或并发，每轮只改变一个变量。
- 长 prompt 或 one-shot 退化要单独记录 prompt token 数、decode 参数、finish reason、输出内容、latency、server liveness 和日志路径；重复 `!`、空输出、超时、异常截断不能归类为环境已通过。
- 量化模型要核对 `quant_method`、`bits`、`group_size`、`zero_point` 与实际 kernel 路由，避免把 `compressed-tensors` 误判为 AWQ/AWQ-Marlin 兼容；W8A16、MoE、vision/multimodal processor 路径应单独列风险。
- 多卡/MoE/精度问题先建立严格等价对照：相同 endpoint、prompt、tokenizer、模型权重、TP/EP、`temperature=0`、`max_tokens` 和可选 `logprobs`；不要混用 chat/completions 与 completions 的结果直接比较。
- 若生成 token 与基线分叉，把多步 decode 改写为单步 prefill：将分叉前 token 拼入 prompt，仅生成 1 个 token，以隔离 KV cache、scheduler 和历史回灌变量。
- 如果怀疑算子或设备卡住，记录 DP/TP 初始化日志、shared memory broadcast 日志和服务状态；`peek_stuck.sh`、软重置、LYP repair 或 reboot 必须有对应授权，不作为默认自动步骤。task-owned KILL 必须满足 sealed identity、graceful timeout、impact record 和独立 `task_owned_kill`，禁止终止非任务进程。
- Real DLC Hardware serving epoch 通过 `dlc-hardware-observability` 保存 `before_launch`、`after_ready`、`during_request` 和 `after_cleanup`；SMI 正常不证明模型正确，observer 缺失不改写为硬件 failure。若只做 static/read-only compatibility analysis，填写 `not_applicable` 及依据，不新增 device execution。
```

## 停止语义与 Evidence

- 模型名称缺失或绝对目录不存在、不可读、malformed 时返回 `blocked_missing_asset`；其他字段必须先自动发现或提出，不能直接归类为缺少用户输入。
- static/pre-handoff mode 未执行模型或硬件时，将 device-backed scope 记录为 `not_verified`，SMI 仅在有依据时为 `not_applicable`；device-backed mode 只接受匹配 handoff 和授权后本次实际生成的模型/硬件 evidence。
- HTTP success、静态检查、fake-server、Dummy 或 DLCsim 不得提升 Real DLC Hardware 或 DLC Runtime acceptance。
- Ticket 06 exact v12 profiles 只证明 `authoritativeness: operational_only`、`acceptance_eligible: false`、alignment unchanged 和 finalization `none` 的 bounded operational state，不证明新模型 target。
- 单个短 prompt serving smoke 只能证明对应启动配置的最小可用性，不证明长上下文、one-shot、Chunked Prefill、DLC Runtime dispatch 或 Real DLC Hardware acceptance 已验证。

## 相关资料

- [模型适配与 Main-to-Main 决策记录](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)
- [cltech_smi 设备观测与诊断证据](../runtime-debugging/chipltech-smi-observability.md)
