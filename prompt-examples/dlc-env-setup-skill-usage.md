# 用 `dlc-env-setup` skill 重建 DLC Ecosystem 工作站环境

专门面向 DLC Ecosystem 工作站重建与修复的 prompt 模板。执行权威是 `/work/skills/skills/engineering/dlc-env-setup/SKILL.md` 及其脚本，覆盖 repo discovery、Git 安全检查、分阶段重建、PyTorch 2.5.0 wheel、可选本地 `vllm` / `vllm-dlc` repair，以及源码树外 DLC Runtime smoke。

**阅读说明**：
- `▶ 可复制 prompt` 后的代码块可直接发给 AI，使用前替换 `<>` 占位符。
- `pytorch-preflight.sh`、`vllm-preflight.sh`、`runtime-smoke.sh` 是对应阶段的可执行 source of truth；本模板只保留调用方式、输入和本地安全策略，不复制脚本内部命令。

---

## 使用前提

- 以当前 `/work/skills/skills/engineering/dlc-env-setup/SKILL.md` 与同目录脚本为准；不要固定旧 commit。若怀疑本地 skill 未更新，先在 `/work/skills` 执行 `git status --short --branch` 并确认已同步到团队当前主线。
- 全局安装：在 `/work/skills` 运行 `./scripts/link-kilo-skills.sh --with-commands`。
- 项目本地安装：运行 `./scripts/link-kilo-skills.sh --project /path/to/project --with-commands`，链接写入项目的 `.kilo/skills` 和 `.kilo/command`。
- linker 会清理其明确列出的 retired skill symlink 和生成式 command alias；真实文件/目录不会被删除。若旧 alias 不是该脚本可识别的 symlink 或生成文件，应先报告并由用户决定，不要手工盲删。

## 和 Model Adaptation 的推荐串联方式

可以、而且推荐把每日空镜像里的新模型工作拆成两个阶段：

```text
先 dlc-env-setup，把空每日镜像变成健康 DLC Ecosystem 工作站
再 model-adaptation，对新模型做 vLLM-DLC / DLC Platform 适配分析
```

边界如下：

- `dlc-env-setup` 负责环境层：repo discovery、Git 安全检查、DLC Ecosystem 分阶段重建、PyTorch 2.5.0 wheel、本地 `vllm` / `vllm-dlc` repair、源码树外 runtime smoke。
- `model-adaptation` 负责模型层：在已健康环境上分析一个明确模型的 loading/serving 兼容性、deployment profile、模型级 Attention/tokenizer/processor/distributed 边界和 evidence 缺口。
- 第一阶段未通过 `runtime-smoke.sh /tmp` 和请求范围内的 `vllm` / `vllm-dlc` import 验证时，不进入第二阶段。
- 第二阶段不得把环境初始化结果当作新模型的 Real DLC Hardware acceptance、DLC Runtime dispatch 或 Verified vLLM Alignment 证据。
- 可直接使用 [vllm-dlc-fresh-image-to-model-adaptation.md](vllm-dlc-fresh-image-to-model-adaptation.md) 作为两阶段 prompt 模板。

## 全量或分阶段重建

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请使用 `dlc-env-setup` skill 重建或修复这台机器上的 DLC Ecosystem 工作站环境。按当前 skill 闭环执行，并在每阶段做最小验证。

先读取并遵循：
- /work/chipltech-knowledge-base/CONTEXT.md
- /work/skills/skills/engineering/dlc-env-setup/SKILL.md
- /work/skills/skills/engineering/dlc-env-setup/scripts/pytorch-preflight.sh
- /work/skills/skills/engineering/dlc-env-setup/scripts/vllm-preflight.sh
- /work/skills/skills/engineering/dlc-env-setup/scripts/runtime-smoke.sh
- /work/chipltech-knowledge-base/runtime-debugging/dlc-workstation-env-rebuild.md
- /work/chipltech-knowledge-base/debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md
- /work/chipltech-knowledge-base/debugging-workflows/post-install-runtime-smoke.md

