# DLC 业务 skill 使用示例

面向 DLC Platform 日常工程任务的 prompt 模板。每条都能直接复制、填空即用，不需要先理解 skill 的原理。

**📖 阅读说明**：
- `▶ 可复制 prompt` 标记后面的代码块是发给 AI 的指令，替换 `<>` 里的占位符即可使用。
- 套餐标题下的"适用场景""关键点"是给人看的说明，不用复制发给 AI。

---

**▶ 可复制 prompt：每次任务开始前先加这句，强制 AI 读知识库**

```
先读取 /work/chipltech-knowledge-base/CONTEXT.md 和 README.md，再开始处理任务。
```

---

## 套餐一：新模型首次接入 DLC Platform

**适用场景**：拿到一个新模型，不知道能不能在 DLC Platform 上跑起来。需要从环境检查、模型加载到首图跑通，逐步验证。

**▶ 可复制 prompt（替换 `<>` 占位符后发给 AI）：**

```
我要把一个新模型在 DLC Platform 上跑通，请按下面的顺序分步验证，每一步通过后再进下一步。

【模型路径】<模型 checkpoint 路径>
【模型类型】<例如 Glm4vForConditionalGeneration 或其他 Transformers class>
【测试图片】<图片路径，如果有>
【推理方式】<单图短 decode / 文本推理 / 批量评测>
【预期 dtype】<bfloat16 / float32>

请你：
1. 读取：
   - /work/chipltech-knowledge-base/CONTEXT.md
   - /work/chipltech-knowledge-base/debugging-workflows/common-debug-commands.md
2. 按以下顺序执行，每一步输出结果：

#### 第一步：环境检查
- 检查 Python、PyTorch、Transformers 版本。
- 检查 `torch.dlc` 是否可用：尝试 `torch.arange(4).to("dlc") + 1`。
- 检查模型类是否可导入。
- 如果任何一步失败，先解决环境问题，不要跳到模型加载。

#### 第二步：模型加载到 DLC
- 加载模型（bf16 或 fp32），记录参数量和 device 分布。
- 执行 `model.to("dlc")`，记录耗时。
- 如果 30 秒未返回，检查残留进程：`lsof /dev/dlc*`，必要时做软复位。
- 记录第一层参数 device 和 dtype，确认模型确实放在了 DLC 上。

#### 第三步：首图冒烟推理
- 构造最小输入：1 张图 + 短 prompt + max_tokens=1。
- 先跑 CPU 参考，确认模型本身不出错。
- 再跑 DLC，确认能完成 prefill 和至少 1 个 decode step。
- 如果卡住，记录第一个阻塞位置（哪个 module / 哪个 op），输出 triage 信息。

交付物：
- 环境检查结果。
- 模型加载结果（参数量、device、loader 耗时）。
- 首图冒烟结果（是否跑通、第一个阻塞点、完整的失败信息和 trace）。
```

**关键点**：先 CPU 跑通，再 DLC 跑通。卡住不要反复重试，第一时间记录阻塞位置和完整错误信息。

---

## 套餐二：遇到 unsupported operator 时捕获 triage 信息

**适用场景**：模型在 DLC Platform 上卡住或报错，需要把第一个失败点收敛成可交付给算子团队的信息包。

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```
请帮我捕获这个模型在 DLC Platform 上的第一个阻塞点，生成可交付给算子团队的 triage 包。

【运行命令】<完整命令>
【卡住位置或报错信息】<贴原始输出>
【模型和输入信息】<模型路径、用了几张图、prompt 内容>

请你：
1. 读取：
   - /work/chipltech-knowledge-base/CONTEXT.md
   - /work/chipltech-knowledge-base/operator-dispatch/enabled-kernels-dispatch.md
   - 相关知识库 case study
2. 先对失败进行分类：
   - 是 PyTorch op 不支持（比如 DLC 后端还没有实现这个算子）？
   - 是 dtype/shape/layout 不兼容？
   - 是 attention backend 不匹配？
   - 是运行时挂起（执行后一直不返回）？
3. 对阻塞算子记录以下信息：
   - 算子名称（例如 Conv3d、F.linear、scaled_dot_product_attention）
   - 输入 tensor 的 shape、dtype、stride、layout、device
   - 输出 tensor 的 shape、dtype、stride、layout、device
   - 如果有 weight/bias，同样记录
4. 尽可能写一个最小复现脚本（10 行以内），只构造该算子的输入，不加载完整模型。
5. 给出建议：这个算子应该由哪个团队处理、应该放在 pytorch_test 还是 PyTorch DLC Backend 里修复。

交付物：
- 失败分类。
- 阻塞算子详情：名称、shape、dtype、stride 等。
- 最小复现命令。
- 建议交付路径（pytorch_test / backend dispatch / attention backend）。
```

