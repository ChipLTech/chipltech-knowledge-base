# chipltech-knowledge-base

Chipltech-Family Accelerator（DLC/TYD/HHP）的工程知识底座。

## 最短入口

在 Kilo Code 或当前能够加载团队 Skills 的 Harness 中，直接发送：

```text
请使用 `chipltech-context` 路由并执行这个 Chipltech-Family Accelerator 工程任务。

目标：<一句话描述>
已有材料：<路径或“请自动发现”>
```

`chipltech-context` 会读取本仓库的 `CONTEXT.md`、`README.md` 和统一能力入口，选择正式 Prompt、业务 Contract 与 owning Skill。当前 Harness 已发现时直接加载；否则只读发现 `<SKILLS_ROOT>` 下的正式 Skill 包；两者都不存在时返回 `blocked_missing_contract`。Skill discovery 只证明 publication/install 可用，不是 runtime Evidence。完全不知道该选哪个能力时，查看 [全部已支持能力 Quickstart](prompt-examples/all-supported-capabilities-quickstart.md)。

Hermes 是可选执行器，不是知识库、Prompt 或 owning Skill 的前置依赖；未选择 Hermes 时，直接使用当前 Kilo/Harness。

## 核心用途：AI 任务时的"项目大脑"

用 Kilo / Claude Code 等 AI 工具做 DLC 相关任务时，直接让 AI 先读这个仓库的 `CONTEXT.md`，它就能拿到：

- **统一的术语体系**：不会再出现 TPU、CUDA core、Tensor Core 等错误叫法混淆上下文，AI 全程使用正式术语，沟通不走样。
- **硬件架构和软件栈**：XYS、PGX、NWS 等硬件单元定义，DLC Ecosystem 组件关系，与 CUDA 的差异对比。
- **算子 dispatch 机制**：KernelDesc、enabled kernels、CPU/DLC fallback 等基础知识。
- **精度定位方法论**：CPU Reference、Hardware-Aware Reference、Model-Site Dump → pytorch_test 复现的完整流程。
- **debug 命令速查和 Runtime 排障**：常用调试命令、环境检查、异步错误定位等。
- **测试框架用法**：pytorch_test Framework、dlc_kernel_test Framework 的使用指南。
- **机器可验证合同**：distributed collective qualification、communicator-owned topology/payload selection、identity freshness 和 fail-closed 状态边界。
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
- **vLLM plugin 命名**：当前仓库和 Distribution 使用 `vllm-cl`，Python import 使用 `vllm_cl`，正式 Git URL 为 `git@github.com:ChipLTech/vllm-cl.git`。旧 `vllm-dlc` / `vllm_dlc` 仅用于解释改名前历史证据。
- **知识库边界**：本仓库提供领域定义、已记录流程和历史 case evidence；仓库文字不能证明当前 Host、package、模型、DLC Runtime、transport 或 Real DLC Hardware 状态。
- **Skills 边界**：Prompt/Contract 描述任务规则，owning Skill 负责可执行工作流、授权边界、停止语义和 Claim Boundary；Skill 被发现或加载只证明能力可用，不证明任务已经执行。
- **Evidence 边界**：当前业务/代码 workspace 和 task-owned artifact 目录承载实际执行与 Evidence。结论应区分 `direct repository evidence`、`runtime observation`、`inference` 和 `missing evidence`。

## 适合谁读

- DLC Platform 新人，需要了解 DLC Ecosystem 的基础概念和软件栈。
- PyTorch DLC Backend 开发者，需要理解 ATen dispatch、KernelDesc、DLC_CHECK_RESULT 和 enabled kernels 的工作方式。
- DLC Custom Kernel 开发者，需要区分 DLC Custom Op、DLC Custom Kernel 和 DLC_Custom_Kernel Repository。
- vLLM-CL 适配开发者，需要查到 vLLM-CL Custom Op 的接入链路。
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
- [全部已支持能力 Quickstart](prompt-examples/all-supported-capabilities-quickstart.md) 是统一能力入口：按目标复制最短调用，再由当前 Kilo 或其他合格 Harness 读取正式 Prompt、Contract 和 owning Skill。
- [业务套餐 Quickstart](prompt-examples/dlc-business-skill-examples-quickstart.md) 提供常见 DLC Platform 任务的薄入口；详细执行规则见其链接的正式 Prompt 和 owning Skill。
- [Hermes Chipltech Engineering Quickstart](prompt-examples/hermes-chipltech-engineering-quickstart.md) 仅用于明确选择 Hermes 时的可选 profile 接入与验收，不影响 Kilo 或其他合格 Harness 直接执行。
- 其他 Prompt 不在 README 逐项维护，以统一能力入口为准。

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
- [vllm-cl/custom-op-integration-and-testing.md](vllm-cl/custom-op-integration-and-testing.md) — vLLM-CL Custom Op 接入。