【模式】<全量重建 / 从 LLVM 开始 / 从 DLC_Custom_Kernel 开始 / 只重建 PyTorch wheel / 只重装 wheel / 修 vllm>
【搜索根目录】<例如 /work,$HOME；不确定则写“请自动发现”>
【需要切换的批准 ref】<逐仓库填写 remote URL、目标 branch/tag 和可选 commit SHA；没有则写“无”>
【是否包含 vllm / vllm-dlc】<是/否>
【是否允许修改 /usr/local】<是/否>

请严格按下面流程执行：

1. 先做 repo discovery，不要默认 `/home/workspace`。
   - 按当前 SKILL.md 发现必需仓库；可选工作被请求时再发现 `vllm` 和 `vllm-dlc`。
   - 对每个 Git 仓库记录并回显：路径、`git rev-parse --show-toplevel`、`git remote -v`、`git branch --show-current`、`git rev-parse HEAD`、`git status --short`。
   - 验证发现路径确实位于该 Git root 下，remote 与批准来源一致，branch/tag 和 HEAD 满足输入约束。
   - 同名候选、remote 不匹配、detached HEAD 未获批准或 authoritative root 不明确时，按当前 SKILL.md 停止。
   - 缺少必需仓库时按当前 SKILL.md 停止；只有用户明确授权 bootstrap 时，才转用 `/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-environment-bootstrap.md`。
   - 回显最终 repo map 和统一 shell 变量：`CMAKE_SRC`、`DLC_THUNK_SRC`、`DLCSIM_SRC`、`DLCSYNAPSE_SRC`、`DLC_CL_SRC`、`LLVM_SRC`、`DLC_CUSTOM_KERNEL_SRC`、`PYTORCH_SRC`，以及可选的 `VLLM_SRC`、`VLLM_DLC_SRC`。

2. 按当前 SKILL.md 验证每个源码树的预期 build entrypoint。不要以目录名存在替代入口验证，缺失时立即停止。

3. 需要切换 ref 时先做 Git 安全检查。
   - 检查 status、current branch、remote 和 branch availability，并记录切换前 HEAD。
   - 有未提交改动立即停止，不覆盖、不 stash、不使用 destructive git 命令。
   - 仅切换到用户批准且本地或 tracked remote 确实存在的 ref；不要无条件 `git pull`。
   - 切换后再次验证 Git root、remote、branch/tag、HEAD 和 clean status。
   - 所有联动仓库都确认完成后再构建。

4. 按当前 SKILL.md 的 `Rebuild Order` 和 `Partial Rebuild Rules` 决定起点与顺序。
   - 从中间阶段开始前，先验证更早的原生依赖和安装产物健康；无法确认时回退到更早阶段。
   - wheel reinstall only 仅可使用 `dist/` 中已确认 fresh 的 `torch-2.5.0*.whl`。
   - `/usr/local` 修改未授权时，不执行对应安装步骤并停止报告。

5. PyTorch 阶段必须直接运行：
   - `/work/skills/skills/engineering/dlc-env-setup/scripts/pytorch-preflight.sh "$PYTORCH_SRC"`
   - 该脚本是 packaging 版本约束和 NumPy pin 的可执行 source of truth；它只打印现有 `USE_NUMPY` cache，不会因值缺失或为 `OFF` 自动失败。
   - 如果已有 `build/CMakeCache.txt`，必须单独检查 `USE_NUMPY:BOOL=ON`；值为 `OFF` 时停止并清理/reconfigure，条目缺失时在 configure 后再次检查。没有确认 `ON` 前，不得进入 wheel build。
   - 运行前确认 `python3 -m pip --version` 指向预期解释器与环境；未经用户授权不得切换 package manager、使用裸 `pip` 或修改其他 Python 环境。
   - packaging preflight 和 `USE_NUMPY:BOOL=ON` gate 都通过后，再按当前 SKILL.md 和仓库已验证流程构建、force-reinstall fresh PyTorch 2.5.0 wheel。