**关键点**：不要急着修，先分类、先记录。完整的 shape/dtype/stride 信息比猜测更有用。

---

## 套餐三：精度差异定位 — 从 token 分叉到第一个 divergent module

**适用场景**：模型在 DLC Platform 上能跑通，但 CPU 和 DLC 的输出 token 不一致。需要找到第一个产生数值差异的 module。

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```
请帮我定位这个模型的 CPU vs DLC 精度差异，按步骤收敛到第一个 divergent module。

【现象】<例如第 2 个 token 分叉、SC_AID 样本输出不同>
【已知信息】<processor inputs 是否 exact、哪些 module 已验证 exact>
【dump 路径】<CPU dump 目录、DLC dump 目录>

请你：
1. 读取：
   - /work/chipltech-knowledge-base/CONTEXT.md
   - /work/chipltech-knowledge-base/precision-debugging/precision-debugging-overview.md
   - /work/chipltech-knowledge-base/precision-debugging/model-site-dump-to-repro.md
   - 相关 case study（如果相似）
2. 用 /diagnose 和 /zoom-out 的方式工作：
   - 先确认 processor inputs 一致（input_ids、pixel_values 等）。
   - 从模型输出往前倒推：logits → decoder → attention → encoder → image processor。
   - 对每个 module，比较 CPU 和 DLC 的 tensor 输出。
   - 只回答"第一个 divergent module 是谁、差异多大"。
3. 不要一次看很多层，逐层看：
   - 如果 attention 有差异，拆成 q/k/v projection、RoPE、attention mask、softmax、value matmul。
   - 如果 MLP 有差异，拆成 gate/up projection、activation、down projection。
4. 每个结论必须标证据来源（dump 文件、compare report、manifest）。

交付物：
- processor inputs 是否 exact。
- 已知 exact 的 module 列表。
- 第一个 divergent module：名称、shape、abs_max 差异。
- 下一步最小验证动作。
```

**关键点**：先确认输入一致，再从输出倒推。一次只看一个边界，不要跳。

---

## 套餐四：从真实 dump 收敛到 pytorch_test 复现

**适用场景**：已经知道某个算子有 CPU/DLC 差异，需要把真实模型问题压成最小复现，交付给算子 owner 修复。

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```
请把这个 Model-Site Dump 问题收敛成 pytorch_test 复现，用 /diagnose 定位，用 /tdd 写测试。

【dump 路径】CPU: <path>；DLC: <path>
【目标算子】<例如 F.linear / F.softmax / F.conv3d / bmm>
【kernel 线索】<例如 custom_addmm_rhsT_ROW_BIAS、custom_softmax，如果有>

请你：
1. 读取：
   - /work/chipltech-knowledge-base/CONTEXT.md
   - /work/chipltech-knowledge-base/precision-debugging/model-site-dump-to-repro.md
   - /work/chipltech-knowledge-base/testing/dlc-kernel-test-framework-guide.md
2. 先验证 replay 正确性：
   - CPU 侧：用 dump 中的真实输入在 CPU 上重新执行，确认结果与 saved CPU tensor 完全一致。
   - DLC 侧：同样在 DLC 上重新执行，确认结果与 saved DLC tensor 完全一致。
   - 如果 replay 不能 exact match，先修 replay 语义，不要继续写 pytorch_test。
3. 收敛到最小 shape：
   - 先用真实 shape 确认可复现。
   - 逐步缩小 shape，找到最小的可复现 shape。
4. 最终交付 code-only repro：
   - 用固定 seed 随机生成输入。
   - 不依赖任何 .pt dump 文件或模型 checkpoint。
   - `python test_file.py` 即可运行。
5. pytorch_test 里用全局唯一 Variant。

交付物：
- pytorch_test 文件路径和 Variant 名。
- 运行命令。
- CPU/DLC 比较结果：mismatch 数、abs_max。
- 是否已脱离 .pt dump 文件（必须回答"是"才算完成）。

```

