# 技术交付一句话总结方法

## 适用场景

本文用于把已经完成的技术实现、跨仓接入、验证或发布工作提炼为一句面向 review、日报、Sprint 或跨团队同步的能力总结。它回答“交付了什么能力”，不回答“故障为什么发生”。已闭合故障使用 [技术问题简要介绍方法](../debugging-workflows/technical-issue-summary.md)。

典型输入包括实现说明、diff、commit、测试结果、review 记录和交付 artifact。调用 `technical-delivery-summary` 时，应提供材料路径、目标读者和长度限制；未指定时默认输出一句跨团队总结。

## 核心结论

一句话总结先提取 **能力主干**：

```text
交付状态 + 对象 + 新增行为 + 关键决策依据或适用条件 + 可感知结果
```

各槽位回答：

| 槽位 | 要回答的问题 | QKNorm 示例 |
|---|---|---|
| 交付状态 | 实现、接入、验证还是发布 | 实现了 |
| 对象 | 谁获得了能力 | QKNorm 算子 |
| 新增行为 | 现在能做什么 | 自动选择 AllReduce 通信实现 |
| 依据或条件 | 根据什么做决定 | 实际 LYP topology 和 payload |
| 结果 | 对使用者意味着什么 | 选择与当前通信条件匹配的实现 |

如果结果已被“新增行为”完整表达，不重复补一句同义收益。目标不是把过程文档缩短，而是从实现链中抽出一个端到端能力主张。

## 先绑定交付状态

摘要中的动词必须与证据匹配：

| 状态 | 最小证据 | 推荐动词 |
|---|---|---|
| Source implemented | identified source 中已存在对应行为 | source 中已实现；避免裸“支持” |
| Integrated | 所需组件边界已完成接入 | 接入、打通 |
| Validated | identified implementation/artifact 通过声明检查 | 验证、确认 |
| Merged | identified change 已进入目标分支 | 合入 |
| Released | identified change 已发布、部署或对目标使用者可用 | 发布、上线、交付 |

这些状态是独立 Evidence 维度，不是从弱到强的单一阶梯。先记录与读者问题有关的每个状态，再选择只表达该维度的动词。已合入不等于已发布；已发布也不自动证明 runtime 或 Real DLC Hardware validation。

以下状态不能互相替代：

- source regression 通过不等于 build 通过；
- build 通过不等于 DLC Runtime execution；
- 历史 Real DLC Hardware smoke 不自动转移给修改后的 source、binary 或 image；
- review 完成不等于 merge、部署或 release；
- 单个 topology、payload 或 workload 通过不等于全部边界验证完成。

证据只支持 source implementation 时使用“实现了”，不要写“已完整验证”“已上线”或“已稳定交付”。

## 提炼步骤

### 1. 从过程账本提取事实

先列出：

- 实际完成的代码或 artifact；
- 跨组件输入、决策和输出；
- 已执行的 build、test、runtime、Real DLC Hardware 或 release 动作；
- 尚未验证的 topology、shape、payload、性能或 exact artifact；
- 目标读者真正需要据此知道或决定什么。

这份账本用于约束摘要，不直接进入摘要。

### 2. 合并跨仓链路

跨仓实现通常是一个能力，不是多个并列工作项：

```text
DLC_CL / DLCCL selection
  -> PyTorch DLC Backend adapter
  -> KernelDesc metadata
  -> DLC Custom Kernel dispatch
```

面向 review 同事时，应压缩为“QKNorm 算子根据实际 LYP topology 和 payload 自动选择 AllReduce 通信实现”。只有读者负责 ABI 或仓库 ownership 时，才保留 strategy、root、Ring order、KernelDesc slot 或 helper 名。

### 3. 明确对象的领域角色

裸技术名可能有歧义：

- `QKNorm` 写为“QKNorm 算子”，避免被理解为模型或算法；
- QKNorm 场景中的 `AllReduce` 面向非通信读者时可写为“多卡方差归约通信”；其他场景按实际归约对象解释，不把 AllReduce 泛化为方差计算；
- `DLCCL`、`DLC Runtime`、`DLC Custom Op` 和 `DLC Custom Kernel` 使用 [全仓术语入口](../CONTEXT.md) 的正式定义。

术语解释只保留到读者能识别能力为止，不在一句话内展开调用链。

