# DLC Platform 业务套餐傻瓜版调用说明

## 这份文档怎么用

这份文档给第一次使用 Harness、Kilo、Claude Code 或其他 AI agent 的同事使用。你不需要先理解 skill、contract、evidence state，也不需要把正式套餐里的每个字段全部填写完。

最简单的调用方式只有三部分：

```text
我要做什么
+ 已有材料在哪里
+ 让大模型读取知识库和正式套餐后自己路由
```

通用模板：

```markdown
我现在要处理：<一句话说明目标或问题>。

已有材料在：<日志、错误文档、模型、dump、代码仓库或实验目录>。

请先自动发现并读取当前 `chipltech-knowledge-base` 的 `CONTEXT.md`、`README.md` 和 `prompt-examples/dlc-business-skill-examples.md`，根据我的目标选择正确套餐和 skill，自动读取已有材料、补全可发现的信息，然后持续分析或执行到当前授权范围内的明确结论。不要让我重复填写可以从本机文件、源码、日志或环境中发现的信息。

所有结论区分已确认事实、推断和未验证项。涉及设备执行、KILL、reset、LYP repair、驱动维护、重启、安装、构建、镜像推送或清理非任务进程时，先说明影响并等待对应授权。
```

如果你知道自己属于哪个套餐，直接复制下面对应的最短示例。`<>` 中只填写你已经知道的内容；不知道的可以写“请自动发现”。

正式执行规则仍以 [DLC Platform 业务 skill 使用示例](dlc-business-skill-examples.md) 和对应专题文档为准。本页只负责让任务快速启动，不复制维护完整流程。

## 套餐一：新模型首次接入

什么时候用：拿到一个模型，想确认能不能在 DLC Platform 上加载和推理。

```markdown
请帮我把模型 `<模型名称>` 在 DLC Platform 上跑通，模型目录是 `<模型绝对路径>`。

先读取当前知识库的 `CONTEXT.md`、`README.md`、`prompt-examples/dlc-business-skill-examples.md`，选择“套餐一：新模型首次接入 DLC Platform”，并自动发现模型 config、权重、tokenizer/processor、环境、设备和测试输入。按环境检查、模型加载、最小功能请求的顺序执行，每一步通过后再进入下一步。

先做只读检查。需要设备执行、安装、构建、KILL 或 Host maintenance 时，列出精确动作和影响后等待授权。
```

只知道模型目录时也可以这样写：

```markdown
这个目录里有一个新模型：`<模型绝对路径>`。请读取知识库和业务套餐，自动识别模型名称、类型和所需流程，然后验证它能否在 DLC Platform 上完成最小推理。
```

## 套餐二：模型卡住或遇到 unsupported operator

什么时候用：模型报 unsupported operator、卡在某个 op，或者不知道第一失败点在哪里。

```markdown
这个模型在 DLC Platform 上运行失败了。已有运行命令、日志和错误材料在：`<材料路径>`。

请读取当前知识库和 `prompt-examples/dlc-business-skill-examples.md`，选择“套餐二：遇到 unsupported operator 时捕获算子问题信息”。先完整读取材料，找到第一失败点，再判断是 unsupported op、dtype/shape/layout、DLC Attention Backend 还是 DLC Runtime 问题。输出可交付给算子团队的问题信息包和最小复现。

不要只看最后一个报错，也不要在未核验进程身份时清理进程。
```

最简单的实际例子：

```markdown
目前登录的机器上模型运行有错误。请好好读取 `/work/err` 下与这次运行有关的日志和说明文档，再参考当前 `chipltech-knowledge-base`、`prompt-examples/dlc-business-skill-examples.md` 和相似 case study，定位主要错误发生在哪个阶段、哪个 module 或哪个 op，并生成算子问题信息包。
```

## 套餐三：输出或 token 不一致

什么时候用：模型能运行，但 CPU Reference、其他基线或 DLC Platform 的输出不同。