**关键点**：最终交付物是 code-only repro，不依赖任何外部文件。真正交付给算子团队的是 pytorch_test 测例，不是聊天结论。

---

## 套餐五：确认 dispatch 改动到底有没有生效

**适用场景**：已经知道一个算子可能要从 DLC 切到 CPU fallback，或者想确认 `enabled_kernels.hpp` 的改动到底控制的是哪个 dispatch key、对应哪个 runtime kernel。希望 AI 不只是解释概念，而是把“开环境变量、跑模型、拿 syn log、生成 kernel.txt、判断生效没”这一整条链路走完整。

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```
请帮我确认一个 DLC 算子的 dispatch 改动是否真的生效，并按完整流程给出验证闭环。

【目标算子】<例如 custom_matmul_t_pingpong / custom_bmm_f32 / custom_layer_norm_f32>
【PyTorch op】<例如 F.linear / torch.bmm / F.layer_norm>
【模型运行命令】<例如 RSThinker_infer_chat_stream.py 的完整命令>
【最小复现命令】<如果有单独 repro.py 也提供；没有就留空>

请你按下面这些已知规则工作，不要省略步骤：
- `enabled_kernels.hpp` 控制的是 dispatch key，不一定等于 synapse log 里的最终 runtime kernel 名。
- 只 grep header 不算验证，必须同时看 runtime log。
- 修改 `/usr/local/chipltech/synapse/include/enabled_kernels.hpp` 之后，必须重编 `/work/pytorch`，否则不能判断改动是否生效。
- 完整验证链路必须包括：开环境变量 -> 跑模型或最小复现 -> 生成 `syn_*.ansi` -> 用 `python3 tool.py <syn_*.ansi>` 生成 `*_kernels.txt` -> 再判断 dispatch 是否生效。
- 看到 `is redispatched to cpu` 这类信息，才说明 dispatch key 真正走了 CPU fallback；如果仍出现 `launch custom_xxx on dlc:0`，就说明 runtime 还在走 DLC。

请你：
1. 先明确回答这 4 件事：
   - 对应的 PyTorch op 是什么。
   - 对应的 DLC Backend 插入函数是什么。
   - `DLC_CHECK_RESULT(...)` 里真正受 `enabled_kernels.hpp` 控制的 dispatch key 是什么。
   - runtime log 里最终可能看到哪些 `launch custom_xxx` 名字。
2. 然后按下面顺序给我完整执行方案，不要跳步骤：

#### 第一步：准备 runtime log 环境
- 设置环境变量：
  - `DLC_SYN_BLOCKING=1`
  - `DLC_SYN_DEBUG=1`
  - `DLC_SYN_LOG_DIR=<目录>`
  - `DLC_SYN_PROF_DIR=<目录>`
  - `DLC_SYN_PROF_CYCLE=1`
  - `DLC_SYN_VERBOSE=4`
  - `DLC_VISIBLE_DEVICES=<卡号，如果需要>`
- 明确告诉我这些变量各自是干什么的，尤其是为什么必须拿到 `syn_*.ansi`。

#### 第二步：做 dispatch 修改
- 修改 `/usr/local/chipltech/synapse/include/enabled_kernels.hpp` 对应 dispatch key，把 `DispatchType::DLC` 改成 `DispatchType::CPU`。
- 如果你判断目标 key 不对，要先指出真正应该改哪个 key，不要让我改错名字。

#### 第三步：重编 PyTorch
- 给出完整重编命令：
  - `cd /work/pytorch`
  - `export PYTORCH_BUILD_VERSION=2.5.0`
  - `export PYTORCH_BUILD_NUMBER=0`
  - `USE_CUDA=0 python setup.py develop`
- 明确说明：只改 header 不重编，不算验证。

#### 第四步：复跑模型或最小复现
- 优先跑【模型运行命令】；如果有【最小复现命令】，也给出来作为第二选择。
- 跑完后，帮我定位这次生成的 `syn_*.ansi` 文件路径。

#### 第五步：生成 kernel 摘要
- 使用：
  - `cd /work/llama2-fine-tune`
  - `python3 tool.py <syn_*.ansi>`
- 告诉我生成出来的 `*_kernels.txt` 路径，并说明它的用途：快速看本次 run 实际 launch 了哪些 kernel。

#### 第六步：判断 dispatch 是否真的生效
- 不要只看 header，要同时看：
  - `syn_*.ansi`
  - `*_kernels.txt`
- 明确告诉我：
  - 哪个现象说明已经 CPU fallback，例如出现 `is redispatched to cpu`
  - 哪个现象说明还在走 DLC，例如仍出现 `launch custom_xxx on dlc:0`
  - 如果 log 里出现的是别的 runtime kernel 名，应该怎么解释，不要按名字误判

交付物：
- dispatch key。
- 对应 runtime kernel 名列表。
- 环境变量设置命令。
- 修改命令和重编命令。
- 复跑命令。
- `syn_*.ansi` 路径和 `*_kernels.txt` 路径。
- 如何从 log 判断“已生效 / 未生效”。
```

