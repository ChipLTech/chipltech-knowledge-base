# 用 `dlc-env-setup` skill 重建 DLC 工作站环境

专门面向 DLC 工作站环境重建 / 修复的 prompt 模板。目标是直接驱动 AI 调用已经通过 `skills/scripts/link-kilo-skills.sh` 暴露给 Kilo 的 `dlc-env-setup` skill 或 `/dlc-env-setup` slash command，覆盖 `/work/test/dlc-env-setup` 里的同等级能力：repo discovery、branch 切换安全检查、分阶段重建、PyTorch 2.5.0 wheel、可选 `vllm` / `vllm-dlc` repair，以及源码树外 runtime smoke。

**📖 阅读说明**：
- `▶ 可复制 prompt` 标记后面的代码块是发给 AI 的指令，替换 `<>` 里的占位符即可使用。
- 标题下的“适用场景”“关键点”是给人看的说明，不用复制发给 AI。

---

## 适用场景

- 一台机器上的 DLC Ecosystem 开发环境已经坏掉，需要从仓库发现、版本切换、构建到验证完整走一遍。
- 已知大部分依赖还在，但需要从 LLVM、`DLC_Custom_Kernel`、PyTorch wheel 或 `vllm` 路线继续修复。
- 希望 AI 不只是解释文档，而是按 skill 的闭环真正执行，并在每个阶段给出最小验证结果。

## 使用前提

- 在 `skills` 仓库根目录运行过：`./scripts/link-kilo-skills.sh --with-commands`
- 这样 Kilo 会获得：
  - skill symlink：`~/.config/kilo/skills/dlc-env-setup`
  - slash command wrapper：`~/.config/kilo/command/dlc-env-setup.md`
- 如果你只运行了 `./scripts/link-kilo-skills.sh`，没有 `--with-commands`，那就用自然语言明确要求“请使用 `dlc-env-setup` skill”。
- 如果已经运行了 `--with-commands`，优先直接使用 `/dlc-env-setup ...`。

## 全量或分阶段重建

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请使用 `dlc-env-setup` skill，帮我重建或修复这台机器上的 DLC 工作站环境。不要跳步骤，要按 skill 和知识库规则执行，并在每个阶段做最小验证。

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
   - 如果缺少关键仓库，先不要在这个模板里直接发明初始化步骤，转而参考：`/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-environment-bootstrap.md`
   - 只有当初始化完成、关键仓库已经齐备后，才继续当前重建模板。
   - 把最终 repo map 原样回显给我。
   - 找到后，把这些路径整理成后续步骤统一复用的 shell 变量：
     - `CMAKE_SRC`
     - `DLC_THUNK_SRC`
     - `DLCSIM_SRC`
     - `DLCSYNAPSE_SRC`
     - `DLC_CL_SRC`
     - `LLVM_SRC`
     - `DLC_CUSTOM_KERNEL_SRC`
     - `PYTORCH_SRC`
     - 如果包含 `vllm` / `vllm-dlc`，再补：`VLLM_SRC`、`VLLM_DLC_SRC`
   - 像这样回显变量和值，确保后面的命令不会混用路径：
     - `export CMAKE_SRC=/path/to/cmake-3.27.0`
     - `export DLC_THUNK_SRC=/path/to/dlc-thunk`
     - `export DLCSIM_SRC=/path/to/DLCsim`
     - `export DLCSYNAPSE_SRC=/path/to/DLCSynapse`
     - `export DLC_CL_SRC=/path/to/DLC_CL`
     - `export LLVM_SRC=/path/to/LLVM`
     - `export DLC_CUSTOM_KERNEL_SRC=/path/to/DLC_Custom_Kernel`
     - `export PYTORCH_SRC=/path/to/pytorch`
   - 如果有同名仓库多个候选路径，先停下来说明候选项，不要擅自选。

