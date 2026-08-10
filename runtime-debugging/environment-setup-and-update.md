# 环境配置与仓库更新

## 适用场景

- 初次搭建 DLC 开发环境。
- 更新 DLC 相关仓库版本。
- 排查 `undefined symbol`、import 失败、编译失败等版本对齐问题。

## 核心结论

DLC Ecosystem 仓库存在严格的版本依赖链，更新顺序和编译方式必须一致，否则会出现 import 失败、undefined symbol、peek 异常等版本不匹配错误。

更新不是“按记忆执行几条命令”。在更新运行中的仓库前，先确认实际 import 的 editable source，再核对 binary identity，再决定编译顺序；否则容易产生“源码已更新但运行仍用旧 checkout”或“source 已新但 binary 仍旧”的假象。

## 构建依赖链

```
LLVM
  -> dlc-thunk
  -> DLCsim
  -> DLCSynapse
  -> DLC_Custom_Kernel
  -> DLC_CL
  -> PyTorch
  -> vLLM
```

## 各仓库更新步骤

### LLVM

```bash
# main 分支
git clone /path/to/llvm
cd llvm
./build.sh
```

**只检查 Git HEAD 不足以证明 toolchain identity。** LLVM source 已是最新 main 不代表实际 clang 由该 source 构建。必须读取实际 compiler binary 自报 SHA（例如 `clang` 报告的构建来源 SHA）或其他可审计 binary identity，并确认其与 source HEAD 一致。否则会出现 source 中已存在某 option、但 binary 仍报 `Unknown command line argument` 的不配对问题。clean rebuild 后重新核对 binary 自报 SHA。

### dlc-thunk

```bash
# master 分支；当前仍被 PyTorch DLC Backend 的 ext/dlc-thunk 和 DLCSynapse 构建链引用
git clone /path/to/dlc-thunk
cd dlc-thunk
# 按当时文档构建
```

### DLCsim

```bash
# main 分支
# 构建命令按仓库文档
```

### DLCSynapse

```bash
# main 分支
git clone /path/to/synapse
cd synapse
./compile.sh
```

### DLC_Custom_Kernel

```bash
# develop 分支
git clone /path/to/DLC_Custom_Kernel
cd DLC_Custom_Kernel
mkdir build && cd build
cmake ..
ninja -j100
```

### DLC_CL

```bash
# inc_nsteps 分支
git clone /path/to/DLC_CL
cd DLC_CL
./build.sh
```

### PyTorch

初始化 PyTorch submodule 时，必须优先复用已有可用镜像中的 `third_party`，不要默认联网重新下载全部第三方库：

1. 确认来源镜像中的 PyTorch `third_party` 可用，并记录来源、目标 branch/HEAD。
2. 将 `third_party` 完整复制到目标 PyTorch 源码树，保留目录结构和隐藏文件。
3. 再执行 `git submodule sync --recursive` 和 `git submodule update --init --recursive`，按当前 checkout 校验并补齐。
4. 只有镜像缓存不存在或无法满足当前 checkout，且任务允许网络下载时，才下载缺失内容。

**强制优先级：已有镜像 `third_party` > 远端下载。已有第三方库可复用时，不得浪费时间重新下载。**

```bash
# release_25 分支
cd /work/pytorch
git submodule sync --recursive
git submodule update --init --recursive
USE_CUDA=0 DEBUG=1 MAX_JOBS=32 python3 setup.py develop
```

如需显式 wheel 版本，可按发布流程额外设置 `PYTORCH_BUILD_VERSION` 和 `PYTORCH_BUILD_NUMBER`；当前 `/work/pytorch/build.sh` 使用的是上面的 develop 构建命令。

构建 wheel：
```bash
USE_CUDA=0 python3 setup.py bdist_wheel
```

### vLLM

```bash
# update-v0.11.0 分支
cd /work/vllm
VLLM_TARGET_DEVICE=dlc pip install -e .
```

## 更新前确认实际 editable source

