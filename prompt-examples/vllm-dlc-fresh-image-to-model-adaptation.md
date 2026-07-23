---
prompt_schema: vllm-dlc-three-stage-prompt/v2
required_user_inputs:
  - model_name
  - absolute_local_model_path
stage_0_skill_identity: model-adaptation-pre-handoff-analysis
stage_1_skill_identity: dlc-env-setup-and-host-c1b
stage_2_skill_identity: model-adaptation-device-backed
shared_contract: vllm-dlc-contract/v1
recommended_order:
  - model-adaptation-pre-handoff-analysis
  - dlc-env-setup-and-host-c1b
  - model-adaptation-device-backed
claim_boundary: operational_or_not_verified_only
discovery_policy: local_query_only_then_contract_proposal
---

# 每日空镜像到新模型 vLLM-DLC 适配 Prompt

## 用途

用于在一个空的每日镜像里，只凭模型名称和本地绝对目录完成 profile derivation、DLC Ecosystem 初始化与 device-backed adaptation。环境模式、source/ref、CMake、mount/profile、设备和交接字段均由 agent 自动发现或提出。

结论：可以这样做，而且推荐这样做。

```text
先 model-adaptation 的只读 pre-handoff analysis，导出 deployment profile
再 dlc-env-setup + Host Runbook，按该 profile 完成 C1a/C1b 和 handoff
最后 model-adaptation，执行需要 device-backed evidence 的适配分析
```

## 边界

- pre-handoff analysis 是只读模型兼容性分析，只生成 capability matrix 和 deployment profile，不加载模型或执行设备操作。
- 环境阶段负责初始化/修复与按 profile 执行 C1a/C1b，不做模型 acceptance。
- device-backed adaptation 不负责重建 DLC Ecosystem，且必须消费合格 handoff。
- C1a/C1b 通过只说明环境与 bounded DLC Runtime execution；不证明新模型已经 Real DLC Hardware accepted、Verified vLLM Alignment、DLC Runtime dispatch 或 request-correlated Chunked Prefill。
- 只有模型名称或绝对路径缺失/无效时立即报告 input blocker；revision、deployment profile、硬件条件和 artifact destination 必须先自动发现或提出，无法闭合时再报告对应 blocker。
- `/mnt/jfs/models` 是团队模型文件服务器挂载根目录，模型路径优先从这里选择并记录完整绝对路径。
- 所有 artifact destination 必须在 `vllm-dlc` 源码树外。
- 阶段 1 输出是阶段 2 的输入证据，不是模型通过结论。阶段 2 只消费明确的 handoff，不得从口头“环境已好”推导任何运行状态。

## 阶段交接 Contract

阶段 1 只有在创建 task-owned daily-image environment 且完成 C1a/C1b 时，才由 Host Daily Image Runbook 按 [模型运行资格与镜像交付 Contract](../vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md) 写 `environment_handoff/v1`。它索引既有 evidence，不新建 PASS 状态。纯 bootstrap、静态检查或未执行 C1b 的阶段 1 只交付 bootstrap report，不能产生 environment handoff。

```text
daily_base_image_id
task_container_identity
driver/runtime/profile fingerprint
source/package/plugin/extension identities
C1a result and artifact path
C1b results by logical device and collective scope
SMI observer status and artifact paths
cleanup baseline
unresolved blockers
explicitly unverified scope
```

阶段 2 可在 handoff 前执行只读 capability matrix 和 pre-handoff deployment profile derivation，且必须将 profile 绑定到后续 runtime qualification contract。任何 device-backed adaptation、model load、serving、benchmark 或 image delivery 仅在 handoff 的 identity 与当前 contract 相符、每个 required C1a/C1b/collective result 都明确 pass 且无 blocker 时开始。`not_executed` 或 `not_applicable` 不能放行需要设备执行的模型适配。

## 可复制 Prompt

