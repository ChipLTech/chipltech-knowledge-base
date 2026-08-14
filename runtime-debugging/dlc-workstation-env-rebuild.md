# DLC 工作站环境重建 Rationale

## 定位

`dlc-env-setup` owning Skill 是 current executable order、current CLI、repo discovery、branch safety、rebuild invalidation、C1a/C1b 和 handoff 的唯一 current executable authority。本文只解释依赖与 invalidation，不复制第二套构建序列。

当前 Harness 已发现该 Skill 时直接加载；否则发现 `<SKILLS_ROOT>/skills/engineering/dlc-env-setup/SKILL.md`。两者都不存在时返回 `blocked_missing_contract`。

## 依赖与 Invalidation

1. 先发现 authoritative roots 和实际 import/load 路径，再判断 source、package 与 binary 是否配对。
2. dirty worktree、重叠修改或不明确 owner 会阻止 ref switch；不得用 destructive Git 操作换取表面 clean。
3. direct upstream artifact 变化会使 downstream build evidence stale。最小 rebuild 起点必须由 identity 与 health evidence 决定，不能从固定组件列表推断。
4. toolchain source 与执行 binary 必须独立记录；source HEAD 不能替代 compiler 自报 identity 或 artifact digest。
5. wheel build、isolated import、DLC Runtime execution、Real DLC Hardware observation、模型功能与性能分别验收。
6. partial rebuild 只有在上游 identity 和 health 仍有效时成立；无法证明时 fail closed，而不是默认复用旧产物。

## 安全边界

- discovery 和 status inspection 可保持 read-only。
- clone/fetch、ref switch、build、install、系统路径修改、device execution、cleanup 和 Host maintenance 分别授权。
- 不固定 workspace root、branch、wheel path、parallelism 或 packaging command。
- 环境 handoff 只对绑定的 source/package/artifact/Host epoch 有效，不转移模型 acceptance。

## Claim Boundary

本文只解释 dependency/invalidation rationale，不验证当前环境可构建、安装、import、执行 DLC Runtime 或运行 Real DLC Hardware。

## 相关资料

- [历史环境配置背景](environment-setup-and-update.md)
- [Python build preflight](../debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md)
- [Post-install runtime smoke](../debugging-workflows/post-install-runtime-smoke.md)
