# Chipltech 工程已支持能力傻瓜式调用总览

## 这份文档解决什么问题

这份文档是 `prompt-examples/` 的统一能力入口。你不需要逐个打开 Prompt 示例，也不需要先理解 Skill、Contract、handoff、Evidence 或 Claim Boundary。

使用方式：

1. 在下方找到最接近的任务。
2. 复制对应的“直接这样说”。
3. 只替换 `<...>` 中你已经知道的内容；不知道的保留“请自动发现”。
4. 把 Prompt 发给当前能够加载团队 Skills 的 Agent/Harness。Kilo Code 可直接读取正式 Prompt、知识库和 owning Skill 后执行；也可以使用已安装的 `/skill-name` wrapper 显式调用。
5. Hermes 是可选执行器，不是其他能力的前置依赖。只有明确选择 Hermes 时，才需要安装和验收 `chipltech-engineering` profile；Hermes 缺失或故障不得阻断 Kilo 或其他合格 Harness 直接调用 owning Skill。

本页只提供薄入口和能力导航。正式执行规则以链接的详细 Prompt、Contract、Runbook、知识专题和当前 owning Skill 为准。本页不会复制维护第二套状态机。

## 完全不知道该选哪个

直接复制：

```markdown
请使用 `chipltech-context` 帮我路由并执行这个 Chipltech-Family Accelerator 工程任务。

我要处理：<一句话描述目标或问题>
已有材料：<模型、日志、错误文档、dump、代码仓库或实验目录；没有就写“请自动发现”>

请先读取当前知识库的 `CONTEXT.md`、`README.md` 和 `prompt-examples/all-supported-capabilities-quickstart.md`，选择最匹配的正式 Prompt、业务 Contract 和 owning Skill。自动补全本机可只读发现的信息，在当前授权范围内持续执行到明确 terminal state，不要只给计划。

结论必须区分 repository evidence、runtime observation、inference 和 missing evidence。需要安装、构建、Git ref 切换、device execution、task-owned KILL、reset、LYP repair、驱动维护、重启、镜像导出/推送或清理非任务资源时，列出精确动作、目标和影响，并只请求最小增量授权。
```

更详细的九类日常业务套餐见：

- [DLC Platform 业务套餐傻瓜版调用说明](dlc-business-skill-examples-quickstart.md)
- [DLC Platform 业务 Skill 使用示例](dlc-business-skill-examples.md)

## 能力速查表

