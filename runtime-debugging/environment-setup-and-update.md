# 环境配置与仓库更新

## 适用场景

- 初次搭建 DLC 开发环境。
- 更新 DLC 相关仓库版本。
- 排查 `undefined symbol`、import 失败、编译失败等版本对齐问题。

## 核心结论

DLC Ecosystem 仓库存在严格的版本依赖链，更新顺序和编译方式必须一致，否则会出现 import 失败、undefined symbol、peek 异常等版本不匹配错误。

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

```bash
# release_25 分支
cd /work/pytorch
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

## 来源

- `/work/plan/dlc基础/dlc环境配置更新各个仓.md`
