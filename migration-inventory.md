# 迁移清单

追踪所有原始资料到 `chipltech-knowledge-base` 的迁移状态。

## 状态说明

- `done` — 已迁移到目标路径
- `partial` — 部分迁移
- `pending` — 待迁移
- `skip` — 不适合迁移（临时 artifact、大体积文件等）
- `needs-conversion` — 需要人工转换（docx 等）

## 来源：/work/plan/dlc基础

| 来源文件 | 目标路径 | 状态 | 含图片 | 需脱敏 | 需人工转换 | 摘要 | 备注 |
|---------|---------|------|--------|--------|-----------|------|------|
| CONTEXT.md | 部分合并到 `CONTEXT.md` 和 `foundation/glossary.md` | done | 否 | 否 | 否 | DLC-Family Accelerator 领域词汇表 | 术语已整合 |
| DLC基础知识手册.md | `foundation/dlc-ecosystem-overview.md` | done | 否 | 否 | 否 | 硬件、软件栈、调试、精度、算子接入速记 | 核心入口资料 |
| TPU基础架构.md | 合并到 `foundation/dlc-ecosystem-overview.md` | done | 是 | 否 | 否 | 芯片代际、硬件模块、架构图 | 图片待迁移 |
| DLC各个概念介绍.md | 合并到 `foundation/dlc-ecosystem-overview.md` | done | 是 | 否 | 否 | vLLM、SGLang、PyTorch、DLCSynapse 等概念 | 图片待迁移 |
| DLC算子实现与CUDA对比报告.md | `pytorch-dlc-backend/operator-integration-guide.md` + `foundation/dlc-vs-cuda-comparison.md` | done | 否 | 否 | 否 | DLC Custom Kernel 与 CUDA 编程模型差异 | 核心对比资料 |
| cuda和dlc生态分析.md | `foundation/dlc-vs-cuda-comparison.md` | done | 否 | 否 | 否 | custom launch 生态与 CUDA runtime 生态对比 | 合并到对比文档 |
| 后端不同.md | `foundation/dlc-vs-cuda-comparison.md` | done | 否 | 否 | 否 | Attention backend 选择差异 | 合并到对比文档 |
| pytorch算子插入.md | `pytorch-dlc-backend/operator-integration-guide.md` | done | 是(远程URL) | 否 | 否 | PyTorch DLC 算子接入完整流程 | 远程图片 URL 不从原始资料迁移 |
| vllm DLC算子添加及测试方法.md | `vllm-dlc/custom-op-integration-and-testing.md` | done | 是 | 否 | 否 | vLLM DLC Custom Op 接入链路 | |
| DLC_kernel测试框架新人使用指南.md | `testing/dlc-kernel-test-framework-guide.md` | done | 否 | 否 | 否 | pytorch_test Framework 使用指南 | |
| torch的测试框架新人使用指南.md | `testing/pytorch-test-framework-guide.md` | done | 否 | 否 | 否 | PyTorch DLC 原生测试框架指南 | |
| DLC常用调试指令和方法.md | `debugging-workflows/common-debug-commands.md` | done | 是 | 可能 | 否 | 设备、Synapse、trace、peek、RDMA 等调试命令 | 需检查路径是否脱敏 |
| vscode调试配置debug.md | `debugging-workflows/vscode-debug-setup.md` | done | 是 | 可能 | 否 | VSCode SSH/debug 配置 | 截图需检查是否含内部 IP/路径 |
| dlc环境配置更新各个仓.md | `runtime-debugging/environment-setup-and-update.md` | done | 是 | 可能 | 否 | 各仓库编译和版本对齐 | 路径需脱敏 |
| 报错问题记录.md | `runtime-debugging/common-error-log.md` | done | 是 | 可能 | 否 | 常见报错速查 | 需检查路径是否脱敏 |
| Paraformer_DLC_CPU精度差异定位报告.md | `case-studies/paraformer-cpu-dlc-precision-report.md` | done | 否 | 可能 | 否 | Paraformer DLC/CPU 精度差异定位 |
| Paraformer_QKV_Linear精度问题到pytorch_test复现经验报告.md | `case-studies/paraformer-qkv-linear-precision.md` | done | 否 | 可能 | 否 | QKV Linear 精度问题到 pytorch_test 复现 |
| Jarvis_DLC问题定位到pytorch_test复现流程报告.md | `case-studies/jarvis-layernorm-to-repro.md` | done | 否 | 可能 | 否 | Jarvis LayerNorm NaN 到 pytorch_test 复现 |
| Jarvis_DLC_AdamW算子错误分析报告.md | `case-studies/jarvis-adamw-operator-error.md` | done | 否 | 可能 | 否 | Jarvis AdamW foreach 算子错误分析 |
| DLC各个概念介绍.docx | `migration-inventory.md` 记录 | skip | — | — | needs-conversion | 已有 .md 版本 | 使用 .md 版本 |
| DLC常用调试指令和方法.docx | `migration-inventory.md` 记录 | skip | — | — | needs-conversion | 已有 .md 版本 | 使用 .md 版本 |
| vllm DLC算子添加及测试方法.docx | `migration-inventory.md` 记录 | skip | — | — | needs-conversion | 已有 .md 版本 | 使用 .md 版本 |
| dlc环境配置更新各个仓.docx | `migration-inventory.md` 记录 | skip | — | — | needs-conversion | 已有 .md 版本 | 使用 .md 版本 |
| TPU基础架构.docx | `migration-inventory.md` 记录 | skip | — | — | needs-conversion | 已有 .md 版本 | 使用 .md 版本 |
| vscode调试配置debug.docx | `migration-inventory.md` 记录 | skip | — | — | needs-conversion | 已有 .md 版本 | 使用 .md 版本 |
| 报错问题记录.docx | `migration-inventory.md` 记录 | skip | — | — | needs-conversion | 已有 .md 版本 | 使用 .md 版本 |

