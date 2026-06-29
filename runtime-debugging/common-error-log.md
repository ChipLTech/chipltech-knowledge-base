# 常见报错速查

## 适用场景

快速查询 DLC Ecosystem 开发中的常见错误和解决方案。

## 核心结论

大部分编译/运行时错误来自版本不匹配、缺少依赖或路径配置问题。按错误信息搜索即可找到对应解决方案。

## 报错列表

### torchvision 相关

**`torchvision::nms does not exist`**
原因：使用了标准 torchvision，需要用 DLC 版本。
解决：使用 DLC 版本的 torchvision。

### 系统依赖

**`FileNotFoundError: ffprobe`**
解决：`apt install ffmpeg`

**`libGL.so.1: cannot open shared object file`**
解决：`apt install libgl1-mesa-glx`

**`libopenblas.so.0: cannot open shared object file`**
解决：`apt install libopenblas-dev`

### vLLM 编译和运行

**vLLM schema mismatch**
原因：缓存编译产物不一致。
解决：clean build 目录后重新编译。

**`torch==2.6.0+cpu` 未找到**
解决：`pip install -e . --no-build-isolation`

**vLLM 启动 404**
原因：端口冲突 / 模型路径不匹配 / `--served-model-name` 错误。
解决：检查端口、路径和 `--served-model-name`。

### PyTorch 编译

**`TypeError: no_python_abi_suffix`**
原因：setuptools 和 pytorch 版本不兼容。
解决：检查版本兼容性。

**`dlc_runtime_api.h: No such file or directory`**
原因：DLCSynapse include path 未正确设置。
解决：检查 `CPLUS_INCLUDE_PATH`。

**`numpy/arrayobject.h: No such file or directory`**
原因：NumPy include path 未设置。
解决：添加到 `CPLUS_INCLUDE_PATH`。

### CMake

**CMake 版本过低**
解决：升级 cmake。
```bash
# 安装新版 cmake
```

### 运行时错误

**`synFail: host memory allocation failed`**
原因：请求的 host 内存太大。
解决：减少分配大小。

**`synErrorLaunchFailure`**
原因：异步 kernel 执行错误。
工作方式：错误在后续 API 调用处报出，真实失败的 kernel 在更早时间点。
解决：
1. 用 `DLC_SYN_BLOCKING=1` 切换同同步模式。
2. 用 `DLC_SYN_DEBUG=1` 追踪失败 kernel。

**`AllocDeviceMemory failed` / `AllocDeviceMemory` 警告**
原因：DLCSynapse 内存分配警告或分配失败，需结合上下文判断是否致命。
解决：通常非致命，kernel 继续执行。如影响结果，检查可用 HBM 是否不足。

**peek_stuck 结果异常**
原因：DLCSynapse 版本不匹配。
解决：重新编译 DLCSynapse。

### NumPy

**NumPy 1.x vs 2.x 不兼容**
解决：`pip install "numpy<2"`

### vLLM Python 安装

**`python setup.py install` vs `pip install -e .`**
开发模式使用 `pip install -e .` 以支持代码修改后无需重新安装。

### pydantic-core

**pydantic-core 版本冲突**
解决：`pip install pydantic-core==<compatible_version>`

### 依赖构建顺序

**XYS IndexError in embedding**
原因：依赖构建顺序不对。
解决：按正确顺序重新构建（LLVM → dlc-thunk → DLCsim → DLCSynapse → DLC_Custom_Kernel → DLC_CL → PyTorch → vLLM）。

## 相关资料

- [runtime-debugging/environment-setup-and-update.md](environment-setup-and-update.md)
- [runtime-debugging/runtime-troubleshooting.md](runtime-troubleshooting.md)

## 来源

- `/work/plan/dlc基础/报错问题记录.md`
