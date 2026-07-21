---
prompt_schema: modelzoo-model-to-dlc-tyd-images/v1
minimum_input: model_name
optional_inputs:
  - framework_selector
  - model_path_override
claim_boundary: evidence_bounded_only
---

# ModelZoo Model 到 DLC Chip/TYD Chip Images Prompt

## 用途

只填写 ModelZoo model name，要求 Agent 直接执行只读解析、resolved manifest、独立 DLC Chip/TYD Chip 镜像 workflow、允许范围内的验证、导出和 evidence 收尾。缺少 evidence、adapter、硬件或授权时必须安全停止并返回结构化 blocked，不得用猜测补齐。

## 最少填写

```text
ModelZoo model name：<EXACT_MODEL_NAME>
```

可选输入：

```text
Framework selector：<例如 vllm；不提供则留空>
Model path override：<当前 Host 上已批准的绝对路径；不提供则留空>
```

## 可直接复制 Prompt

```md
请直接执行从 ModelZoo model 到独立 DLC Chip/TYD Chip images 的完整安全流程，不要只写计划。证据、framework adapter、资产、硬件或授权不足时，生成 manifest 和 blocker evidence 后安全停止，不得猜测、降级安全门或伪造结果。

最小输入：
- ModelZoo model name: <EXACT_MODEL_NAME>

可选输入：
- Framework selector: <FRAMEWORK_OR_EMPTY>
- Model path override: <APPROVED_ABSOLUTE_PATH_OR_EMPTY>

开始前读取并遵循：
- <KNOWLEDGE_BASE_ROOT>/CONTEXT.md
- <KNOWLEDGE_BASE_ROOT>/vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/host-daily-image-to-model-validation.md
- <KNOWLEDGE_BASE_ROOT>/debugging-workflows/post-install-runtime-smoke.md
- 当前环境中实际可用且与本任务相关的 skill `SKILL.md` 和脚本；先读取再调用，不发明不存在的 script flag

按以下 contract 执行：

1. 只读发现 authoritative ModelZoo root，默认候选为 `/home/xuansun/ModelZoo`，但必须以 Git root 验证。记录 root、remote、branch/tag、full HEAD 和 `git status --short`。不得修改、checkout、fetch、pull、reset、clean 或生成 ModelZoo 文件。
2. 按 `metafile.yml` 的 `Name` 精确匹配 model name。记录并稳定排序全部候选；无匹配返回 `blocked_model_not_found`；同名跨 framework/path 且 selector 不能唯一决定时返回 `blocked_ambiguous_model`。不得按目录顺序、README 标题或近似名称猜测。
3. 安全解析 selected `metafile.yml`，读取 selected README（存在时），记录两者相对路径和 SHA-256。YAML 异常、非 mapping 或关键字段类型异常返回 `blocked_malformed_metadata`。README 命令、Host、路径、component SHA 和 benchmark 结果均为 `historical_modelzoo_claim`，不是当前事实。
4. 采用以下优先级并保留所有冲突：user override > current Host observation > selected `metafile.yml` > selected README > top-level/historical ModelZoo claim。Override 不得覆盖文件不存在、ref 不可解析、错误硬件代际或缺少授权等事实；关键 claim 冲突无法裁决时返回 `blocked_conflicting_source_claims`。
5. 在生成 resolved manifest 前完成当前 Host 只读 preflight：验证模型路径和资产、approved component source/ref、base image identity、framework/package identity、硬件代际、目标资源占用和授权状态。可执行 current Host observation 必须由受保护本地 observer 签发；调用者 JSON、caller boolean 或单独 SHA-256 只能闭合 bytes，不能证明当前事实。resolver output 无论 trust class 都是 qualification-only，后续 action workflow 必须用独立 sealed action record 绑定资产、component source/ref、authorization 和 execution scope。fixture diagnostic key 的输出必须为 `action_eligible: false`，不可进入 action workflow。Model path override 优先于 README claim，但必须保留 README claim 与 `resolution_reason=user_override`；实际不存在时返回 `blocked_missing_asset`，不得下载、复制或替换模型。Component full ref 无法从 approved source 解析时返回 `blocked_unresolved_component_ref`，不得使用 latest、movable branch/tag 或其他 SHA。Supported adapter 缺 weight、component、serve、smoke 或 provenance 关键字段时返回 `blocked_missing_required_field`，并逐项列出缺失字段。
6. 完成第 5 步 observations 后，在任何 build、device execution 或模型加载前生成 deterministic `modelzoo-dlc-tyd-resolved-manifest/v1`。必须包含 ModelZoo Git identity、exact model/framework/path、metafile fields、README path/hash、source claims、当前 Host observations、resolved model path、component refs、required environment、serve/smoke contracts、framework adapter、missing fields、conflicts、claim boundary、blocked result 和 canonical `resolution_id`。数组稳定排序；相同输入/source/observation 必须得到相同 resolution ID。若第 1-5 步触发 blocker，仍写出 `resolution_status: blocked` 的 manifest 后停止。
7. 只对有明确 adapter contract 的 framework 执行。当前完整 adapter 是 `vllm/v1`；SGLang、Transformers、PyTorch、NeMo、Diffusers、KTransformers 和其他 framework 完成 manifest 后返回 `blocked_unsupported_framework`，不得套用 vLLM workflow 或声称全部 framework supported。`Infer: false` 保留为 ModelZoo claim，不自动提升为可部署。
8. 对 resolved vLLM entry 建立两个相互独立的 target contract。DLC Chip 与 TYD Chip 必须使用不同 fixed tag、Image ID、tar、tar SHA-256、attestation 和 validation report；`latest` 不能作为交付身份，模型权重不得写入 image。先回显 base image、source refs、build scope、输出目录和所需授权；缺少必要授权返回 `blocked_missing_authorization`。
9. DLC Chip target 获得 build/export 授权后直接构建和导出，并按 Host 模板执行：先 C1a package/import；仅在 DLC Chip identity、设备归属、占用、授权和 fresh-process gate 满足时执行 C1b；mandatory run 缺少匹配硬件或资源时返回 `blocked_missing_hardware`；仅在 C1b 通过后执行最小 vLLM model functional smoke。Smoke 至少分别记录 liveness、请求完成、非空输出和可观察 correctness。Benchmark 需要独立 workload 和授权，未执行则标记 `not_verified`，不能由 smoke 推导。
10. DLC Chip C1b 必须 fresh process 分层覆盖 allocation、H2D、nontrivial device operation、synchronize、D2H 和 correctness。只看到 package import、backend availability、device enumeration、allocation 或 H2D 不构成 C1b。执行服务时保存精确 process identity、端口和 sealed HBM baseline；结束后只停止本任务 APIServer、EngineCore、workers 和 clients，确认端口、handles 和任务 HBM 增量回到 baseline。
11. TYD Chip target 必须证明完整 vLLM 栈的每个实际编译进程继承 `DLC_TPU_VERSION=2`，至少包括 dlc-thunk、LLVM、DLCsim、DLCSynapse、DLC_CL、DLC_Custom_Kernel Repository、PyTorch DLC Backend 和 vLLM。每个组件记录 source full SHA、build process environment evidence 和 artifact hash。仅在 image config 添加 `ENV DLC_TPU_VERSION=2` 不充分；缺任一 required component provenance 时返回 `blocked_missing_required_field` 并列出缺失 provenance，不得使用正式 TYD Chip tag、发明 partial validation state 或声称 full-stack build PASS。
12. 在 DLC Chip Host 上，TYD Chip image 只允许不触发 device execution 的 static/package/import、artifact hash、image label 和 attestation checks。严禁执行 TYD Chip device operation、C1b、DLCCL、模型加载、serving 或 benchmark；这些状态必须精确报告 `intentionally_not_executed_on_dlc_gen1`，不能写普通 pending、功能 PASS 或 TYD Golden Baseline。只有已证明位于 TYD Chip Host 且另有明确执行授权时，才可按独立 contract 执行 TYD Chip functional smoke。
13. 未明确授权时，禁止下载模型权重或依赖、更新 Host driver、restart service、reset/reboot、push registry、覆盖 dirty tree、kill 无关进程、prune 或抢占设备。自动发现不是变更授权。任何禁止动作成为继续条件时返回 `blocked_missing_authorization`，列出所需授权类别后停止。
14. 分层报告状态，不混合或升级：`c1a_package_import_pass`、`c1b_runtime_execution_pass`、`static_package_pass`、`model_functional_pass`、`benchmark_pass`、`not_verified`、`blocked_*`、`intentionally_not_executed_on_dlc_gen1`。Model functional PASS 不证明 benchmark；static/package PASS 不证明 C1b；ModelZoo 历史结果不证明当前 Host PASS。
15. 每套交付报告 fixed tag、Image ID、image configuration、tar path/size/SHA-256、source/base provenance、build attestation、validation report 和 cleanup evidence。精确清理本任务创建且已记录 identity 的 builder/static-check containers、临时 staging tags、临时 build directories 和未完成 export artifacts；保留正式交付物、logs 和失败 epoch evidence。不得删除预先存在或不属于本任务的资源。进程、端口、HBM 或 build/export resource 未回到 sealed cleanup contract 时返回 `blocked_cleanup_incomplete`，不得声称交付闭环。Registry digest 仅在另获批准并实际 push/pull 后记录，不得提前猜测。
16. 最终报告严格分栏：`modelzoo_claims`、`current_observations`、`inferences`、`execution_evidence`、`unverified_scope`。同时输出 supported/blocked/prohibited matrix、所有 meaningful failures/fixes、artifact 路径和剩余风险。不得把本文 prompt 或 static checks 称为真实 image 或 hardware validation evidence。
```

## 状态速查

| Supported | Blocked | Prohibited |
|---|---|---|
| ModelZoo 只读 exact discovery、manifest；完整 vLLM adapter；获授权的独立 build/export | missing model、ambiguity、malformed/conflicting metadata、missing field/asset/ref/hardware/authorization、unsupported framework | 修改 ModelZoo、猜默认值、自动下载/Host 变更、在 DLC Chip 上执行 TYD device scope、未授权 push |

## 相关资料

- [ModelZoo 驱动的 DLC/TYD 镜像与验证 Contract](../vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md)
- [Host 每日镜像到模型验证](host-daily-image-to-model-validation.md)
- [新模型验证 quickstart](new-model-validation-quickstart.md)
- [安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md)
