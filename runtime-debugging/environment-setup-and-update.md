# 环境配置与仓库更新（历史背景）

## 定位

本文封存旧环境更新材料中的 rationale 和故障模式，不提供 current branch、固定业务根目录、安装命令或 executable order。`dlc-env-setup` owning Skill 是 repo discovery、Git 安全检查、current CLI、重建顺序、停止条件和环境 handoff 的唯一 current executable authority。

当前 Harness 已发现该 Skill 时直接加载；否则发现 `<SKILLS_ROOT>/skills/engineering/dlc-env-setup/SKILL.md`。两者都不存在时返回 `blocked_missing_contract`，不要从本文恢复历史命令。

## 保留的工程理由

- 更新前确认 authoritative repository root、full SHA、dirty state 和实际 editable import source；同名 checkout 不可按路径猜测。
- Git HEAD 只标识 source，不证明 compiler、wheel、shared library 或 loaded binary identity。
- native dependency 或 toolchain 变化会使 downstream build/runtime evidence 失效；最小 rebuild 起点应由当前 dependency health 决定。
- build 成功、wheel import、C1a、C1b、模型功能、Real DLC Hardware acceptance 和 performance 是独立证据维度。
- 已有 `third_party` 缓存是否可复用、是否允许网络、是否允许 install/build/ref switch 都由当前任务合同和授权决定，不是长期默认值。

## 常见故障边界

| 现象 | 优先核验 |
|---|---|
| `undefined symbol` | loaded library identity 与直接上游 ABI |
| source 有 option、binary 报 unknown | compiler binary identity 与 source SHA |
| 更新后仍运行旧代码 | `direct_url.json`、`module.__file__`、进程命令与 source SHA |
| wheel 可安装但运行失败 | build configuration、fresh import、Stack Preflight 与 C1b |
| 上层 package/import 失败 | 最早不健康的下游依赖，不机械全量重建 |

## Claim Boundary

本文只保存历史 rationale。它不证明当前仓库位置、branch、命令、依赖顺序、package、DLC Runtime 或 Real DLC Hardware 状态，也不授权 clone、fetch、build、install、ref switch、Host maintenance 或 device execution。

## 相关资料

- [DLC 工作站环境重建 rationale](dlc-workstation-env-rebuild.md)
- [Runtime 排障](runtime-troubleshooting.md)
- [Host API 22 历史案例](../case-studies/host-api22-fullstack-main-to-main-update.md)
