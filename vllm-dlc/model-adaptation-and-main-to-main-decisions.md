# vLLM-DLC 模型适配与 Main-to-Main 决策记录

shared_contract: vllm-dlc-contract/v1

## 适用场景

本文记录 Model Adaptation 与 Main-to-Main Upgrade 产品化中的稳定领域决策、历史观察和未验证项。术语以 [CONTEXT.md](../CONTEXT.md) 为准；执行顺序和停止语义由 skills-owned public seam 与 stable skills 维护，本文不复制命令、接口路径、schema grammar 或 executable assertions。

## 当前实现范围

- **事实 / Fact**：Ticket 01-07 的当前实现只位于 skills repository 与 chipltech-knowledge-base repository。
- **事实 / Fact**：当前 candidate 对 vllm-dlc repository 严格 read-only；不修改源码、manifest、alignment、metadata、branch、index 或 generated files。
- **事实 / Fact**：Model Adaptation 与 Main-to-Main Upgrade 已由 Ticket 07 发布为 stable Kilo engineering skills；它们仍保持 read-only、report-only/no-finalize 边界，不提升 Ticket 06 evidence。
- **目标架构决策 / Target architecture decision**：未来可由 vllm-dlc repository 持有 deterministic DLC Runtime contracts、instrumentation、manifest、alignment 和 finalization backend。这不是当前 Ticket 01-05 的实现事实。
- **建议 / Recommendation**：知识文档只保存稳定决策和证据含义；可变接口细节由 `shared_contract: vllm-dlc-contract/v1` 的 public seam 单独维护。

## 历史观察与可信度

- **事实 / Fact**：当前 `vllm_alignment.yaml` 的 vLLM version、commit 和 vllm-dlc branch 均为 `unknown`，不能提供 Main-to-Main 的旧 commit。
- **事实 / Fact**：vllm-dlc Git 历史显式记录过 upstream commit `2488d1dca2df05059fcfbad0a1612ef2a5202b47`。
- **经验 / Experience**：upstream commit `3072ed636a8993f69e6c2ab4d4a90bb50f04ab81` 与 vllm-dlc profiler 提交存在时间和主题关联，但历史没有显式声明它是 Verified vLLM Alignment。
- **事实 / Fact**：规划时的本地 vLLM checkout `a208f41eee15d15b0da619ded9384fda5efd2e7f` 晚于 alignment 文件初建日期，不能证明初始 alignment。
- **经验 / Experience**：跨仓提交时间、主题和代码关系可提高候选可信度，但不能替代 mandatory sealed evidence。

恢复候选按以下稳定可信度排序：历史完整 mandatory evidence、显式 Git pin、相关性候选、checkout clue、installation clue、README clue。只有第一类可直接建立历史 Verified vLLM Alignment；其余候选都需要同等 mandatory evidence。upstream full SHA 是 exact identity，tag 仅作 lineage metadata。

## Evidence 分类

证据不是一条可自动升级的线性 ladder。必须分别声明 execution environment、model asset identity、diagnostic/acceptance eligibility 和 status。

| Evidence class | 稳定含义 | 明确不声称 |
|---|---|---|
| Static validation | 文档、package、schema、routing 和 identity consistency | 不执行模型、API、DLC Runtime、weights 或硬件 |
| Fake-server validation | 在受控 server 上验证 shared seam orchestration、protocol parsing 和 lifecycle | 不证明 real model、Chunked Prefill DLC Runtime、DLC Runtime dispatch 或硬件 acceptance |
| Dummy diagnostics | 合格 real-weight failure 后显式选择的 diagnostic wiring | 永不作为 real-weight/hardware acceptance，永不 finalize-eligible |
| DLCsim evidence | 对声明范围的 simulator execution | 不证明物理 Real DLC Hardware 行为或 attestation |
| Real-weight evidence | 使用批准 weights 与 exact model/deployment identity | 缺少合格硬件与 DLC Runtime evidence 时不证明 Real DLC Hardware acceptance |
| Real DLC Hardware acceptance | 物理硬件执行且全部 mandatory sealed gates 具备合格 attestation | 不外推到其他 model、profile、target、candidate 或 evidence identity |

- **事实 / Fact**：分配给 requested mandatory run 的硬件缺失是 `blocked_missing_hardware`。
- **事实 / Fact**：本阶段没有执行的硬件依赖工作是 `not_verified`，不是 `blocked_missing_hardware`。
- **事实 / Fact**：`not_verified` 不等于 `not_applicable`。
- **事实 / Fact**：Static、fake-server、Dummy、DLCsim、HTTP success 或 long-input construction 不得提升 Real DLC Hardware 或 DLC Runtime acceptance。

