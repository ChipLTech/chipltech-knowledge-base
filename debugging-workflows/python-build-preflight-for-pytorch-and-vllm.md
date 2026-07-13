# PyTorch 与 vLLM 构建前预检

## 适用场景

- 准备重建 PyTorch 2.5.0 wheel，但不想等到长时间编译后才发现 Python packaging 或基础依赖不对。
- 准备执行本地 `vllm` / `vllm-dlc` editable install，需要先确认构建工具链。
- 已经遇到 metadata generation、`build_editable`、`pybind11`、NumPy bridge 等错误，想快速判断是前置依赖没对齐还是源码本身有问题。

## 核心结论

1. PyTorch 与 `vllm` 的高频失败点，很多并不在编译逻辑本身，而在 build 前的 Python packaging 工具链。
2. PyTorch wheel 的关键预检是 `numpy==1.26.4` 与 `USE_NUMPY` 路线；`vllm` / `vllm-dlc` 的关键预检是 `setuptools`、`setuptools-scm`、`packaging`、`pybind11`、`grpcio-tools` 等依赖是否齐全。
3. 如果需要自动执行同一套 preflight，可调用 `dlc-env-setup` skill 内的 `scripts/pytorch-preflight.sh` 和 `scripts/vllm-preflight.sh`；本文只保留平台无关的规则与检查项。

## 操作步骤

### 1. 先确认 Python 基线

至少记录：

```bash
python3 --version
python3 -m pip --version
python3 -m pip list | rg '^(pip|setuptools|setuptools-scm|wheel|ninja|jinja2|packaging|numpy|pybind11|grpcio-tools)\b'
```

如果当前目标包含 `vllm` 或 `vllm-dlc`，建议确认 Python 版本在 `>=3.10,<3.12`。

### 2. PyTorch wheel build preflight

建议先安装或升级：

```bash
python3 -m pip install typing_extensions
python3 -m pip install --upgrade packaging 'setuptools>=77.0.3,<81.0.0' 'setuptools-scm>=8.0' wheel ninja jinja2
python3 -m pip install --force-reinstall 'numpy==1.26.4'
```

然后检查：

- `numpy` 版本确实为 `1.26.4`
- `setuptools` 足够新，避免 editable/build backend 能力不完整
- `ninja` 与 `wheel` 已安装

如果 PyTorch 目录已经跑过 configure，可额外查看：

```bash
rg '^USE_NUMPY:BOOL=' "$PYTORCH_SRC/build/CMakeCache.txt"
```

`USE_NUMPY:BOOL=ON` 才能支撑后续 `torch.tensor(...).numpy()` 的运行时桥接。

### 3. `vllm` / `vllm-dlc` build preflight

在 PyTorch wheel 已经通过基础 smoke 的前提下，再补：

```bash
python3 -m pip install --upgrade pip packaging 'setuptools>=77.0.3,<81.0.0' 'setuptools-scm>=8.0' wheel ninja jinja2
python3 -m pip install pybind11 grpcio-tools==1.78.0
```

然后确认：

- `pybind11` 已安装
- `grpcio-tools` 已安装
- `packaging`、`setuptools`、`setuptools-scm` 不缺失

如果目标是最小修复 editable metadata，而不是重装整套依赖，可以只修 metadata 所需工具链，再用 `--no-build-isolation` 或 `--no-deps` 做最小 reinstall。

### 4. wheel 与本地路径前置检查

在安装 `vllm-dlc` 前，确认它引用的 PyTorch wheel 路径确实存在于当前机器。不要把别人的本地 wheel 路径直接沿用到当前环境。

### 5. 进入长编译前的 go/no-go 判断

出现下面任一情况时，先停，不要开始长时间 build：

- `numpy` 版本不对
- `setuptools` / `setuptools-scm` 太旧
- `pybind11` 或 `grpcio-tools` 缺失
- PyTorch 旧 configure cache 已经显示 `USE_NUMPY:BOOL=OFF`
- 目标 wheel 路径根本不存在

## 常见坑

1. **把 `pip install -e .` 失败当作源码 bug**：很多时候只是 packaging 工具链没补齐。
2. **只升级 `pip` 不升级 `setuptools`**：editable install 的 backend 能力主要受 `setuptools` 影响。
3. **忘记固定 `numpy==1.26.4`**：这会导致 PyTorch wheel 看似构建成功，但运行时 NumPy bridge 失败。
4. **忽略旧的 CMakeCache**：如果 cache 里已经是 `USE_NUMPY:BOOL=OFF`，应先清理后再重建。
5. **把机器特定 wheel 路径写进长期文档**：路径必须根据当前机器实际 repo map 确认。

## 相关资料

- [../runtime-debugging/dlc-workstation-env-rebuild.md](../runtime-debugging/dlc-workstation-env-rebuild.md)
- [post-install-runtime-smoke.md](post-install-runtime-smoke.md)
- [../runtime-debugging/environment-setup-and-update.md](../runtime-debugging/environment-setup-and-update.md)

## 来源

- `/work/test/dlc-env-setup/scripts/pytorch-preflight.sh`
- `/work/test/dlc-env-setup/scripts/vllm-preflight.sh`
- `/work/test/dlc-env-setup/SKILL.md`