## 来源：/work/plan/newraw

| 来源文件 | 目标路径 | 状态 | 含图片 | 需脱敏 | 需人工转换 | 摘要 | 备注 |
|---------|---------|------|--------|--------|-----------|------|------|
| DLCCL-2.docx | `foundation/dlccl-guide.md` | done | 是 | 否 | 否 | DLCCL 集合通信（Ring/Tree/Broadcast/AllReduce） | 已转换并迁移 |
| ResultChecker 做了什么.docx | `testing/resultchecker-explained.md` | done | 否 | 否 | 否 | ResultChecker RPD 误差计算和判定逻辑 | 已转换并迁移 |
| Synapse 使用文档.docx | `debugging-workflows/synapse-usage-guide.md` | done | 是 | 否 | 否 | DLCSynapse 测试编写和调试 | 已转换并迁移 |
| TPU Profile 工具简介.docx | `runtime-debugging/performance-profiling.md` | done | 是 | 否 | 否 | Perfetto 性能 profiling 工具 | 已转换并迁移 |
| TPU基础架构-2.docx | `foundation/hardware-specifications.md` | done | 是 | 否 | 否 | 硬件规格参数、芯片代际对比 | 已转换并迁移 |
| 如何正确的debug.docx | `debugging-workflows/effective-debugging.md` | done | 否 | 否 | 否 | Debug 方法论和排查流程 | 已转换并迁移 |
| 常用builtin介绍.docx | `foundation/dlc-builtin-functions.md` | done | 是 | 否 | 否 | dlc_dma_new、v_f32_ld_tnsr_st_msk、Print 等 | 已转换并迁移 |
| 精度标准.docx | `precision-debugging/precision-standards.md` | done | 否 | 否 | 否 | 算子精度验收标准（ULP/atol/rpd 分级） | 已转换并迁移 |
| Pytorch算子插入&测试(简单目录).md | 内容已包含在 `pytorch-dlc-backend/operator-integration-guide.md` 和 `testing/` | done | 否 | 否 | 否 | 简化版算子插入和测试目录 | 内容与主文档重叠 |

## 来源：/work/RSThinker

| 来源文件 | 目标路径 | 状态 | 含图片 | 需脱敏 | 需人工转换 | 摘要 | 备注 |
|---------|---------|------|--------|--------|-----------|------|------|
| CONTEXT.md | `agent-context/` | pending | 否 | 否 | 否 | RSThinker DLC Platform Adaptation 术语 | 作为跨模型可复用经验 |
| docs/rsthinker_custom_softmax_diagnosis_experience.md | `case-studies/rsthinker-custom-softmax.md` | pending | 否 | 否 | 否 | custom_softmax 精度定位经验 |
| docs/rsthinker_precision_dump_runbook.md | `precision-debugging/precision-dump-runbook.md` | pending | 否 | 否 | 否 | 精度 dump 标准操作流程 |
| docs/rsthinker_visual_mlp_drift_diagnosis_experience.md | `case-studies/rsthinker-visual-mlp-drift.md` | pending | 否 | 否 | 否 | 视觉 MLP drift 定位经验 |
| docs/adr/0001-start-with-pytorch-dlc-backend.md | `agent-context/` | pending | 否 | 否 | 否 | 架构决策：优先 PyTorch DLC Backend 路径 |
| dlc_model_placement_hang_analysis.md | `case-studies/model-placement-hang.md` | pending | 否 | 否 | 否 | 模型放置卡住分析 |
| DLC_PLATFORM_ADAPTATION_PLAN.md | `agent-context/` | pending | 否 | 否 | 否 | RSThinker DLC 平台适配计划 |

## 来源：/work/plans

