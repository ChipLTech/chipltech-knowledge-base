# 新模型验证 Quickstart

## 用途

只填写模型名称和本地目录，默认执行 `qualification_only`：从 ordinary daily base 完成 C1a、C1b、真实模型功能和声明的 benchmark。Image delivery 必须显式选择并授权。

本入口不安装全局 skills、不 clone 仓库、不下载模型或依赖；现有 skill 可直接读取使用。缺失依赖或源码时只读发现并返回最小 blocker。

## 可直接复制 Prompt

```md
请直接执行下面模型的 runtime-first 验证，不要只写计划。

模型名称：<MODEL_NAME>
模型目录：<ABSOLUTE_LOCAL_MODEL_PATH>
模式：qualification_only

先自动发现并回显 `<KNOWLEDGE_BASE_ROOT>`、`<SKILLS_ROOT>` 与已安装 skill 路径，然后读取：
- <KNOWLEDGE_BASE_ROOT>/CONTEXT.md
- <KNOWLEDGE_BASE_ROOT>/vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/host-daily-image-to-model-validation.md
- <KNOWLEDGE_BASE_ROOT>/runtime-debugging/chipltech-smi-observability.md
- 当前可用 dlc-env-setup、model-adaptation 和 modelzoo-image-validation skills/scripts

执行：
1. 使用 `modelzoo-image-validation` 验证 config、weights、tokenizer/processor applicability、digest；ModelZoo 仅作可选只读 reference。
2. 先委托 `model-adaptation` 执行只读 pre-handoff analysis，输出与 runtime qualification contract 绑定的模型 capability 与 deployment profile；此时不得加载模型或执行 device-backed adaptation。
3. 委托 Host Daily Image Runbook 选择 immutable ordinary daily base 并创建 task-owned validation environment；委托其使用 `dlc-env-setup` 按 pre-handoff profile 完成 source/package/import、fresh-process C1a/C1b、required collective 和 `environment_handoff/v1`。保留这些模块的 record，不复制其 probe。
4. 只有 handoff 的 required C1a/C1b/collective 都明确通过且没有 blocker，才按 Host Runbook 依次执行真实模型功能和写入 Contract 的 benchmark；任一失败立即停止提升并保留 failure epoch。
5. Real DLC Hardware 时使用同一 run ID 封存四阶段 SMI Observation Envelope；它只证明设备/process/HBM 观测，不替代 C1b、模型功能或 benchmark。
6. Benchmark 先 warm-up，再按声明 workload 执行；单次成功只称 `benchmark_workload_pass`，不称稳定 baseline。
7. 输出 resolved manifest、runtime qualification contract、runtime action record、`environment_handoff/v1`、C0-C5 terminal states、artifact paths、claim boundary、unverified scope、失败边界和 remaining risks。
8. 只停止 task-owned processes，确认 port、task HBM delta 和 `after_cleanup` observation 回到 sealed baseline/tolerance。

不要安装全局 skills，不修改 ModelZoo，不下载替代模型，不使用模型专用 image 作为 daily base，不执行未授权 Host/device 维护。
```

需要同时交付 image 时，将模式改为：

```text
qualification_and_image_delivery
```

并使用 [ModelZoo 模型到 DLC/TYD Images Prompt](modelzoo-model-to-dlc-tyd-images.md) 声明 target，以及 build/install、device execution、tar export、registry push 和 TYD full-stack rebuild 的逐项授权。该 prompt 会先交付 DLC，再以其 immutable Image ID 作为 TYD full-stack rebuild 的基线。
