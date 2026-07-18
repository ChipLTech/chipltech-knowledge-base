---
prompt_schema: vllm-dlc-reusable-prompt/v1
skill_identity: main-to-main-upgrade
shared_contract: vllm-dlc-contract/v1
required_inputs:
  - target_vllm_full_sha
  - lineage_tag
  - guarded_repository_snapshot
  - baseline_candidates
  - history_range_evidence
  - changed_surface_inventory
  - unresolved_impact_state
  - manifest_impact_identity
  - regression_policy_state
  - deepseek_tp2_assignment
  - llama_tp1_assignment
  - model_adaptation_handoffs
  - artifact_evidence_references
  - commit_authorization_state
missing_input_status: blocked
missing_input_reason: blocked_missing_target
hardware_evidence: not_verified
---

# vLLM-DLC Main-to-Main Upgrade 可复用 Prompt

## 用途

用于 exact upstream alignment、alignment recovery 或全局 compatibility impact 分析。它选择 stable `main-to-main-upgrade` skill 并保持 report-only、no-finalize。

## 使用方法

替换所有尖括号占位符。target 必须是完整 SHA；tag 只作 nullable lineage metadata。

## 可复制 Prompt

```markdown
选择 `main-to-main-upgrade`，只处理 exact upstream alignment、Verified vLLM Alignment recovery 或 global compatibility impact。

目标与仓库快照：
- exact target upstream full SHA: <TARGET_VLLM_FULL_SHA>
- lineage tag（nullable）: <LINEAGE_TAG_OR_NULL>
- guarded repository path / branch / HEAD / status / read-only snapshot identity: <GUARDED_REPOSITORY_SNAPSHOT>

恢复与影响输入：
- baseline candidates and evidence references in confidence order: <BASELINE_CANDIDATES>
- complete history/range evidence: <HISTORY_RANGE_EVIDENCE>
- changed-surface inventory: <CHANGED_SURFACE_INVENTORY>
- unresolved-impact state: <UNRESOLVED_IMPACT_STATE>
- read-only manifest-impact identity: <MANIFEST_IMPACT_IDENTITY>
- regression policy state: <REGRESSION_POLICY_STATE>

强制 assignment 与交接：
- DeepSeek real-weight TP=2 assignment identity: <DEEPSEEK_TP2_ASSIGNMENT>
- Llama real-weight TP=1 assignment identity: <LLAMA_TP1_ASSIGNMENT>
- Ticket 03 Model Adaptation child handoff references: <MODEL_ADAPTATION_HANDOFFS>
- artifact/evidence references: <ARTIFACT_EVIDENCE_REFERENCES>
- commit authorization state: <COMMIT_AUTHORIZATION_STATE>

先列出缺失输入并停止；缺 target 使用 `blocked_missing_target`。保持 unknown baseline 为 unknown，并按历史 mandatory evidence、explicit Git pin、correlated candidate、checkout/install/README clue 的可信度顺序报告。使用 `model-adaptation` 作为唯一模型专属 child seam。通过 `shared_contract: vllm-dlc-contract/v1` 的 skills-owned public seam 引用确定性检查。manifest 只生成 future-impact report；不得修改 vllm-dlc、commit、finalize、写 metadata 或声称新的 Verified vLLM Alignment。Ticket 06 exact v12 DeepSeek TP=2 与 Llama TP=1 只证明 bounded operational state；不要把该 evidence 继承到此新 target。未执行的 real weights、Real DLC Hardware、Chunked Prefill runtime 和 DLC Runtime dispatch 均报告 `not_verified`。
```

## 停止语义与 Evidence

- target 缺失时停止为 `blocked_missing_target`；其他 unresolved inputs 不得被推断为已满足。
- 当前 prompt dry run 不执行 runner、模型或硬件，也不产生 acceptance 或 finalize eligibility。
- fake-server、Dummy、DLCsim、静态检查或 HTTP success 不能建立 mandatory acceptance。
- Ticket 06 exact v12 profiles 只证明 `authoritativeness: operational_only`、`acceptance_eligible: false`、alignment unchanged、manifest report-only 和 finalization `none`，不证明新的 Verified vLLM Alignment。

## 相关资料

- [模型适配与 Main-to-Main 决策记录](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)