**关键点**：`enabled_kernels.hpp` 控制的是 dispatch key，不一定等于 synapse log 里的最终 kernel 名。完整闭环必须包括：开环境变量 -> 跑模型/复现 -> 生成 `syn_*.ansi` -> 用 `tool.py` 生成 `*_kernels.txt` -> 再判断 dispatch 是否真的生效。

---

## 套餐六：把 synapse log 直接丢给 pytorch_test --replay

**适用场景**：手里还没有现成的 replay 套餐，希望 AI 帮你从环境变量开始，先跑一次模型拿到 `syn_*.ansi`，再用 `tool.py` 生成 `*_kernels.txt`，最后在 `DLC_Custom_Kernel/pytorch_test` 里跑 `python main.py --replay log.txt`。也适用于手里已经有 log，但还没整理成完整 replay 流程。

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```
请帮我把 synapse runtime log 接到 pytorch_test 的 replay 流程，并把前面的取 log 步骤也一起补完整。

【模型运行命令】<例如 RSThinker_infer_chat_stream.py 的完整命令>
【log 路径】<如果已经有，例如 /work/tmpx/syn_1123318.ansi；如果还没有就写“需要先生成”>
【目标 kernel】<可选，例如 custom_layer_norm_bf16；如果没有就留空>
【是否需要 verify-log】<是/否>

请你按下面这些已知规则工作，不要省略步骤：
- replay 不是只跑一条 `python main.py --replay log.txt`，完整链路是：开环境变量 -> 跑模型生成 `syn_*.ansi` -> 用 `python3 tool.py <syn_*.ansi>` 生成 `*_kernels.txt` -> 再跑 `pytorch_test --replay`。
- `*_kernels.txt` 的作用是快速浏览这次 runtime 实际 launch 了哪些 kernel，便于决定要不要加 `-f <目标 kernel>`。
- `-f <目标 kernel>` 是按 runtime kernel 名过滤，不代表这个名字一定等于 dispatch key。
- `--verify-log` 只在怀疑 log 本身不完整或参数不一致时再开，不要一上来就全加。
- 如果 replay 失败，优先按这 4 类排查：log 路径不对、kernel 名过滤错了、pytorch_test 没有这个 kernel 的用例、replay 输入和测试定义不匹配。

请你：
1. 如果【log 路径】还没有，请先按完整流程帮我生成：

#### 第一步：设置环境变量
- `DLC_SYN_BLOCKING=1`
- `DLC_SYN_DEBUG=1`
- `DLC_SYN_LOG_DIR=<目录>`
- `DLC_SYN_PROF_DIR=<目录>`
- `DLC_SYN_PROF_CYCLE=1`
- `DLC_SYN_VERBOSE=4`
- `DLC_VISIBLE_DEVICES=<卡号，如果需要>`

#### 第二步：运行模型
- 用【模型运行命令】跑一次，目标是生成 `syn_*.ansi`。
- 跑完后告诉我：本次 log 路径是什么。

#### 第三步：生成 kernel.txt
- 执行：
- `cd /work/llama2-fine-tune`
- `python3 tool.py <syn_*.ansi>`
- 告诉我生成的 `*_kernels.txt` 路径，并说明它是用来快速浏览 runtime kernel 列表的。

2. 然后在 `/work/DLC_Custom_Kernel/pytorch_test` 下给我 replay 套餐：
   - 先跑：`python3 main.py --replay <log 路径>`
   - 如果只看某个 kernel，再跑：`python3 main.py --replay <log 路径> -f <目标 kernel>`
   - 如果需要校验日志，再跑：`python3 main.py --replay <log 路径> --verify-log`
3. 告诉我每一步的目的：
   - 为什么先拿 `syn_*.ansi`
   - 为什么再生成 `*_kernels.txt`
   - 全量 replay 是干什么的
   - `-f` 过滤是干什么的
   - `--verify-log` 适合什么时候开
4. 如果 replay 失败，请先帮我判断属于哪一类：
   - log 路径不对
   - kernel 名过滤错了
   - pytorch_test 里没有这个 kernel 的用例
   - replay 输入和测试定义不匹配

交付物：
- 环境变量设置命令。
- 模型运行命令。
- `syn_*.ansi` 路径。
- `*_kernels.txt` 路径。
- 可直接执行的 replay 命令。
- 如果有目标 kernel，对应的 `-f` 命令。
- replay 失败时的四类排查顺序。
- 是否建议下一步改成手写 real-input repro。
```