### 技术沟通入口

- [foundation/technical-delivery-summary.md](foundation/technical-delivery-summary.md) — 将已完成的实现、跨仓接入、验证或发布工作提炼为一句能力总结，并保持交付状态和 Evidence 层级。
- [debugging-workflows/technical-issue-summary.md](debugging-workflows/technical-issue-summary.md) — 将已闭合的复杂故障证据压缩成 Sprint、Issue、owner 确认或 handoff 说明，并保持首次异常边界、量化闭合和 Claim Boundary。

### Runtime / Debug 入口

- [runtime-debugging/dlc-runtime-api-reference.md](runtime-debugging/dlc-runtime-api-reference.md) — 基于固定镜像、源码 commit 和动态库 ABI 的 DLC Runtime、DLCSynapse Core、KernelDesc 版本化接口参考，包含生命周期、同步语义、DLC Runtime 完整函数族、支持矩阵、Success stub、ABI 缺失和已知缺陷。
- [runtime-debugging/runtime-troubleshooting.md](runtime-debugging/runtime-troubleshooting.md) — DLCSynapse、DLC Runtime、DLCsim、Real DLC Hardware 排障。
- [runtime-debugging/performance-profiling.md](runtime-debugging/performance-profiling.md) — Chipltech-Family Accelerator 性能 profiling、分层热点定位、异步计时边界和性能 Claim Boundary。
- [runtime-debugging/chipltech-smi-observability.md](runtime-debugging/chipltech-smi-observability.md) — `chipltech_smi_lib` / `cltech_smi` 四阶段 SMI Observation Envelope、设备/HBM/进程 debug、上传和 operational evidence 边界。
- [runtime-debugging/dlc-workstation-env-rebuild.md](runtime-debugging/dlc-workstation-env-rebuild.md) — DLC 工作站环境重建总流程，覆盖 repo discovery、branch 安全规则、构建顺序、PyTorch wheel 重建、`vllm` / `vllm-cl` 安装与最终 smoke。
- [debugging-workflows/common-debug-commands.md](debugging-workflows/common-debug-commands.md) — 常用调试命令速查。
- [debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md](debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md) — PyTorch 2.5.0 wheel 与本地 `vllm` / `vllm-cl` editable install 的 build preflight 清单。
- [debugging-workflows/post-install-runtime-smoke.md](debugging-workflows/post-install-runtime-smoke.md) — 安装后的最小 runtime smoke 和失败回退路径。
- [runtime-debugging/stack-preflight-and-cold-completion.md](runtime-debugging/stack-preflight-and-cold-completion.md) — 不修改运行库的 exact stack 静态身份门禁与 fresh-process cold first-compute 组合资格检查。
- [case-studies/vllm-async-speculative-decode-launch-failure.md](case-studies/vllm-async-speculative-decode-launch-failure.md) — 从部分 token 后 HTTP 500 收敛到 async MTP 4-token Graph 执行窗口，并区分原 decode failure 与诊断配置引入的新 stall。
- [prompt-examples/vllm-async-launch-failure-localization.md](prompt-examples/vllm-async-launch-failure-localization.md) — 可直接复用的异步 launch failure 定位 prompt。
- [case-studies/vllm-attention-duplicate-kv-cache-update.md](case-studies/vllm-attention-duplicate-kv-cache-update.md) — 从 full-attention layer 收敛到跨层重复 KV cache update 的历史性能案例，并明确 source identity 与正式 benchmark 缺口。
- [case-studies/vllm-hybrid-kv-cache-strided-output.md](case-studies/vllm-hybrid-kv-cache-strided-output.md) — 从单次 `reshape_and_cache` 内的 copy/slice 事件收敛到 hybrid Attention/Mamba non-contiguous destination view 与 output adaptation，并区分历史观察、机制推断和候选 strided-output 设计。
- [case-studies/vllm-fused-moe-schema-kernel-abi-boundary.md](case-studies/vllm-fused-moe-schema-kernel-abi-boundary.md) — 从 MiniMax main 三仓交付提炼 Public Operator Schema、KernelDesc Descriptor ABI、冻结 DLC Custom Kernel Entry ABI、跨仓 owner 和 exact-artifact evidence 边界。
- [case-studies/qknorm-topology-aware-allreduce-selection.md](case-studies/qknorm-topology-aware-allreduce-selection.md) — 从 QKNorm 三仓闭环提炼 communicator-owned topology/payload selection、Verified Collective Fallback、稳定 strategy ABI 和边界测试方法。
- [case-studies/host-api22-fullstack-main-to-main-update.md](case-studies/host-api22-fullstack-main-to-main-update.md) — Host API 21→22 全栈主线更新 + TP4/EP 验证，提炼 binary identity、editable source 确认、Git bundle 中转与清理等可复用经验。
- [case-studies/model-adaptation-analysis-report-production.md](case-studies/model-adaptation-analysis-report-production.md) — 从 Hy3 GPTQ-Int4 报告复盘提炼 Evidence Ledger、Decision Summary/Technical Attachment 分层、真实 kernel launch name 核验和性能口径边界。
- [prompt-examples/vllm-performance-hotspot-localization.md](prompt-examples/vllm-performance-hotspot-localization.md) — 可直接复用的 vLLM 性能热点分层定位 prompt。
- [debugging-workflows/synapse-log-and-kernel-summary-workflow.md](debugging-workflows/synapse-log-and-kernel-summary-workflow.md) — 从 DLCSynapse `.ansi`/`.log` 按 `tool.py` 的 1400 MHz 口径只导出算子 CSV。