```markdown
这个模型存在输出精度差异：`<一句话描述现象，例如第 2 个 token 开始分叉>`。

相关日志、输出、dump 或 compare 结果在：`<材料路径>`。

请读取当前知识库和 `prompt-examples/dlc-business-skill-examples.md`，选择“套餐三：精度差异定位”。先确认输入完全一致，再从 logits 向前逐层定位第一个 divergent module。每次只缩小一个边界，并为结论标出 evidence 路径。
```

如果不知道 dump 在哪里：

```markdown
模型在 CPU Reference 和 DLC Platform 上输出不一致，但我不知道要抓哪些 dump。请读取知识库和套餐三，先检查现有日志与代码，告诉我最小需要采集哪些输入和 module 输出；能在当前授权内自动采集的就执行，不能执行的列出最小 blocker。
```

## 套餐四：把真实 dump 变成 pytorch_test Framework 复现

什么时候用：已经有 Model-Site Dump 或已知道目标算子，需要压成可交付的测试。

```markdown
请把这个 Model-Site Dump 问题做成 pytorch_test Framework 最小复现。

材料目录：`<dump 或实验目录>`
目标算子：`<知道就填写，不知道写“请自动定位”>`

请读取当前知识库和 `prompt-examples/dlc-business-skill-examples.md`，选择“套餐四：从真实 dump 收敛到 pytorch_test Framework 复现”。先验证 dump 可以 exact replay，再逐步缩小 shape，最终生成不依赖 `.pt` dump 和模型权重的 code-only repro、全局唯一 Variant 和运行命令。
```

## 套餐五：确认 dispatch 或 fallback 是否生效

什么时候用：修改了 `enabled_kernels.hpp`，想确认目标路径是否真的 fallback 或是否仍在发射原 kernel。

```markdown
我修改了一个 PyTorch DLC Backend 算子的 dispatch/fallback 配置，需要确认是否真正生效。

代码和运行材料在：`<PyTorch 仓库、模型命令、日志目录或实验目录>`
目标 op/kernel：`<知道就填写，不知道写“请从 diff 和日志自动发现”>`

请读取当前知识库和 `prompt-examples/dlc-business-skill-examples.md`，选择“套餐五：确认 dispatch 改动到底有没有生效”。自动检查本次 diff、dispatch key、PyTorch op、DLC Custom Kernel 映射和现有 DLCSynapse log；缺日志时给出或执行本次 run 的采集流程。最终用 `syn_*.ansi` 和 kernel summary 证明生效或未生效。
```

## 套餐六：从 DLCSynapse log 进入 replay

什么时候用：手里已有 `syn_*.ansi`，或希望先生成 log 再做 pytorch_test Framework replay。

```markdown
请把这次 DLC Platform 运行接到 DLCSynapse log 和 pytorch_test Framework replay 流程。

已有材料：`<syn_*.ansi、模型命令或实验目录>`

请读取当前知识库和 `prompt-examples/dlc-business-skill-examples.md`，选择“套餐六：从 DLCSynapse log 进入 pytorch_test Framework replay”。自动发现 log、kernel summary 工具和 pytorch_test Framework 路径；没有 log 时先按知识库生成。输出实际 `syn_*.ansi`、`*_kernels.txt` 和可执行 replay 命令，并区分 dispatch key、Variant 和 DLC Custom Kernel 名。
```

## 套餐七：DLC Runtime 报错、hang 或进程残留

什么时候用：出现 `synErrorLaunchFailure`、模型 hang、多卡异常、worker 退出或任务进程残留。

```markdown
当前机器上的 DLC Platform 运行出现 `<hang / synErrorLaunchFailure / worker 退出 / 多卡异常 / 进程残留>`。

错误日志、启动命令和已有说明在：`<材料路径>`。

请读取当前知识库、`prompt-examples/dlc-business-skill-examples.md` 和相似 Runtime case study，选择“套餐七：DLC Runtime 排障”，使用 `/diagnosing-bugs` 找第一真实错误和最小失败边界。先做只读设备、进程、端口和 HBM 检查，再决定是否需要 blocking、DLCSynapse log 或最小复现。

未经授权不得 reset、LYP repair、重启、驱动维护或清理非任务进程；task-owned KILL 也要先核验 PID、PGID、starttime、cmdline 和 namespace/cgroup 身份。
```

