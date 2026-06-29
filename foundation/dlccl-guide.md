# DLCCL 集合通信指南

## 适用场景

- 理解 DLC-Family Accelerator 多卡通信机制。
- 开发或调试 DLCCL 集合通信算子（AllReduce、Broadcast、AllGather、ReduceScatter 等）。
- 排查多卡训练或推理中的通信卡死问题。

## 核心结论

DLCCL 是 DLC-Family Accelerator 的集合通信库（类 NCCL），实现 AllReduce、Broadcast、AllGather、Reduce、ReduceScatter 和 Send/Recv 六种操作。底层基于 LYP（片间互联）进行数据传输，使用 Ring 和 Tree 两种拓扑算法。

## 基本环境要求

- Docker 容器需要 `--shm-size 32G`，因为 DLCCL 和 sim LYP 使用 shared memory。
- 在芯片或伪设备上运行**不要**设置 `DLC_SYN_BLOCKING` 环境变量。

## LYP 基础

LYP 是 DLC-Family Accelerator 的片间互联模块：

- 提供 **rsync** 和 **rdma** 两种功能：
  - rsync：写其他 XYS 的 syncflag。
  - rdma：写其他 XYS 的 memory。
- 每张卡最多与另外四张卡相连（机内连接方式固定）。

### RDMA Garbage（DLC Chip 特有问题）

在 DLC Chip 的 rdma 中，dst sync 无法工作，必须使用 src sync 等待 rdma 完成。但是 src sync 只能确认数据已发出，不能确认已接收。因此在实际数据 rdma 后额外发一条不带实际数据的 rdma（称为 garbage rdma，大小 1MB）确保前一条数据已被接收。

## 开发运行路径

### Kernel → DLC (syntests)

最简单的调试方式，通过 syntests 直接调起 kernel：

```bash
# SIM 环境
export SYNAPSE_SIM_COUNT=4          # 启用 4 张卡
export DLC_SIM_USE_SNT=1            # 启用 LYP 支持
export DLC_SIMUNET_ENABLE=1

# 运行 allreduce 测试
syntest -t allreduce_ring_f32_rdma

# 硬件
# 先配置 LYP，然后
syntest -t allreduce_ring_f32_rdma

# 伪设备
# 安装驱动时：insmod chipltech_dlc.ko dlc_sim_counts=4
```

验证方法：`D2D` 后缀的算子不需要 LYP，`rdma` 后缀的算子使用硬件 LYP。

### DLCCL → Kernel → DLC

```bash
cd /path/to/dlccl
./build.sh
export LD_LIBRARY_PATH=/usr/local/chipltech/dlccl/lib:$LD_LIBRARY_PATH
# 运行 example 或 dlccl-tests
```

### PyTorch → DLCCL → Kernel → DLC

完整调用链路：
```bash
pytest -s ./test/distributed/test_c10d_dlccl.py::ProcessGroupDLCCLTest::test_broadcast_ops
```

## Channel 机制

Channel 是 DLCCL 算子的关键组件，包含发送端（tx）和接收端（rx）：

- **buffer 开在接收方卡上**：发送方通过 rdma 将数据写入接收方 buffer。
- **容量为 1 的 fifo**：简化实现，用一个 bool 记录 fifo 状态。
- **syncflag 在两张卡各存一份**：操作时同时修改自身和对方卡上的 flag。
- **rsync 不失败为前提**：Channel 及 rsync 的使用建立在此前提上。

## Ring 拓扑算法

### Broadcast
root 卡向 next 卡发数据，每张卡接收后接力发送。

```
if (self == rootDevice):
    send(input)
elif (self->next == rootDevice):
    output = recv()
else:
    output = recv(); send(output)
```

### Reduce
每张卡将自己数据和接收到的数据 reduce 后再发给下一张卡。

### AllGather
通过接力形式将所有卡的数据传到所有卡上。

### ReduceScatter
接力数据时顺手将自身数据 reduce 传给下一张卡，最后每张卡各自拥有部分 reduce 结果。

### AllReduce = ReduceScatter + AllGather
先 ReduceScatter 在各卡产生部分 reduce 结果，再 AllGather 分发回所有卡。

