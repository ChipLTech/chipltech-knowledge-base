---
prompt_schema: vllm-dlc-two-stage-prompt/v1
stage_1_skill_identity: dlc-env-setup
stage_2_skill_identity: model-adaptation
shared_contract: vllm-dlc-contract/v1
recommended_order:
  - dlc-env-setup
  - model-adaptation
claim_boundary: operational_or_not_verified_only
---

# 每日空镜像到新模型 vLLM-DLC 适配 Prompt

## 用途

用于在一个空的每日镜像里，先把 DLC Ecosystem 初始化成健康工作站，再对一个明确新模型进行 vLLM-DLC / DLC Platform 适配分析。

结论：可以这样做，而且推荐这样做。

```text
先 dlc-env-setup，把空每日镜像变成健康 DLC Ecosystem 工作站
再 model-adaptation，对新模型做 vLLM-DLC / DLC Platform 适配分析
```

## 边界

- 第一阶段是环境初始化/修复，不是模型适配。
- 第二阶段是模型兼容性分析，不负责重建 DLC Ecosystem。
- 第一阶段通过只说明环境健康；不证明新模型已经 Real DLC Hardware accepted、Verified vLLM Alignment、DLC Runtime dispatch 或 request-correlated Chunked Prefill。
- 第二阶段如果缺少模型资产、revision、deployment profile、硬件条件或 artifact destination，应先报告 blocker，不要继续假装完成。
- 所有 artifact destination 必须在 `vllm-dlc` 源码树外。

## 可复制 Prompt

```md
请按两阶段处理：先使用 `dlc-env-setup`，再在通过条件满足后使用 `model-adaptation`。

必须先读取并遵循：
- /work/chipltech-knowledge-base/CONTEXT.md
- /work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md
- /work/chipltech-knowledge-base/prompt-examples/vllm-dlc-model-adaptation.md
- /work/skills/skills/engineering/dlc-env-setup/SKILL.md
- /work/skills/skills/engineering/model-adaptation/SKILL.md

总体目标：
1. 先把这个空每日镜像初始化成健康 DLC Ecosystem 工作站。
2. 环境健康后，再对一个明确新模型做 vLLM-DLC / DLC Platform 适配分析。
3. 不把环境初始化结果提升成新模型 acceptance、Verified vLLM Alignment 或 Real DLC Hardware 权威证据。

阶段 1：环境初始化，使用 `dlc-env-setup`

【模式】<全量重建 / 从 LLVM 开始 / 从 DLC_Custom_Kernel 开始 / 只重建 PyTorch wheel / 只重装 wheel / 修 vllm>
【搜索根目录】<例如 /work,$HOME；不确定则写“请自动发现”>
【需要切换的批准 ref】<逐仓库填写 remote URL、目标 branch/tag 和可选 commit SHA；没有则写“无”>
【是否包含 vllm / vllm-dlc】是
【是否允许修改 /usr/local】<是/否>

阶段 1 必须交付：
- 最终 repo map。
- 每个仓库的 Git root、remote、branch/tag、HEAD、status。
- 起始阶段和依赖健康证据。
- PyTorch 2.5.0 wheel 路径与 reinstall 结果。
- `/work/skills/skills/engineering/dlc-env-setup/scripts/pytorch-preflight.sh` 结果。
- `/work/skills/skills/engineering/dlc-env-setup/scripts/vllm-preflight.sh` 结果。
- `/work/skills/skills/engineering/dlc-env-setup/scripts/runtime-smoke.sh /tmp` 结果。
- `vllm` / `vllm-dlc` import 与 package metadata 验证结果。

阶段 1 停止条件：
- repo root、remote、branch/tag、HEAD 不明确或不符合批准 ref。
- 工作树有未提交改动且切换 ref 会覆盖它们。
- 必需 build entrypoint 缺失。
- `/usr/local` 修改未授权但当前阶段需要安装到 `/usr/local`。
- PyTorch、NumPy bridge、DLC Platform、请求范围内的 `vllm` / `vllm-dlc` import 或 runtime smoke 不健康。

只有阶段 1 明确通过后，才能进入阶段 2。

阶段 2：新模型适配分析，使用 `model-adaptation`

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
- 阶段 1 环境初始化报告路径: <DLC_ENV_SETUP_REPORT_PATH>

阶段 2 要求：
- 先列出缺失输入并停止；资产缺失用 `blocked_missing_asset`，硬件不足用 `blocked_missing_hardware`。
- 只做新模型兼容性分析和报告，不直接修改 `vllm-dlc`。
- 不继承 Ticket 06 v12 operational evidence 给这个新 target。
- 未针对该新模型执行的 real weights、Real DLC Hardware、Chunked Prefill runtime 和 DLC Runtime dispatch 均报告 `not_verified`。
- 输出写到声明的外部 artifact destination。
```

## 相关资料

- [dlc-env-setup skill 使用模板](dlc-env-setup-skill-usage.md)
- [Model Adaptation 可复用 Prompt](vllm-dlc-model-adaptation.md)
- [模型适配与 Main-to-Main 决策记录](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)
