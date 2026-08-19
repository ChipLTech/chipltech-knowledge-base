# vLLM-CL 模型适配与 Main-to-Main 决策记录

shared_contract: vllm-cl-contract/v1

## 适用场景

本文记录 Model Adaptation 与 Main-to-Main Upgrade 产品化中的稳定领域决策、历史观察和未验证项。术语以 [CONTEXT.md](../CONTEXT.md) 为准；执行顺序和停止语义由 skills-owned public seam 与 stable skills 维护，本文不复制命令、接口路径、schema grammar 或 executable assertions。

## 当前实现范围

- **事实 / Fact**：Ticket 01-07 的当前实现只位于 skills repository 与 chipltech-knowledge-base repository。
- **事实 / Fact**：当前 candidate 对 vllm-cl repository 严格 read-only；不修改源码、manifest、alignment、metadata、branch、index 或 generated files。
- **事实 / Fact**：Model Adaptation 与 Main-to-Main Upgrade 已由 Ticket 07 发布为 stable Kilo engineering skills；它们仍保持 read-only、report-only/no-finalize 边界，不提升 Ticket 06 evidence。
- **目标架构决策 / Target architecture decision**：未来可由 vllm-cl repository 持有 deterministic DLC Runtime contracts、instrumentation、manifest、alignment 和 finalization backend。这不是当前 Ticket 01-05 的实现事实。
- **建议 / Recommendation**：知识文档只保存稳定决策和证据含义；可变接口细节由 `shared_contract: vllm-cl-contract/v1` 的 public seam 单独维护。

## 历史观察与可信度

- **事实 / Fact**：当前 `vllm_alignment.yaml` 的 vLLM version、commit 和 vllm-cl branch 均为 `unknown`，不能提供 Main-to-Main 的旧 commit。
- **事实 / Fact**：vllm-cl Git 历史显式记录过 upstream commit `2488d1dca2df05059fcfbad0a1612ef2a5202b47`。
- **经验 / Experience**：upstream commit `3072ed636a8993f69e6c2ab4d4a90bb50f04ab81` 与 vllm-cl profiler 提交存在时间和主题关联，但历史没有显式声明它是 Verified vLLM Alignment。
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

### Model Adaptation Analysis Reporting

模型适配分析的执行证据与对外报告是两个产品：先建立 Evidence Ledger，再把技术细节下沉到 Technical Attachment，最后按读者决策问题生成独立的 Decision Summary。报告必须分别表达“路线可行”“当前完成到哪一阶段”和“是否达到稳定交付”，不能把 readiness、源码存在、timeout 或参考性能升级为适配完成、根因或实测。

算子适配表使用实际 `KernelDesc::launch("custom_xxx")` 或明确的 kernel family，并沿 vLLM/vLLM-CL、PyTorch DLC Backend、KernelDesc、DLC_Custom_Kernel Repository registry 闭合来源。模型模块名只作解释；DLCCL collective 不混入模型算子表；同平台其他模型的 kernel name、量化格式或 shape predicate 只能作为检索线索，不能直接类推。

性能摘要同时给出原始口径和可重算换算：固定 input/output token policy、TP/EP、并发、TPOT、单请求 Decode token/s、是否包含 TTFT，以及固定输出长度的纯 Decode 时间。token/s 不是服务总吞吐，TTFT 不能由 input tokens 乘 TPOT 推导，参考模型数据不是目标模型实测或 SLA。

详细生产步骤和可复制模板见 [Model Adaptation Analysis Summary](../prompt-examples/vllm-cl-model-adaptation-analysis-summary.md)；该 prompt 只生成报告，不替代 `model-adaptation` 的兼容性分析、`diagnosing-bugs` 的根因诊断或 Real DLC Hardware qualification。

## 跨仓变更归属与 ABI 配对

一次模型适配或 Main-to-Main 调查可以覆盖多个仓库，但调查范围、修改范围和 PR 数量是三个不同集合。没有 patch 的仓库也应保留调查结论；“本轮不修改”可以是版本配对分析后的设计结果，不是遗漏。