| 你要做的事 | 最少提供 | 主要入口 | Owning Skill / 角色 |
| --- | --- | --- | --- |
| 新模型只做运行资格验证 | 模型名、绝对目录 | [新模型验证 Quickstart](new-model-validation-quickstart.md) | `modelzoo-image-validation`，委托 `model-adaptation`、`dlc-env-setup` |
| 从每日空镜像初始化环境并做新模型适配 | 模型名、绝对目录 | [每日空镜像到新模型适配](vllm-dlc-fresh-image-to-model-adaptation.md) | Stage 0/2 `model-adaptation`，Stage 1 `dlc-env-setup` |
| 模型兼容性/加载/Serving 适配分析 | 模型名、绝对目录 | [vLLM-DLC Model Adaptation](vllm-dlc-model-adaptation.md) | `model-adaptation` |
| 模型资格通过后交付 DLC/TYD Images | 模型名、绝对目录 | [ModelZoo 模型到 DLC/TYD Images](modelzoo-model-to-dlc-tyd-images.md) | `modelzoo-image-validation` |
| 按 Host Daily Image Runbook 做完整环境和模型验证 | 模型名、绝对目录 | [Host Daily Image 到模型验证](host-daily-image-to-model-validation.md) | Runbook 编排多个 Skills |
| 重建或修复 DLC Ecosystem 工作站环境 | 模式、搜索根或现有材料 | [DLC Env Setup Skill 使用模板](dlc-env-setup-skill-usage.md) | `dlc-env-setup` |
| 新容器缺仓库/CMake，需要先 bootstrap | 目标根、允许动作、版本策略 | [DLC Ecosystem 环境 Bootstrap](dlc-env-setup-environment-bootstrap.md) | `dlc-env-setup` 的显式授权扩展 |
| 验证新容器是否正确安装和识别 `dlc-env-setup` | 容器/工作站 | [新容器 Skill 验收](dlc-env-setup-fresh-container-validation.md) | Kilo 集成验收，验证 `dlc-env-setup` |
| unsupported operator 或不知道第一失败点 | 日志/错误/运行材料 | [业务套餐二](dlc-business-skill-examples-quickstart.md#套餐二模型卡住或遇到-unsupported-operator) | `diagnosing-bugs` |
| 输出、logits 或 token 不一致 | 日志/output/dump/compare | [业务套餐三](dlc-business-skill-examples-quickstart.md#套餐三输出或-token-不一致) | `diagnosing-bugs` |
| Model-Site Dump 转 pytorch_test Framework 复现 | dump 或实验目录 | [业务套餐四](dlc-business-skill-examples-quickstart.md#套餐四把真实-dump-变成-pytorch_test-framework-复现) | `diagnosing-bugs` → `tdd` |
| 验证 dispatch/fallback 是否真的生效 | diff、日志、op/kernel | [业务套餐五](dlc-business-skill-examples-quickstart.md#套餐五确认-dispatch-或-fallback-是否生效) | `diagnosing-bugs` |
| 从 DLCSynapse log 进入 replay | `syn_*.ansi` 或运行材料 | [业务套餐六](dlc-business-skill-examples-quickstart.md#套餐六从-dlcsynapse-log-进入-replay) | `diagnosing-bugs` |
| DLC Runtime 报错、hang、worker 退出或进程残留 | 日志、命令、说明 | [业务套餐七](dlc-business-skill-examples-quickstart.md#套餐七dlc-runtime-报错hang-或进程残留) | `diagnosing-bugs` |
| vLLM 部分 token 后异步 launch failure / HTTP 500 | 启动命令、固定请求、日志 | [异步 Launch Failure 定位](vllm-async-launch-failure-localization.md) | `diagnosing-bugs`，必要时 `model-adaptation` |
| vLLM TTFT/TPOT/ITL/吞吐性能热点 | serve command、固定 workload、baseline | [性能热点分层定位](vllm-performance-hotspot-localization.md) | `diagnosing-bugs` |
| 模型 distributed/MoE route 资格边界 | 模型/deployment identity、active route inventory | [Distributed Collective Qualification](../vllm-dlc/distributed-collective-qualification.md)（supporting reference，不是独立 capability entrypoint） | `model-adaptation` |
| Prefill/Decode Separation 部署或诊断 | 模型名、绝对目录 | [Prefill/Decode Separation](vllm-dlc-prefill-decode-separation.md) | `pd-separation` |
| 精确 upstream vLLM Main-to-Main Upgrade | 目标 full SHA、仓库/evidence | [Main-to-Main Upgrade](vllm-dlc-main-to-main-upgrade.md) | `main-to-main-upgrade` |
| 可选：固定 Hermes ModelZoo runner 做单模型验证 | 模型名、模型路径 | [Hermes ModelZoo 单模型验证](hermes-modelzoo-batch-validation.md) | 仅限匹配的固定环境；通用资格使用 `modelzoo-image-validation` |
| 长任务换 Session 或交接同事 | 当前 Session/材料 | [业务套餐八](dlc-business-skill-examples-quickstart.md#套餐八长任务换-session-或交接同事) | `handoff` |
| 把可复用经验写回知识库 | 报告、日志、测试或实验目录 | [业务套餐九](dlc-business-skill-examples-quickstart.md#套餐九把本次经验写回知识库) | 按问题域更新，必要时加载对应 Skill |
| 为 Host/容器补齐 Git/SSH 私有仓库能力 | 源容器、目标、仓库 URL、授权 | [Git/SSH 安全赋能](bootstrap-git-from-configured-container.md) | 安全操作 Contract，无单一业务 Skill |
| 可选：验证 Hermes 知识库、Memory 和 Skills 接入 | Hermes 配置和两个仓库 | [Hermes Chipltech Engineering Quickstart](hermes-chipltech-engineering-quickstart.md) | 可选 Hermes profile 验收；不阻断其他入口 |

## 1. 新模型只做运行资格验证

什么时候用：只有模型名和本地目录，想确认模型能否在 DLC Platform 上完成 runtime-first 功能验证；默认不交付镜像。

直接这样说：

```markdown
请验证模型 `<MODEL_NAME>` 能否在 DLC Platform 上运行。
模型目录：`<ABSOLUTE_LOCAL_MODEL_PATH>`

请参考 `prompt-examples/new-model-validation-quickstart.md`，自动发现环境、source、ordinary daily base、设备、deployment profile、测试输入和 artifact 路径，并使用 `modelzoo-image-validation` 编排 `model-adaptation` 与 `dlc-env-setup`。在当前授权范围内持续执行到 terminal state。
```

详细规则：[新模型验证 Quickstart](new-model-validation-quickstart.md)

关键边界：默认 qualification-only；没有合格 `environment_handoff/v1` 不进入 real-weight functional/benchmark；单次 benchmark 不称稳定 baseline。

## 2. 每日空镜像初始化后做新模型适配

什么时候用：同事示例中的场景。希望从一个每日空镜像开始，先完成 DLC Ecosystem 环境，再处理明确的新模型适配。

直接这样说：

```markdown
参考 `prompt-examples/vllm-dlc-fresh-image-to-model-adaptation.md`，在每日空镜像环境下完成模型 `<MODEL_NAME>` 的 vLLM-DLC / DLC Platform 适配。

模型目录：`<ABSOLUTE_LOCAL_MODEL_PATH>`

请严格按三个阶段执行：先用 `model-adaptation` 做只读 capability matrix 和 pre-handoff deployment profile；再用 `dlc-env-setup` 完成环境、C1a/C1b 和 `environment_handoff/v1`；handoff 合格后再做 device-backed adaptation。其余可发现字段自动发现，只在下一步确实需要时请求最小授权。
```

详细规则：[每日空镜像到新模型 vLLM-DLC 适配 Prompt](vllm-dlc-fresh-image-to-model-adaptation.md)

关键边界：环境通过不等于模型 acceptance；Stage 2 不直接修改 `vllm-dlc`；短 Prompt smoke 不能外推长上下文、Chunked Prefill 或完整 Real DLC Hardware acceptance。

## 3. 单模型兼容性或 Serving 适配分析

什么时候用：模型已经明确，希望分析 Attention、MLA、MoE、quantization、multimodal、MTP 或 distributed compatibility，而不是做 upstream 全局对齐。

直接这样说：

```markdown
请使用 `model-adaptation` 分析模型 `<MODEL_NAME>` 的 vLLM-DLC / DLC Platform 兼容性。
模型目录：`<ABSOLUTE_LOCAL_MODEL_PATH>`

参考 `prompt-examples/vllm-dlc-model-adaptation.md`。先完成只读 compatibility matrix 和 deployment profile；只有匹配的 handoff、qualification execution 和 device authorization 齐全时，才继续 device-backed validation。
```

详细规则：[vLLM-DLC Model Adaptation 可复用 Prompt](vllm-dlc-model-adaptation.md)

关键边界：不负责 DLC Ecosystem 重建，不做 Main-to-Main Upgrade，不直接修改 `vllm-dlc`；未执行的 real weights、DLC Runtime dispatch 和 Real DLC Hardware 项必须写 `not_verified`。

## 4. 模型验证后交付 DLC/TYD Images

什么时候用：不仅要验证模型，还要在功能和 benchmark gates 通过后交付 DLC Chip image，并按条件独立交付 TYD Chip image。

直接这样说：

```markdown
请验证并交付模型 `<MODEL_NAME>` 的 DLC/TYD Images。
模型目录：`<ABSOLUTE_LOCAL_MODEL_PATH>`

参考 `prompt-examples/modelzoo-model-to-dlc-tyd-images.md`，使用 `modelzoo-image-validation`。先在 ordinary daily base 完成 runtime qualification、真实权重功能和 benchmark gates，再提出并执行已授权的 DLC image delivery；TYD 必须作为独立 rebuild 和独立验证处理。
```

详细规则：[ModelZoo 模型到 DLC/TYD Images Prompt](modelzoo-model-to-dlc-tyd-images.md)

关键边界：target 不等于 build/export/push 授权；模型权重不得进入 image；DLC 与 TYD 独立判定，TYD 失败不回写 DLC 已完成状态。

## 5. 按 Host Daily Image Runbook 做完整验证

什么时候用：需要明确控制 ordinary daily base、task-owned container、C0-C5、handoff、真实权重、benchmark、failure epoch 和 cleanup。

直接这样说：

```markdown
请按 `prompt-examples/host-daily-image-to-model-validation.md` 验证模型 `<MODEL_NAME>`。
模型目录：`<ABSOLUTE_LOCAL_MODEL_PATH>`

自动生成 Runtime Qualification Contract，选择 immutable ordinary daily base，并在当前授权范围内执行到 terminal state。每个 required C1a/C1b/collective 必须有明确结果，handoff 合格后才能进入模型 load、functional 和 benchmark。
```

详细规则：[Host Daily Image 到模型验证 Runbook](host-daily-image-to-model-validation.md)

关键边界：Runbook 拥有 Host/container mechanics，不独立拥有最终状态和 image-delivery 语义；HTTP 200、import、allocation、weight load 或非空输出不自动提升到更高验收状态。

## 6. 重建或修复 DLC Ecosystem 环境

什么时候用：工作站或容器的 PyTorch 2.5.0 wheel、DLC Platform、DLC_Custom_Kernel Repository、vLLM 或 vLLM-DLC 环境损坏或需要阶段化重建。

直接这样说：

```markdown
请使用 `dlc-env-setup` 修复 DLC Ecosystem 环境。

模式：<全量重建 / 从某阶段开始 / 只修 vLLM-vLLM-DLC>
已有仓库或错误材料：<路径；不知道写“请自动发现”>

参考 `prompt-examples/dlc-env-setup-skill-usage.md`。先只读发现仓库、Git identity、依赖健康和最小 rebuild 起点；任何 ref 切换、安装、构建、`/usr/local` 修改或 Host maintenance 都按最小授权执行。
```

详细规则：[DLC Env Setup Skill 使用模板](dlc-env-setup-skill-usage.md)

关键边界：dirty worktree 不切 ref、不 stash、不覆盖；partial rebuild 必须先证明上游依赖健康；package/import smoke 不等于模型或 Real DLC Hardware acceptance。

## 7. 新环境缺仓库或 CMake，先做 Bootstrap

什么时候用：新机器/容器连 stable `dlc-env-setup` 所需的仓库、合格 CMake 或 repair source 都没有。

直接这样说：

```markdown
请参考 `prompt-examples/dlc-env-setup-environment-bootstrap.md`，只完成 DLC Ecosystem 环境 bootstrap，不开始长构建。

目标根目录：`<ROOT>`
允许 clone：<是/否>
允许下载 CMake：<是/否>
版本策略：<CI 默认最新 / 固定 refs / 混合；不知道写“先提出方案”>

先做只读发现；任何 private repository、下载、安装、`/usr/local` 修改和 Host 操作必须在精确授权范围内。完成 bootstrap report 后停止，并把控制权交还标准 `dlc-env-setup` 流程。
```

详细规则：[DLC Ecosystem 工作站源码与基础依赖 Bootstrap](dlc-env-setup-environment-bootstrap.md)

关键边界：bootstrap report 不是 `environment_handoff/v1`；缺 approved CMake SHA-256 不解压；不得在非 disposable workspace 进行 destructive clean/reset。

## 8. 验证新容器中的 `dlc-env-setup`

什么时候用：刚装好 Skills，想验证来源、暴露、识别、停止语义和可选完整闭环，而不是立即重建业务环境。

直接这样说：

```markdown
请参考 `prompt-examples/dlc-env-setup-fresh-container-validation.md` 验证当前新容器中的 `dlc-env-setup`。

先只做 Skills 来源、安装暴露、当前 Kilo/Harness 识别和停止语义检查；如果本机选择使用 Hermes，再附加 Hermes profile 验收。不要构建、切换 branch 或修改系统环境。只有我明确批准完整执行层后，才运行真实 rebuild/install/runtime smoke。
```

详细规则：[新容器中验证 `dlc-env-setup`](dlc-env-setup-fresh-container-validation.md)

关键边界：识别回答、文件存在和轻量骨架不能替代真实 build/install/runtime smoke；Kilo 可直接作为日常执行器。Hermes 仅在已选择、已安装并通过独立 profile 验收时作为可选执行器。

## 9. 模型卡住或 Unsupported Operator

什么时候用：模型报 unsupported operator、卡在 op，或不知道第一失败点。

直接这样说：

```markdown
这个模型在 DLC Platform 上运行失败。运行命令、日志和错误材料在：`<MATERIAL_PATH>`。

请参考 `prompt-examples/dlc-business-skill-examples-quickstart.md` 的套餐二，使用 `diagnosing-bugs` 建立能复现当前问题的反馈闭环，找到第一失败点，并输出可交付给算子团队的问题信息包和最小复现。不要只看最后一个报错。
```

详细规则：[业务套餐二](dlc-business-skill-examples-quickstart.md#套餐二模型卡住或遇到-unsupported-operator)

## 10. 输出、Logits 或 Token 不一致

什么时候用：模型能运行，但 CPU Reference、其他基线或 DLC Platform 输出存在差异。

直接这样说：

```markdown
模型存在输出精度差异：<现象，例如第 2 个 token 开始分叉>。
日志、输出、dump 或 compare 结果：`<MATERIAL_PATH>`

请参考 `prompt-examples/dlc-business-skill-examples-quickstart.md` 的套餐三，使用 `diagnosing-bugs`。先确认输入一致，再从 logits 向前逐层定位第一个 divergent module；每轮只缩小一个边界并给出 evidence 路径。
```

详细规则：[业务套餐三](dlc-business-skill-examples-quickstart.md#套餐三输出或-token-不一致)

关键边界：预期硬件行为应称 DLC Precision Difference，不自动称 precision bug；CPU fallback 是定位手段，不是生产修复。

## 11. Model-Site Dump 转 pytorch_test Framework 复现

什么时候用：已有 Model-Site Dump 或明确目标算子，需要压缩成可交付、code-only 的最小回归测试。

直接这样说：

```markdown
请把这个 Model-Site Dump 问题做成 pytorch_test Framework 最小复现。

材料目录：`<DUMP_OR_EXPERIMENT_PATH>`
目标算子：<知道就填写，不知道写“请自动定位”>

参考 `prompt-examples/dlc-business-skill-examples-quickstart.md` 的套餐四。先证明 dump exact replay，再逐步缩小 shape，最终交付不依赖 `.pt` 和模型权重的 code-only repro、全局唯一 Variant 和运行命令。
```

详细规则：[业务套餐四](dlc-business-skill-examples-quickstart.md#套餐四把真实-dump-变成-pytorch_test-framework-复现)

关键边界：dump replay 不 exact 时不能宣称已形成 pytorch_test Framework 复现；Variant、dispatch key 和 DLC Custom Kernel 名不能混用。

## 12. 验证 Dispatch/Fallback 是否生效

什么时候用：修改 `enabled_kernels.hpp` 或 dispatch 配置后，需要证明目标路径真的 fallback 或仍在发射原 kernel。

直接这样说：

```markdown
请验证这次 PyTorch DLC Backend dispatch/fallback 修改是否真实生效。

代码、diff 和运行材料：`<MATERIAL_PATH>`
目标 op/kernel：<知道就填，不知道写“从 diff 和日志自动发现”>

参考 `prompt-examples/dlc-business-skill-examples-quickstart.md` 的套餐五，检查 dispatch key、PyTorch op、DLC Custom Kernel 映射和 DLCSynapse log，最终用 `syn_*.ansi` 和 kernel summary 给出证据。
```

详细规则：[业务套餐五](dlc-business-skill-examples-quickstart.md#套餐五确认-dispatch-或-fallback-是否生效)

## 13. 从 DLCSynapse Log 进入 Replay

什么时候用：已有 `syn_*.ansi`，或希望从一次模型运行生成 log，再进入 pytorch_test Framework replay。

直接这样说：

```markdown
请把这次 DLC Platform 运行接到 DLCSynapse log 和 pytorch_test Framework replay。

已有材料：`<SYN_LOG_OR_RUN_PATH>`

参考 `prompt-examples/dlc-business-skill-examples-quickstart.md` 的套餐六，自动发现 log、kernel summary 工具和 pytorch_test Framework，输出实际 `syn_*.ansi`、`*_kernels.txt` 和可执行 replay 命令。
```

详细规则：[业务套餐六](dlc-business-skill-examples-quickstart.md#套餐六从-dlcsynapse-log-进入-replay)

## 14. DLC Runtime 报错、Hang 或进程残留

什么时候用：出现 `synErrorLaunchFailure`、模型 hang、多卡异常、worker 退出或任务进程残留。

直接这样说：

```markdown
当前 DLC Platform 运行出现：<hang / synErrorLaunchFailure / worker 退出 / 多卡异常 / 进程残留>。
日志、启动命令和已有说明：`<MATERIAL_PATH>`

参考 `prompt-examples/dlc-business-skill-examples-quickstart.md` 的套餐七，使用 `diagnosing-bugs` 找最后成功阶段、第一真实错误和最小失败边界。先做只读设备、进程、端口和 HBM 检查，再决定下一步。
```

详细规则：[业务套餐七](dlc-business-skill-examples-quickstart.md#套餐七dlc-runtime-报错hang-或进程残留)

关键边界：未经授权不得 reset、LYP repair、重启、驱动维护或清理非任务进程；task-owned KILL 前也必须重新核验进程身份。

## 15. vLLM 异步 Launch Failure 定位

什么时候用：Serving 已生成部分 token，随后 worker abort、EngineCore dead 或 HTTP 500，希望定位第一 fatal 和首个失败 DLC Custom Kernel。

直接这样说：

```markdown
请使用 `diagnosing-bugs` 定位这次 vLLM-DLC 异步 launch failure。

完整启动命令、固定失败请求、server log 和 client output：`<ARTIFACT_PATH>`

参考 `prompt-examples/vllm-async-launch-failure-localization.md`，先建立可重复触发当前症状的反馈闭环，按生命周期找第一 fatal。只有 blocking evidence 到达实际 worker 并覆盖原失败阶段时，才命名首个失败 DLC Custom Kernel。
```

详细规则：[vLLM 异步 Launch Failure 定位 Prompt](vllm-async-launch-failure-localization.md)

关键边界：HTTP 500、EngineDead、cleanup warning 和异步错误报告 API 都可能只是传播或 observation point；异步 kernel summary 末行只能作为候选。

## 16. vLLM 性能热点分层定位

什么时候用：TTFT、TPOT/ITL、decode latency 或 throughput 有回归，需要从端到端逐层定位热点。

直接这样说：

```markdown
请使用 `diagnosing-bugs` 定位这次 vLLM-DLC 性能热点或回归。

模型、完整 serve command、固定 workload、baseline 和 artifact 目录：`<MATERIAL_PATH>`

参考 `prompt-examples/vllm-performance-hotspot-localization.md`。先建立原始无插桩 baseline，再从端到端、模型 layer、framework wrapper 到 DLC Platform 执行层逐级收敛；每轮只深入当前确认最慢的边界。最终移除 instrumentation 后重跑正式 benchmark。
```

详细规则：[vLLM 性能热点分层定位 Prompt](vllm-performance-hotspot-localization.md)

关键边界：debug timing、强制同步和 microbenchmark 只用于定位，不能直接外推正式 TTFT/TPOT/throughput；没有无插桩复测时结论保持 `not_verified`。

## 17. Prefill/Decode Separation

什么时候用：部署或诊断单机 TCP、qualified `lyp_full`、qualified `dlccl_direct` 或跨机器 TCP 的 Prefill/Decode Separation。

直接这样说：

```markdown
请使用 `pd-separation` 部署并验证模型 `<MODEL_NAME>` 的 Prefill/Decode Separation。
模型目录：`<ABSOLUTE_LOCAL_MODEL_PATH>`

参考 `prompt-examples/vllm-dlc-prefill-decode-separation.md`。自动发现并提出 topology、设备、端口、transport 和 KV Cache Transfer Contract；先完成 Transport Qualification Gate，再加载 Prefill/Decode roles，持续执行到 `pd_validated` 或明确 terminal blocker。
```

详细规则：[vLLM-DLC Prefill/Decode Separation Prompt](vllm-dlc-prefill-decode-separation.md)

关键边界：两个 role Ready、HTTP 200、非空输出或 connector handshake 都不证明 KV Cache transfer；`pd_validated` 需要 request-correlated routing、KV consumption 和 monolithic functional equivalence evidence。

## 18. Main-to-Main Upgrade

什么时候用：把 vLLM-DLC main 对齐到精确 upstream vLLM full SHA、恢复未知基线，或做全局 compatibility impact 分析。

直接这样说：

```markdown
请使用 `main-to-main-upgrade` 分析 vLLM-DLC 对齐任务。

目标 upstream vLLM 完整 SHA：`<TARGET_VLLM_FULL_SHA>`
仓库快照、历史范围和已有 evidence：`<PATH_OR_SUMMARY>`

参考 `prompt-examples/vllm-dlc-main-to-main-upgrade.md`。先核验全部强制输入和当前 Git 状态；缺失时返回精确 blocker。保持 report-only、no-finalize，不修改仓库、不 commit、不写入伪造的 Verified vLLM Alignment。
```

详细规则：[vLLM-DLC Main-to-Main Upgrade Prompt](vllm-dlc-main-to-main-upgrade.md)

关键边界：target 必须是 full SHA；候选 checkout、README 或历史记录不能称为 Verified vLLM Alignment；单模型适配应改用 `model-adaptation`。

## 19. 可选：固定 Hermes ModelZoo Runner 单模型验证

什么时候用：明确选择 Hermes，并且处在文档指定的固定 ModelZoo 环境，使用已有 `run-one-model.py` 验证单个模型。其他环境跳过本节，直接使用 `modelzoo-image-validation`。

直接这样说：

```markdown
参考 `prompt-examples/hermes-modelzoo-batch-validation.md`，使用现有固定 ModelZoo runner 验证一个模型。

模型名称：`<MODEL_NAME>`
模型目录：`<MODEL_PATH>`

先核验文档中固定 container、runner、设备保留和 SMI 前提仍成立；如果环境身份不匹配，停止并改用通用 `new-model-validation-quickstart.md`，不要套用历史结果。
```

详细规则：[Hermes ModelZoo 单模型验证 Prompt](hermes-modelzoo-batch-validation.md)

关键边界：该文件名含 batch，但正文主流程是单模型；固定路径和历史 PASS 不能自动继承为当前 Contract 的 runtime qualification 或 image delivery evidence。

## 20. 长任务换 Session 或交接同事

什么时候用：当前 Session 很长、实验很多，或要交给另一个 Session/同事继续。

直接这样说：

```markdown
/handoff "请生成完整交接包：目标、事实基线、已完成实验、关键证据、当前最小失败边界、不要重复的路径、下一步单变量动作、完成判据、暂停条件、必读知识文档和建议 Skills。再生成一段可直接复制到新 Agent Session 的中文 bootstrap Prompt；如果目标执行器已明确，再按 Kilo、Hermes 或其他 Harness 标注。"
```

详细规则：[业务套餐八](dlc-business-skill-examples-quickstart.md#套餐八长任务换-session-或交接同事)

关键边界：handoff 是跨 Session 上下文桥梁，不是业务验收记录；新 Session 仍需核验当前代码、进程、环境和 Evidence。

## 21. 把经验反哺知识库

什么时候用：问题已定位或形成可复用方法，希望后续任务不再从零排查。

直接这样说：

```markdown
这个任务产生了可复用经验，材料在：`<REPORT_LOG_TEST_OR_EXPERIMENT_PATH>`。

请参考 `prompt-examples/dlc-business-skill-examples-quickstart.md` 的套餐九，先区分通用方法、具体 case、runtime observation 和未验证假设，再按问题域更新专题文档、case study 或 prompt example。不要按模型名新建一级或二级目录，不得写入密码、token 或机器敏感信息。
```

详细规则：[业务套餐九](dlc-business-skill-examples-quickstart.md#套餐九把本次经验写回知识库)

## 22. 为 Host/容器补齐 Git/SSH 能力

什么时候用：Host 或新容器缺 Git、GitHub SSH 或 private repository clone/fetch/push 能力，但已有一个受信、配置完成的源容器。

直接这样说：

```markdown
请参考 `prompt-examples/bootstrap-git-from-configured-container.md`，安全地为目标环境补齐 Git/SSH private repository 能力。

源容器：`<SOURCE_CONTAINER_AND_USER>`
目标 Host/容器：`<TARGETS>`
验证仓库：`<PRIVATE_REPO_SSH_URL>`
允许复制 SSH private key：<是/否>
允许安装 Git：<是/否>

先验证源身份、目标冲突和授权边界。不得输出任何 secret，不覆盖目标已有 SSH/Git 配置，不做真实测试 commit/push，只执行 clone/fetch 和 push dry-run 验收。
```

详细规则：[从已配置容器为 Host 和其他容器赋能 Git/SSH](bootstrap-git-from-configured-container.md)

关键边界：SSH private key 迁移需要凭据 owner 的明确授权；环境 bootstrap 需要 clone 不等于自动获得私钥复制授权。

## 23. 可选：验证或修复 Hermes 的 Chipltech 接入

什么时候用：团队明确选择使用 Hermes，并且 Hermes 找不到知识库、Memory、Project 或 Skills，路由错误，默认模型/Provider 异常，或需要验证配置完整性。未使用 Hermes 时跳过本节，不影响其他能力。

直接这样说：

```markdown
请参考 `prompt-examples/hermes-chipltech-engineering-quickstart.md`，检查并修复 Hermes 的 Chipltech 工程接入。

验证知识库 `/work/chipltech-knowledge-base`、Skills `/work/skills`、`chipltech-context`、稳定 external dirs、Project、Memory、默认模型和四场景路由验收。不得输出 API Key。

这里由 Kilo Code Agent 作为 Hermes 的教练和修复者执行；修复完成后给出根因、修改、验证证据、残余风险和下一条 Hermes 可执行动作。是否把后续任务交给 Hermes 由用户选择，Kilo 仍可直接执行。
```

详细规则：[Hermes Chipltech Engineering Quickstart](hermes-chipltech-engineering-quickstart.md)

关键边界：Hermes 是可选执行器；Kilo 的修复证据只证明 Hermes 能力或执行链恢复，不证明具体业务任务、模型、DLC Runtime 或 Real DLC Hardware 已验收。Hermes 不可用不代表 owning Skill 或 Kilo 调用链不可用。

## 使用前只记住六件事

1. **默认使用当前可用 Harness**：Kilo Code 可直接加载 owning Skill 并执行；Hermes 是可选执行器，只有明确选择 Hermes 时才要求它的 profile 验收通过。
2. **只给最少必要输入**：通常是一句话目标加一个材料路径；模型类任务通常只需要模型名和绝对目录。
3. **不知道的写“请自动发现”**：不要重复填写本机可只读发现的仓库、版本、设备、命令和日志。
4. **自动发现不等于授权**：安装、构建、ref 切换、device execution、KILL、Host maintenance、镜像导出/推送仍需对应授权。
5. **Prompt 是任务 Contract，不是完成证据**：文档、Skill、HTTP 200、Ready、非空输出和单次 benchmark 都不能自动证明更高层验收。
6. **结论必须可追溯**：要求输出 evidence 路径、terminal state、Claim Boundary、未验证项和 remaining risks。
