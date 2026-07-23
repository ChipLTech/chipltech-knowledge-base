---
prompt_schema: modelzoo-model-to-dlc-tyd-images/v3
minimum_input: model_name_and_model_path
default_mode: qualification_and_image_delivery
---

# ModelZoo 模型到 DLC/TYD Images Prompt

## 用途

这是薄入口。它收集模型、target 和授权，具体状态机统一由 [模型运行资格与镜像交付 Contract](../vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md) 定义，Host 操作统一由 [Host Daily Image Runbook](host-daily-image-to-model-validation.md) 定义。DLC 先完成交付；TYD 从该 DLC image 的 immutable Image ID 派生并独立报告状态。

ModelZoo 是可选只读 reference。本入口不重写 C1a/C1b probe、源码重建命令、SMI adapter 或 benchmark runner；它只传递输入、授权、target、artifact root 和完成条件给对应 owner。

## Owner 与交接

| 阶段 | Owner | 本入口必须消费的结果 |
|---|---|---|
| 本地资产与 ModelZoo reference | `modelzoo-image-validation` | resolved model manifest 或精确 blocker |
| daily base、task container、C1a/C1b、functional、benchmark、cleanup | Host Daily Image Runbook | runtime qualification contract、runtime action record、`environment_handoff/v1`、C0-C5 terminal states |
| 环境/源码/toolchain 与可执行 C1a/C1b probes | `dlc-env-setup` delegated by Host Daily Image Runbook | source/package/device evidence 或 blocker |
| 模型 capability、兼容性与 deployment profile | `model-adaptation` pre-handoff analysis | contract-bound pre-handoff deployment profile 或 blocker |
| target status、claim 和镜像交付 | Contract | sealed delivery record、DLC/TYD independent final status |

交接只使用 Contract 定义的 `environment_handoff/v1`，不另造 PASS 语义。Host Daily Image Runbook 是其唯一 producer；consumer 必须按 Contract 验证 identity 和 required C1a/C1b/collective PASS，不能将 terminal、`not_executed` 或 `not_applicable` 当作模型加载前提。

## 最少填写

```text
模型名称：<MODEL_NAME>
模型目录：<ABSOLUTE_LOCAL_MODEL_PATH>
目标：<dlc | tyd | both，默认 both>
授权：<build/install、device execution、privileged Host integration、tar export、registry push、create_tyd_full_stack_rebuild；逐项 yes | no>
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
- privileged Host integration：<yes | no；若 yes，附批准的 mount/privilege profile>
- tar export：<yes | no>
- registry push：<yes | no>
- create_tyd_full_stack_rebuild：<yes | no>

开始前先发现并回显实际的 `<KNOWLEDGE_BASE_ROOT>`、`<SKILLS_ROOT>` 和已安装 skill 路径；不要假定 `/work`、`/home/workspace` 或某个历史 checkout。然后完整读取：
- <KNOWLEDGE_BASE_ROOT>/CONTEXT.md
- <KNOWLEDGE_BASE_ROOT>/vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/host-daily-image-to-model-validation.md
- <KNOWLEDGE_BASE_ROOT>/runtime-debugging/chipltech-smi-observability.md
- 当前安装的 modelzoo-image-validation、dlc-env-setup、model-adaptation SKILL.md，以及本次实际调用的脚本/`--help`
- 仅在缺少 task-owned source 或合格 CMake 且已授权 bootstrap 时读取 <KNOWLEDGE_BASE_ROOT>/prompt-examples/dlc-env-setup-environment-bootstrap.md

执行要求：
1. 调用 `modelzoo-image-validation` 验证本地模型资产并生成 resolved model manifest。ModelZoo 缺失、不完整、歧义或 malformed 只记录 reference status，除非本次明确要求它为必需来源。
2. 在 Runtime Qualification Contract 已写入后，先委托 `model-adaptation` 执行只读 pre-handoff analysis，生成 contract-bound capability matrix 和 deployment profile；此阶段不得 model load、serving、benchmark、image delivery 或 device-backed adaptation。
3. 再委托 Host Daily Image Runbook 创建 immutable ordinary daily base 上的 task-owned persistent container 并完成 C0；使用 `dlc-env-setup` 按该 profile 闭合 source/toolchain/package/import 与 fresh-process C1a/C1b/required collective，并生成 Contract 定义的 `environment_handoff/v1`。共享、已变更、模型专用、golden 或 candidate image/container 不能作为本次资格证据。handoff 后的 device-backed adaptation 必须消费该 handoff。保留其 evidence，不复制 probe 或重建命令。任何必需 observer 缺失返回 `blocked_missing_observability`，不归因为硬件失败。
4. 消费 `environment_handoff/v1` 与 runtime qualification contract 后，才允许按 Host Runbook 执行 real-weight functional -> declared benchmark。server 必须使用实际 `--help`、absolute `--model` 和离线 Hub guard；functional 与 benchmark 的状态、证据和升级规则以 Contract 为准。
5. 所有中断按 Contract 的 task-owned recovery 规则处理：保留第一失败边界，每次 recovery 新建 failure epoch 且只改变一个变量。不得把非 container 的 C1b failure 归因为 profile mismatch，不得触碰共享资源。
6. 只有 runtime gates 已闭合，才写 sealed delivery record 并构建、exact-image 验证和导出 DLC image；提前构建的 image 必须标记 `prequalification_only`。DLC 必须记录 fixed tag、Image ID、tar、SHA-256、attestation、validation report 和 final status，且模型权重不进入 image。
7. target 包含 tyd 时，只能以当前模型已交付 DLC image 的 immutable Image ID 为 baseline。按 Contract 的 TYD Build Closure And Recovery 在所需授权下以 `DLC_TPU_VERSION=2` 完整重编；TYD 必须独立记录 fixed tag、Image ID、tar、SHA-256、attestation、static/exact-image validation 和 final status。TYD 被 blocked 或 failed 不得改变已完成 DLC 的 final status。
8. DLC Chip Host 上不得执行 TYD device operation、C1b、DLCCL、model load、serving 或 benchmark；统一记录 `intentionally_not_executed_on_dlc_gen1`。
9. 只清理 task-owned resources，保留正式交付物和失败 epochs。按 Host Runbook 保存 `after_cleanup` 并与 baseline 对比；最终输出 DLC/TYD independent matrix、claim boundaries、record/artifact paths、cleanup/observation evidence、unverified scope 和 remaining risks。

自动发现不授权 network download、clone/fetch、package install、Host maintenance、registry push、reset/reboot 或抢占设备。任一成为继续条件时输出 structured blocker 和最小恢复输入。
```

## 相关资料

- [规范 Contract](../vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md)
- [Host Daily Image Runbook](host-daily-image-to-model-validation.md)
- [新模型 Quickstart](new-model-validation-quickstart.md)
