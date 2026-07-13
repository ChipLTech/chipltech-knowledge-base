# DLC 工作站环境重建

## 适用场景

- 一台机器上的 DLC Ecosystem 开发环境已经失真，需要从仓库发现、版本检查、构建到验证重新走一遍。
- 需要安全切换 `LLVM`、PyTorch 或其他核心仓库分支，再做部分或全量重建。
- 需要把 PyTorch DLC Backend、DLC_Custom_Kernel Repository、本地 `vllm` 与 `vllm-dlc` 重新对齐到可工作的状态。

## 核心结论

1. 环境重建不是“按记忆执行几条命令”，而是先做 repo discovery，再做 branch 安全检查，再按依赖顺序重建。
2. PyTorch 2.5.0 wheel 的可用性必须同时通过 build 成功和 post-install runtime smoke 验证，不能只看 `bdist_wheel` 是否结束。
3. `vllm` 与 `vllm-dlc` 的 editable install 应建立在一个已经通过 smoke 的 PyTorch DLC Backend 之上，否则后面的 Python packaging 报错容易掩盖真正的底层问题。
4. 自动化入口适合放在 skill；长期可复用的规则、顺序、停止条件和常见坑应留在知识库。

## 操作步骤

### 1. 先做 repo discovery

不要默认仓库位于 `/home/workspace`、`/work` 或某个固定用户目录。先确认当前机器的真实 repo map。

至少要找到这些目录：

- `cmake-3.27.0`
- `dlc-thunk`
- `DLCsim`
- `DLCSynapse`
- `DLC_CL`
- `LLVM`
- `DLC_Custom_Kernel`
- `pytorch`
- 如果要修 `vllm` 路线，再额外确认 `vllm` 与 `vllm-dlc`

建议优先搜索用户给出的 workspace root、当前工作根、`$HOME` 等高概率位置，再扩大范围。找到目录后，不要直接开始构建，先确认每个仓库的关键入口文件是否存在，例如：

- `dlc-thunk/compile.sh`
- `DLCsim/build.sh`
- `DLCSynapse/compile.sh`
- `DLC_CL/build.sh`
- `LLVM/build.sh`
- `DLC_Custom_Kernel/CMakeLists.txt`
- `pytorch/setup.py`

如果同名仓库找到了多个候选路径，先确定哪个是当前机器的 authoritative root，再继续。

### 2. 先做 branch 切换安全规则，再开始重建

如果用户要求先切 branch，再重建，不要跳过 git 状态检查。每个仓库至少确认：

```bash
git status --short
git branch --show-current
git branch -a --list '*目标分支*'
```

遵循下面的安全规则：

1. 有未提交改动时先停下来，不要擅自覆盖或清理。
2. 本地有目标分支时再切到本地分支。
3. 只有 remote-tracking branch 时，再创建 tracking branch。
4. 多个仓库需要联动切换时，先全部完成切换和确认，再开始构建。
5. 不要为了“快速回到干净状态”使用 `git reset --hard`、`git checkout --` 一类 destructive 命令。

### 3. 按依赖顺序重建

推荐顺序：

1. CMake 3.27
2. `dlc-thunk`
3. `DLCsim`
4. `DLCSynapse`
5. `DLC_CL`
6. `LLVM`
7. `DLC_Custom_Kernel`
8. PyTorch 2.5.0 wheel build + reinstall
9. 可选：`vllm` / `vllm-dlc` editable install
10. post-install runtime smoke

这一顺序的目的，是让 PyTorch DLC Backend 和 `vllm` 只建立在已经重建完成的原生依赖之上，避免把底层库未更新、头文件不匹配、旧安装残留等问题拖到 Python 层才暴露。

### 4. CMake 3.27 先确认真的接管了 `PATH`

只看到 `/usr/local/bin/cmake` 正确还不够，还要确认默认 `cmake`、`ctest`、`cpack` 也都来自 3.27.0。某些机器会被用户态 wrapper 抢占，导致后续构建仍然调用旧版本工具。

如果构建 CMake 时在 `Utilities/cmcurl` 阶段报 OpenSSL 头文件缺失，应先补 `libssl-dev`，再重跑构建。

### 5. PyTorch 2.5.0 wheel 重建与重装

PyTorch 阶段的关键不只是“build 结束”，还包括 build 前 preflight、build 后 wheel 重装、以及源码树外 runtime 验证。

建议先做 preflight：

- Python packaging 依赖对齐：`packaging`、`setuptools`、`setuptools-scm`、`wheel`、`ninja`、`jinja2`
- `typing_extensions`
- 强制重装 `numpy==1.26.4`

然后清理旧产物，重新构建 wheel。重点检查：

