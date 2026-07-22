# chipltech-knowledge-base

Chipltech-Family Accelerator（DLC/TYD/HHP）的工程知识底座。

## 核心用途：AI 任务时的"项目大脑"

用 Kilo / Claude Code 等 AI 工具做 DLC 相关任务时，直接让 AI 先读这个仓库的 `CONTEXT.md`，它就能拿到：

- **统一的术语体系**：不会再出现 TPU、CUDA core、Tensor Core 等错误叫法混淆上下文，AI 全程使用正式术语，沟通不走样。
- **硬件架构和软件栈**：XYS、PGX、NWS 等硬件单元定义，DLC Ecosystem 组件关系，与 CUDA 的差异对比。
- **算子 dispatch 机制**：KernelDesc、enabled kernels、CPU/DLC fallback 等基础知识。
- **精度定位方法论**：CPU Reference、Hardware-Aware Reference、Model-Site Dump → pytorch_test 复现的完整流程。
- **debug 命令速查和 Runtime 排障**：常用调试命令、环境检查、异步错误定位等。
- **测试框架用法**：pytorch_test Framework、dlc_kernel_test Framework 的使用指南。
- **真实 case study**：跨模型、跨算子的精度问题和运行时故障复盘，AI 可直接参考相似案例。
- **常用 prompt 示例**：团队内部沉淀的高频、好用的业务 prompt 模板，便于同事直接复用。

相当于给 AI 配了一个"项目 onboarding 包"，不用每次重新解释项目背景。

## 持续积累

每个人做完一个任务，把过程中的优化经验、定位过程和踩坑总结写成文档，回写到这个仓库里——按问题域（精度、算子、Runtime、vLLM 等）组织，不按模型名组织。

仓库越用越厚，后面做同类问题越查越快，AI 能直接定位到相似 case study，减少重复排查。

## 仓库定位

- **这是什么**：Chipltech-Family Accelerator 的工程知识底座，保存项目背景知识、术语体系、调试经验、精度定位报告、算子接入说明、测试框架指南、环境配置说明、真实问题复盘和 agent 上下文材料。
- **这不是什么**：这不是 `skills.git`（agent 工作流和可执行方法论），也不是业务代码仓库（PyTorch DLC Backend、DLC_Custom_Kernel Repository、DLCSynapse 等）。
- **与 skills.git 的关系**：`skills.git` 保存 agent skills、工作流和流程化能力；`chipltech-knowledge-base` 保存 DLC 项目背景知识。两者分工协作。

## 适合谁读

- DLC Platform 新人，需要了解 DLC Ecosystem 的基础概念和软件栈。
- PyTorch DLC Backend 开发者，需要理解 ATen dispatch、KernelDesc、DLC_CHECK_RESULT 和 enabled kernels 的工作方式。
- DLC Custom Kernel 开发者，需要区分 DLC Custom Op、DLC Custom Kernel 和 DLC_Custom_Kernel Repository。
- vLLM DLC 适配开发者，需要查到 vLLM DLC Custom Op 的接入链路。
- 精度定位开发者，需要了解 CPU Reference、Hardware-Aware Reference、Model-Site Dump 和 pytorch_test 复现流程。
- AI agent，需要在新 session 中快速恢复 DLC 项目上下文。

## 新人阅读顺序

1. **[CONTEXT.md](CONTEXT.md)** — 先读术语表和组件边界，建立 DLC Ecosystem 的心智模型。
2. **[foundation/dlc-ecosystem-overview.md](foundation/dlc-ecosystem-overview.md)** — 理解 Chipltech-Family Accelerator 的硬件、软件栈和与 CUDA 的差异。
3. **[foundation/glossary.md](foundation/glossary.md)** — 查阅所有正式术语的定义和关系。
4. **[pytorch-dlc-backend/operator-integration-guide.md](pytorch-dlc-backend/operator-integration-guide.md)** — 了解 PyTorch DLC Backend 算子接入的完整流程。
5. **[testing/pytorch-test-framework-guide.md](testing/pytorch-test-framework-guide.md)** 和 **[testing/dlc-kernel-test-framework-guide.md](testing/dlc-kernel-test-framework-guide.md)** — 理解两个测试框架的用法和区别。
6. **[case-studies/paraformer-qkv-linear-precision.md](case-studies/paraformer-qkv-linear-precision.md)** — 读一个从模型 failure 到 pytorch_test 复现的完整案例。
7. 根据当前任务，按需查阅对应专题文档。

## Prompt 示例约定

