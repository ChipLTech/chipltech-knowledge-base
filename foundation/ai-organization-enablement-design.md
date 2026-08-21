# AI Organization Enablement 设计

## 目标

本设计在 Stage A 完成、Stage B 收口后，将 Chipltech Skills 与知识库从个人 Agent 能力升级为团队可发现、可交接、可复核、可积累的共享工程底座。它实现 `Discover`、`Ask`、`Handoff`、`Context Routing`、`Evidence Return` 和 `Shared Team Knowledge`，暂不引入 production trust root、签名、key rotation、revocation 或复杂企业权限。

## 单一真源

| 信息 | 真源 |
|---|---|
| Skill 发布包 | `skills.git/SKILLHUB.yaml` |
| 业务能力导航 | `agent-context/capability-manifest.yaml` 与 canonical Quickstart 的闭包 |
| Workflow、授权、终态 | owning `SKILL.md` |
| 机器字段与状态约束 | JSON Schema 和 validator |
| Evidence identity、authority 和 digest | Qualification Artifact Envelope |
| 当前任务事实 | task-owned Evidence Artifact |
| 历史过程 | Case Study |
| 可复用经验状态 | `validated-lessons/index.yaml` |

README、Quickstart、agent context 和 quality view 是派生或强校验视图，不维护第二套业务规则。

## 六层架构

```text
Terminology
  -> Knowledge
  -> Capability Catalog
  -> Task Contract
  -> Owning Skill
  -> Task Evidence
```

- Router 只选择 owner。
- Owning Skill 独占全局 task state 和终态。
- Role Agent 返回 Evidence Return，不直接写全局终态。
- Reviewer、Observer 和 Assessor 默认只读。
- Handoff 只引用 Evidence，不复制或升级 Evidence。

## Stage B 收口

### Publication Candidate Producer/Assessor

Producer 是外部、明确授权的 publication-preparation workflow，负责在 clean isolated worktree 构造候选并输出 sealed handoff，不负责批准。Stage C deferred 期间，`main-to-main-upgrade` 的 B02 assessor 只重新计算 Tested Revision、candidate HEAD/base/tree、scoped diff、Patch Equivalence、affected gate freshness、remote observation 和 exact lease，以建立 `structural candidate assessment`；它不认证 authorization、不输出 publication eligibility，永不 commit、push、rewrite 或 finalize。每个 candidate 都必须绑定针对其 exact source、tree 和 artifact graph 新执行的 build/runtime gate Evidence；其他 candidate、Tested Revision、patch-id 或历史 gate 结果不能继承为该 candidate 的 fresh Evidence。

### Stage C Deferred 边界

Stage C deferred 表示 production trust、正式 authorization certification 和 publication decision 尚未实现。B02 structural candidate assessment 只能说明候选结构、identity、Patch Equivalence、freshness 与 Evidence references 是否闭合，不能说明候选已获授权、可发布、已验证为生产可用或值得生产信任。Authorization 仅作为未认证的输入事实或缺口记录；publication eligibility 必须留给未来 Stage C 的独立 authority。

### Workflow Behavioral Fixtures

Publication tests 证明 Skill 可以被发现和安装；Behavior fixtures 证明 deterministic input 会得到正确 route、state、blocker、resume point 和 Claim Boundary。统一 runner 只负责安全 argv 调用、fixture containment、machine JSON assertion 和 quality projection，不复制 owner-specific 状态机。

首批行为覆盖：

- PD gate aggregation；
- Plugin Migration evidence dimensions；
- Model Adaptation report-only route；
- Technical Delivery Summary；
- Publication Candidate assessor；
- topology selection；
- Technical Issue Summary 首版仅标记 `contract_static`。

## 组织合同

### `agent-evidence-return/v1`

子 Agent 返回 assignment state、Evidence references、claim-to-Evidence closure、changed artifacts、blocker reference、resume point、unverified scope 和 Claim Boundary。Assignment `passed` 只表示子任务完成，不表示 runtime 或 formal acceptance。

### `context-reference-package/v1`

声明 terminology、navigation、catalog、contract、runbook、case 和 prompt 引用，并记录 required/optional/forbidden access、canonical/supporting/historical authority、Execution Locus、freshness 和 digest。它用于路由和解释，不是 runtime Evidence。

### `engineering-handoff/v1`

跨 Agent、session、Harness 或同事传递 objective、current state、fact baseline、decision refs、Evidence Return refs、失败尝试、Context Package、next owner/action/criterion、stop conditions、待补授权和 invalidation conditions。它不替代 topic-specific handoff，可引用它们。

### `team-task-brief/v1`