| 来源文件 | 目标路径 | 状态 | 含图片 | 需脱敏 | 需人工转换 | 摘要 | 备注 |
|---------|---------|------|--------|--------|-----------|------|------|
| 算子dispatch.md | `operator-dispatch/enabled-kernels-dispatch.md` | done | 否 | 否 | 否 | enabled_kernels.hpp dispatch 修改方法 |
| dlc-knowledge-base-prd-20260626.md | 本仓库的需求文档 | skip | — | — | — | 已作为 PRD 使用 |
| rsthinker_post_conv_embeddings_drift说明.md | `case-studies/rsthinker-visual-attention-boundary.md` | pending | 否 | 否 | 否 | RSThinker post_conv/embeddings drift 详细分析 |

## 来源：/tmp（精选有长期价值的 handoff/prompt）

| 来源文件 | 目标路径 | 状态 | 含图片 | 需脱敏 | 需人工转换 | 摘要 | 备注 |
|---------|---------|------|--------|--------|-----------|------|------|
| rsthinker_attention_bmm_softmax_sdpa_handoff_20260625_0919.md | `case-studies/rsthinker-visual-attention-boundary.md` | pending | 否 | 否 | 否 | attention 分解定位完整过程 | 与主文档合并 |
| rsthinker_mlp_next_session_prompt_20260626_0134.md | `case-studies/rsthinker-visual-mlp-drift.md` | pending | 否 | 否 | 否 | MLP drift 定位决策树 | 与方法文档合并 |
| rsthinker_dlc_precision_handoff_20260623_cpu_oracle_update.md | `precision-debugging/precision-debugging-overview.md` | pending | 否 | 否 | 否 | CPU oracle 作为精度基准的策略 | 合并到方法论 |

## 图片资产迁移状态

| 来源目录 | 目标目录 | 状态 | 文件数 | 备注 |
|---------|---------|------|--------|------|
| /work/plan/dlc基础/DLC各个概念介绍_assets/ | assets/foundation/dlc-concepts/ | pending | | |
| /work/plan/dlc基础/DLC常用调试指令和方法_assets/ | assets/debugging-workflows/common-debug/ | pending | | |
| /work/plan/dlc基础/TPU基础架构_assets/ | assets/foundation/dlc-architecture/ | pending | | |
| /work/plan/dlc基础/dlc环境配置更新各个仓_assets/ | assets/runtime-debugging/env-setup/ | pending | | |
| /work/plan/dlc基础/报错问题记录_assets/ | assets/runtime-debugging/error-log/ | pending | | |
| /work/plan/dlc基础/vscode调试配置debug_assets/ | assets/debugging-workflows/vscode/ | pending | | |
| /work/plan/dlc基础/vllm DLC算子添加及测试方法_assets/ | assets/vllm-dlc/ | done | 3 | |
| /tmp/newraw_converted/ 各 docx 图片 | assets/foundation/dlccl/ assets/foundation/synapse-usage/ assets/runtime-debugging/ assets/foundation/builtin/ | done | 47 | DLCCL/Synapse/Profile/TPU架构/builtin 图片 |

## 知识正确性审查报告

| 文档组 | 主文档 | 处理结论 |
|--------|--------|----------|
| `foundation/glossary.md` vs `CONTEXT.md` | `CONTEXT.md` | `glossary.md` 明确为速查索引，不再独立维护完整定义和禁用规则。 |
| `foundation/dlc-ecosystem-overview.md` / `foundation/dlc-vs-cuda-comparison.md` / `foundation/hardware-specifications.md` | `foundation/dlc-ecosystem-overview.md` | overview 维护软件栈和真实仓库快照；comparison 聚焦 CUDA 对比；hardware 只维护规格参数，弱化“共享相同基本架构”的绝对表述。 |
| `debugging-workflows/common-debug-commands.md` vs `debugging-workflows/effective-debugging.md` | `debugging-workflows/effective-debugging.md` | commands 保留命令速查；错误表转交 `runtime-debugging/common-error-log.md`，避免重复维护。 |
| `runtime-debugging/runtime-troubleshooting.md` vs `runtime-debugging/common-error-log.md` | `runtime-debugging/runtime-troubleshooting.md` | troubleshooting 保留排障流程；error log 保留错误索引，并统一 `AllocDeviceMemory` 为警告/失败需结合上下文判断。 |
| `case-studies/paraformer-qkv-linear-precision.md` vs `case-studies/paraformer-cpu-dlc-precision-report.md` | `case-studies/paraformer-qkv-linear-precision.md` | QKV 文档是后续完整结论；早期报告顶部已添加合并提示并保留历史过程。 |

## 说明

- 已存在 .md 版本的 docx 文件不再转换，直接使用 .md 版本。
- 远程图片 URL（如 `alidocs.oss-cn-zhangjiakou.aliyuncs.com`）不迁移到本地 assets。
- 所有路径中的内部 IP、token、密码等敏感信息需在迁移时移除或脱敏。