| 变更面 | 首要 owner | 稳定判断规则 |
|---|---|---|
| 通用模型 API、日志或跨后端框架行为 | vLLM | 对所有后端成立的修复优先进入 upstream framework，不放入 DLC 专用 patch |
| DLC worker 生命周期、DLC model patch、plugin registration | vLLM-CL | 只承载 DLC Platform integration，不重复 upstream 或 fused layer 已拥有的 collective |
| public operator schema、PyTorch DLC dispatch、host wrapper、KernelDesc packing | PyTorch DLC Backend 或当前实际 extension owner | 先按 source/packaging identity 确认 owner，不用历史目录布局猜测 |
| DLC Custom Kernel 实现、entry ABI、variant 和 kernel metadata | DLC_Custom_Kernel Repository | 静态审计确认 source gap 后记录缺失；只有要把它认定为当前 workload 的必要修复时才要求运行必要性 evidence |
| compiler、DLCSynapse、DLC Runtime、DLCCL 或其他 native binary | 对应 native component | 代码存在但 build/runtime artifact 不配对时，分类为版本配对，不写成源码缺失 |

每个变更面至少记录 repository full SHA、dirty state、source owner、consumer owner、change kind、patch/diff identity、build artifact identity、验证 owner 和未验证范围。PR 按仓库责任和单一主要目的拆分，不按交付 patch 文件数量机械拆分。

### Tested Revision 与 Publication Candidate

Tested Revision 是实际产生 build/runtime evidence 的 checkout；Publication Candidate 是从当前目标 main 的隔离 clean worktree 构造的交付 revision。二者必须分别记录 full SHA、base、dirty state、scoped diff/tree 和 artifact/validation identity。验证 checkout 可以保留用于调查的 merge history 或无关现场文件，Publication Candidate 只能包含批准 scope，不能通过原地 squash 破坏 primary evidence。

Patch Equivalence 证明声明 scope 的净差异关系，不自动转移 acceptance。stable patch-id 是 supporting evidence；还要核对完整 scoped diff/tree、排除无关路径，并在 base 变化后重跑受影响 contract/source 回归。若依赖解析、ABI、toolchain、native stack 或 build graph 可能变化，则重新 build 和执行相应 runtime gates。

远端并发推进是 publication identity 变化，不是可忽略的 push race。旧 base 或 PR tip 的 exact lease 应阻止发布；随后基于新 main 重建 candidate、更新 identity 并重跑受影响门禁。改写现有远端 PR history 需要独立显式授权，只允许 exact expected old SHA 的 `--force-with-lease`；无 lease force push 不属于批准路径。read-only/no-finalize Skill 只报告这些门禁，不创建 worktree、commit 或 push。

### Public Schema、Descriptor 与 Kernel Entry

Public Operator Schema、KernelDesc Descriptor ABI 和 DLC Custom Kernel Entry ABI 的正式定义见 [CONTEXT.md](../CONTEXT.md)。三层必须分别核验，并与 exact source、adapter 和 binary identity 绑定。

- **事实 / Fact**：optional 参数值为 `None` 不自动证明底层 descriptor slot 消失；slot 是否存在由 host adapter 构造决定。
- **经验 / Experience**：main public schema 已演进而冻结 DLC Custom Kernel 仍使用旧 entry ABI 时，直接按新 descriptor 发射可能发生位置错位，并在异步 completion 边界表现为与表面算子无关的 failure。
- **建议 / Recommendation**：需要保持 caller source compatibility 时，保留 Public Operator Schema，在 host adapter 中按 exact DLC Custom Kernel Entry ABI 构造 descriptor；未由 exact DLC Custom Kernel revision 证明的 metadata、routing、scoring、quantization mode 或 group size 必须在 launch 前 fail closed。
- **建议 / Recommendation**：capability matrix 绑定 exact source、adapter 和 kernel binary identity。`unsupported`、`not_verified` 与“硬件永久不支持”是不同结论。
- **建议 / Recommendation**：对 descriptor slot 顺序、collective ownership 和 lifecycle barrier 等结构性不变量使用 source/AST regression；它们不替代 build、C1b、collective correctness、模型功能和 Real DLC Hardware evidence。

### 主线缺失的双证据

静态审计足以记录一个 surface 在 exact HEAD 中缺失。只有同时满足以下条件，才能进一步把该缺失认定为当前 workload 的必要失败边界或必要修复：

1. 在 exact repository HEAD 和声明的搜索范围内有可审计的静态缺失证据。
2. 在 exact artifact identity 上有运行证据证明该缺失是目标 workload 的必要失败边界。

如果代码已经存在但 source、descriptor、DLC Custom Kernel、toolchain 或 native stack 不配对，应记录为 `artifact/ABI pairing`。如果旧 patch 没有原样进入 main，但当前实现已有替代路径或运行不再需要它，应记录为 `confirmed present`、`superseded` 或 `confirmed irrelevant`，不能把“旧 patch 不存在”写成当前 workload 的已证实缺口。

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
- **事实 / Fact**：freeze report 必须记录 Tested Revision；提出 publication 时再单独记录 Publication Candidate、base、Patch Equivalence、受影响回归、remote lease 和 publication authorization。纯 read-only impact analysis 不虚构 candidate，这些门禁本身也不授权 commit 或 push。

