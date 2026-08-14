# Model Adaptation Analysis Report Production

## 案例范围

本案例复盘 2026-08-13 一次 Hy3 GPTQ-Int4 适配分析报告从长版技术材料压缩为决策摘要的过程。它沉淀报告生产方法，不建立 Hy3 当前 runtime、语义正确性、性能或 Real DLC Hardware acceptance。

Evidence status: `historical_report_derived_observation`。原始个人过程资产当前为 `external_unavailable`，目标 source/package/model/artifact identity 无仓内 manifest 可复核，故为 `identity_unavailable`；不得猜填缺失 SHA/digest。

不可继承：本案例不能自动成为 approved Stack Policy、current topology qualification、current model acceptance 或 current performance baseline。

## 最有价值的通用规则

### 1. 先建 Evidence Ledger

把输入分成五类：

| 类别 | 可支持的表达 |
|---|---|
| 运行已证明 | 在 exact artifact、model、profile 和设备条件下实际观察到的结果 |
| 源码已存在 | source、registry、schema 或 launch path 的静态事实 |
| 参考观察 | 其他模型、历史 run 或同平台材料提供的锚点 |
| 推断 | 基于结构、调用条件或对照得到的暂定解释 |
| 未知 | 当前材料无法闭合的结论 |

每个摘要句子都要能回到类别、source identity 和 Claim Boundary。源码存在不等于 runtime 通过；readiness 不等于请求成功；timeout/RPC error 是传播边界时不等于底层根因；参考模型性能不等于目标模型实测。

### 2. 分离三个交付状态

报告至少分别写出：

```text
路线可行性：技术路径和关键前置已证明成立
适配完成度：当前实际走到的验证阶段
稳定交付：是否已达到目标验收和稳定性要求
```

真实权重加载、KV Cache、warmup 和 HTTP readiness 只能证明前置生命周期到达；首个请求失败时，摘要应同时保留“路线可行”和“适配未完成”，不能二选一或写成已交付。

### 3. 摘要与技术附件分层

Technical Attachment 保存 exact image/source/wheel/binary identity、模型结构和量化参数、调用链、失败边界、原始日志、kernel 条件、性能参考、验证计划和 owner。Decision Summary 只回答读者最关心的决策问题：真正难点是什么、需要新增能力还是先验证已有能力、速度大致如何。

摘要不是长报告的缩小版。每个主体段落只保留一个主判断、最少支撑事实和一个边界；章节顺序按读者决策顺序，不按工程执行时间排序。章节数量由本次交付要求决定，案例中的“三段”不是全局格式。

### 4. 算子名称必须闭合到 launch name

使用以下链路核验算子：

```text
model forward
  -> framework op
  -> vLLM-DLC adapter / PyTorch DLC Backend
  -> KernelDesc::launch("custom_xxx")
  -> DLC_Custom_Kernel Repository registry
```

摘要中的算子表使用真实 `custom_xxx` launch name 或明确的 kernel family，并用普通语言补充作用。`Attention`、`W13/W2`、`KV Cache`、`Router/top-k` 等模块名只能解释功能，不能代替实现名称。

同平台参考只能提供检索线索。量化格式、dtype、group size、zero-point、shape predicate、alignment 和调用阶段必须从目标模型 exact source、schema、descriptor、registry 和实际 run 重新闭合。不要把 AWQ/W8A16 kernel 名称复制到 GPTQ-Int4，也不要因模型请求失败就直接声明缺少算子。

DLCCL collective 属于通信 contract，应在通信/拓扑分析中单独表达，不混入模型算子缺口列表。只有 source、registry 和 workload 必需性均闭合为缺失时，才把算子列入“未实现”；否则写“暂未发现必须新增 kernel，优先验证已有实现”。

### 5. 性能口径必须可重算

性能摘要至少固定：input tokens、requested/generated output tokens、TP/EP、并发、单请求还是服务吞吐、TPOT 定义、是否包含 TTFT，以及参考数据与目标数据的身份关系。

```text
单请求 steady-state Decode token/s = 1000 / TPOT(ms/token)
固定输出长度的纯 Decode 时间 = output_tokens * TPOT / 1000
```

必须说明换算边界：纯 Decode 时间不含 TTFT；严格排除首 token 时 tail 可能使用 `output_tokens - 1`；token/s 不是服务总吞吐；规划区间不是实测、SLA 或理论上限。每个数字都要能由来源 workload 和公式复算。

### 6. 用减法 review 收敛

逐项删除不服务决策问题的内容：过程流水账、重复身份、技术附件已有表格、owner/计划细节、无信息量铺垫和重复 Claim Boundary。保留真实 kernel launch name、关键失败边界、最少证据和明确未验证范围。

独立审校至少检查：是否把 readiness 写成推理成功，是否把 timeout 写成根因，是否把 source presence 写成 runtime pass，是否把参考性能写成目标实测，是否把单请求 token/s 写成服务吞吐，是否把通信 kernel 混入模型算子表。

## 不应泛化

- Hy3 的 80 layers、192 experts、top-8、GPTQ-Int4、TP8 和具体 launch name。
- 某一模型的三段摘要结构、字数或性能区间。
- 从 MiniMax/GLM 等参考材料得到的具体 token/s、TPOT 或 kernel 名称。
- 某次首请求 timeout 的 owner、根因或具体 RPC 链路。

## Claim Boundary

本案例建立的是报告生产方法和表达边界。它不建立任何目标模型的兼容性、算子 runtime correctness、性能、稳定性、Real DLC Hardware acceptance、Verified vLLM Alignment 或 image acceptance。

## 来源

- 个人过程资产：`external_unavailable`（历史位置已脱敏）
- [模型适配与 Main-to-Main 决策记录](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)
- [vLLM-DLC Model Adaptation Prompt](../prompt-examples/vllm-dlc-model-adaptation.md)
