# Hermes Chipltech Engineering Quickstart

## 用途

使用独立 Hermes profile 接入团队知识库、业务 Prompt 和稳定 Skills。知识库与 Skills 仓库继续作为 Single Source of Truth；Hermes 只负责检索、路由、执行和会话编排。

## 本机当前安装

```text
Profile: chipltech-engineering
Command: chipltech-engineering
Knowledge base: <KNOWLEDGE_BASE_ROOT>
Skills repository: <SKILLS_ROOT>
Default model: gpt-5.6-sol
```

Profile 只加载以下稳定 Skill 根：

```text
<SKILLS_ROOT>/skills/engineering
<SKILLS_ROOT>/skills/productivity
<SKILLS_ROOT>/skills/misc
```

`deprecated/`、`in-progress/` 和 `personal/` 不暴露给该 profile。

## 启动方式

从实际业务代码仓库启动，不要从知识库目录代替业务工作区：

```bash
cd /path/to/business-repository
chipltech-engineering
```

知识问答或任务路由：

```text
/chipltech-context <问题或任务>
```

示例：

```text
/chipltech-context 我需要验证一个本地新模型是否能在 ordinary daily base 上运行。请引用知识库，选择业务 Contract 和执行 Skill，并说明最少需要我提供什么。
```

Hermes 应先读取 `CONTEXT.md` 和 `README.md`，再按问题域检索专题、Case Study 和 `prompt-examples/`，最后加载最窄的执行 Skill。知识结论必须引用仓库相对路径和标题或行号。

## 直接使用业务入口

以下业务 Prompt 仍是任务 Contract，不复制进 Hermes 配置：

- [全部已支持能力傻瓜式调用总览](all-supported-capabilities-quickstart.md)
- [新模型验证 Quickstart](new-model-validation-quickstart.md)
- [ModelZoo 模型到 DLC/TYD Images](modelzoo-model-to-dlc-tyd-images.md)
- [vLLM-DLC Model Adaptation](vllm-dlc-model-adaptation.md)
- [vLLM-DLC Prefill/Decode Separation](vllm-dlc-prefill-decode-separation.md)
- [vLLM-DLC Main-to-Main Upgrade](vllm-dlc-main-to-main-upgrade.md)
- [DLC Env Setup Skill 使用模板](dlc-env-setup-skill-usage.md)

把需要的 Prompt 作为用户任务提交给 `chipltech-engineering` 即可。Prompt 中的 `<KNOWLEDGE_BASE_ROOT>` 和 `<SKILLS_ROOT>` 由 profile 配置与只读发现闭合。

## 验证命令

```bash
chipltech-engineering config get model.default
chipltech-engineering config get skills.external_dirs --json
chipltech-engineering skills list
chipltech-engineering chat -q "Reply with exactly: CHIPLTECH_PROFILE_OK"
```

预期：

- 默认模型为 `gpt-5.6-sol`。
- 外部目录仅包含三个稳定分类根。
- `chipltech-context`、DLC/vLLM Skills 和 Matt Pocock 1.2 通用 Skills 可发现。
- 模型请求返回 `CHIPLTECH_PROFILE_OK`。

1.2 通用 Skills 至少包括：

```text
codebase-design
domain-modeling
wizard
grilling
to-questionnaire
wait-what
writing-for-agents
```

它们只提供通用设计语言、访谈机制、人工流程和 Agent 文档能力。Chipltech 任务仍先由 `chipltech-context` 选择知识和 owning Skill；通用 Skill 的发现或加载不构成业务执行或 Runtime Evidence。

## 路由验收矩阵

以下验收均只做知识检索和路由，不执行构建、设备或 Host 修改：

| 场景 | 预期知识入口 | 预期 Skill | 缺少关键输入时的状态 |
| --- | --- | --- | --- |
| 环境重建 | `runtime-debugging/`、`debugging-workflows/` | `dlc-env-setup` | `blocked_missing_repository` |
| 单模型兼容 | `vllm-dlc/model-adaptation-and-main-to-main-decisions.md` | `model-adaptation` | `blocked_missing_asset` |
| 本地模型资格验证 | `vllm-dlc/modelzoo-driven-dlc-tyd-image-contract.md` | `modelzoo-image-validation` | `blocked_missing_asset` |
| Prefill/Decode Separation | `vllm-dlc/prefill-decode-separation.md` | `pd-separation` | `blocked_missing_contract` |

每项还必须满足：

- `chipltech-context` 从外部 Skills 仓库实际加载。
- 返回知识库相对路径和标题或行号。
- 加载对应执行 Skill，但不在 routing-only 验收中执行它。
- 返回 Claim Boundary，不把文档、HTTP、Ready 或非空输出当成执行 Evidence。
- 当前工作目录不因知识检索切换到知识库。
- `domain-modeling`、`grill-with-docs` 和 `wait-what` 使用当前业务仓库的 project context，不把知识库 `CONTEXT.md` 当成隐式写入目标。

在 Skills 仓库运行 live acceptance runner：

```bash
<SKILLS_ROOT>/scripts/validate-hermes-chipltech-integration.py
```

脚本从临时业务工作目录依次执行四个 routing-only 场景，并断言 router/owner Skill 加载轨迹、知识路径、blocker、Claim Boundary 和动态 workspace 配置。通过时输出 `hermes-chipltech-acceptance/v1` JSON，`passed` 为 `true`。

## 边界

- Hermes profile 不是文件系统 sandbox；真实只读要求必须由权限或只读挂载保证。
- Hermes Memory 和 Session History 不是验收记录。
- 知识库文档、Prompt 和 Skill 不能证明当前执行成功。
- Package/Import、C1b、SMI Observation、模型正确性、KV Transfer 和性能 Evidence 必须分别报告。
- 知识回写必须经过独立授权和 Evidence 审核，不在 `chipltech-context` 只读流程中完成。
- `wizard` 只能固化已授权的人工作业步骤，不能授予凭据、Host 修改或生产 cutover 权限，也不能证明步骤已经执行。
- `codebase-design`、`grilling`、`to-questionnaire`、`wait-what` 和 `writing-for-agents` 不能替代 DLC owning Skill，也不能弱化 Claim Boundary。
- 不把 HTTP 200、服务 Ready 或非空输出提升为模型正确性或 PD Separation 已验证。