- 定稿后或已经在实际工作中证明“常用、好用、值得复用”的 prompt 示例，应同步沉淀到 `prompt-examples/`。
- `prompt-examples/` 用来存放团队成员在日常工作中总结出的可直接复制使用的 prompt 模板，不放临时实验记录，不放一次性的聊天草稿。
- 如果后续同事又产出了新的高频好用 prompt，也统一放到 `chipltech-knowledge-base/prompt-examples/` 目录下维护。
- 这类文档的目标是“拿来即用”，所以优先保持结构清晰、占位符明确、少背景解释。
- `prompt-examples/dlc-env-setup-skill-usage.md` 用于直接调用 `dlc-env-setup` skill 做环境重建或修复。
- `prompt-examples/dlc-env-setup-fresh-container-validation.md` 用于在全新容器中验证 `dlc-env-setup` 的 Kilo 暴露、调用方式和功能闭环。
- [prompt-examples/bootstrap-git-from-configured-container.md](prompt-examples/bootstrap-git-from-configured-container.md) 用于从已验证容器安全提取 Git 配置和 SSH client key，为 Host 或其他容器补齐私有仓库 clone、fetch 与 push 能力。
- [prompt-examples/host-daily-image-to-model-validation.md](prompt-examples/host-daily-image-to-model-validation.md) 用于从 Host 固定每日镜像、创建持久容器、补齐 Git/SSH 和 DLC Ecosystem 环境，再完成可恢复的分层模型功能验证。
- [prompt-examples/new-model-validation-quickstart.md](prompt-examples/new-model-validation-quickstart.md) 是只需填写模型名称和目录的 runtime qualification 入口，不默认安装 skills 或构建 image。
- [prompt-examples/modelzoo-model-to-dlc-tyd-images.md](prompt-examples/modelzoo-model-to-dlc-tyd-images.md) 是模型名称、本地目录和 target 的薄入口；先完成 ordinary daily-base 功能/性能 gate，再驱动独立 DLC Chip/TYD Chip image delivery。
- [prompt-examples/vllm-dlc-fresh-image-to-model-adaptation.md](prompt-examples/vllm-dlc-fresh-image-to-model-adaptation.md) 用于每日空镜像先初始化 DLC Ecosystem，再对新模型做 vLLM-DLC / DLC Platform 适配分析的两阶段流程。
- [prompt-examples/vllm-dlc-model-adaptation.md](prompt-examples/vllm-dlc-model-adaptation.md) 用于一个明确模型的 vLLM-DLC Model Adaptation stable skill 只读分析。
- [prompt-examples/vllm-dlc-main-to-main-upgrade.md](prompt-examples/vllm-dlc-main-to-main-upgrade.md) 用于 exact upstream Main-to-Main Upgrade、恢复和全局影响分析。
- [prompt-examples/vllm-dlc-prefill-decode-separation.md](prompt-examples/vllm-dlc-prefill-decode-separation.md) 用于固定 Prefill/Decode 拓扑、KV Cache Transfer Contract、分层验证和 evidence 边界。

## 快速入口

### 精度定位入口

- [precision-debugging/precision-debugging-overview.md](precision-debugging/precision-debugging-overview.md) — 精度定位方法论总览。
- [precision-debugging/model-site-dump-to-repro.md](precision-debugging/model-site-dump-to-repro.md) — 从 Model-Site Dump 到 pytorch_test 复现的流程。
- [precision-debugging/token-divergence-and-moe-contract-debugging.md](precision-debugging/token-divergence-and-moe-contract-debugging.md) — vLLM 生成 token 分叉、MoE rank/expert 和 activation contract 定位。
- [case-studies/](case-studies/) — 跨模型、跨算子的真实精度问题复盘。

### 算子接入与测试入口

- [pytorch-dlc-backend/operator-integration-guide.md](pytorch-dlc-backend/operator-integration-guide.md) — PyTorch DLC Backend 算子接入。
- [operator-dispatch/enabled-kernels-dispatch.md](operator-dispatch/enabled-kernels-dispatch.md) — enabled kernels dispatch 机制和 CPU fallback 方法。
- [testing/](testing/) — pytorch_test Framework、PyTorch 原生测试、Static Shape Test、Dynamic Fuzz Test。
- [testing/arsenal-ci-and-blackbox-testing.md](testing/arsenal-ci-and-blackbox-testing.md) — Arsenal CI、vLLM benchmark、黑盒 HTTP 测试和 DLCCL hang 分析入口。
- [vllm-dlc/custom-op-integration-and-testing.md](vllm-dlc/custom-op-integration-and-testing.md) — vLLM DLC Custom Op 接入。

### Runtime / Debug 入口

- [runtime-debugging/runtime-troubleshooting.md](runtime-debugging/runtime-troubleshooting.md) — DLCSynapse、DLC Runtime、DLCsim、Real DLC Hardware 排障。
- [runtime-debugging/chipltech-smi-observability.md](runtime-debugging/chipltech-smi-observability.md) — `cltech_smi` 设备观测、debug 上传和 operational evidence 边界。
- [runtime-debugging/dlc-workstation-env-rebuild.md](runtime-debugging/dlc-workstation-env-rebuild.md) — DLC 工作站环境重建总流程，覆盖 repo discovery、branch 安全规则、构建顺序、PyTorch wheel 重建、`vllm` / `vllm-dlc` 安装与最终 smoke。
- [debugging-workflows/common-debug-commands.md](debugging-workflows/common-debug-commands.md) — 常用调试命令速查。
- [debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md](debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md) — PyTorch 2.5.0 wheel 与本地 `vllm` / `vllm-dlc` editable install 的 build preflight 清单。
- [debugging-workflows/post-install-runtime-smoke.md](debugging-workflows/post-install-runtime-smoke.md) — 安装后的最小 runtime smoke 和失败回退路径。