6. LLVM 与 DLC_Custom_Kernel Repository 阶段保留以下本地策略：
   - LLVM 曾失败、半构建或 build cleanliness 不明时，优先使用仓库已有的 clean rebuild 路径 `./build.sh -r`；否则可用 `./build.sh`。
   - 配置 DLC_Custom_Kernel Repository 时显式令 `LLVM_PATH="$LLVM_SRC"`，并使用已验证的 `/usr/bin/ninja`，不要依赖意外的 PATH alias。
   - 构建与安装命令必须来自当前仓库入口和知识库验证路径，不发明工具链参数。
   - 安装后确认 DLC Custom Kernel 相关库、header 和测试工具落在预期 `/usr/local/chipltech/synapse/` 位置。

7. 请求本地 `vllm` / `vllm-dlc` repair 时：
   - 先确认 PyTorch 2.5.0 wheel 和 DLC Runtime smoke 健康，再运行 `/work/skills/skills/engineering/dlc-env-setup/scripts/vllm-preflight.sh`。
   - 该脚本是 editable-install packaging prerequisites 的可执行 source of truth；仍需本地确认 Python 版本满足仓库要求，并确认两个仓库的 remote/ref 与 Python import surface 匹配。
   - 优先从已批准的本地源码做最小 editable-install repair。是否使用 `--no-deps` 必须由“仅 metadata 损坏且依赖已验证健康”的证据支持。
   - `UNKNOWN` 只是一条 metadata 异常线索。先用 `pip list`、`pip show`、对应 `*.dist-info`/`*.egg-info` 所属关系和当前 editable install 证据定位具体损坏包；只修复有证据关联的 metadata，不 blanket 删除所有 `UNKNOWN`。
   - `triton` 的存在本身不是卸载依据。仅在当前批准 ref、依赖声明或已复现冲突证明它不应存在时，报告证据并执行最小处置；不要默认卸载。
   - 发现写死的其他机器 wheel URL 时，先验证当前真实 wheel，再只修复已确认错误的本地配置或 metadata。

8. 安装后必须直接运行：
   - `/work/skills/skills/engineering/dlc-env-setup/scripts/runtime-smoke.sh /tmp`
   - 该脚本是源码树外 PyTorch、NumPy bridge、DLC Platform 可用性、可选 `vllm` imports 和关键 package inventory 的可执行 source of truth。
   - 脚本会打印可选 import error；请求了 `vllm` 工作时，这些 error 是失败，不能因脚本继续运行而视为通过。
   - 另按当前 SKILL.md 验证 CMake 3.27.0、PyTorch 2.5.0、`/usr/local/chipltech/synapse/bin` 中 DLC Custom Kernel 测试工具，以及请求的 `pip show` metadata。

9. 任一阶段失败时按当前 SKILL.md 的停止条件立即停止，报告失败命令、第一批关键报错和应回退的阶段。不要在失败状态下盲目继续。

