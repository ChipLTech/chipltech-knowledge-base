# 用 `dlc-env-setup` skill 初始化 DLC Ecosystem 工作站源码与基础依赖

专门面向新机器、新容器和缺仓库环境的初始化 prompt 模板。稳定版 `dlc-env-setup` skill 在必需仓库缺失时会立即停止；本模板是在用户明确授权 clone 或下载后才执行的 bootstrap 扩展，负责补齐源码树与 CMake 3.27.0，再把控制权交还给当前 skill。

**阅读说明**：
- `▶ 可复制 prompt` 后的代码块可直接发给 AI，使用前替换 `<>` 占位符。
- 本模板只负责受控初始化和最小验证，不负责 LLVM、PyTorch 或 DLC_Custom_Kernel Repository 的长构建。
- 执行权威来自当前 `/work/skills/skills/engineering/dlc-env-setup/SKILL.md` 及其 `scripts/`；本模板仅增加稳定 skill 明确不做的缺失仓库与 CMake 源码 bootstrap。

---

## 适用场景

- 新机器或新容器中还没有完整的 DLC Ecosystem 源码仓库。
- 缺少 `cmake-3.27.0` 官方源码树。
- 本地 `vllm` / `vllm-dlc` repair 所需仓库尚未 clone。
- 希望先完成可审计的初始化，再进入标准重建流程。

## 初始化模板

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请使用当前 `dlc-env-setup` skill，只做 DLC Ecosystem 工作站 bootstrap，不要开始 LLVM、PyTorch 或 DLC_Custom_Kernel Repository 的长构建。

先读取并遵循：
- /work/chipltech-knowledge-base/CONTEXT.md
- /work/skills/skills/engineering/dlc-env-setup/SKILL.md
- /work/skills/skills/engineering/dlc-env-setup/scripts/pytorch-preflight.sh
- /work/skills/skills/engineering/dlc-env-setup/scripts/vllm-preflight.sh
- /work/skills/skills/engineering/dlc-env-setup/scripts/runtime-smoke.sh
- /work/chipltech-knowledge-base/runtime-debugging/dlc-workstation-env-rebuild.md

说明：当前稳定 skill 对缺失的必需仓库要求立即停止。下面的 clone/download 流程是我明确授权的 bootstrap 扩展；未获得对应授权时，仍按当前 SKILL.md 停止。

【初始化目标根目录】<例如 /work>
【搜索根目录】<例如 /work,$HOME；不确定则写“请自动发现”>
【是否允许 clone 缺失仓库】<是/否>
【是否允许下载 CMake 3.27.0 源码】<是/否>
【是否允许修改 /usr/local】<是/否>
【是否包含 vllm / vllm-dlc】<是/否>
【已批准的仓库 ref】<逐仓库填写 remote URL、branch/tag 和可选 commit SHA；不得自行猜测>
【CMake 3.27.0 SHA-256】<来自 Kitware 官方发布信息或用户批准值；没有则写“未提供”>

请严格按下面流程执行：

1. 先做 repo discovery，不要默认 `/home/workspace`。
   - 搜索 `cmake-3.27.0`、`dlc-thunk`、`DLCsim`、`DLCSynapse`、`DLC_CL`、`LLVM`、`DLC_Custom_Kernel`、`pytorch`。
   - 如果要求包含 `vllm` / `vllm-dlc`，再搜索这两个仓库。
   - 对每个 Git 候选记录并回显：发现路径、`git rev-parse --show-toplevel`、`git remote -v`、`git branch --show-current`、`git rev-parse HEAD`、`git status --short`。
   - 只有 Git root、remote、branch/tag 和 HEAD 与【已批准的仓库 ref】一致或获得用户明确确认后，才能把候选列入 repo map。
   - 同名候选有多个、remote 不匹配、detached HEAD 未获批准或 authoritative root 不明确时立即停止。

2. 缺少仓库时，仅在【是否允许 clone 缺失仓库】为“是”且该仓库有已批准 remote/ref 时补 clone。
   - clone 前确认目标根目录存在、可写，并确认目标路径不存在。
   - 不覆盖已有目录；同名目录不是预期 repo 时立即停止。
   - 使用【已批准的仓库 ref】中的 URL，不把示例 URL 当作授权。常用 remote 参考如下：
     - `git@github.com:ChipLTech/dlc-thunk.git`
     - `git@github.com:ChipLTech/DLCsim.git`
     - `git@github.com:ChipLTech/DLCSynapse.git`
     - `git@github.com:ChipLTech/DLC_CL.git`
     - `git@github.com:ChipLTech/LLVM.git`
     - `git@github.com:ChipLTech/DLC_Custom_Kernel.git`
     - `git@github.com:ChipLTech/pytorch.git`
     - 可选：`git@github.com:ChipLTech/vllm.git`
     - 可选：`git@github.com:ChipLTech/vllm-dlc.git`
   - clone 后切到已批准 branch/tag/commit；不要无条件执行 `git pull`。
   - 对 `pytorch` 仅在 root/remote/branch/HEAD 检查通过后执行 `git submodule update --init --recursive`。
   - clone 完成后重新 discovery，并再次记录 Git root、remote、branch、HEAD 和 status。
   - 未授权 clone、缺少批准 ref 或 ref 不存在时，按当前 SKILL.md 停止。

