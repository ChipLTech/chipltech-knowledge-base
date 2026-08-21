# Agent 新 Session 上下文

本页是 context references 的 human view，不是流程真源，也不保存机器、runner、container 或历史命令。先发现实际 `<KNOWLEDGE_BASE_ROOT>` 与 `<SKILLS_ROOT>`；不要假定固定部署目录。

## 30 秒入口

```text
请使用 chipltech-context 路由并执行这个 Chipltech-Family Accelerator 任务。
目标：<一句话目标>
已有材料：<路径或“请自动发现”>
```

只需按顺序读取：

1. `<KNOWLEDGE_BASE_ROOT>/CONTEXT.md`：正式术语与组件边界。
2. `<KNOWLEDGE_BASE_ROOT>/README.md`：人类导航与六层架构。
3. `<KNOWLEDGE_BASE_ROOT>/prompt-examples/all-supported-capabilities-quickstart.md`：canonical capability catalog。
4. catalog 选择的详细 Prompt/Contract/Runbook。
5. `<SKILLS_ROOT>` 中实际加载的 owning Skill。
6. 当前业务 workspace 与 task-owned artifacts：唯一可建立当前执行 Evidence 的位置。

## Ask 输出合同

自然语言路由必须返回以下 literal labels：

```text
Selected capability: <capability_id 和 human entry>
Selection basis: <目标与 catalog entry 的匹配依据>
Minimum missing inputs: <仅列无法只读发现且阻止首个安全动作的输入>
First safe action: <当前授权内的首个只读或可逆动作>
Expected terminal states: <从 owning Skill 引用，不在这里重定义>
Evidence boundary: <已建立、未建立及 Claim Boundary>
```

机器闭包索引见 `agent-context/capability-manifest.yaml`。它只引用 Quickstart、owning Skill 与 Skills 仓库 `SKILLHUB.yaml`，不复制执行规则。

## 贡献者入口

- 能力身份与 publication closure：`agent-context/capability-manifest.yaml`。
- 经验晋升：`validated-lessons/index.yaml`；未列出的旧 case 默认 `historical_unreviewed`。
- 组织验证：`<SKILLS_ROOT>/scripts/validate-chipltech-organization.py`。
- `validated` 只修饰 lesson statement，不修饰整篇 case，不证明当前 runtime acceptance。

Claim Boundary: 本页只建立 context navigation 与 Ask 输出格式；它不证明当前 Host、Container、source、package、DLC Runtime、模型、transport、Real DLC Hardware、性能或 release 状态。
