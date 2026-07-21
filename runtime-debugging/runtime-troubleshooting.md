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
5. Package 可导入、设备可见、allocation 或 H2D 成功都不等于 DLC Runtime execution 健康。
6. 每个恢复动作后必须用 fresh process 重跑最小失败 case；命令 exit 0 只证明动作被接受。

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
# 只作为一个观测源；保留 PID、FD 和设备节点，不只取 PID
lsof /dev/cltech*
```

`lsof` 可能受权限、tracefs/ZFS 和 mount namespace 影响，空输出不能单独证明设备无人使用。还需交叉核对 `cltech_smi` 进程/HBM、Host 与容器 process table、监听端口、cgroup 和 pre-launch baseline。

不要从历史文档复制裸软复位命令。软复位前必须在当前 Host 上确认 `command -v`、工具版本和 `--help`，明确目标物理设备、作用范围、独占性和授权。当前主要入口参考 [chipltech-smi-observability.md](chipltech-smi-observability.md)。

### Fresh-Process Layered Runtime Probe

当 vLLM 初始化、模型放置或简单 tensor 卡住时，先按以下顺序切层：

```text
package/import
device_count
device_properties
mem_get_info
allocation
H2D
device operation
synchronize
D2H
correctness
```

Probe 要求：

- 在源码树外和 fresh process 中运行。
- 每层打印 `BEGIN`/`PASS` 并立即 flush。
- 用外层 timeout 保存明确的超时边界。
- 真正执行设备计算，例如先 H2D，再执行 `device + 1`，最后 synchronize 和 D2H correctness。
- 保存完整命令、environment、stdout/stderr、exit code、signal、PID identity、HBM 和 handles。

推荐实现和判定规则见 [安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md)。

如果 allocation 和 H2D PASS，但首个 device operation hang，应把当前最小边界写为 PyTorch DLC Backend / DLCSynapse / DLC Runtime / Host driver completion path，不要写成 allocation hang，也不要先修改模型 registry、attention 或 scheduler。

`DLC_SYN_BLOCKING=1` 可以帮助异步错误前移，但不保证解除 hang。Blocking 模式下仍停在同一 device operation 时，应保留为同一失败边界；正式运行前清理该变量。

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
5. 先用 TP=1 小模型或纯 PyTorch layered probe 判断是否低于 vLLM/模型层；单卡 execution 未通过前不进入 DLCCL/TP 归因。
6. 获得授权后才执行软复位、LYP repair、驱动重载、service restart 或 reboot。
7. 每个动作后、下一个动作前用 fresh process 重跑最小失败 case。若连续执行多个动作而中间没有 probe，只能报告组合动作之后恢复。

开发容器如果只做端口映射而没有 host 网络/host namespace，软复位通信可能超时。需要设备级操作时，优先使用明确授权的 host 环境或一次性 privileged、host network 容器执行。

### 模型放置 hang

参考案例：`model.to("dlc")` 在 10.29B bf16 model 上可达 ~24s。如果 hang 更久：
1. 检查是否有残留 Python process 占用 DLC 设备。
2. 用 fresh-process layered probe 区分 allocation、H2D、device operation、synchronize 和 D2H；单独 `torch.arange(4).to("dlc")` 只覆盖部分路径。
3. 基础 device operation 通过后，再做从 tiny 到目标规模的增量 allocation/H2D/device-op/D2H correctness。
4. 如果恢复动作后现象消失，只记录具体 action sequence 和复验结果。没有单变量证据时，不将其描述为已确认的瞬态根因或某个动作的唯一修复。

### Namespace-Safe 进程清理

向 PID 发送 signal 前至少核对：

- Owning container ID 和名称。
- Host PID、container PID、`NSpid` 映射和 PID namespace inode。
- `/proc/<pid>/stat` starttime，防止 PID 重用。
- cmdline、executable、PPID、cgroup 和监听端口。
- Device handle 与 HBM 增量是否属于本任务 server epoch。

优先 TERM 本任务 APIServer 或 probe，等待并复查；只有进程身份仍匹配且未退出时才升级 KILL。Host 无权 signal 容器 root PID 时，在 owning container 内使用对应 container PID 精确终止，不能假设 Host PID 与 container PID 必然相同。

清理后交叉确认本任务 process group、端口、device PID、handles 和 HBM 均回到 sealed pre-launch baseline。共享 Host 不应无条件要求全机 HBM 为 0，也不得清理原有占用。

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
cltech_smi --lypcheck=dlc_info_agg --verbose

# DLCCL 检查
cltech_smi --dlcclcheck
```

命令执行前仍应以当前 Host 的 `cltech_smi -h` 为准；历史 `dlc_smi` 仅作为旧日志识别线索，不作为当前命令入口。

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

## SMI 观测入口

需要设备列表、HBM、功耗、温度、固件版本、进程占用或 debug 采集时，优先参考 [chipltech-smi-observability.md](chipltech-smi-observability.md)。`cltech_smi` 的 query-only 输出可以作为设备观测材料；reset、LYP repair、debug 上传和硬复位属于有副作用或外部通信的操作，必须明确授权。

## 相关资料

- [debugging-workflows/common-debug-commands.md](../debugging-workflows/common-debug-commands.md)
- [runtime-debugging/environment-setup-and-update.md](environment-setup-and-update.md)
- [runtime-debugging/chipltech-smi-observability.md](chipltech-smi-observability.md)
- [CONTEXT.md](../CONTEXT.md) — DLCSynapse、DLC Runtime 等术语定义

## 来源

- `/work/plan/dlc基础/DLC常用调试指令和方法.md`
- `/work/plan/dlc基础/报错问题记录.md`
- `/work/RSThinker/dlc_model_placement_hang_analysis.md`
- `/work/test/同事文档/新人服务器环境配置.md`
- `/work/test/同事文档/常见BUG和解决思路：.md`