## Tree 拓扑与多机

在 Tree 拓扑中，一张卡可以有 1 个父节点和 3 个子节点。AllReduce 需要数据先沿上行汇聚到根节点，再沿下行传递回其他卡。

在实际构建中：
- **机器间**：形成树结构。
- **机器内**：组成一条链。

### LYP 数据传输
跨机 rdma 和机内 rdma 没有区别。Tree 需要准备四份 Channel，收发方需指定接收方地址。

### D2H2D 数据传输
通过中断让 host 完成跨机数据传输。d2h2d 中断有两个参数：chipid 和 src 地址。接收数据固定写到 `62GiB + (n%64)*buffsize`。

## Kernel 参数（Ring）

DLCCL buffer 固定在 HBM 中：
- 临时空间：XYS0 的 62.5 GiB 和 XYS1 的 63.5 GiB，各占 0.5 GiB。
- Garbage 数据：63 GiB 处，占 1 MiB。

## 调试方法

### 伪设备/sim 调试
```bash
export DLC_SIMUNET_ENABLE_LOG=1  # 打印 rsync 和 rdma 指令信息
```
卡死特征：所有 chip 处于 wait sync 阶段（running 且 pc 不变）。

### 芯片上调试

**回推卡死指令位置**：
```bash
peek  # 查看 pc
# pc - 1092 = kernel 实际指令序号
# 打开 kernel/build/dlc_src/<kernel>.dlcasm 找到对应指令
# 向上搜索 wait 指令定位卡死原因
```

**sync 501 的含义**：
记录 DLCCL kernel 绝大部分 wait 操作：
- `EE`：wait gte（rdma 和旧版 sync all devices）
- `DD`：wait done
- `CC`：wait clear
- `33`：上一个 wait 已满足
- 后面数字：等待的 sync 号

**sync 502**：在 kernel 关键位置记录代码行号。

### 卡死分析流程

1. 检查 run_status：Halted = 没卡死，Running + pc 不变 = 卡死。
2. 排除硬件故障：pc=0、出现 error、LYP lst error。
3. 用 `peek_stuck.sh` 确定卡死的算子。
4. DLCCL 需要所有卡同时处于同一个算子，如果部分卡不在运行 DLCCL 算子，先排查其他卡。
5. 常见卡死：卡 sync_all_devices（pc<1500 且多卡 pc 一致）、部分卡 halted、rdma 卡死（sync 501 出现 EE12C）。

### LYP Link Error 处理

```bash
# 检查 error link
dlc_i2c/ICC_interchip/check_lst_error_status.sh 0 1 2 3 4 5 6 7

# Port 对应关系
# port0：t+4
# port1：t+2
# port2：连 t+1
# port3：连 t-1

# 绕过 error link（如不需要 8 卡）
dlc_i2c/ICC_4chip/lut_bypass_??.sh
```

## LYP 路由配置

### 断线检测
```bash
rdma_perf  # 测试所有两两组合，FAIL 表示断线
```

### 路由生成
使用 Python 脚本（依赖 `z3-solver`）自动生成 Hamiltonian 回路：
```bash
pip install z3-solver
python lut_solver.py "[(0,1),(1,2),...]"  # 参数是所有好的线
```

复杂需求路由使用 C++ 程序进行遗传算法优化。

### 路由应用
1. 将路由表粘贴到 `DLCsim/src/lyp_chip_invert`
2. 运行 `./build/src/simple_test/lut_test`
3. 复制 `lyp_bootstrap_table` 到 `dlc_i2c/ICC_interchip/`
4. 运行 `rst_lyp_all_all.sh` 两次
5. `chmod +x lyp_bootstrap_table && ./lyp_bootstrap_table`

## 相关资料

- [CONTEXT.md](../CONTEXT.md) — DLCCL、LYP、RDMA 术语定义
- [runtime-debugging/runtime-troubleshooting.md](../runtime-debugging/runtime-troubleshooting.md) — 卡死排查
- [foundation/dlc-ecosystem-overview.md](dlc-ecosystem-overview.md) — DLC Ecosystem 整体概述

## 来源

- `/work/plan/newraw/DLCCL-2.docx`（已转换为 Markdown）
