# 用 `dlc-env-setup` skill 重建 DLC 工作站环境

专门面向 DLC 工作站环境重建 / 修复的 prompt 模板。目标是直接驱动 AI 调用 `/work/skills/skills/engineering/dlc-env-setup/` 中的 `dlc-env-setup` skill，覆盖 `/work/test/dlc-env-setup` 里的同等级能力：repo discovery、branch 切换安全检查、分阶段重建、PyTorch 2.5.0 wheel、可选 `vllm` / `vllm-dlc` repair，以及源码树外 runtime smoke。

**📖 阅读说明**：
- `▶ 可复制 prompt` 标记后面的代码块是发给 AI 的指令，替换 `<>` 里的占位符即可使用。
- 标题下的“适用场景”“关键点”是给人看的说明，不用复制发给 AI。

---

## 适用场景

- 一台机器上的 DLC Ecosystem 开发环境已经坏掉，需要从仓库发现、版本切换、构建到验证完整走一遍。
- 已知大部分依赖还在，但需要从 LLVM、`DLC_Custom_Kernel`、PyTorch wheel 或 `vllm` 路线继续修复。
- 希望 AI 不只是解释文档，而是按 skill 的闭环真正执行，并在每个阶段给出最小验证结果。

## 全量或分阶段重建

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请使用 `/work/skills/skills/engineering/dlc-env-setup/` 中的 `dlc-env-setup` skill，帮我重建或修复这台机器上的 DLC 工作站环境。不要跳步骤，要按 skill 和知识库规则执行，并在每个阶段做最小验证。

先读取：
- /work/chipltech-knowledge-base/CONTEXT.md
- /work/chipltech-knowledge-base/README.md
- /work/chipltech-knowledge-base/runtime-debugging/dlc-workstation-env-rebuild.md
- /work/chipltech-knowledge-base/debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md
- /work/chipltech-knowledge-base/debugging-workflows/post-install-runtime-smoke.md

【模式】<全量重建 / 从 LLVM 开始 / 从 DLC_Custom_Kernel 开始 / 只重建 PyTorch wheel / 只重装 wheel / 修 vllm>
【搜索根目录】<例如 /work,/home/workspace,$HOME；如果不确定就写“请自动发现”>
【需要切 branch 的仓库】<例如 LLVM:feature-x, pytorch:release/2.5-dlc；没有就写“无”>
【是否包含 vllm / vllm-dlc】<是/否>
【是否允许修改 /usr/local】<是/否>

请严格按下面流程执行：

1. 先做 repo discovery，不要默认 `/home/workspace`。
   - 找到这些仓库：`cmake-3.27.0`、`dlc-thunk`、`DLCsim`、`DLCSynapse`、`DLC_CL`、`LLVM`、`DLC_Custom_Kernel`、`pytorch`。
   - 如果【是否包含 vllm / vllm-dlc】是“是”，再额外找到 `vllm` 和 `vllm-dlc`。
   - 把最终 repo map 原样回显给我。
   - 如果有同名仓库多个候选路径，先停下来说明候选项，不要擅自选。

2. 对每个关键仓库验证入口文件：
   - `dlc-thunk/compile.sh`
   - `DLCsim/build.sh`
   - `DLCSynapse/compile.sh`
   - `DLC_CL/build.sh`
   - `LLVM/build.sh`
   - `DLC_Custom_Kernel/CMakeLists.txt`
   - `pytorch/setup.py`
   - 如果有 `vllm` / `vllm-dlc`，也确认仓库根目录存在安装入口
   - 缺任何一个都要立即停止并汇报。

3. 如果【需要切 branch 的仓库】不是“无”，先做 branch 安全检查：
   - 对每个目标仓库执行：
     - `git status --short`
     - `git branch --show-current`
     - `git branch -a --list '*目标分支*'`
   - 如有未提交改动，立即停止，不要覆盖、不用 `reset --hard`。
   - 本地有目标分支就切本地；只有 remote branch 时再建 tracking branch。
   - 所有需要联动切换的仓库都切好并确认后，再开始构建。

4. 按依赖顺序重建，除非【模式】明确要求从中间阶段开始：
   - CMake 3.27
   - `dlc-thunk`
   - `DLCsim`
   - `DLCSynapse`
   - `DLC_CL`
   - `LLVM`
   - `DLC_Custom_Kernel`
   - PyTorch 2.5.0 wheel build + force reinstall
   - 可选：`vllm` / `vllm-dlc` editable install
   - final runtime smoke

5. CMake 3.27 阶段必须检查：
   - `/usr/local/bin/cmake --version`
   - `command -v cmake ctest cpack`
   - `cmake --version`
   - 如果遇到 OpenSSL 头文件缺失，先提示 `libssl-dev` 问题，再决定是否继续。

6. PyTorch 阶段必须先做 preflight：
   - 安装或升级 `typing_extensions`、`packaging`、`setuptools`、`setuptools-scm`、`wheel`、`ninja`、`jinja2`
   - 强制 `numpy==1.26.4`
   - 如已有 `build/CMakeCache.txt`，检查 `USE_NUMPY:BOOL=ON`
   - 然后再清理旧产物并构建 wheel
   - 构建完成后，从 `dist/` 中 force-reinstall 最新 `torch-2.5.0*.whl`