2. 对每个关键仓库验证入口文件：
   - `cmake-3.27.0/bootstrap`
   - `cmake-3.27.0/CMakeLists.txt`
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
   - CMake 3.27.0 安装和版本验证
   - `dlc-thunk`
   - `DLCsim`
   - `DLCSynapse`
   - `DLC_CL`
   - `LLVM`
   - `DLC_Custom_Kernel`
   - PyTorch 2.5.0 wheel build + force reinstall
   - 可选：`vllm` / `vllm-dlc` editable install
   - final runtime smoke

   分阶段模式必须额外遵守这些安全门：
   - 如果【模式】是“从 LLVM 开始”，先确认 DLC core stack 已经安装，再从 LLVM 往后走。
   - 如果【模式】是“从 DLC_Custom_Kernel 开始”，先确认 LLVM 已经是健康可用状态。
   - 如果【模式】是“只重建 PyTorch wheel”，先确认这些前置依赖仍然存在：
     - `/usr/local/chipltech/simulator`
     - `/usr/local/chipltech/synapse`
     - `/usr/local/chipltech/dlccl`
     - `LLVM` build 下的 `build/bin/clang`
   - 如果【模式】是“只重装 wheel”，先确认 `dist/` 里已经存在 fresh `torch-2.5.0*.whl`。
   - 如果你无法确认更早阶段依赖仍然健康，就不要跳阶段，回退到更早阶段重建。

5. CMake 3.27.0 阶段必须检查：
   - 如果 `/usr/local/bin/cmake --version` 不是 `3.27.0`，先停止，并转去参考初始化模板：`/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-environment-bootstrap.md`
   - 只有当初始化模板已经把 CMake 3.27.0 安装好后，才继续当前重建模板。
   - 当前模板里必须检查：
     - `/usr/local/bin/cmake --version`
     - `command -v cmake ctest cpack`
     - `cmake --version`
     - `ctest --version`
     - `cpack --version`

6. PyTorch 阶段必须先做 preflight：
   - 安装或升级 `typing_extensions`、`packaging`、`setuptools`、`setuptools-scm`、`wheel`、`ninja`、`jinja2`
   - 强制 `numpy==1.26.4`
   - 如已有 `build/CMakeCache.txt`，检查 `USE_NUMPY:BOOL=ON`
   - 然后再清理旧产物并构建 wheel
   - 构建完成后，从 `dist/` 中 force-reinstall 最新 `torch-2.5.0*.whl`

7. LLVM 和 `DLC_Custom_Kernel` 阶段不要自由发挥，优先复用已验证路径：
   - 如果 LLVM 之前失败过、半构建过，或你无法确认当前 build 目录是干净的，优先使用 `./build.sh -r`，不要先赌普通 `./build.sh`。
   - LLVM 阶段应明确给出你实际采用的是哪条路径：
     - clean rebuild：
       - `cd "$LLVM_SRC"`
       - `./build.sh -r`
     - 普通 rebuild：
       - `cd "$LLVM_SRC"`
       - `./build.sh`
   - `DLC_Custom_Kernel` 配置时显式指定 `LLVM_PATH` 指向当前 LLVM 源树。
   - `DLC_Custom_Kernel` 必须显式使用 `/usr/bin/ninja`，不要依赖 `/usr/local/bin/ninja` 是否存在。
   - `DLC_Custom_Kernel` 阶段按这个命令骨架执行，不要自行换工具链：
     - `cd "$DLC_CUSTOM_KERNEL_SRC"`
     - `mkdir -p build`
     - `cd build`
     - `LLVM_PATH="$LLVM_SRC" cmake ../ -G Ninja -DCMAKE_MAKE_PROGRAM=/usr/bin/ninja -DBUILD_TARGET=ALL -DCMAKE_CXX_FLAGS="-fdiagnostics-color=always" -DCMAKE_C_FLAGS="-fdiagnostics-color=always"`
     - `/usr/bin/ninja install`
   - `DLC_Custom_Kernel` 安装后，至少确认这些产物应落位到 `/usr/local/chipltech/synapse/` 下：
     - `lib/libcustom_dlc_perf_lib.so`
     - `include/enabled_kernels.hpp`
     - `bin/ping`
     - `bin/rdma_perf`
     - `bin/rdma_self_perf`
     - `bin/rsync_check`

