# 用 `dlc-env-setup` skill 初始化 DLC 工作站源码与基础依赖

专门面向新机器 / 新容器 / 缺仓库环境的初始化 prompt 模板。目标是先把 `dlc-env-setup` 后续所需的关键源码仓库、可选的 `vllm` / `vllm-dlc` 仓库，以及 `cmake-3.27.0` 源码树准备齐，再把结果交给重建模板或本地 `vllm` repair 模板继续处理。

**📖 阅读说明**：
- `▶ 可复制 prompt` 标记后面的代码块是发给 AI 的指令，替换 `<>` 里的占位符即可使用。
- 这个模板只负责初始化下载、可选 `vllm` / `vllm-dlc` 下载、CMake 3.27.0 安装和最小验证，不负责后续 LLVM / PyTorch / `DLC_Custom_Kernel` 全量构建。

---

## 适用场景

- 新机器或新容器里还没有完整的 DLC Ecosystem 源码仓库。
- 机器上缺少 `cmake-3.27.0` 源码树，后续没法按标准路径安装 CMake 3.27.0。
- 你要做本地 `vllm` / `vllm-dlc` repair，但机器上还没把 `vllm` 或 `vllm-dlc` 仓库 clone 下来。
- 你希望先把初始化阶段收口，再切到重建模板继续做后续构建。

## 初始化目标

- 在指定根目录下准备这些仓库：`dlc-thunk`、`DLCsim`、`DLCSynapse`、`DLC_CL`、`LLVM`、`DLC_Custom_Kernel`、`pytorch`
- 如果用户要求，也准备：`vllm`、`vllm-dlc`
- 准备 `cmake-3.27.0` 官方源码树
- 在允许修改 `/usr/local` 的前提下安装 CMake `3.27.0`
- 输出 repo map、shell 变量映射和最小版本验证结果

## 初始化模板

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请使用 `dlc-env-setup` skill，先只做 DLC 工作站环境初始化，不要直接开始 LLVM、PyTorch 或 `DLC_Custom_Kernel` 的长构建。目标是把后续重建所需的源码仓库和 CMake 3.27.0 准备好，并完成最小验证。

先读取：
- /work/chipltech-knowledge-base/CONTEXT.md
- /work/chipltech-knowledge-base/README.md
- /work/chipltech-knowledge-base/runtime-debugging/dlc-workstation-env-rebuild.md
- /work/chipltech-knowledge-base/debugging-workflows/post-install-runtime-smoke.md

【初始化目标根目录】<例如 /work>
【搜索根目录】<例如 /work,$HOME；如果不确定就写“请自动发现”>
【是否允许 clone 缺失仓库】<是/否>
【是否允许下载 CMake 3.27.0 源码】<是/否>
【是否允许修改 /usr/local】<是/否>
【是否包含 vllm / vllm-dlc】<是/否>

请严格按下面流程执行：

1. 先做 repo discovery，不要默认 `/home/workspace`。
   - 搜索这些仓库：`cmake-3.27.0`、`dlc-thunk`、`DLCsim`、`DLCSynapse`、`DLC_CL`、`LLVM`、`DLC_Custom_Kernel`、`pytorch`。
   - 如果【是否包含 vllm / vllm-dlc】是“是”，再额外搜索 `vllm` 和 `vllm-dlc`。
   - 回显当前 repo map。
   - 如果有同名候选路径多个，先停止并列出候选项，不要擅自选。

2. 如果缺少关键仓库，并且【是否允许 clone 缺失仓库】是“是”，在【初始化目标根目录】下补 clone。
   - clone 前先确认目标根目录存在。
   - 不要覆盖已有目录；如果已有同名目录但不是预期 repo，立即停止。
   - 缺哪个补哪个，基础仓库清单固定如下：
     - `git clone git@github.com:ChipLTech/dlc-thunk.git`
     - `git clone git@github.com:ChipLTech/DLCsim.git`
     - `git clone git@github.com:ChipLTech/DLCSynapse.git`
     - `git clone git@github.com:ChipLTech/LLVM.git`
     - `git clone git@github.com:ChipLTech/DLC_Custom_Kernel.git`
     - `git clone git@github.com:ChipLTech/DLC_CL.git`
     - `git clone git@github.com:ChipLTech/pytorch.git`
   - 如果【是否包含 vllm / vllm-dlc】是“是”，再补：
     - `git clone git@github.com:ChipLTech/vllm.git`
     - `git clone git@github.com:ChipLTech/vllm-dlc.git`
   - `pytorch` clone 完成后，还要执行：
     - `cd "$PYTORCH_SRC"`
     - `git pull`
     - `git submodule update --init --recursive`
   - clone 完成后重新做 repo discovery，并重新回显 repo map。
   - 如果【是否允许 clone 缺失仓库】是“否”，则在发现缺库时立即停止。