7. 如果【是否包含 vllm / vllm-dlc】是“是”，在 PyTorch smoke 通过后再继续：
   - 先做 `vllm` preflight：`pip`、`packaging`、`setuptools`、`setuptools-scm`、`wheel`、`ninja`、`jinja2`、`pybind11`、`grpcio-tools`
   - 优先用本地源码做 editable install
   - 如果只是 metadata 损坏，优先走最小 repair 路线，不要大范围 churn 依赖
   - 如果发现 wheel 路径写死成别的机器路径，指出并改成当前机器真实路径

8. final runtime smoke 必须在源码树外执行，例如 `/tmp`：
   - `python3 -c "import torch; print(torch.__version__)"`
   - `python3 -c "import torch; print(torch.tensor([0.1], dtype=torch.float32).numpy())"`
   - `python3 -c "import torch; print(torch.backends.dlc.is_available() if hasattr(torch.backends, 'dlc') else 'no torch.backends.dlc')"`
   - 如果包含 `vllm`：
     - `python3 -c "import vllm; print(vllm.__version__)"`
     - `python3 -c "import vllm_dlc; print(vllm_dlc.__file__)"`
     - `python3 -m pip show vllm`
     - `python3 -m pip show vllm_dlc`

9. 任一阶段失败时：
   - 立即停止
   - 报告失败命令
   - 报告第一批关键报错
   - 说明应该回退到哪一层处理
   - 不要在失败状态下盲目继续到后续阶段

交付物必须包括：
- 最终 repo map。
- branch 检查和切换结果。
- 本次从哪个阶段开始、为什么可以从这里开始。
- 每个阶段的最小验证结果。
- PyTorch wheel 路径和 reinstall 结果。
- final runtime smoke 结果。
- 如果失败：失败阶段、失败命令、第一批关键报错、建议回退路径。
```

**关键点**：这个模板的目标不是“解释怎么装”，而是让 AI 真正调用 `dlc-env-setup` skill 按闭环执行。repo discovery、branch 安全检查、PyTorch `USE_NUMPY`、源码树外 smoke、可选 `vllm` repair 都不能省略。

---

## 只修本地 `vllm` / `vllm-dlc`

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请使用 `/work/skills/skills/engineering/dlc-env-setup/` 中的 `dlc-env-setup` skill，只修本地 `vllm` / `vllm-dlc` 安装链路，不要重复全量重建原生依赖，除非你验证发现底层 PyTorch wheel 本身不健康。

先读取：
- /work/chipltech-knowledge-base/CONTEXT.md
- /work/chipltech-knowledge-base/runtime-debugging/dlc-workstation-env-rebuild.md
- /work/chipltech-knowledge-base/debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md
- /work/chipltech-knowledge-base/debugging-workflows/post-install-runtime-smoke.md

【vllm 仓库根目录】<路径或“请自动发现”>
【vllm-dlc 仓库根目录】<路径或“请自动发现”>
【PyTorch wheel 路径】<路径或“请自动发现”>
【当前症状】<例如 packaging 缺失 / UNKNOWN metadata / build_editable hook 缺失 / import vllm_dlc 失败>

请按下面顺序执行：
1. 先验证当前 PyTorch wheel 是否健康：
   - 在源码树外执行 `import torch`
   - 验证 `torch.tensor([0.1], dtype=torch.float32).numpy()`
   - 验证 `torch.backends.dlc` 是否可用
2. 如果 PyTorch wheel 不健康，立即停止，并明确说明这不是单纯的 `vllm` 问题。
3. 如果 PyTorch wheel 健康，再做 `vllm` preflight：
   - `pip`、`packaging`、`setuptools`、`setuptools-scm`、`wheel`、`ninja`、`jinja2`
   - `pybind11`
   - `grpcio-tools`
4. 优先用 editable install 最小修复：
   - metadata 坏了就优先修 metadata
   - `UNKNOWN` 先清掉再重装
   - 只在需要时才扩展到依赖 churn
5. 如果 `vllm-dlc` 里引用的 wheel 路径是别的机器路径，指出并改成当前机器真实 wheel 路径。
6. 最后在源码树外验证：
   - `import vllm`
   - `import vllm_dlc`
   - `pip show vllm`
   - `pip show vllm_dlc`
   - `pip list | rg 'vllm|UNKNOWN|triton|numpy|torch'`

交付物：
- 当前问题属于 PyTorch 底层问题还是 `vllm` packaging 问题。
- preflight 检查结果。
- 实际执行的 repair 命令。
- 最终 import 和 metadata 验证结果。
- 如果失败：失败阶段、失败命令、第一批关键报错、建议回退路径。
```

**关键点**：`vllm` / `vllm-dlc` repair 必须建立在健康的 PyTorch 2.5.0 wheel 之上。不要把底层 NumPy bridge 或 wheel 安装问题误判成 `vllm-dlc` bug。

---

## 什么时候停

1. repo discovery 找到多个同名候选路径，但无法判断 authoritative root。
2. 任一关键仓库缺入口文件。
3. branch 切换前发现未提交改动。
4. PyTorch wheel 未通过源码树外 smoke。
5. `vllm` repair 实际暴露的是底层 PyTorch 问题，而不是 packaging 问题。