8. 如果【是否包含 vllm / vllm-dlc】是“是”，在 PyTorch smoke 通过后再继续：
   - 先确认 Python 版本在 `>=3.10,<3.12`
   - 先记录：
     - `python3 --version`
     - `python3 -m pip --version`
     - `python3 -m pip list | rg 'torch|vllm|UNKNOWN|triton|numpy|opencv'`
     - `python3 -c "import torch; print(torch.__version__); print(torch.backends.dlc.is_available())"`
   - 先做 `vllm` preflight：`pip`、`packaging`、`setuptools`、`setuptools-scm`、`wheel`、`ninja`、`jinja2`、`pybind11`、`grpcio-tools`
   - 安装 `vllm` 时优先用本地源码，并显式走：
     - `VLLM_TARGET_DEVICE=empty python3 -m pip install -v -e . --no-build-isolation`
   - 如果只是 metadata 损坏，优先走最小 repair 路线：
     - `VLLM_TARGET_DEVICE=empty python3 -m pip install -v -e . --no-build-isolation --no-deps`
   - 如果 `pip list` 或 `pip show` 报 `UNKNOWN`，优先清掉 `UNKNOWN` 并修复 `vllm` metadata，再 reinstall。
   - 安装 `vllm-dlc` 前先安装：
     - `pybind11`
     - `grpcio-tools==1.78.0`
   - 安装 `vllm-dlc` 时显式走：
     - `python3 -m pip install -v -e . --no-build-isolation`
   - 如果发现 wheel 路径写死成别的机器路径，指出并改成当前机器真实路径
   - 如果装完 `vllm-dlc` 后环境里仍有 `triton`，按原 skill 路线显式卸掉：
     - `python3 -m pip uninstall -y triton`

9. final runtime smoke 必须在源码树外执行，例如 `/tmp`：
   - `python3 -c "import torch; print(torch.__version__)"`
   - `python3 -c "import torch; print(torch.tensor([0.1], dtype=torch.float32).numpy())"`
   - `python3 -c "import torch; print(torch.backends.dlc.is_available() if hasattr(torch.backends, 'dlc') else 'no torch.backends.dlc')"`
   - `ls /usr/local/chipltech/synapse/bin`
   - 如果包含 `vllm`：
      - `python3 -c "import vllm; print(vllm.__version__)"`
      - `python3 -c "import vllm_dlc; print(vllm_dlc.__file__)"`
      - `python3 -m pip show vllm`
      - `python3 -m pip show vllm_dlc`
      - `python3 -m pip list | rg 'vllm|UNKNOWN|triton|numpy|opencv|fastapi'`

   final verification 至少要明确判断这些结果是否达标：
   - `/usr/local/bin/cmake` 是 `3.27.0`
   - 默认 `cmake`、`ctest`、`cpack` 也都是 `3.27.0`
   - PyTorch 版本是 `2.5.0`
   - NumPy bridge 正常
   - `/usr/local/chipltech/synapse/bin` 中存在 custom-kernel 相关工具
   - 如果包含 `vllm`，则 `import vllm`、`import vllm_dlc`、`pip show`、`pip list` 都收口正常

10. 任一阶段失败时：
   - 立即停止
   - 报告失败命令
   - 报告第一批关键报错
   - 说明应该回退到哪一层处理
   - 不要在失败状态下盲目继续到后续阶段
   - 优先采用这些已验证回退路径，而不是临时发明新方案：
      - 关键 repo 缺失：转去参考 `/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-environment-bootstrap.md`
      - `cmake-3.27.0` 缺失或默认 `cmake` 不是 `3.27.0`：转去参考 `/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-environment-bootstrap.md`
      - LLVM 失败：优先重试 `./build.sh -r`
      - PyTorch 失败：先 `make clean`，再重建 wheel
      - `DLC_Custom_Kernel` 失败：继续坚持显式 `/usr/bin/ninja` 路线
      - repo discovery 出现多个同名候选路径：停下来让用户确认 authoritative root

