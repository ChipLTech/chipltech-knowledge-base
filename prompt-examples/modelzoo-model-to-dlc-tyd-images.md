---
prompt_schema: modelzoo-model-to-dlc-tyd-images/v2
minimum_input: model_name_and_model_path
default_mode: qualification_and_image_delivery
---

# ModelZoo 模型到 DLC/TYD Images Prompt

## 用途

这是薄入口。它收集模型、target 和授权，具体状态机统一由 [模型运行资格与镜像交付 Contract](../vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md) 定义，Host 操作统一由 [Host Daily Image Runbook](host-daily-image-to-model-validation.md) 定义。DLC 先完成交付；TYD 从该 DLC image 的 immutable Image ID 派生并独立报告状态。

ModelZoo 是可选只读 reference。先从 ordinary daily base 创建新的 task-owned container，按 Host Daily Image Runbook 初始化 DLC Ecosystem 并完成 C1a/C1b；该已验证环境才可用于模型功能和性能验证，之后才构建正式 image。

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

开始前完整读取：
- <KNOWLEDGE_BASE_ROOT>/CONTEXT.md
- <KNOWLEDGE_BASE_ROOT>/vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/host-daily-image-to-model-validation.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/dlc-env-setup-environment-bootstrap.md
- 当前安装的 modelzoo-image-validation、dlc-env-setup、model-adaptation SKILL.md 和实际脚本/--help

执行要求：
1. 先验证本地模型资产；ModelZoo 缺失、不完整、歧义或 malformed 只记录 reference status，除非本次明确要求它为必需来源。
2. 从 immutable ordinary daily base 创建新的 task-owned persistent container，禁止复用共享或已变更 container，也不得使用 Jiutian 或其他模型专用/golden/candidate image 继承当前结论。
3. 严格按 Host Daily Image Runbook 在该容器中初始化 DLC Ecosystem：创建独立 src/build/wheels/artifacts/logs 和可写 cache，模型只读挂载；在加载模型前闭合 driver API、container C1b-compatible mount/privilege profile、source refs/dirty state、递归 submodule worktree entrypoints、offline wheel/dependency/extension provenance、Python/pip/CMake/compiler、DLC Platform/plugin/extension identity。任何 clone/fetch、package install 或 build 均需对应授权。
4. 仅在所需授权和已批准 remote/ref 仍有效时，按 Contract 的 task-owned recovery 规则处理已知中断；不得把非 container 的 C1b failure 归因为 container profile，不得触碰共享资源。每次 recovery 都新建 failure epoch 且只改变一个变量。
5. 写 resolved model manifest、runtime qualification contract 和 runtime action record；先在 fresh process 完成 C1a 和 C1b。只有该 daily-image environment 的 C1a/C1b 通过，才允许加载模型并按 real-weight functional -> declared benchmark 顺序执行。启动前读取实际 server --help，使用显式 absolute --model、HF_HUB_OFFLINE=1、TRANSFORMERS_OFFLINE=1，并保存未回退远端模型的 server log evidence。
6. Functional 至少保存两个正交 deterministic assertions 的 raw request/response，并区分 HTTP、request completion、non-empty、semantic correctness 和 health-after。
7. Benchmark 保存 --help、profile diff、warm-up、formal attempts、raw client/server logs、structured result 和 health-after。明确区分 benchmark_workload_pass 与 benchmark_stability_baseline_pass。
8. DLC runtime gates 通过后，写 sealed delivery record，并构建、exact-image 验证和导出 DLC image；提前构建的 image 标记 prequalification_only。
9. DLC 使用 fixed tag、Image ID、tar、SHA-256、attestation、validation report 和 final status。模型权重不进入 image。
10. target 包含 tyd 时，必须以当前模型已交付 DLC image 的 immutable Image ID 为 baseline。完整遵循 Contract 的 TYD Build Closure And Recovery；长编译前确认实际 CMake >3.27.0、LLVM -> DLCsim -> 全下游顺序和所需授权，再以 DLC_TPU_VERSION=2 重编 TYD stack。
11. TYD 使用独立 fixed tag、Image ID、tar、SHA-256、attestation、static/exact-image validation 和 final status。TYD 被 blocked 或 failed 不得改变已完成 DLC 的 final status。
12. DLC Chip Host 上不得执行 TYD device operation、C1b、DLCCL、model load、serving 或 benchmark；统一记录 intentionally_not_executed_on_dlc_gen1。
13. 只清理 task-owned resources，保留正式交付物和失败 epochs。最终输出 DLC/TYD 独立 matrix、claim boundaries、artifact paths、cleanup evidence 和 remaining risks。

自动发现不授权 network download、clone/fetch、package install、Host maintenance、registry push、reset/reboot 或抢占设备。任一成为继续条件时输出 structured blocker 和最小恢复输入。
```

## 相关资料

- [规范 Contract](../vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md)
- [Host Daily Image Runbook](host-daily-image-to-model-validation.md)
- [新模型 Quickstart](new-model-validation-quickstart.md)