3. 缺少 `cmake-3.27.0` 时，仅在允许下载后准备官方 source tarball。
   - 下载前确认目标目录和 tarball 路径不会覆盖现有内容。
   - 仅使用 Kitware 官方 release URL：
     - `curl -L -o cmake-3.27.0.tar.gz https://github.com/Kitware/CMake/releases/download/v3.27.0/cmake-3.27.0.tar.gz`
   - 解压前使用用户提供或官方发布的 SHA-256 校验，例如：
     - `printf '%s  %s\n' '<approved-sha256>' 'cmake-3.27.0.tar.gz' | sha256sum -c -`
   - SHA-256 未提供、来源未经批准或校验失败时不要解压，立即停止并报告。
   - 校验通过后执行 `tar -xzf cmake-3.27.0.tar.gz`，并确认 `bootstrap` 与 `CMakeLists.txt` 存在。

4. 整理并回显统一 shell 变量：
   - `export CMAKE_SRC=/path/to/cmake-3.27.0`
   - `export DLC_THUNK_SRC=/path/to/dlc-thunk`
   - `export DLCSIM_SRC=/path/to/DLCsim`
   - `export DLCSYNAPSE_SRC=/path/to/DLCSynapse`
   - `export DLC_CL_SRC=/path/to/DLC_CL`
   - `export LLVM_SRC=/path/to/LLVM`
   - `export DLC_CUSTOM_KERNEL_SRC=/path/to/DLC_Custom_Kernel`
   - `export PYTORCH_SRC=/path/to/pytorch`
   - 可选：`export VLLM_SRC=/path/to/vllm`
   - 可选：`export VLLM_DLC_SRC=/path/to/vllm-dlc`

5. 验证当前 SKILL.md 要求的仓库入口文件：
   - `cmake-3.27.0/bootstrap`
   - `cmake-3.27.0/CMakeLists.txt`
   - `dlc-thunk/compile.sh`
   - `DLCsim/build.sh`
   - `DLCSynapse/compile.sh`
   - `DLC_CL/build.sh`
   - `LLVM/build.sh`
   - `DLC_Custom_Kernel/CMakeLists.txt`
   - `pytorch/setup.py`
   - 可选仓库还需有 `setup.py` 或 `pyproject.toml`。
   - 任一入口缺失时按当前 SKILL.md 停止。

6. 仅在允许修改 `/usr/local` 时安装 CMake 3.27.0。
   - 先检查 `/usr/local/bin/cmake --version`、`command -v cmake ctest cpack` 和默认版本；均为 `3.27.0` 时可跳过安装。
   - 需要安装时，在 `$CMAKE_SRC` 中执行：`./bootstrap --prefix=/usr/local`、`make -j$(nproc)`、`make install`。
   - 遇到缺少系统包时，先识别当前系统的 package manager、权限和变更授权；不要无条件运行 `apt-get`，也不要混用 package manager。
   - 只有确认是 Debian/Ubuntu、`apt-get` 可用且用户允许系统包变更时，OpenSSL 开发头缺失才可使用 `apt-get update && apt-get install -y libssl-dev`，然后从 CMake 阶段重来。
   - 安装后验证 `/usr/local/bin/cmake --version`、`command -v cmake ctest cpack`、`cmake --version`、`ctest --version`、`cpack --version`。

7. 初始化完成后停止，不继续长构建。交付：
   - 最终 repo map，以及每个仓库的 Git root、remote、branch/tag、HEAD、status。
   - 最终 shell 变量映射。
   - clone/download 清单、采用的批准 ref 和 CMake SHA-256 校验结果。
   - CMake 安装与版本验证结果。
   - 如果失败：当前 SKILL.md 或本模板触发的停止条件、失败命令、第一批关键报错和安全回退点。
```

## 什么时候停

以当前 `/work/skills/skills/engineering/dlc-env-setup/SKILL.md` 的 `Stop Immediately When` 为准，并增加以下 bootstrap 条件：未授权 clone/download、缺少已批准 ref、仓库身份检查不通过、下载校验值缺失或不匹配、系统 package manager 或修改权限不明确。

## 相关资料

- `/work/skills/skills/engineering/dlc-env-setup/SKILL.md`（commit `3f04504`）
- `/work/skills/skills/engineering/dlc-env-setup/scripts/pytorch-preflight.sh`
- `/work/skills/skills/engineering/dlc-env-setup/scripts/vllm-preflight.sh`
- `/work/skills/skills/engineering/dlc-env-setup/scripts/runtime-smoke.sh`
- [dlc-env-setup-skill-usage.md](dlc-env-setup-skill-usage.md)
- [../runtime-debugging/dlc-workstation-env-rebuild.md](../runtime-debugging/dlc-workstation-env-rebuild.md)
