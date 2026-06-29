# DLCSynapse 使用指南

## 适用场景

- 编写 DLCSynapse 层面的 kernel 测试（syntest）。
- 理解 Synapse test 的基本结构和工作流程。
- 在 DLCsim 或 veloce 上调试 kernel。

## 核心结论

DLCSynapse 测试文件的核心结构包括三个步骤：准备输入/输出 Tensor → 初始化 kernel 并 launch → 比较输出与 reference。

## 前置准备

```bash
# 编译 synapse 前确保 simulator 和 thunk 已 make install
cd synapse && ./compile.sh

# 确保 kernel so 在正确路径
cp /path/to/DLC_KERNEL_LIB/*.so /usr/local/chipltech/simulator/lib/
# 或
export LD_LIBRARY_PATH=/path/to/DLC_KERNEL_LIB:$LD_LIBRARY_PATH
```

## 测试文件基本结构

### 步骤 1：准备输入和输出 Tensor

- 设置 Tensor 的 shape 和数据类型。
- 初始化输入数据。
- 设置输出 tensor 大小，检查维度是否合理。
- 设置对齐（alignment）并对齐 tensor。

### 步骤 2：设置输入/输出和 extra args

关键设置：
- 每个 tensor 使用的 memory（HBM/VMEM 等）。
- host 指针。
- tensor 维度和大小。
- 数据类型（默认 float，Int 或其他需要显式设置）。

DMA 设置：在 veloce 上连续不同 memory 的 chip-to-host DMA 不能保证顺序，建议只做 HBM 的 DMA，kernel 只设一个 memory 的输出。

### 步骤 3：比较结果

Launch 完成后将输出搬到 output，与 reference 比较：
- `output.unpad()`：如果数据不需要对齐可以去掉。
- `output.print()`：可选的输出打印。

## 切换设备

在 `dlc_launch.h` 中修改：
- `synDeviceSimulator`：使用 DLCsim。
- `synDeviceVeloce`：使用 veloce（硬件模拟加速器）。

## Debug 方法

### DLCsim Debug

在 DLCsim 源码中启用 XYS 调试信息，重新编译后可以：
- 每个 cycle 打印寄存器值。
- 查看每个 cycle 执行的指令。
- 查看 memory 状态。

### memory 查看

在 `synapse_core/synapse_core.cpp` 的 `synLaunchDefault` 中使用：
- `DebugPrintSmem`：查看 SMEM。
- `DebugPrintVmem`：查看 VMEM。
- `DebugPrintCmem`：查看 CMEM。
- `PrintHBM`：查看 HBM。

### CRT 断点

如果怀疑参数初始化有问题：
1. 在 `dlcsim/lib-rt/crt1` 的汇编（branch 到 main 之前）加 halt。
2. CMakeLists 中取消 crt 注释。
3. 重新生成 crt hex。

### Veloce Debug

```bash
peek_status   # 查看 pc、运行状态、SMEM/VMEM 报错
peek -h       # 查看所有查看选项（寄存器、imem、smem、vmem、指令 decode）
```

正常完成：pc 回到 crt end（通常 100 以内），status=halt。

## 相关资料

- [CONTEXT.md](../CONTEXT.md) — DLCSynapse、DLC Runtime、DLCsim 定义
- [runtime-debugging/runtime-troubleshooting.md](../runtime-debugging/runtime-troubleshooting.md)
- [debugging-workflows/common-debug-commands.md](../debugging-workflows/common-debug-commands.md)

## 来源

- `/work/plan/newraw/Synapse 使用文档.docx`（已转换为 Markdown）