交付物必须包括：
- 最终 repo map。
- 最终 shell 变量映射。
- branch 检查和切换结果。
- 本次从哪个阶段开始、为什么可以从这里开始。
- 每个阶段的最小验证结果。
- PyTorch wheel 路径和 reinstall 结果。
- final runtime smoke 结果。
- 如果失败：失败阶段、失败命令、第一批关键报错、建议回退路径。
```

**关键点**：这个模板的目标不是“解释怎么装”，而是让 AI 真正调用 `dlc-env-setup` skill 按闭环执行。repo discovery、branch 安全检查、PyTorch `USE_NUMPY`、源码树外 smoke、可选 `vllm` repair 都不能省略。

### 如果已经生成 slash command

运行过 `./scripts/link-kilo-skills.sh --with-commands` 后，也可以直接用下面这种更短的方式：

```text
/dlc-env-setup <把上面模板里的需求和约束直接作为参数传进去>
```

### 原始 skill 的常见请求写法

下面这些句式可以直接塞进上面的模板里，用来逼近原始 skill 的典型调用方式：

- `请先 rediscover the repos，确认 repo map，然后从 LLVM onward 开始重建。`
- `请切换 pytorch 到 release/2.5-dlc，只重建 PyTorch wheel，不要碰更早阶段，除非你验证发现前置依赖不健康。`
- `请切换 LLVM 和 DLC_Custom_Kernel 到匹配分支，先把两个仓库都切好并确认，再从 LLVM 开始重建。`
- `请只 reinstall 最新 wheel，不要重编，但必须先确认 dist/ 里已有 fresh wheel。`

---

## 只修本地 `vllm` / `vllm-dlc`

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请使用 `dlc-env-setup` skill，只修本地 `vllm` / `vllm-dlc` 安装链路，不要重复全量重建原生依赖，除非你验证发现底层 PyTorch wheel 本身不健康。

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
3. 先检查 `vllm` 和 `vllm-dlc` 仓库是否都存在。
   - 如果缺少任意一个，不要在这个模板里临时发明初始化步骤，转去参考：`/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-environment-bootstrap.md`
   - 只有当 `vllm` 和 `vllm-dlc` 仓库都已齐备后，才继续当前 repair 模板。
4. 如果 PyTorch wheel 健康，再做 `vllm` preflight：
    - 先确认 Python 版本在 `>=3.10,<3.12`
    - `pip`、`packaging`、`setuptools`、`setuptools-scm`、`wheel`、`ninja`、`jinja2`
    - `pybind11`
    - `grpcio-tools`
    - 同时确认当前本地 `vllm` 与 `vllm-dlc` 至少在 Python import surface 上兼容；如果 `vllm-dlc` 代码引用的 `vllm.*` 模块在当前 `vllm` 仓库里根本不存在，先停止并要求对齐这两个仓库的匹配分支，而不是盲目继续安装
5. 优先用 editable install 最小修复：
   - `vllm` 默认先走：`VLLM_TARGET_DEVICE=empty python3 -m pip install -v -e . --no-build-isolation`
   - metadata 坏了就优先修 metadata
   - 如果只是 metadata 损坏，优先走：`VLLM_TARGET_DEVICE=empty python3 -m pip install -v -e . --no-build-isolation --no-deps`
   - `UNKNOWN` 先清掉再重装
   - 只在需要时才扩展到依赖 churn
6. 如果 `vllm-dlc` 里引用的 wheel 路径是别的机器路径，指出并改成当前机器真实 wheel 路径，例如把错误的 `file:///root/.../torch-2.5.0...whl` 改成当前机器真实存在的 wheel 路径。
7. 安装 `vllm-dlc` 前先确保已经安装 `pybind11` 和 `grpcio-tools==1.78.0`，然后显式走：
   - `python3 -m pip install -v -e . --no-build-isolation`
8. 如果安装后仍残留 `triton`，按原始 skill 路线显式清理：
   - `python3 -m pip uninstall -y triton`
9. 最后在源码树外验证：
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

### 原始 skill 的常见请求写法

- `请先确认 PyTorch wheel 健康，再 repair 本地 vllm 和 vllm-dlc editable install。`
- `如果只是 metadata 坏了，请优先走最小 repair 路线，不要大范围 churn 依赖。`
- `如果 pip show 或 pip list 里出现 UNKNOWN，请先修 metadata，再重新安装。`
- `如果运行时报 PyTorch was compiled without NumPy support，请停止 vllm repair，回退到 PyTorch rebuild。`

---

## 什么时候停

1. repo discovery 找到多个同名候选路径，但无法判断 authoritative root。
2. 任一关键仓库缺入口文件。
3. branch 切换前发现未提交改动。
4. PyTorch wheel 未通过源码树外 smoke。
5. `vllm` repair 实际暴露的是底层 PyTorch 问题，而不是 packaging 问题。
6. `vllm` 或 `vllm-dlc` 仓库本地根本不存在，应先转去 bootstrap 模板初始化。
7. `vllm-dlc` 依赖的 `vllm` Python 模块路径在当前本地 `vllm` 仓库中不存在，说明两个仓库版本不匹配，应先对齐匹配分支。
