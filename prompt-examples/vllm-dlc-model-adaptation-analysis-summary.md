---
prompt_schema: vllm-dlc-report-prompt/v1
skill_identity: model-adaptation
required_user_inputs:
  - model_name
  - absolute_local_model_path
  - evidence_paths
derived_or_proposed_inputs:
  - audience_and_decision_questions
  - report_format
  - technical_attachment_path
discovery_policy: evidence_bound_then_reader_first_summary
missing_input_status: blocked
missing_input_reason: blocked_missing_evidence
---

# vLLM-DLC Model Adaptation Analysis Summary Prompt

## 用途

把已经收集的模型适配材料整理成面向指定读者的 `Decision Summary`，并保留可复核的 `Technical Attachment`。这是报告生产 prompt，不执行模型适配、根因诊断、benchmark、Real DLC Hardware qualification 或 image delivery。

## 可复制 Prompt

```markdown
请使用 `model-adaptation` 的知识边界，把以下模型适配材料整理成一份面向 `<AUDIENCE>` 的分析摘要。

模型名称：<MODEL_NAME>
模型目录：<ABSOLUTE_LOCAL_MODEL_PATH>
证据材料：
- <EVIDENCE_PATH_1>
- <EVIDENCE_PATH_2>

读者需要回答的决策问题：
1. <QUESTION_1>
2. <QUESTION_2>
3. <QUESTION_3>

报告要求：
1. 先建立 Evidence Ledger，将每条事实标为“运行已证明、源码已存在、参考观察、推断或未知”，并绑定 exact source/package/image/model/device identity 与 Claim Boundary。
2. 单独表达“路线可行性、适配完成度、稳定交付”三种状态。readiness、source presence、timeout、RPC error、HTTP success 和参考模型性能不得升级为更强结论。
3. 生成一份 Technical Attachment：保存模型结构、dtype/quantization、TP/EP、调用链、失败边界、真实 launch 条件、原始 evidence、性能来源和未验证范围。
4. 生成一份 Decision Summary：按读者决策问题排序，每个主体部分只保留一个主判断、最少支撑事实和明确边界。摘要不是技术附件的缩写流水账。
5. 算子表沿 `model forward -> framework op -> vLLM-DLC/PyTorch DLC Backend -> KernelDesc::launch("custom_xxx") -> DLC_Custom_Kernel Repository registry` 核验，使用实际 `custom_xxx` 或 kernel family。模块名只能作解释；DLCCL collective 单独表达，不混入模型算子缺口。
6. 量化格式、dtype、bits、group_size、zero_point、shape/alignment predicate 和调用阶段从目标 exact source/descriptor/registry 重新闭合。不要复制其他模型的 kernel name 或把同平台参考当成目标实测。
7. 性能数字必须绑定 input/output tokens、TP/EP、并发、单请求/服务吞吐、TPOT、TTFT 边界和来源身份。需要换算时使用 `token/s = 1000 / TPOT(ms/token)` 与 `纯 Decode 时间 = output_tokens * TPOT / 1000`，并明确这不是实测、SLA 或理论上限。
8. 做减法 review：删除过程流水账、重复身份、无信息量铺垫、重复 Claim Boundary、未服务决策问题的章节和未经证据支持的 owner/根因结论。
9. 最后列出：已回答的决策问题、仍未知的事实、建议的下一条最小验证动作，以及“本报告不证明什么”。

只有证据明确支持时才使用“已实现、已通过、根因、性能达到”等强表述；否则使用“源码存在、当前边界、候选、参考观察或未验证”。
```

## 完成标准

- Evidence Ledger 中每个摘要事实都有来源、身份和证据等级。
- 路线可行性、完成度和稳定交付没有混写。
- 每个算子名称都能追溯到实际 launch name 或明确 kernel family。
- 性能数字可按固定 workload 和公式复算，并标明 TTFT、吞吐和参考数据边界。
- 摘要按读者决策问题组织，技术附件保留复核细节。
- 所有未执行的 runtime、Real DLC Hardware、benchmark、image 和 alignment scope 都明确为未验证。

## 相关资料

- [Model Adaptation stable decisions](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)
- [Model Adaptation 可复用 Prompt](vllm-dlc-model-adaptation.md)
- [Model Adaptation Analysis Report Production](../case-studies/model-adaptation-analysis-report-production.md)