3. 如果缺少 `cmake-3.27.0`，并且【是否允许下载 CMake 3.27.0 源码】是“是”，准备官方 source tarball。
   - 下载到【初始化目标根目录】下。
   - 只允许使用官方 release source tarball：
     - `cd <bootstrap-root>`
     - `curl -L -o cmake-3.27.0.tar.gz https://github.com/Kitware/CMake/releases/download/v3.27.0/cmake-3.27.0.tar.gz`
     - `tar -xzf cmake-3.27.0.tar.gz`
   - 解压后确认目录里至少有：
     - `bootstrap`
     - `CMakeLists.txt`
   - 如果【是否允许下载 CMake 3.27.0 源码】是“否”，则在发现缺失时立即停止。

4. 整理统一复用的 shell 变量，并原样回显：
   - `export CMAKE_SRC=/path/to/cmake-3.27.0`
   - `export DLC_THUNK_SRC=/path/to/dlc-thunk`
   - `export DLCSIM_SRC=/path/to/DLCsim`
   - `export DLCSYNAPSE_SRC=/path/to/DLCSynapse`
   - `export DLC_CL_SRC=/path/to/DLC_CL`
   - `export LLVM_SRC=/path/to/LLVM`
   - `export DLC_CUSTOM_KERNEL_SRC=/path/to/DLC_Custom_Kernel`
    - `export PYTORCH_SRC=/path/to/pytorch`
    - 如果包含 `vllm` / `vllm-dlc`，再补：
      - `export VLLM_SRC=/path/to/vllm`
      - `export VLLM_DLC_SRC=/path/to/vllm-dlc`

5. 验证关键入口文件：
   - `cmake-3.27.0/bootstrap`
   - `cmake-3.27.0/CMakeLists.txt`
   - `dlc-thunk/compile.sh`
   - `DLCsim/build.sh`
   - `DLCSynapse/compile.sh`
   - `DLC_CL/build.sh`
    - `LLVM/build.sh`
    - `DLC_Custom_Kernel/CMakeLists.txt`
    - `pytorch/setup.py`
    - 如果包含 `vllm` / `vllm-dlc`，再确认：
      - `vllm/setup.py` 或 `vllm/pyproject.toml`
      - `vllm-dlc/setup.py` 或 `vllm-dlc/pyproject.toml`
    - 缺任何一个都要立即停止并汇报。

6. 如果【是否允许修改 /usr/local】是“是”，安装 CMake 3.27.0；否则只验证源码树存在后停止。
   - 安装前先检查：
     - `/usr/local/bin/cmake --version`
     - `command -v cmake ctest cpack`
   - 如果当前默认 `cmake` / `ctest` / `cpack` 已经都是 `3.27.0`，可以跳过安装，但仍要输出验证结果。
   - 如果不是 `3.27.0`，按下面路径安装：
     - `cd "$CMAKE_SRC"`
     - `./bootstrap --prefix=/usr/local`
     - `make -j$(nproc)`
     - `make install`
   - 如果 `./bootstrap` 或 configure 报 OpenSSL 头文件 / library 缺失，优先安装：
     - `apt-get update && apt-get install -y libssl-dev`
     然后从 CMake 阶段重来。

7. CMake 安装后必须验证：
   - `/usr/local/bin/cmake --version`
   - `command -v cmake ctest cpack`
   - `cmake --version`
   - `ctest --version`
   - `cpack --version`

8. 初始化完成后不要继续做长构建，只交付初始化结果。

交付物必须包括：
- 最终 repo map。
- 最终 shell 变量映射。
- clone 了哪些仓库。
- 如果包含 `vllm` / `vllm-dlc`，是否成功 clone 了这两个仓库。
- 是否下载了解压 `cmake-3.27.0`。
- CMake 3.27.0 安装和版本验证结果。
- 如果失败：失败阶段、失败命令、第一批关键报错、建议下一步。
```

## 什么时候停

1. repo discovery 找到多个同名候选路径，但无法判断 authoritative root。
2. 关键仓库缺失且不允许 clone。
3. `cmake-3.27.0` 缺失且不允许下载。
4. 目标根目录已存在同名目录但内容不符合预期。
5. CMake 安装时缺少 `libssl-dev` 之外的更深层依赖，且无法在当前阶段安全解决。

## 相关资料

- [dlc-env-setup-skill-usage.md](dlc-env-setup-skill-usage.md)
- [../runtime-debugging/dlc-workstation-env-rebuild.md](../runtime-debugging/dlc-workstation-env-rebuild.md)
- [../debugging-workflows/post-install-runtime-smoke.md](../debugging-workflows/post-install-runtime-smoke.md)