**关键点**：不要把 replay 当成“只需要一条命令”。完整流程是：开环境变量 -> 跑模型生成 `syn_*.ansi` -> 用 `tool.py` 生成 `*_kernels.txt` -> 再跑 `pytorch_test --replay`。`-f` 适合快速看单个 kernel，`--verify-log` 适合怀疑日志本身有问题时再开，不要一上来就全都加。

---

## 套餐七：DLC Platform 运行时排障

**适用场景**：模型 hang 住不返回、报异步错误、设备有残留进程、多卡通信异常。

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```
请帮我排查一个 DLC Platform 运行时问题，先不要改业务代码。

【现象】<hang / synErrorLaunchFailure / 进程残留 / 多卡异常>
【触发命令】<完整命令>
【环境】<单卡/多卡、DLCsim/Real DLC Hardware>

请你：
1. 读取：
   - /work/chipltech-knowledge-base/CONTEXT.md
   - /work/chipltech-knowledge-base/debugging-workflows/common-debug-commands.md
2. 按以下顺序排查：

#### 第一步：设备和进程检查
- 检查设备可用：ls /dev/dlc*
- 检查残留进程：lsof /dev/dlc*
- 如果有残留，记录下来，但不要随意 kill。
- 必要时做软复位：dlcpd_clnt -s

#### 第二步：确认是不是异步错误
- 用 DLC_SYN_BLOCKING=1 重新运行。
- 异步错误经常在后续 API 才暴露，阻塞执行能定位真实失败的 kernel。

#### 第三步：开 verbose trace 定位卡住点
- 用 DLC_SYN_DEBUG=1 DLC_SYN_VERBOSE=1 运行。
- 找到最后一个 launch 成功但未返回的 kernel。
- 调试完后要提醒清理高开销环境变量。

#### 第四步：跑最小测例
- 如果卡在某个特定算子（如 Conv3d / embedding / where），写一个 10 行以内的最小复现。
- 用最小测例确认是通用问题还是模型特定问题。

交付物：
- 问题分类（设备/进程/算子/通信）。
- 第一条真实错误或最后一个卡住的 kernel。
- 最小复现命令。
- 已排除的路径。
```

**关键点**：先做低扰动检查（设备、进程），再开高开销 debug 开关。debug 环境变量用完后必须清理。

---

## 套餐八：长任务交接包

**适用场景**：一个任务已经跑了多轮实验，需要在当前 session 收尾时一次性生成交接包，让下一个 session 直接复制 prompt 开跑，避免自动扫描最新 handoff 拿错文件。

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```
/resume-from-handoff "这个任务已经跑了多轮，请生成交接包。交接包必须包含事实基线、当前问题边界、下一步计划、chipltech-knowledge-base 必读文档、suggested skills、检查结果，以及可直接复制到新 session 的完整中文 bootstrap prompt。

【当前目标】<一句话>
【已完成】<跑了哪些实验、有哪些 manifest 路径>
【关键结论】<哪些 exact、哪些 drift、哪个 branch 成立>
【不要重复做】<已排除的路径、失败的实验>
【下一步】<接手者应该立刻做的 2-3 个动作>

交接文档必须包含：
- 任务目标。
- 已完成的工作和文件路径。
- 当前卡在哪。
- 接手者第一步要做什么，以及怎么验证。
- 已经踩过的坑。
- 新 session 可直接复制执行的 bootstrap prompt。
- 明确完成判据和暂停条件。"
```

**关键点**：旧 session 收尾时就生成唯一交接包路径。新 session 不再运行 `/resume-from-handoff` 自动找最新文件，而是直接复制交接包里的 `新 Session Bootstrap Prompt` 执行。