## 当前未验证项

### Ticket 06 operational amendment

- **已批准决策 / Approved decision**：Ticket 06 产出较弱的 **Real DLC Hardware operational evidence**，不产出 Real DLC Hardware acceptance。
- **事实 / Fact**：该 evidence 证明 real-weight API/lifecycle 行为与 query-only、non-excluded physical-device、runner-owned process occupancy observation 同时成立；不声称完整 device health assessment。
- **事实 / Fact**：该 evidence 不证明 request-correlated Chunked Prefill、DLC Runtime dispatch、DLCCL/LYP、具体 Attention path、Triton non-execution 或 compile/Dynamo non-execution。
- **事实 / Fact**：fixture、fake-server、Dummy、DLCsim、static、unknown-provider 和手工 result 不能贡献 operational completion。
- **事实 / Fact**：v2 operational result 始终 `acceptance_eligible: false`，不能建立或 finalize Verified vLLM Alignment。
- **事实 / Fact**：Ticket 06 只修改 skills repository 和本知识库的稳定决策文字；vLLM、vllm-cl、PyTorch DLC Backend、DLCSynapse、DLC Runtime、DLCCL、DLC Custom Kernel 与 `chipltech_smi_lib` 保持 read-only。
- **事实 / Fact**：lease、signature、trusted time、revocation、runtime-stream binding、atomic allocation 和跨仓 instrumentation 属于未来更强 acceptance class，不阻塞 operational regression。
- **已批准决策 / Approved decision**：内部修改的模型、tokenizer 和 processor 不要求对应上游 Git 或 Hugging Face revision。Ticket 06 operational v2 使用批准的 exact local path 与 recursive byte digest 闭合执行资产身份；revision 仅在独立已知时记录，否则为 null，且不得推测。vLLM 与 vllm-cl guarded source identity 仍使用 full Git SHA。
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
- 本阶段不修改 vllm-cl，不更新 alignment，不 commit、push 或改变 runtime repository。
- Ticket 06 已完成 exact v12 operational regression；Ticket 07 仅发布 stable Kilo skill surfaces。

## 相关资料

- [Model Adaptation reusable prompt](../prompt-examples/vllm-cl-model-adaptation.md)
- [Main-to-Main Upgrade reusable prompt](../prompt-examples/vllm-cl-main-to-main-upgrade.md)
- [precision-debugging/token-divergence-and-moe-contract-debugging.md](../precision-debugging/token-divergence-and-moe-contract-debugging.md)
- [testing/arsenal-ci-and-blackbox-testing.md](../testing/arsenal-ci-and-blackbox-testing.md)
- [case-studies/qwen3-32b-dlc-block256-diagnosis.md](../case-studies/qwen3-32b-dlc-block256-diagnosis.md)
- [case-studies/vllm-fused-moe-schema-kernel-abi-boundary.md](../case-studies/vllm-fused-moe-schema-kernel-abi-boundary.md)
- [case-studies/host-api22-fullstack-main-to-main-update.md](../case-studies/host-api22-fullstack-main-to-main-update.md)

## Qwen3-32B Block-256 Campaign 经验 (2026-07-28)

### 经验 / Experience

Qwen3-32B (64 layers, BF16, TP4, block-256) 在 DLC Platform 上出现稳定但错误的算术输出 (`2+2=` → `5`)。通过 ordered seam diagnostic (H001 → H008) 和 Qwen3-1.7B cross-model comparison，确认一个独立问题并形成一个待验证假设：

1. **DLCCL o_proj all-reduce 数值偏差** (max_abs_diff 0.015625) — code-only repro 已存在
2. **待验证的 64 层 BF16 累积误差假设** — 每层 diff ≤ 0.002，可能在 64 层叠加后改变 final logits top-1；当前 cross-model comparison 仅提供支持性证据

1.7B (28 layers) 在相同环境下算术正确，排除 DLC Platform 全局故障。

Full diagnosis: [case-studies/qwen3-32b-dlc-block256-diagnosis.md](../case-studies/qwen3-32b-dlc-block256-diagnosis.md)

### 建议 / Recommendation

对新模型的 ordered seam diagnostic 建议按下述优先级执行: loaded-weight integrity → fused ops (synthetic) → RMSNorm → rank-local GEMMs → TP collectives (one-at-a-time) → CPU collective diagnostic probe → real forward Model-Site Dump → cross-model comparison。Cross-model comparison 是快速区分平台全局故障与模型规模相关问题的高效手段。