```md
请按三个阶段处理并持续到 terminal state。不要要求我预填可从当前 Host、container、source、package、model config、历史过程资产或实际 `--help` 自动发现的字段。

模型名称：<MODEL_NAME>
模型目录：<ABSOLUTE_LOCAL_MODEL_PATH>

除这两行外，全部自动发现或提出：知识库/skill roots、daily image、repo map、ref identities、rebuild 起点、CMake、mount/profile、设备、deployment profile、artifact destination、handoff 和 serving assertions。优先保持当前 checkout 和复用健康本地环境；任何 qualification execution、task-owned KILL、fetch/ref switch/install/build/`/usr/local`/privileged profile/device execution/Host maintenance 只在成为继续条件时消费已验证 standing authorization 或请求最小增量。

先发现并回显 `<KNOWLEDGE_BASE_ROOT>`、`<SKILLS_ROOT>` 与当前安装的 skill 路径；不要假定 `/work`。然后读取并遵循：
- <KNOWLEDGE_BASE_ROOT>/CONTEXT.md
- <KNOWLEDGE_BASE_ROOT>/vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/host-daily-image-to-model-validation.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/dlc-env-setup-skill-usage.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/vllm-dlc-model-adaptation.md
- <KNOWLEDGE_BASE_ROOT>/runtime-debugging/chipltech-smi-observability.md
- 当前安装的 dlc-env-setup/SKILL.md 与 model-adaptation/SKILL.md

总体目标：
1. 先完成只读 capability matrix 和与 runtime qualification contract 绑定的 pre-handoff deployment profile。
2. 使用该 profile 初始化并验证 DLC Ecosystem 工作站。
3. handoff 合格后，再对该明确新模型做 device-backed vLLM-DLC / DLC Platform 适配分析。
4. 不把环境初始化结果提升成新模型 acceptance、Verified vLLM Alignment 或 Real DLC Hardware 权威证据。
5. 新模型验证按阶梯推进：先短 prompt serving smoke，再逐步扩展到长 prompt / one-shot 敏感场景。

阶段 0：Runtime Qualification Contract 与只读 pre-handoff analysis，使用 `model-adaptation`

先按 Contract 写 Runtime Qualification Contract；再在执行任何 C1b 前，基于模型资产、revision、目标 source identity、硬件要求和 artifact destination 输出 capability matrix 与 pre-handoff deployment profile。profile 至少绑定 requested logical-device mapping、TP/PP/EP、dtype/quantization、context/batching limit、collective communication requirement 和 runtime qualification contract identity。此阶段不执行 model load、serving、benchmark、image delivery 或任何 device-backed adaptation。

阶段 1：环境初始化和 C1a/C1b，使用 `dlc-env-setup` 与 Host Daily Image Runbook

自动生成 Stage 1 proposal：repo map、当前 checkout policy、最小 rebuild/repair 起点、CMake candidate、required container profile 和所需授权。默认保持当前 checkout；`CI默认最新` 只可作为 proposal，不能自动 fetch 或切换。只请求当前环境与目标 profile 的精确 diff。

阶段 1 必须交付：
- 最终 repo map。
- 每个仓库的 Git root、remote、branch/tag、HEAD、status。
- 起始阶段和依赖健康证据。
- PyTorch 2.5.0 wheel/package identity 与 validation result；已有环境健康时 reinstall 记录 `not_required`，不得为了交付字段而重装。
- 已发现的 `dlc-env-setup/scripts/pytorch-preflight.sh` 结果。
- 已发现的 `dlc-env-setup/scripts/vllm-preflight.sh` 结果。
- 已发现的 `dlc-env-setup/scripts/runtime-smoke.sh /tmp` 结果。
- `vllm` / `vllm-dlc` import 与 package metadata 验证结果。
- 实际执行 Real DLC Hardware 时，`dlc-hardware-observability` 的 observer identity、四阶段 raw/normalized evidence 和 cleanup closure；仅环境静态检查时记录 `not_applicable`。

阶段 1 停止条件：
- repo root、remote、branch/tag、HEAD 不明确或不符合批准 ref。
- 工作树有未提交改动且切换 ref 会覆盖它们。
- 自动发现后 source identity 仍不唯一；默认 `preserve_current_checkout` 无法满足 profile，且所需 fetch/ref change 未获授权。
- 必需 build entrypoint 缺失。
- `/usr/local` 修改未授权但当前阶段需要安装到 `/usr/local`。
- PyTorch、NumPy bridge、DLC Platform、请求范围内的 `vllm` / `vllm-dlc` import 或 runtime smoke 不健康。
- CMake 缺失或版本不满足 `>3.27.0`，且没有授权安装/切换到合格版本。
- 后续 serving smoke 需要的 `/mnt/jfs` 或 `/dev/dlc*` 在当前容器不可见。
- 驱动重载、软重置、`cltech-init`/`dlc-init`、LYP repair 或物理机 reboot 未获明确授权。

只有阶段 1 已按阶段 0 profile 完成 C1a/C1b/required collective 并生成合格 `environment_handoff/v1` 后，才能进入阶段 2。

阶段 2：device-backed 新模型适配分析，使用 `model-adaptation`

由 agent 输出 resolved Stage 2 record：本地模型 digest/revision-or-null、source full SHAs、deployment profile、hardware availability、manifest/dependency identity、外部 artifact destination、Stage 1 report 和 `environment_handoff/v1` 路径。输入路径不在 `/mnt/jfs/models` 时只记录实际来源，不替换模型或下载替代品。

阶段 2 要求：
- 只有模型名/绝对路径无效才视为初始缺失；其他字段先 discovery。资产缺失用 `blocked_missing_asset`，硬件不足用 `blocked_missing_hardware`，授权缺失用 `blocked_missing_authorization`。
- 只做新模型兼容性分析和报告，不直接修改 `vllm-dlc`。
- 不继承 Ticket 06 v12 operational evidence 给这个新 target。
- 未针对该新模型执行的 real weights、Real DLC Hardware、Chunked Prefill runtime 和 DLC Runtime dispatch 均报告 `not_verified`。
- 输出写到声明的外部 artifact destination。
- handoff 前仅可完成只读 capability matrix 和 pre-handoff deployment profile derivation，并将其绑定到后续 runtime qualification contract；不得执行 model load、serving、benchmark 或 image delivery。
- 在任何 device-backed adaptation 前校验 Contract 定义的 `environment_handoff/v1`：缺失、字段身份不一致、任一 required C1a/C1b/collective 未明确 pass 或有未解决 blocker 时停止并报告最小恢复输入；不得重跑或猜测阶段 1 的结果。

阶段 2 serving smoke 阶梯：
1. 先做短 prompt serving smoke。
    - 使用短文本 prompt。
    - 优先 `temperature=0`、`top_p=1.0`。
    - `max_tokens` 先用 64 或 128。
    - 如果启动服务时指定了 `--served-model-name`，请求里的 `model` 必须使用该 alias；没有 alias 时才使用启动命令约定的模型路径/名称。
    - 目标是验证模型能加载、API 能返回非空生成结果、server liveness 正常。
2. 短 prompt 通过后，再扩展到中等长度 prompt。
    - 仍保持 greedy / deterministic 参数。
    - 观察是否出现空输出、重复符号、明显降速或截断。
3. 中等长度通过后，再验证长 prompt / one-shot 敏感场景。
    - 可参考同事文档中 Qwen3.5-27B 的 one-shot、长 padding 和 CoT 敏感案例。
    - 每次只改变一个变量：prompt 长度、one-shot 结构、temperature、`max_tokens` 或 Chunked Prefill 相关参数。
    - 出现重复 `!`、空输出、速度下降或 timeout 时，记录 prompt token 数、decode 参数、返回内容、finish reason、server liveness 和日志位置。
4. 多卡、量化、MoE、vision/multimodal 或长上下文场景需额外记录：
   - `DLC_VISIBLE_DEVICES`、TP/PP/EP、dtype、quantization、`max_model_len`、`max_num_batched_tokens`、prefix caching、expert parallel、processor/tokenizer revision。
   - 量化配置中的 `quant_method`、`bits`、`group_size`、`zero_point` 与实际 kernel 路由是否一致；`compressed-tensors`、W8A16、AWQ/AWQ-Marlin 不得只按目录名判断兼容。
    - 如果卡在 DP/TP 初始化、shared memory broadcast 或疑似算子 hang，只记录日志和现象；先关联 SMI Observation Envelope、server PID/PGID、端口和 runtime log。`peek_stuck.sh`、软重置、LYP repair 或 reboot 需要对应授权；task-owned KILL 必须满足 sealed identity、graceful timeout、impact record 和独立 `task_owned_kill`，禁止终止非任务进程。
5. 不要把短 prompt smoke 通过解释为长上下文、Chunked Prefill runtime、DLC Runtime dispatch 或 Real DLC Hardware acceptance 已验证。
```

## 相关资料

- [dlc-env-setup skill 使用模板](dlc-env-setup-skill-usage.md)
- [Model Adaptation 可复用 Prompt](vllm-dlc-model-adaptation.md)
- [模型适配与 Main-to-Main 决策记录](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)