---

## 套餐九：把定位经验和复现成果反哺知识库

**适用场景**：一个任务已经产出了可复用的经验、复现测例或 operator 边界结论，需要按问题域回写到知识库，避免下次从零开始。

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```
这个任务已经产出了一些可复用经验，请帮我把它们反哺到知识库。

【我做了什么】<环境搭建 / 精度定位 / dispatch fallback / pytorch_test 复现 / Runtime 排障>
【模型】<模型名，仅作为正文提及，不作为目录或标题层>
【关键发现】<第一个 divergent module、确认的 kernel、fallback 后归零的边界>
【复现方式】<pytorch_test 文件路径 / Variant 名 / 运行命令>
【踩过的坑】<用错的命令、不该重复的方案、容易误判的点>

请你：
1. 先读取 /work/chipltech-knowledge-base/README.md 的"持续维护原则"。
2. 按问题域判断应该写到哪个目录，不按模型名建目录：

| 问题域 | 写入位置 | 什么情况下选它 |
|--------|---------|-------------|
| 精度差异定位、dump/compare/replay | case-studies/ | 有完整的从模型问题到算子边界的定位过程 |
| dispatch fallback、enabled kernels | operator-dispatch/ | 新发现的 lambda name、dispatch 规则、fallback 注意事项 |
| pytorch_test Framework 复现 | testing/ | 复现 shape/tolerance/Variant 选择等通用经验 |
| DLC Runtime 挂起、异步错误、设备残留 | runtime-debugging/ | 可复现的 runtime 排障流程 |
| vLLM attention/KV cache | vllm-dlc/ | attention backend 或 KV cache layout 相关问题 |
| 调试命令、trace 分析 | debugging-workflows/ | 可复用的调试命令和流程 |

3. 如果写 case study，按以下结构组织：
   - 问题现象（一句话，发生了什么）
   - 背景与环境（模型、对比方式、已知条件）
   - 定位路径（分阶段，记录每一步的排除和发现）
   - 最小边界或当前结论
   - 验证方式（可复现的命令和结果）
   - 可复用经验（3-5 条，让别人下次少走弯路）

4. 遵守以下写作规则：
   - 标题和文件名用 kebab-case 英文。
   - 正文用中文，技术术语保留英文原名。
   - 区分事实、经验和未验证假设。
   - 可复现的命令必须写完整的，不要用"运行测试"这种模糊描述。
   - 不要新建以模型名作为一级或二级目录的结构。
   - 敏感信息（路径中的用户名、API key 等）必须移除。

交付物：
- 建议写入的目录和文件名。
- case study 正文或专题补充内容。
- 可复用经验列表（用于快速查阅，不超过 5 条）。
```

**关键点**：知识库是按问题域组织的长期资产，不是某个模型的日志。只有可复用经验才写入，单次临时实验写 handoff 就够了。

---

## 如果不知道该用哪个套餐

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```
请先判断任务类型，再选择对应流程。

【我要做什么】<在 DLC 上跑一个新模型 / 模型输出不对 / 运行时卡住了 / 算子结果有差异>
【当前状态】<模型加载成功了吗？/ 推理能跑通吗？/ 输出大概差多少？>
【已有材料】<dump 路径、日志、已知 exact 的模块>

先读取 /work/chipltech-knowledge-base/CONTEXT.md 和 README.md，然后告诉我应该用哪个套餐。
```

---

## 快速上手三步走

1. **先确认环境能跑**（套餐一第一步 + 套餐一第二步）。环境都跑不通，后面都是浪费时间。
2. **先跑通再谈精度**（套餐一第三步 + 需要时用套餐二）。能出结果了再比输出对不对。
3. **从输出倒推，一次一个边界**（套餐三 + 套餐四）。不要一次改很多东西，不要跳到 kernel 内部。
4. **做完反哺**（套餐九）。可复用经验不回写，下次等于从零开始。

---

## 什么时候停

1. 环境检查都没过，不要开始加载模型。
2. 模型跑不通（卡住/报错），不要开始精度对比（套餐三）。
3. replay 不能 exact match saved tensor，不要写 pytorch_test 结论（套餐四）。
4. CPU fallback 修好后只说明这个算子是边界，不要声称"问题解决"。
5. 产出可复用经验后，不要只停在聊天总结，用套餐九反哺到对应问题域的文档或 case study。