任务输入声明 objective、current/desired behavior、scope/out-of-scope、owner、Execution Locus、required identities/authorizations，以及每条 acceptance criterion 需要的 Evidence class 和 schema。Brief 不预填 gate 为通过。

## Discover 与 Ask

Capability ID 使用 `cap.<domain>.<name>`，将 canonical Quickstart entry、owning Skill 和 SKILLHUB publication 连接起来。canonical Quickstart 中所有正式 Capability ID 与 manifest 必须双向集合相等；Runbook、固定环境 helper、凭据 bootstrap 和可选 Harness 集成等 supporting navigation 不分配 ID。自然语言 Ask 统一返回：

```text
Selected capability:
Selection basis:
Minimum missing inputs:
First safe action:
Expected terminal states:
Evidence boundary:
Claim Boundary:
```

能力目录只负责发现；实际执行前必须读取 owning Skill 的当前 scope、negative scope、授权、停止语义和 Claim Boundary。

## L1/L2/L3/ST 质量视图

| 层级 | 含义 |
|---|---|
| L1 | publication、catalog、owner 和 reference closure；可由 exact source/fixture run 派生 |
| L2 | deterministic route 与 workflow behavior fixtures；可由 exact source/fixture run 派生 |
| L3 | Evidence Return、Context Package、Handoff 和 Team Brief 协作合同的结构存在性与 focused tests |
| ST | 用户式 prompt、Evidence 分类、历史案例适用性和 claim evaluation |
| Hardware | task-owned Real DLC Hardware Evidence，独立于 ST |

Quality view 是只读派生结果，不生成综合分，也不把 fixture 或 source test 升级为 runtime acceptance。L1/L2 可由绑定 exact source identity 的当前 fixture run 派生。L3 可以在当前 session 核验组织合同存在并运行 focused tests，但持久 quality view 默认仍为 `not_reported`，除非存在独立、可持久化且满足该 view 合同的 Evidence。Stage C deferred 期间，ST、Hardware 和 runtime 均为 `not_reported`；未建立的任何其他层级也必须显示 `not_reported`。这些状态不构成 production trust 声明。

## Verified Experience 晋升

```text
Task Evidence
  -> Case Study
  -> Validated Lesson
  -> Rule / Skill Requirement
  -> Contract / Regression Fixture
  -> Superseded Tombstone
```

每条 Validated Lesson 独立记录 statement、identity scope、applies-to、does-not-apply-to、Evidence class、validation method、counterexample、owner、rule/contract/test refs、review date 和 Claim Boundary。`validated` 状态要求 qualified regression test ID 直接核验 lesson ID、精确 statement 和对应 rule/contract 内容；只证明测试文件存在不形成 lesson closure。无法形成该闭包时保持 `reviewed`。`validated` 只修饰有界 lesson statement，不修饰整篇 case，也不证明当前 runtime 状态；未进入索引的旧案例默认 `historical_unreviewed`。

## 可选 Task/Plan/Round Ledger

只有任务同时满足多候选、多轮、可恢复，并至少有两个真实 adapter 时才启用 ledger profile。适用时要求 owner 单写 task state、Evidence Ledger append-only、round0 固定 baseline、round 全局单调、失败轮次保留、派生 Plan 不覆盖以及最终使用同一 workload 复验。简单任务不引入该框架。

## 实施和验收顺序

1. Publication Candidate handoff schema、assessor、fixtures 和 publication surface。
2. PD evaluator、behavior manifest、runner 和代表性 owner cases。
3. 四个组织合同、validator、positive/negative fixtures 和 generated copies。
4. Capability ID closure、三级入口和 Ask Output Contract。
5. L1/L2/L3/ST/Hardware 派生质量视图。
6. Validated Lesson Index 与旧案例 historical default。
7. README、Quickstart、agent context、SkillHub 和安装面同步。
8. Focused tests、generated drift、publication/install、全量 tests、secret scan 和双轴 review。

完成标准：所有合同 closed-world、digest-bound；所有 claim 和 handoff references 闭合；B02/B03 行为 fixtures 通过；Capability Catalog 能闭合到 owner 和 publication；quality view 不升级 Evidence；lesson 可追溯 case、rule 和 test；High/Medium review findings 全部关闭。

## Claim Boundary

Claim Boundary: 本设计和配套软件合同提高团队能力发现、Agent 协作、交接、Evidence 返回和经验复用；它不实现 production cryptographic trust、复杂企业权限或 formal issuer，也不证明任何当前 Host、Image、Driver、DLC Runtime、模型、collective、PD、benchmark、Real DLC Hardware 或 release 状态。
