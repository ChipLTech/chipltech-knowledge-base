# Agent 新 Session 上下文

## 适用场景

启动新的 Chipltech-Family Accelerator 相关 agent session（Hermes、Kilo、Claude 或其他 AI agent），需要快速恢复 DLC Ecosystem 项目背景。

## 快速启动 Prompt

```
你正在处理 Chipltech-Family Accelerator 相关任务。请先发现并回显实际的 <KNOWLEDGE_BASE_ROOT> 和 <SKILLS_ROOT>，不要假定固定部署目录。然后读取：

1. <KNOWLEDGE_BASE_ROOT>/CONTEXT.md — 术语表、组件关系
2. <KNOWLEDGE_BASE_ROOT>/README.md — 仓库导航
3. <KNOWLEDGE_BASE_ROOT>/foundation/dlc-ecosystem-overview.md — DLC Ecosystem 概述

然后根据任务类型，读取对应专题文档：

- 算子接入 / dispatch: pytorch-dlc-backend/operator-integration-guide.md
- dispatch fallback: operator-dispatch/enabled-kernels-dispatch.md
- 测试: testing/pytorch-test-framework-guide.md, testing/dlc-kernel-test-framework-guide.md
- 精度定位: precision-debugging/precision-debugging-overview.md
- vLLM: vllm-dlc/custom-op-integration-and-testing.md
- 调试: debugging-workflows/common-debug-commands.md
- runtime: runtime-debugging/runtime-troubleshooting.md
- 性能定位: runtime-debugging/performance-profiling.md
- 案例参考: case-studies/

关键约束：
- 不使用 TPU、国产 TPU 或 DLC-Family Accelerator，使用 Chipltech-Family Accelerator
- 不在知识库中用模型名建立一级或二级目录
- CPU fallback 是定位手段，不是生产修复
- DLC_CHECK_RESULT lambda name ≠ launch kernel name
- 知识库文件使用英文 kebab-case 文件名，中文正文
```

## 任务类型快速路由

| 任务 | 优先读取 |
|------|---------|
| 新增 DLC 算子 | `pytorch-dlc-backend/operator-integration-guide.md` |
| dispatch 配置 | `operator-dispatch/enabled-kernels-dispatch.md` |
| 算子精度定位 | `precision-debugging/precision-debugging-overview.md` → `model-site-dump-to-repro.md` |
| 编写测试 | `testing/pytorch-test-framework-guide.md` / `testing/dlc-kernel-test-framework-guide.md` |
| vLLM 集成 | `vllm-dlc/custom-op-integration-and-testing.md` |
| 环境配置 | `runtime-debugging/environment-setup-and-update.md` |
| 常见报错 | `runtime-debugging/common-error-log.md` |
| 调试命令 | `debugging-workflows/common-debug-commands.md` |
| 性能热点或回归 | `runtime-debugging/performance-profiling.md` → `case-studies/` 中最近的性能案例 |

## 关键路径速记

### dispatch fallback 操作
1. 找到 kernel 的 `DLC_CHECK_RESULT(lambda, ...)` lambda name
2. 修改 `/usr/local/chipltech/synapse/include/enabled_kernels.hpp` 中对应常量
3. `cd /work/pytorch && USE_CUDA=0 DEBUG=1 MAX_JOBS=32 python3 setup.py develop`

### 精度定位原则
1. CPU 是主要 oracle
2. 从端到端收敛到算子级：dump → compare → replay → single-variable
3. 最终交付 code-only repro（无需外部文件）

### 测试
- PyTorch 原生测试：`<PYTORCH_ROOT>/test/dlc_ops/test_dlc_ops.py`
- pytorch_test：`<DLC_CUSTOM_KERNEL_ROOT>/pytorch_test/run.py`

## 已知避免事项

- 不要把 DLC-Family Accelerator 称为 TPU
- 不要假设 CUDA device execution model
- 不要一次改多个 dispatch
- 不要把 CPU fallback 当生产修复
- 不要在知识库中建立以模型名为一级或二级的目录