同一台机器或容器内可能同时存在多套 checkout（例如 `/work/*` 与 `/work/minimax-src/*`）。更新运行中的仓库前，先通过以下方式确认实际 import 的 editable source，避免更新错误仓库后产生“源码已更新但运行仍用旧 checkout”的假象：

- 检查安装元数据（如 `direct_url.json`）。
- 检查 `module.__file__` 指向的源码路径。
- 检查实际进程/服务命令使用的路径。

editable package 的 metadata version 字符串来自安装时生成，**不随源码 checkout 更新而变**。判定运行源码时应同时记录 editable `direct_url.json` 和 source Git SHA，不能只看 package version 字符串。

## 容器内无 SSH identity 时的 fetch 中转

容器内 remote 需要 SSH 但容器没有可用 identity 时，直接 fetch 会报 `Permission denied (publickey)`。若宿主机具备已批准的私有仓库读取能力，可用宿主机只读 mirror + Git bundle 中转，避免复制或输出私钥：

1. 宿主机验证 `git ls-remote`。
2. 宿主机创建 bare mirror。
3. 从 mirror 生成只包含目标 refs 的 Git bundle。
4. 容器仓库从挂载的 bundle fetch。

该方法不把 SSH 私钥复制进容器，也不在日志或命令参数中暴露凭据。

## 版本验证

```bash
# 当前 /work 顶层确认存在
/work/check_version.sh
```

未在 `/work` 顶层发现 `check_version.py`。如使用 Python 脚本，先确认具体路径，例如其他工具仓库内的版本检查脚本。

## 环境验证

快速验证环境是否正确：

```bash
# 用 llama 7b 做简单推理测试
# 确认模型能加载、prefill、decode
```

## 常见环境问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `undefined symbol` | .so 版本不一致 | 按依赖链顺序重新编译 |
| PyTorch import 失败 | DLC 相关 .so 缺失 | 重新编译 PyTorch |
| vLLM 编译失败 | 依赖版本未对齐 | 检查前置仓库版本 |
| peek 参数异常 | synapse 版本不匹配 | 重新编译 synapse |
| CMake 版本过低 | 系统 cmake 太旧 | 升级 cmake |
| `dlc_runtime_api.h not found` | synapse include path 未设置 | 检查环境变量 |
| NumPy 1.x/2.x 不兼容 | numpy 版本不对 | `pip install "numpy<2"` |
| `libopenblas.so` 缺失 | 缺少 openblas | `apt install libopenblas-dev` |
| `numpy/arrayobject.h not found` | numpy include path 未设置 | 添加到 CPLUS_INCLUDE_PATH |
| PyTorch submodule 初始化耗时过长 | 未复用已有镜像中的 `third_party` | 先复制镜像缓存，再执行 recursive submodule update |
| source 已含某 option 但编译报 Unknown argument | Git HEAD 新但实际 binary 仍由旧 SHA 构建 | clean rebuild LLVM，并核对 clang 自报 SHA 与 source HEAD |
| 更新后运行仍用旧 checkout | 容器存在多套 checkout，editable metadata 版本不随源码变 | 用 `direct_url.json` / `module.__file__` / 进程命令确认实际 editable source |
| 容器内 fetch 报 Permission denied | 容器无 SSH identity | 宿主机只读 mirror + Git bundle 中转 |

## 多组件版本确认清单

环境配置完成后确认：
1. LLVM 版本
2. DLCSynapse 版本
3. DLC_Custom_Kernel 版本
4. DLC_CL 版本
5. PyTorch 版本 + wheel 版本
6. vLLM 版本
7. DLC kernel driver 版本
8. DLC Runtime API 版本

## 相关资料

- [debugging-workflows/common-debug-commands.md](../debugging-workflows/common-debug-commands.md)
- [runtime-debugging/runtime-troubleshooting.md](runtime-troubleshooting.md)
- [runtime-debugging/dlc-workstation-env-rebuild.md](dlc-workstation-env-rebuild.md)
- [case-studies/host-api22-fullstack-main-to-main-update.md](../case-studies/host-api22-fullstack-main-to-main-update.md)

## 来源

- `/work/plan/dlc基础/dlc环境配置更新各个仓.md`
