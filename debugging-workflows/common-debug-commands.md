# 常用调试命令

## 适用场景

DLC Platform 日常调试：查看设备状态、进程、卡死排查、trace 分析、环境变量配置。

## 核心结论

DLC Platform 调试的核心入口分为：设备管理、DLCSynapse 调试、卡死检查、多卡/RDMA、PyTorch traceback、异步错误检查。调试后必须清理高开销环境变量。

## 设备和进程

```bash
# 发现当前 Host 的设备节点；不同环境可能保留兼容命名
ls /dev/cltech*
ls /dev/dlc*

# 查看设备 handle；空输出不能单独证明无人使用
lsof /dev/cltech*
```

## 端口占用

```bash
lsof -i :端口号
```

## 软复位安全门

软复位不是日常只读命令，不在速查表提供版本不明、目标范围不明的裸命令。执行前必须：

1. 在当前 Host 检查 `command -v cltech_smi`、版本和 `cltech_smi -h`。
2. 明确目标物理设备、共享/独占状态和操作授权。
3. 保存进程、HBM、handles、端口和设备基线。
4. 区分 core soft reset 与 full soft reset，不扩大到非目标设备。
5. 执行后使用 fresh-process layered DLC Runtime probe 验收；exit 0 不等于 DLC Runtime 已恢复。

具体边界见 [cltech_smi 设备观测与诊断证据](../runtime-debugging/chipltech-smi-observability.md)。历史 `dlcpd_clnt`、`cltechpd_clnt` 或 `dlc_smi` 名称只能作为旧日志线索，不能在未核对当前工具帮助时执行。

## DLCSynapse 调试

```bash
# 启用调试模式（打印每个 kernel launch）
export DLC_SYN_DEBUG=1
export DLC_SYN_VERBOSE=1

# 阻塞执行
export DLC_SYN_BLOCKING=1

# 使用模拟器
export DLC_SYN_USE_SIM=1
```

调试后清理：
```bash
unset DLC_SYN_DEBUG DLC_SYN_VERBOSE DLC_SYN_BLOCKING
```

## PyTorch C++ traceback

在 DLC 后端代码中添加：
```cpp
at::print_traceback();
```

或在运行前设置环境变量（如 PyTorch 支持的相关选项）。

## 卡死检查

### 检查 LYP 和 DLCCL

```bash
cltech_smi --lypcheck=dlc_info_agg --verbose
cltech_smi --dlcclcheck
```

执行前以当前 `cltech_smi -h` 为准。历史 `dlc_smi` 只用于识别旧日志。

### peek 命令

```bash
# 查看卡住算子（会打断算子执行）
peek_stuck.sh

# 查看多卡状态
peek_multi.sh

# 查看算子状态
peek_multi.sh -s status
```

`peek status` 和 `peek stuck` 的区别：后者会打断算子执行。

## 驱动版本

```bash
# 获取设备、固件、驱动和产品信息
cltech_smi -q
cltech_smi --versioncheck
```

## RDMA 和多卡

```bash
# DLCCL/LYP 检查
cltech_smi --dlcclcheck
cltech_smi --lypcheck=dlc_info_agg --verbose

# 根据 loopcheck 结果输出 LYP link topology
cltech_smi --loopcheck --verbose
cltech_smi --lypportcheck

# DLCCL hang 关系分析，需要目标设备排障授权
python3 /work/arsenal/dlccl_check.py 0 1 2 3
```

## 硬件速度检查

```bash
# HBM 频率查看/修改
# 使用硬件管理工具
```

## 驱动加载

```bash
# 重启后重新加载驱动
# 加载 driver + LYP

# 仅加载驱动（不重建 LYP）
# 仅 driver 重载
```

## 版本信息

```bash
# Arsenal 版本检查工具
python3 /work/arsenal/check_version.py
python3 /work/arsenal/check_version.py offline
python3 /work/arsenal/check_version.py --summary
```

版本检查只能作为本地安装版本线索；无法读取时保持 `N/A` 或 `Unknown`，不要猜测。

## 多卡单算子测试

```python
import torch
import torch.distributed as dist

# 单算子多卡测试代码
```

## DLC 初始化检查清单

运行前确认：
1. 设备节点和物理到逻辑设备映射可审计。
2. `lsof`、`cltech_smi`、Host/容器进程、端口和 HBM 相对 pre-launch baseline 没有未归属占用。
3. DLCSynapse、DLC Runtime、PyTorch DLC Backend 和 Host driver identity 已记录。
4. Package/import smoke 通过。
5. Fresh-process layered execution smoke 通过 device operation、synchronize、D2H 和 correctness。
6. 环境变量已清理，无残留高开销 debug 变量。
7. 如使用多卡，每张目标设备独立 probe、同时 probe 和 DLCCL correctness 已通过。

`lsof` 空输出只代表当前权限和 namespace 下未观察到 handle，不能单独创建“无残留进程”结论。

## 异步错误

`synErrorLaunchFailure` 可能在后续 API 报出，真实失败 kernel 在更早位置。定位方法：
- 用 `DLC_SYN_BLOCKING=1` 关闭异步执行。
- 用 `DLC_SYN_DEBUG=1` 逐 kernel 追踪。

## 常见错误速查

常见错误信息和解决方案统一维护在 [common-error-log.md](../runtime-debugging/common-error-log.md)。本文档只保留命令入口，避免错误表重复维护后过期。

## 常见的 vLLM 问题

- 404 错误：端口冲突、模型路径不匹配、`--served-model-name` 不正确。
- `python setup.py install` vs `pip install -e`：后者用于开发模式。

## 相关资料

- [runtime-debugging/runtime-troubleshooting.md](../runtime-debugging/runtime-troubleshooting.md)
- [runtime-debugging/common-error-log.md](../runtime-debugging/common-error-log.md)
- [runtime-debugging/environment-setup-and-update.md](../runtime-debugging/environment-setup-and-update.md)
- [runtime-debugging/chipltech-smi-observability.md](../runtime-debugging/chipltech-smi-observability.md)
- [testing/arsenal-ci-and-blackbox-testing.md](../testing/arsenal-ci-and-blackbox-testing.md)
- [debugging-workflows/vscode-debug-setup.md](vscode-debug-setup.md)

## 来源

- `/work/plan/dlc基础/DLC常用调试指令和方法.md`
- `/work/plan/dlc基础/报错问题记录.md`
- `/work/chipltech_smi_lib/README.md`
- `/work/arsenal/README.md`