交付物：
- 最终 repo map，以及每个仓库的 Git root、remote、branch/tag、HEAD、status。
- 本次起始阶段及其依赖健康证据。
- 每阶段执行命令和最小验证结果。
- PyTorch wheel 路径与 reinstall 结果。
- 三个 skill 脚本的执行结果，以及最终 DLC Runtime smoke 判定。
- `UNKNOWN`、`triton` 或 wheel path 如被处置，列出证据和最小改动。
- 如果失败：触发的当前 SKILL.md 停止条件、失败阶段、命令、关键报错和安全回退点。
```

### Slash Command

运行 linker 的 `--with-commands` 模式后，可使用：

```text
/dlc-env-setup <把上面模板里的需求、批准 ref 和约束作为参数传入>
```

## 只修本地 `vllm` / `vllm-dlc`

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请使用当前 `dlc-env-setup` skill，只修本地 `vllm` / `vllm-dlc` 安装链路。除非证据表明底层 PyTorch wheel 或 DLC Runtime 不健康，否则不要重建原生依赖。

先读取：
- /work/chipltech-knowledge-base/CONTEXT.md
- /work/skills/skills/engineering/dlc-env-setup/SKILL.md
- /work/skills/skills/engineering/dlc-env-setup/scripts/vllm-preflight.sh
- /work/skills/skills/engineering/dlc-env-setup/scripts/runtime-smoke.sh
- /work/chipltech-knowledge-base/debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md

【vllm 仓库根目录】<路径或“请自动发现”>
【vllm-dlc 仓库根目录】<路径或“请自动发现”>
【批准 ref】<两个仓库各自的 remote URL、branch/tag 和可选 commit SHA>
【PyTorch wheel 路径】<路径或“请自动发现”>
【当前症状】<例如 packaging 缺失 / UNKNOWN metadata / build_editable hook 缺失 / import vllm_dlc 失败>

执行顺序：
1. 对两个仓库检查 Git root、remote、branch、HEAD 和 status；身份不匹配、候选不唯一或工作树不干净时按当前 SKILL.md 停止。
2. 从源码树外运行 `/work/skills/skills/engineering/dlc-env-setup/scripts/runtime-smoke.sh /tmp`。PyTorch、NumPy bridge 或 DLC Platform 不健康时停止，归类为底层问题。
3. 运行 `/work/skills/skills/engineering/dlc-env-setup/scripts/vllm-preflight.sh`，并确认它使用预期 Python/pip 环境。
4. 验证当前 `vllm` 与 `vllm-dlc` 批准 ref 的 Python import surface 兼容；缺少被引用模块时停止并报告 ref mismatch。
5. 优先做最小 editable-install repair。只有证据确认问题仅限 metadata 且依赖健康时才使用 `--no-deps`。
6. 对 `UNKNOWN` 先定位具体 metadata 所属包和损坏证据，只修复关联项，不 blanket 删除。
7. 不因发现 `triton` 就卸载；只有批准 ref、依赖声明或复现冲突提供证据时才做最小处置。
8. wheel URL 指向其他机器时，先确认当前 wheel 的真实路径与版本，再只修复已确认错误的引用。
9. 最后再次运行 `runtime-smoke.sh /tmp`，并在源码树外验证请求范围内的 imports 和 `pip show` metadata。脚本打印的请求组件 import error 必须判为失败。

交付：问题分类、Git 身份检查、preflight 结果、repair 命令、证据化 metadata/依赖处置、最终 import 与 DLC Runtime smoke 结果。
```

## 什么时候停

以当前 `/work/skills/skills/engineering/dlc-env-setup/SKILL.md` 的 `Stop Immediately When` 为准。模板额外要求在仓库 remote/ref 身份不匹配、Python package manager 指向不明、`vllm`/`vllm-dlc` import surface 不兼容，或请求组件在 `runtime-smoke.sh` 中导入失败时停止。

## 相关资料

- `/work/skills/skills/engineering/dlc-env-setup/SKILL.md`（以当前团队主线为准）
- `/work/skills/skills/engineering/dlc-env-setup/scripts/pytorch-preflight.sh`
- `/work/skills/skills/engineering/dlc-env-setup/scripts/vllm-preflight.sh`
- `/work/skills/skills/engineering/dlc-env-setup/scripts/runtime-smoke.sh`
- `/work/skills/scripts/link-kilo-skills.sh`
- [dlc-env-setup-environment-bootstrap.md](dlc-env-setup-environment-bootstrap.md)
- [../runtime-debugging/dlc-workstation-env-rebuild.md](../runtime-debugging/dlc-workstation-env-rebuild.md)
- [../debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md](../debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md)
- [../debugging-workflows/post-install-runtime-smoke.md](../debugging-workflows/post-install-runtime-smoke.md)