### vLLM-CL workflow

- [vllm-cl/prefill-decode-separation.md](vllm-cl/prefill-decode-separation.md) — Prefill/Decode Separation 的拓扑、MooncakeDLCConnector、KV Cache transfer、验证阶梯和 claim boundaries。
- [vllm-cl/prefill-decode-separation-practical-guide.md](vllm-cl/prefill-decode-separation-practical-guide.md) — 可填写的 X-P / Y-D 通用实操配置指南，覆盖端口矩阵、启动顺序、请求关联、第二条请求 profiling、验收和止损。
- [prompt-examples/vllm-cl-prefill-decode-separation.md](prompt-examples/vllm-cl-prefill-decode-separation.md) — 配合已安装的 `pd-separation` skill 或 skills checkout，可直接填写并执行 PD 分离部署或诊断的 prompt。
- [case-studies/vllm-cl-pd-transport-and-lyp-recovery.md](case-studies/vllm-cl-pd-transport-and-lyp-recovery.md) — Transport Qualification Gate、LYP group 修复、direct DLCCL 和 Site Recovery Contract 的真实复盘。
- [vllm-cl/modelzoo-driven-dlc-tyd-image-contract.md](vllm-cl/modelzoo-driven-dlc-tyd-image-contract.md) — 本地模型优先、ModelZoo 可选只读 reference、ordinary daily-base runtime qualification 和独立 DLC/TYD delivery 状态机。
- [prompt-examples/modelzoo-model-to-dlc-tyd-images.md](prompt-examples/modelzoo-model-to-dlc-tyd-images.md) — 填写模型名称、本地目录和 target 的 runtime-first image delivery prompt。
- [vllm-cl/model-adaptation-and-main-to-main-decisions.md](vllm-cl/model-adaptation-and-main-to-main-decisions.md) — Model Adaptation 与 Main-to-Main Upgrade 的稳定决策、证据分类和当前只读边界。
- [vllm-cl/distributed-collective-qualification.md](vllm-cl/distributed-collective-qualification.md) — supporting reference；区分 native DLC_CL、PyTorch ProcessGroup、vLLM communicator、模型 route 与 MoE/custom-kernel ABI，并说明 frozen v1、exact v2、`vllm-cl-collective-selection/v1`、deterministic clock seam 和 controlled fixture 的证据边界。
- [prompt-examples/vllm-cl-fresh-image-to-model-adaptation.md](prompt-examples/vllm-cl-fresh-image-to-model-adaptation.md) — 每日空镜像初始化后接新模型适配的两阶段 prompt。
- [prompt-examples/vllm-cl-model-adaptation.md](prompt-examples/vllm-cl-model-adaptation.md) — Model Adaptation 可复用 prompt。
- [prompt-examples/vllm-cl-model-adaptation-analysis-summary.md](prompt-examples/vllm-cl-model-adaptation-analysis-summary.md) — 将已收集的模型适配证据整理为面向决策者的分析摘要与技术附件。
- [prompt-examples/vllm-cl-main-to-main-upgrade.md](prompt-examples/vllm-cl-main-to-main-upgrade.md) — Main-to-Main Upgrade 可复用 prompt。