- `USE_CUDA=0`
- `USE_NUMPY=1`
- `PYTORCH_BUILD_VERSION="2.5.0"`
- `PYTORCH_BUILD_NUMBER=1`

仅看命令退出成功还不够。要在 `build/CMakeCache.txt` 中确认 `USE_NUMPY:BOOL=ON`。否则常见症状是 wheel 装上了，但 `torch.tensor(...).numpy()` 在运行时仍然报 “PyTorch was compiled without NumPy support”。

重装 wheel 后，不要站在 PyTorch 源码树里做 `import torch` 验证。应切到 `/tmp` 或其他源码树外目录，避免误从源码树导入。

### 6. `vllm` / `vllm-dlc` editable install

这一阶段应视为可选附加层，不要与 PyTorch rebuild 混为一个原子步骤。前提是：

- Python 版本在 `>=3.10,<3.12`
- `torch==2.5.0` 的 DLC wheel 已经安装并通过 smoke
- 本地 `vllm` 与 `vllm-dlc` 仓库位置已确认

建议先完成 build preflight，再分别执行 editable install。

常见经验：

- `setuptools` 太旧时，editable install 可能缺 `build_editable` hook。
- 缺 `packaging`、`pybind11` 或 `grpcio-tools` 时，错误通常出现在 metadata generation 或扩展构建阶段。
- 如果 `vllm-dlc` checkout 里写死了某个 wheel 路径，应改成当前机器上真实存在的 wheel 路径，而不是把别人的本地路径当长期规则。
- `UNKNOWN` metadata 常常说明 editable install 元数据本身坏了，优先修复该仓库的 packaging 信息，再做最小 reinstall。

### 7. post-install runtime smoke

环境重建的收口必须是一组短而硬的 smoke，而不是“模型大概能起”这类软判断。至少确认：

- `torch.__version__` 为 `2.5.0`
- `torch.tensor([0.1], dtype=torch.float32).numpy()` 成功
- `torch.backends.dlc.is_available()` 或等价 DLC Platform 检查通过
- 如本次包含 `vllm` / `vllm-dlc`，则 `import vllm` 和 `import vllm_dlc` 均成功

如果 smoke 失败，优先回到对应层处理：

- `import torch` 失败：回到 PyTorch wheel 安装与源码树外验证
- NumPy bridge 失败：回到 PyTorch build 配置与 `USE_NUMPY`
- `vllm` metadata 或 import 失败：回到 Python packaging preflight
- DLC Platform 可见但 kernel 路径有问题：转到 Runtime 排障或测试框架文档

## 常见坑

1. **把固定机器路径当规则**：路径发现是流程的一部分，不是可忽略的杂项。
2. **跳过 git 安全检查**：环境修复经常需要切 branch，但 branch 切换不应覆盖开发中的改动。
3. **只验证 `/usr/local/bin/cmake`**：真正生效的是当前 shell 的 `PATH` 解析结果。
4. **站在 PyTorch 源码树里验证 import**：这会制造假阴性。
5. **wheel build 成功就认定环境成功**：真正的成功标准是 runtime smoke。
6. **把 `vllm` 问题都当成 `vllm-dlc` 问题**：底层 PyTorch NumPy bridge 没修好时，上层问题通常只是连带症状。
7. **把别人的 wheel 路径写进长期文档**：知识库只写规则，机器路径只作为一次性来源。

## 可选自动化入口

如果希望 agent 自动执行 repo discovery、branch 安全检查、分阶段重建和最终 smoke，可使用 `/work/skills/skills/engineering/dlc-env-setup/` 中的 `dlc-env-setup` skill。

- skill 负责执行编排、脚本入口和停止条件。
- 本文负责解释为什么这样排顺序、哪些边界必须守住、失败时应回退到哪一层。

## 相关资料

- [python-build-preflight-for-pytorch-and-vllm.md](../debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md)
- [post-install-runtime-smoke.md](../debugging-workflows/post-install-runtime-smoke.md)
- [runtime-troubleshooting.md](runtime-troubleshooting.md)
- [environment-setup-and-update.md](environment-setup-and-update.md)
- [common-debug-commands.md](../debugging-workflows/common-debug-commands.md)
- [CONTEXT.md](../CONTEXT.md)

## 来源

- `/work/test/dlc-env-setup/SKILL.md`
- `/work/test/dlc-env-setup/scripts/pytorch-preflight.sh`
- `/work/test/dlc-env-setup/scripts/vllm-preflight.sh`
- `/work/test/dlc-env-setup/scripts/runtime-smoke.sh`
- `/work/chipltech-knowledge-base/runtime-debugging/environment-setup-and-update.md`
