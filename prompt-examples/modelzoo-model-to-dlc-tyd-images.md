---
prompt_schema: modelzoo-model-to-dlc-tyd-images/v2
minimum_input: model_name_and_model_path
default_mode: qualification_and_image_delivery
---

# ModelZoo 模型到 DLC/TYD Images Prompt

## 用途

这是薄入口。它收集模型、target 和授权，具体状态机统一由 [模型运行资格与镜像交付 Contract](../vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md) 定义，Host 操作统一由 [Host Daily Image Runbook](host-daily-image-to-model-validation.md) 定义。DLC 先完成交付；TYD 从该 DLC image 的 immutable Image ID 派生并独立报告状态。

ModelZoo 是可选只读 reference。功能和性能必须先在 ordinary daily base 环境通过，之后才构建正式 image。

## 最少填写

```text
模型名称：<MODEL_NAME>
模型目录：<ABSOLUTE_LOCAL_MODEL_PATH>
目标：<dlc | tyd | both，默认 both>
授权：<build/install、device execution、tar export、registry push、create_tyd_full_stack_rebuild；逐项 yes | no>
```

## 可直接复制 Prompt

```md
请按 runtime-first workflow 直接执行该模型的运行资格验证。DLC delivery gates 通过后，构建、exact-image 验证并导出 DLC image；若 target 包含 tyd，则以该 DLC image 的 immutable Image ID 为 build baseline，完整重编 TYD full stack，再独立验证和导出 TYD image。

模型名称：<MODEL_NAME>
模型目录：<ABSOLUTE_LOCAL_MODEL_PATH>
Framework selector：<可空>
ModelZoo root：<可空，默认只读发现>
目标：<dlc | tyd | both>
Artifact root：<可空，自动选择持久目录>
Benchmark workload：<可空；自动提出保守 workload 并写入 contract>
授权：
- build/install：<yes | no>
- device execution：<yes | no>
- tar export：<yes | no>
- registry push：<yes | no>
- create_tyd_full_stack_rebuild：<yes | no>

开始前完整读取：
- <KNOWLEDGE_BASE_ROOT>/CONTEXT.md
- <KNOWLEDGE_BASE_ROOT>/vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/host-daily-image-to-model-validation.md
- 当前安装的 modelzoo-image-validation、dlc-env-setup、model-adaptation SKILL.md 和实际脚本/--help

执行要求：
1. 先验证本地模型资产；ModelZoo 缺失、不完整、歧义或 malformed 只记录 reference status，除非本次明确要求它为必需来源。
2. 从 immutable ordinary daily base 创建 task-owned validation environment；禁止使用 Jiutian 或其他模型专用/golden/candidate image 继承当前结论。
3. 写 resolved model manifest、runtime qualification contract 和 runtime action record，再按 C1a -> C1b -> real-weight functional -> declared benchmark 顺序执行。
4. Functional 至少保存两个正交 deterministic assertions 的 raw request/response，并区分 HTTP、request completion、non-empty、semantic correctness 和 health-after。
5. Benchmark 保存 --help、profile diff、warm-up、formal attempts、raw client/server logs、structured result 和 health-after。明确区分 benchmark_workload_pass 与 benchmark_stability_baseline_pass。
6. DLC runtime gates 通过后，写 sealed delivery record，并构建、exact-image 验证和导出 DLC image；提前构建的 image 标记 prequalification_only。
7. DLC 使用 fixed tag、Image ID、tar、SHA-256、attestation、validation report 和 final status。模型权重不进入 image。
8. target 包含 tyd 时，必须以当前模型已交付 DLC image 的 immutable Image ID 为 baseline。在该镜像基础上以 DLC_TPU_VERSION=2 完整重编 TYD stack；此动作需要 build/install、tar export 和 create_tyd_full_stack_rebuild 的明确授权。
9. TYD 使用独立 fixed tag、Image ID、tar、SHA-256、attestation、static/exact-image validation 和 final status。TYD 被 blocked 或 failed 不得改变已完成 DLC 的 final status。
10. DLC Chip Host 上不得执行 TYD device operation、C1b、DLCCL、model load、serving 或 benchmark；统一记录 intentionally_not_executed_on_dlc_gen1。
11. 只清理 task-owned resources，保留正式交付物和失败 epochs。最终输出 DLC/TYD 独立 matrix、claim boundaries、artifact paths、cleanup evidence 和 remaining risks。

自动发现不授权 network download、clone/fetch、package install、Host maintenance、registry push、reset/reboot 或抢占设备。任一成为继续条件时输出 structured blocker 和最小恢复输入。
```

## 相关资料

- [规范 Contract](../vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md)
- [Host Daily Image Runbook](host-daily-image-to-model-validation.md)
- [新模型 Quickstart](new-model-validation-quickstart.md)