GLM 类异步错误可以进一步告诉大模型读取：

```text
prompt-examples/vllm-async-launch-failure-localization.md
```

## 套餐八：长任务换 session 或交接同事

什么时候用：当前对话已经很长、跑了多轮实验，或者要让另一个 session/同事继续。

```markdown
/handoff "请把当前任务生成完整交接包。自动总结目标、已完成实验、事实基线、当前失败边界、关键文件、不要重复做的路径、下一步单变量动作、完成判据和暂停条件。读取当前 chipltech-knowledge-base，为新 session 选择必读文档和 suggested skills，并生成一段可直接复制到新 session 的中文 bootstrap prompt。"
```

如果只想交给同事：

```markdown
请把当前任务整理成一个同事能直接接手的 Markdown：说明问题、已经做了什么、证据在哪里、现在卡在哪、下一步第一条命令和风险边界。不要只写聊天摘要。
```

## 套餐九：把本次经验写回知识库

什么时候用：问题已经定位或形成可复用方法，希望下次不再从零排查。

```markdown
这个任务已经产生了可复用的定位经验，完整材料在：`<报告、日志、测试或实验目录>`。

请读取当前知识库的 `README.md`、`CONTEXT.md` 和 `prompt-examples/dlc-business-skill-examples.md`，选择“套餐九：把定位经验和复现成果反哺知识库”。先区分通用方法、具体 case 和未验证假设，再按问题域更新专题文档、case study 或 prompt example；不要按模型名新建一级或二级目录，也不要写入密码、token 或机器敏感信息。

如果认为需要更新 skill，只把稳定、跨案例适用的执行门禁写入对应 skill，不把单次故障假设固化成通用规则。完成后运行格式、链接和相关发布测试。
```

## 不知道选哪个套餐

什么时候用：你只知道“有个问题”，不知道属于环境、模型、算子、精度还是 DLC Runtime。

```markdown
使用 `/ask-matt` 帮我路由这个 DLC Platform 任务。

我要处理：`<一句话描述问题>`
已有材料：`<日志、错误文档、模型目录、dump 或代码目录>`

请先读取当前 `chipltech-knowledge-base` 的 `CONTEXT.md`、`README.md` 和 `prompt-examples/dlc-business-skill-examples.md`，再读取已有材料。告诉我应该使用哪个套餐、哪个 skill、第一条可执行动作和需要什么授权；能够在当前授权内安全完成的工作直接继续，不要只给计划。
```

## 只有错误文件时怎么说

如果同事只给了一个错误目录，直接复制这段：

```markdown
目前这台机器上的运行内容有错误。请完整读取 `<错误材料目录>` 下与本次任务有关的日志、说明和启动记录，同时读取当前 `chipltech-knowledge-base` 的 `CONTEXT.md`、`README.md`、`prompt-examples/dlc-business-skill-examples.md` 和相似 case study。

请自动选择正确套餐和 skill，先按生命周期找最后成功阶段和第一失败点，再给出主要错误发生在哪里、直接错误传播链、已排除项、未验证项和下一步最小验证动作。不要只复述最后一个报错，不要把异步错误报告点直接当成真实失败点。

先做只读分析。需要设备执行、进程清理或 Host maintenance 时，列出精确动作、目标和影响后等待授权。
```

## 使用时记住四件事

1. **先给路径，不必复制大段日志**：大模型和你共享机器，直接让它读取文件。
2. **不知道的信息写“自动发现”**：模型名称、版本、设备、命令和日志能从本机找到时，不必手工重复填写。
3. **让大模型持续做，不要只让它给建议**：明确写“在当前授权内直接执行到明确结论”。
4. **高风险动作仍需授权**：KILL、reset、LYP repair、驱动维护、重启、安装、构建、镜像推送和非任务资源清理不会因为用了傻瓜版而自动授权。