如需让 agent 自动执行 repo discovery、阶段化重建和 smoke，可配合当前 Harness 已发现的 `dlc-env-setup` Skill，或 `<SKILLS_ROOT>/skills/engineering/dlc-env-setup/` 中的源包使用。

## Agent 使用方式

在新 session 中处理 Chipltech 工程任务时：

1. 先通过 `chipltech-context` 读取本仓库的 `CONTEXT.md` 和 `README.md`，采用正式术语与组件边界。
2. 任务类型或 owner 不明确时，读取统一能力入口，选择链接的正式 Prompt、Contract 或 Runbook；不要只根据能力目录直接执行。
3. 在当前 Harness 中实际加载最窄的 owning Skill，核对其 scope、授权要求、停止语义、完成条件和 Claim Boundary。
4. 在实际业务/代码 workspace 中执行，并将 runtime Evidence 写入 task-owned artifact 目录；不要把知识库文字当成当前运行证据。
5. 只有在任务 Evidence 和 Claim Boundary 已闭合、且用户明确授权知识库贡献时，才按问题域回写专题或 case study。

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
├── vllm-cl/                   # vLLM-CL Custom Op、DLC Attention Backend、KV cache
├── debugging-workflows/        # VSCode 调试、日志分析、trace 分析
├── case-studies/               # 跨模型的真实问题复盘
├── prompt-examples/            # 团队沉淀的常用、好用、可直接复用的 prompt 示例
├── agent-context/              # agent 使用的浓缩上下文和模板
└── assets/                     # 图片等静态资产
```

## 持续维护原则

- **闭环经验应在授权后反哺知识库**：任务 Evidence 和 Claim Boundary 闭合后，通过单独授权的贡献流程，将可复用经验回写到对应专题或 case study。
- **按问题域组织，不按模型组织**：模型名可以出现在文档标题、文件名或正文中，但不作为一级或二级目录边界。
- **术语以 CONTEXT.md 为准**：所有文档引用统一术语，不重复定义。
- **区分事实、经验和未验证假设**：避免把经验性判断当成硬性结论。
- **敏感信息必须移除**：API key、密码、token、个人敏感信息不得写入。
- **常用 prompt 示例要沉淀到 `prompt-examples/`**：如果某个 prompt 在实际工作中已经证明常用、好用、值得复用，就放到 `chipltech-knowledge-base/prompt-examples/`，不要只留在个人 plans 或聊天记录里。

## 外部受限参考治理

来源按以下等级管理：Chipltech-owned authoritative source、独立编写的 DLC-native specification、许可允许当前用途的 public/permissive external reference、external restricted reference，以及 unknown/unreviewed source。来源等级描述可用边界，不代表对内容质量或法律状态的推断。

- External restricted reference 只可作为受控 review input，不得复制、翻译、机械改写或格式转换后写入知识文档、Prompt、模板、脚本、schema、Skill、agent instruction 或 bundled reference。
- 知识库只保存独立编写的 DLC-native 结论和 specification。维护者应在 skills 仓库的 `config/restricted-reference-governance.json` 登记 source locator、revision 或 SHA-256、license metadata、允许和禁止用途、reviewer、review date 与 disposition。
- 缺失或未解决的来源、许可、角色隔离或发布状态必须返回 `blocked_legal_boundary`；调查可以保留在内部 governance record 中，但不得继续形成受影响的实现或对外发布资产。
- 内部 governance record 可以为审查而指明受限来源；对外发布内容不得包含受限表达或 bundled source material。外部固定性能收益不得转述，除非有独立 DLC evidence 和 publication approval。
- 名称相似、一般工程词语，以及 `torch_npu`、`msprof`、`npugraph_ex`、`PA_NZ`、`AIC`、`AIV` 等命中只触发 review，不构成复制、侵权、许可或来源结论。判断优先使用 provenance、exact path/hash、license metadata、package role 和实际 distribution status。
- 发布审查以实际 packaging/install manifest 为准。Governance ADR、source/license register、调查记录和 negative fixture 不是 execution asset；但一旦被实际安装或打包为 script、template、schema、Skill/agent instruction 或 bundled reference，就按该发布角色扫描。
