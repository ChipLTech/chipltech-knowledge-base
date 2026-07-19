# Runtime 排障指南

## 适用场景

- DLC Platform 运行时出现卡死、设备占用、异步错误、设备初始化失败。
- 需要区分 DLCSynapse、DLC Runtime、DLCsim 和 Real DLC Hardware 层面的问题。

## 核心结论

Runtime 排障的核心原则：
1. 先确认是哪个组件的问题（DLCSynapse / DLC Runtime / DLCsim / Real DLC Hardware / DLCCL）。
2. 区分"卡死"(hang)和"报错"(error)。
3. 异步错误 `synErrorLaunchFailure` 的真实故障位置通常在报错之前的 kernel，不在报错处。
4. 调试后必须清理高开销环境变量。

## 组件区分

| 组件 | 常见问题类型 |
|------|-------------|
| DLCSynapse | kernel 编译失败、launch 失败、alloc 失败、AllocDeviceMemory 警告 |
| DLC Runtime | stream 同步失败、event 超时、异步错误报告 |
| DLCsim | 模拟器挂起、模拟行为与真实硬件的偏差 |
| Real DLC Hardware | 设备不可用、硬件复位、温度/功耗异常 |
| dlc-thunk | 用户态驱动桥接失败、设备节点权限 |
| DLCCL | 多卡通信失败、LYP 异常、RDMA 问题 |

## 常见问题处理

### DLCSynapse 层

**kernel launch 失败**：
```bash
# 用 blocking 模式关闭异步，逐 kernel 追踪
export DLC_SYN_BLOCKING=1
export DLC_SYN_DEBUG=1
```

**AllocDeviceMemory 警告**（`synapse_state.cpp`）：
- 通常是非致命警告，kennel 继续执行。
- 如果导致实际失败，检查可用 HBM 是否不足。

### 设备卡死

**peek stuck**（可能影响正在执行的算子，执行前确认授权和目标卡）：
```bash
peek_stuck.sh
```

**peek status**（不打断）：
```bash
peek_multi.sh -s status
```

**区分 hang 和慢**：
- 用 `DLC_SYN_BLOCKING=1` 看是否单个 kernel hang。
- 用 verbose trace 看最后 launch 的 kernel。

### 异步错误

`synErrorLaunchFailure` 的原则：
- 报错处通常不是失败 kernel 的位置，真实失败在更早位置。
- 用 `DLC_SYN_BLOCKING=1` 启用同步模式定位第一个失败 kernel。

### 设备占用/残留进程

```bash
# 查看占用设备进程
lsof -t /dev/dlc*

# 软复位
dlcpd_clnt -s
```

在共享 host 或用户开发容器中，软复位、驱动重载、LYP repair、kill 非本任务进程和 reboot 都属于设备/宿主机操作。默认只记录证据并报告建议，获得明确授权后再执行。

### vLLM DP/TP 初始化卡住

常见现象包括：

- 服务停在 DP/TP 初始化阶段。
- `No available shared memory broadcast block found in 60 seconds` 长时间反复出现。
- serving 进程仍在，但请求无响应或初始化不前进。

排障顺序：

1. 先记录完整启动命令、`DLC_VISIBLE_DEVICES`、TP/PP/EP、容器启动参数和日志路径。
2. 检查是否有本任务之外的残留进程占用 `/dev/dlc*`；不要直接 kill 非本任务进程。
3. 判断是慢还是 hang：结合 DLCSynapse verbose trace、DLC Runtime blocking 模式或服务日志最后一条有效进展。
4. 获得授权后再对目标卡运行 stuck 检查；如果显示 `halting`，保存输出并反馈给对应算子/Runtime owner。
5. 获得授权后才执行软复位、LYP repair、驱动重载或重启。

开发容器如果只做端口映射而没有 host 网络/host namespace，软复位通信可能超时。需要设备级操作时，优先使用明确授权的 host 环境或一次性 privileged、host network 容器执行。

### 模型放置 hang

参考案例：`model.to("dlc")` 在 10.29B bf16 model 上可达 ~24s。如果 hang 更久：
1. 检查是否有残留 Python process 占用 DLC 设备。
2. 用小 tensor 测试 DLC 基本操作：`torch.arange(4).to("dlc")`。
3. 增量分配测试：从 tiny -> 20 GiB 逐步确认。
4. DLC Runtime/驱动在第一次大分配已有瞬态问题，环境重置后可能消失。

## 环境变量速查

| 变量 | 用途 | 注意 |
|------|------|------|
| `DLC_SYN_BLOCKING=1` | 同步执行模式 | 调试后必须取消 |
| `DLC_SYN_DEBUG=1` | 打印每个 kernel launch | 输出量大，性能慢 |
| `DLC_SYN_VERBOSE=1` | 详细日志 | 加剧性能影响 |
| `DLC_SYN_USE_SIM=1` | 使用模拟器 | 不影响真实硬件 |

调试后清理：
```bash
unset DLC_SYN_BLOCKING DLC_SYN_DEBUG DLC_SYN_VERBOSE DLC_SYN_USE_SIM
```

## 多卡/RDMA 问题

### 检查连通性

```bash
# LYP 检查
dlc_smi --lypcheck

# DLCCL 检查
dlc_smi --dlcclcheck
```

### 常见多卡问题

- 设备可见性不正确（检查可见 DLC 设备列表）。
- DLCCL 初始化失败（检查 LYP 和网络拓扑）。
- AllReduce 结果不正确（检查数据分布和通信模式）。
- 掉卡检测（全卡检查设备状态）。

### LYP repair 边界

LYP 初始化或 repair 可以恢复多卡通信，但它不是模型 acceptance 证据。记录时只说明：

- 使用的工具和版本。
- 目标卡组和 `DLC_VISIBLE_DEVICES`。
- 日志路径。
- 前/后卡组、内环/外环或 RDMA 检查结果。
- 是否涉及驱动安装、HBM 频率配置或 loop repair。

不要把 LYP 检查通过外推为新模型 Real DLC Hardware acceptance、DLC Runtime dispatch 或 Chunked Prefill runtime 已验证。

## 环境配置检查

版本对齐问题常见表现：
- `undefined symbol` — 编译时和链接时使用的库版本不一致。
- PyTorch import 失败 — DLC 相关 so 文件缺失或版本不匹配。
- vLLM 编译失败 — 依赖未对齐。
- peek 参数异常 — synapse 版本与其他仓库不匹配。

详见 [environment-setup-and-update.md](environment-setup-and-update.md)。

## 相关资料

- [debugging-workflows/common-debug-commands.md](../debugging-workflows/common-debug-commands.md)
- [runtime-debugging/environment-setup-and-update.md](environment-setup-and-update.md)
- [CONTEXT.md](../CONTEXT.md) — DLCSynapse、DLC Runtime 等术语定义

## 来源

- `/work/plan/dlc基础/DLC常用调试指令和方法.md`
- `/work/plan/dlc基础/报错问题记录.md`
- `/work/RSThinker/dlc_model_placement_hang_analysis.md`
- `/work/test/同事文档/新人服务器环境配置.md`
- `/work/test/同事文档/常见BUG和解决思路：.md`