## 新模型 serving 验证经验

- **建议 / Recommendation**：新模型功能验证先从短 prompt、`temperature=0`、`top_p=1.0` 和小 `max_tokens` 开始，只验证模型加载、API 返回非空输出和 server liveness。
- **建议 / Recommendation**：短 prompt 通过后，再逐步扩大到中等 prompt、长 prompt、one-shot/CoT、采样温度、Chunked Prefill 或并发；每轮只改变一个变量。
- **事实 / Fact**：`--served-model-name` 会改变 OpenAI-compatible API 中可用的模型名；请求 JSON 的 `model` 必须使用该 alias，否则可能出现 model not found / 404，这不是模型文件缺失。
- **经验 / Experience**：Qwen3.5-27B 类模型可能对 prompt 长度和 one-shot/CoT 结构敏感，表现为重复 `!`、空输出、长解释、超时或答案截断。此类现象应作为模型/参数/长上下文风险记录，不得把短 prompt smoke 通过提升为长上下文或 one-shot 已验证。
- **建议 / Recommendation**：量化与 MoE 模型适配必须核对 `quant_method`、`bits`、`group_size`、`zero_point`、processor/tokenizer 类型和实际 kernel 路由。`compressed-tensors`、AWQ、AWQ-Marlin、W4A16、W8A16 不能只按目录名判断兼容。
- **事实 / Fact**：如果当前 DLC 软件栈缺少目标量化/MoE fused kernel，Python 层绕路或修改模型 config 不构成长期适配完成；应报告为 kernel capability gap 或 `not_verified`，并说明需要 DLC Custom Op / DLC_Custom_Kernel Repository 支持。
- **建议 / Recommendation**：需要 serving 稳定性或性能补充证据时，可引用 Arsenal 的 vLLM benchmark 和黑盒 HTTP 测试入口；这些结果应作为 serving 层 evidence 记录，不得提升为 Verified vLLM Alignment 或 Real DLC Hardware acceptance。

## Skill 所有权

### Model Adaptation

- **事实 / Fact**：只由一个特定模型的 load、weights、Attention、MLA、MoE、quantization、multimodal、MTP 或 distributed compatibility 任务触发，或接受 Main-to-Main 的单向 child assignment。
- **事实 / Fact**：TP 来自批准 weights、模型配置、dtype、quantization、capacity 和 deployment profile；固定 regression assignment 不是任意模型默认值。
- **事实 / Fact**：Dummy 仅在合格 real-weight failure 和显式批准后用于 diagnostic-only。
- **事实 / Fact**：stable skill 不写 compatibility source 或 manifest，也不恢复、更新、finalize 或声称 Verified vLLM Alignment。

### Main-to-Main Upgrade

- **事实 / Fact**：只由 exact upstream full SHA alignment、unknown alignment recovery 或 complete global compatibility impact analysis 触发。
- **事实 / Fact**：changed surface 必须分类为 affected dependency、new dependency candidate 或 confirmed irrelevant；unresolved impact 保持 blocker。
- **事实 / Fact**：manifest impact 当前仅为 future-change report。模型专属工作仅通过 Ticket 03 Model Adaptation child seam 单向委派和回收 sealed handoff。
- **事实 / Fact**：DeepSeek TP=2 与 Llama TP=1 的 exact Ticket 06 v12 assignments 已完成 operational regression；该结果仍为 `operational_only` 且 `acceptance_eligible: false`。
- **事实 / Fact**：stable skill 不修改、commit 或 finalize，alignment outcome 保持 unchanged。

## 当前未验证项

### Ticket 06 operational amendment

