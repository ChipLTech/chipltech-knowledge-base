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

先自动发现并读取：
- chipltech-knowledge-base/CONTEXT.md
- chipltech-knowledge-base/vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md
- chipltech-knowledge-base/prompt-examples/host-daily-image-to-model-validation.md
- 当前可用 dlc-env-setup、model-adaptation 和 modelzoo-image-validation skills/scripts

执行：
1. 验证 config、weights、tokenizer/processor applicability、digest 和模型能力。
2. 选择 immutable ordinary daily base，创建 task-owned validation environment，记录 image/container/source/package/import/device/HBM baseline。
3. 依次完成 C1a、fresh-process C1b、真实模型功能和声明 benchmark；失败立即停止提升，保留 failure epoch。
4. 功能至少验证 health、alias、两个正交 deterministic assertions、raw responses、semantic correctness 和 health-after。
5. Benchmark 先 warm-up，再按写入 contract 的 workload 执行；单次成功称 benchmark_workload_pass，不称稳定 baseline。
6. 停止 task-owned processes，确认 port 与 task HBM delta 回到 baseline，输出 artifact paths、状态 matrix、失败边界和 remaining risks。

不要安装全局 skills，不修改 ModelZoo，不下载替代模型，不使用模型专用 image 作为 daily base，不执行未授权 Host/device 维护。
```

需要同时交付 image 时，将模式改为：

```text
qualification_and_image_delivery
```

并使用 [ModelZoo 模型到 DLC/TYD Images Prompt](modelzoo-model-to-dlc-tyd-images.md) 声明 target 与 build/export 授权。