### 4. 保留真正区分能力的依据

删除一个短语后，如果能力会退化为普通旧行为，就保留它。QKNorm 场景中，“实际 LYP topology 和 payload”说明选择不是只按 world size 的静态映射，因此属于能力主干。

fallback 只有在其安全保证是本次交付重点时才进入一句话。否则把 Verified Collective Fallback、root、alignment、Ring order 和候选验证留在 supporting detail 或 case study。

### 5. 做删减测试

逐段删除并判断：

- 删除后改变“交付了什么、何时生效或证据到哪一层”，保留；
- 删除后只看不到“内部怎么实现”，下沉；
- 删除后对象产生歧义，补充领域角色；
- 删除后交付状态被夸大，恢复边界限定。

通用场景默认只保留一个主句。Chipltech 分支必须再输出一行字面 `Claim Boundary:`，不把边界塞回主句，也不将该总结写成 Qualification Artifact。

## 输出模板

### 能力优先

> 实现了 `<对象>` 根据 `<关键依据或条件>`，自动 `<新增行为>`，从而 `<可感知结果>`。

### 验证优先

> 验证了 `<对象>` 在 `<条件>` 下能够 `<行为>`，并达到 `<已测结果>`。

### 发布优先

> 已发布 `<对象>` 的 `<能力>`，使 `<使用者>` 能够 `<结果>`。

### 可选范围说明

> 相关接入覆盖 `<组件或仓库>`；`<未执行层级>` 仍为 `not_verified`。

## QKNorm 示例

过程材料包含三个仓库、selector API、strategy int、root、Ring order、payload 条件、fallback 和 helper dispatch。对 review 同事，能力主干为：

```text
Source implemented + QKNorm 算子 + 自动选择 AllReduce 实现 + 实际 LYP topology 和 payload
```

推荐一句话：

> Historical source 中已实现 QKNorm 算子根据机器实际 LYP topology 和 payload，自动选择对应的 AllReduce 通信实现。

面向不熟悉集合通信的读者：

> Historical source 中已实现 QKNorm 算子根据机器实际 LYP topology 和数据量，自动选择对应的多卡方差归约通信实现。

需要说明范围时单独追加：

> 相关能力已在 DLC_CL、PyTorch DLC Backend 和 DLC_Custom_Kernel Repository 完成接入；当前 source、build、历史 hardware smoke 与目标 topology 的最终 Real DLC Hardware 验证状态应分别报告。

> Claim Boundary: 该示例来自历史 source implementation；最终 exact-artifact topology qualification、当前 Real DLC Hardware validation、性能和 release 未由这句话证明。

## 常见错误

- 按仓库顺序复述 API、参数传递和 helper dispatch，读者仍不知道最终能力。
- 为了显得完整，把 strategy int、root、Ring order 和 fallback 状态机全部塞进一句话。
- 只写“完成三仓开发”，没有对象、行为或结果。
- 裸写 QKNorm、AllReduce 或 DLC，使非本模块读者误解对象。
- 把 source implemented、build passed、historical smoke、reviewed、merged 和 released 混成“已完成验证上线”。
- 为追求通俗删除 topology、payload 或 policy 等真正区分新能力的依据。

## 完成标准

- 非实现参与者读完能复述交付了什么能力；
- 对象、行为和必要条件明确；
- 交付动词不超过证据层级；
- 每个事实子句可回溯到 identified source、artifact 或执行证据；
- 跨仓 plumbing 已被压缩为端到端行为；
- 删除任何剩余实现细节都会改变能力主张，而不只是缩短句子。

## Claim Boundary

本文只能规范技术交付总结的提炼和措辞，不能证明 source、build、test、DLC Runtime、Real DLC Hardware、模型正确性、性能、review、merge、部署或 release 状态。QKNorm 示例说明 historical source implementation 的表达方式，不构成当前 checkout、loaded binary、LYP topology、payload、模型或 image 的验收证据。

## 相关资料

- [QKNorm Topology-Aware AllReduce Selection](../case-studies/qknorm-topology-aware-allreduce-selection.md)
- [技术问题简要介绍方法](../debugging-workflows/technical-issue-summary.md)
- [Chipltech-Family Accelerator 术语和规则](../CONTEXT.md)