### vLLM-DLC workflow

- [vllm-dlc/prefill-decode-separation.md](vllm-dlc/prefill-decode-separation.md) — Prefill/Decode Separation 的拓扑、MooncakeDLCConnector、KV Cache transfer、验证阶梯和 claim boundaries。
- [prompt-examples/vllm-dlc-prefill-decode-separation.md](prompt-examples/vllm-dlc-prefill-decode-separation.md) — 配合已安装的 `pd-separation` skill 或 skills checkout，可直接填写并执行 PD 分离部署或诊断的 prompt。
- [vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md](vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md) — 本地模型优先、ModelZoo 可选只读 reference、ordinary daily-base runtime qualification 和独立 DLC/TYD delivery 状态机。
- [prompt-examples/modelzoo-model-to-dlc-tyd-images.md](prompt-examples/modelzoo-model-to-dlc-tyd-images.md) — 填写模型名称、本地目录和 target 的 runtime-first image delivery prompt。
- [vllm-dlc/model-adaptation-and-main-to-main-decisions.md](vllm-dlc/model-adaptation-and-main-to-main-decisions.md) — Model Adaptation 与 Main-to-Main Upgrade 的稳定决策、证据分类和当前只读边界。
- [prompt-examples/vllm-dlc-fresh-image-to-model-adaptation.md](prompt-examples/vllm-dlc-fresh-image-to-model-adaptation.md) — 每日空镜像初始化后接新模型适配的两阶段 prompt。
- [prompt-examples/vllm-dlc-model-adaptation.md](prompt-examples/vllm-dlc-model-adaptation.md) — Model Adaptation 可复用 prompt。
- [prompt-examples/vllm-dlc-main-to-main-upgrade.md](prompt-examples/vllm-dlc-main-to-main-upgrade.md) — Main-to-Main Upgrade 可复用 prompt。

如需让 agent 自动执行 repo discovery、阶段化重建和 smoke，可配合 `/work/skills/skills/engineering/dlc-env-setup/` 中的 `dlc-env-setup` skill 使用。

## Agent 使用方式

在新 session 中处理 DLC 相关任务时，建议在 prompt 中固定要求：

1. 先读取 `chipltech-knowledge-base` 的 `CONTEXT.md` 和 `README.md`。
2. 根据任务类型读取对应专题文档。
3. 使用 `CONTEXT.md` 中的正式术语。
4. 任务结束后，将可复用经验回写到对应专题或 case study。

## 目录说明

```
chipltech-knowledge-base/
├── README.md                   # 本文件
├── CONTEXT.md                  # 领域词汇表、术语规范、组件关系和写作规则
├── migration-inventory.md      # 原始资料迁移状态追踪
├── foundation/                 # DLC Ecosystem 基础概念、硬件与软件栈
├── pytorch-dlc-backend/        # PyTorch DLC Backend、ATen dispatch、KernelDesc
├── operator-dispatch/          # 算子 dispatch、CPU/DLC/BOTH fallback
├── precision-debugging/        # CPU Reference、Hardware-Aware Reference、Model-Site Dump
├── testing/                    # pytorch_test Framework、Static Shape Test、Dynamic Fuzz Test
├── runtime-debugging/          # DLCSynapse、DLC Runtime、DLCsim、Real DLC Hardware
├── vllm-dlc/                   # vLLM DLC Custom Op、DLC Attention Backend、KV cache
├── debugging-workflows/        # VSCode 调试、日志分析、trace 分析
├── case-studies/               # 跨模型的真实问题复盘
├── prompt-examples/            # 团队沉淀的常用、好用、可直接复用的 prompt 示例
├── agent-context/              # agent 使用的浓缩上下文和模板
└── assets/                     # 图片等静态资产
```

## 持续维护原则

- **任务结束必须反哺知识库**：每个 session 产生的可复用经验应回写到对应专题或 case study。
- **按问题域组织，不按模型组织**：模型名可以出现在文档标题、文件名或正文中，但不作为一级或二级目录边界。
- **术语以 CONTEXT.md 为准**：所有文档引用统一术语，不重复定义。
- **区分事实、经验和未验证假设**：避免把经验性判断当成硬性结论。
- **敏感信息必须移除**：API key、密码、token、个人敏感信息不得写入。
- **常用 prompt 示例要沉淀到 `prompt-examples/`**：如果某个 prompt 在实际工作中已经证明常用、好用、值得复用，就放到 `chipltech-knowledge-base/prompt-examples/`，不要只留在个人 plans 或聊天记录里。