- **已批准决策 / Approved decision**：Ticket 06 产出较弱的 **Real DLC Hardware operational evidence**，不产出 Real DLC Hardware acceptance。
- **事实 / Fact**：该 evidence 证明 real-weight API/lifecycle 行为与 query-only、non-excluded physical-device、runner-owned process occupancy observation 同时成立；不声称完整 device health assessment。
- **事实 / Fact**：该 evidence 不证明 request-correlated Chunked Prefill、DLC Runtime dispatch、DLCCL/LYP、具体 Attention path、Triton non-execution 或 compile/Dynamo non-execution。
- **事实 / Fact**：fixture、fake-server、Dummy、DLCsim、static、unknown-provider 和手工 result 不能贡献 operational completion。
- **事实 / Fact**：v2 operational result 始终 `acceptance_eligible: false`，不能建立或 finalize Verified vLLM Alignment。
- **事实 / Fact**：Ticket 06 只修改 skills repository 和本知识库的稳定决策文字；vLLM、vllm-dlc、PyTorch DLC Backend、DLCSynapse、DLC Runtime、DLCCL、DLC Custom Kernel 与 `chipltech_smi_lib` 保持 read-only。
- **事实 / Fact**：lease、signature、trusted time、revocation、runtime-stream binding、atomic allocation 和跨仓 instrumentation 属于未来更强 acceptance class，不阻塞 operational regression。
- **已批准决策 / Approved decision**：内部修改的模型、tokenizer 和 processor 不要求对应上游 Git 或 Hugging Face revision。Ticket 06 operational v2 使用批准的 exact local path 与 recursive byte digest 闭合执行资产身份；revision 仅在独立已知时记录，否则为 null，且不得推测。vLLM 与 vllm-dlc guarded source identity 仍使用 full Git SHA。
- **已批准决策 / Approved decision**：production hardware observation 由 skills-owned normalization adapter 调用官方默认版 `cltech_smi`，不从 raw sysfs 自行重建 vendor 查询语义，也不把 vendor executable 复制进 skills repository 或知识库。
- **事实 / Fact**：完整运行容器需 privileged、host PID namespace，并挂载 host `pci.ids`、`/dev`、`/sys`、`/run`、`/lib/modules`、`/var/log`；当前容器缺少部分挂载只表示当前环境不能完成全部 SMI 功能，不表示未来 skill 无法观测硬件。
- **建议 / Recommendation**：优先使用 DLC base image 或 host payload 已安装的 `cltech_smi`；若缺失，环境准备可 clone `git@github.com:ChipLTech/chipltech_smi_lib.git` 并按 README build/install 默认版，随后冻结 source full SHA 与 executable digest。Ticket 06 runner 仅执行 allowlisted query，维护和诊断 action 不在回归路径内。
- **事实 / Fact**：SMI normalization 必须闭合 indexed finite positive HBM capacity、run-local device reference 和 runner process-group occupancy；device/process query failure 必须 fail closed。cleanup 同时要求原 runner process group 为空，且 post-cleanup device PID 不得超出 sealed pre-launch baseline；共享 host 的 baseline occupancy 不能贡献 run gate。

- **未验证 / Not verified**：当前 Verified vLLM Alignment 仍是 unknown。
- **事实 / Fact**：exact Ticket 06 v12 profiles 的批准 production regression policy identity、真实模型资产、real-weight API/lifecycle 行为和 query-only Real DLC Hardware operational observation 已通过 sealed public validation。
- **未验证 / Not verified**：Real DLC Hardware acceptance 与 authoritative attestation。
- **未验证 / Not verified**：request-level Chunked Prefill DLC Runtime evidence。
- **未验证 / Not verified**：DLC Runtime dispatch evidence。
- **未验证 / Not verified**：Triton non-execution evidence。
- **未验证 / Not verified**：compile/Dynamo non-execution evidence。

这些更强状态只能由后续 eligible sealed evidence 改变。本文、两个 reusable prompt 和 prompt dry run 都不是 execution evidence；Ticket 06 v12 evidence 只绑定 exact profiles，不能继承到新 target。

## Out of Scope

- Ticket 07 publication 阶段不运行模型、server、DLCsim 或 Real DLC Hardware。
- 本阶段不实现或复制 shared runner、DLC Runtime instrumentation、manifest grammar 或 finalization backend。
- 本阶段不修改 vllm-dlc，不更新 alignment，不 commit、push 或改变 runtime repository。
- Ticket 06 已完成 exact v12 operational regression；Ticket 07 仅发布 stable Kilo skill surfaces。

## 相关资料

- [Model Adaptation reusable prompt](../prompt-examples/vllm-dlc-model-adaptation.md)
- [Main-to-Main Upgrade reusable prompt](../prompt-examples/vllm-dlc-main-to-main-upgrade.md)
- [precision-debugging/token-divergence-and-moe-contract-debugging.md](../precision-debugging/token-divergence-and-moe-contract-debugging.md)
- [testing/arsenal-ci-and-blackbox-testing.md](../testing/arsenal-ci-and-blackbox-testing.md)
