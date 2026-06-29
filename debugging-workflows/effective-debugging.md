# 有效 Debug 方法论

## 适用场景

- DLC Custom Kernel 或 PyTorch DLC Backend 出现卡死、结果错误或精度问题。
- 需要系统化地定位问题根源，而不是漫无目的地看代码。

## 核心结论

Debug 的核心原则：**先判断问题范围，再逐层缩小，最后用单变量方法确认根因**。用 synthsts/sim 做早期缩小，用 pytorch_test/Model-Site Dump 做真实场景验证，用 CPU fallback 做单变量排除。

## 排查流程

### 第一步：判断是否是算子问题

1. 开启 `DLC_SYN_VERBOSE`，如果是卡死，根据 log 判断卡在 kernel 还是其他地方。
2. 查看 XYS 的状态：如果 XYS 状态正常但程序仍卡住，说明不是卡在算子。

### 第二步：判断是哪个算子的问题

由于 PyTorch 调用并非像 syntests 一样显式，可能存在隐式调用和算子组合：

1. 了解 PyTorch 调用路线，确认是否有其他算子一起被调用。
2. 开启 dma-check 编译选项，检查是否存在算子越界。
3. **用 CPU fallback 排除干扰**：把可疑算子外的其他算子转成 CPU 实现。
4. 注意修改位置：`/usr/local/chipltech/synapse/include/enabled_kernels.hpp`。

**关键**：`enabled_kernels.hpp` 通过 `ninja install` 更新，每次 `ninja install` 后会被覆盖。

**查找正确的 dispatch 常量**：
- 阅读 PyTorch DLC Backend 代码中对应算子的 `DLC_CHECK_RESULT` 宏。
- 第一个参数就是 dispatch 常量名。
- 例如 conv2d 的控制匿名函数是 `custom_conv2d`（不是 `custom_conv2d_cpu`）。

5. **快速初步检查其他算子**：保存运行 log，用 PyTorch test replay 功能测试：
   ```bash
   python main.py --replay /path/to/runtime.log
   ```

### 第三步：判断是算子的哪个方面的问题

1. **第一时间检查参数**：拿到 fail 测例后，不要先看 kernel 代码，先检查参数传入是否正确。
2. **尝试复现并缩小**：大 shape 缩小为小 shape。
3. **PyTorch fail 但 syntests pass**：检查 PyTorch 插入代码，对比两边的 launch log 信息，确认参数是否一致。

### 第四步：定位错误代码位置

- **Print 中间结果**：通过 Print 打印关键位置的值。
- **硬件 error 报错**：通过 error 信息判断错误代码位置，通过 pc 停留位置判断卡住位置。
- **sim 上 debug**：设置 `DLC_SYN_USE_SIM=1` 在 sim 上运行，可以打印更多信息。

## 常见卡死问题的可能原因

- **片内网络问题**：大量不带 sync 的 DMA 连续发出 / 存在双向不带 sync 的 DMA（如 h2v 和 v2h）。
- **算子越界**：检查 DMA 地址范围。
- **资源泄漏**：检查算子是否存在 trf / mrf / crf 等 register file 没 pop 完的情况。

## 相关资料

- [operator-dispatch/enabled-kernels-dispatch.md](../operator-dispatch/enabled-kernels-dispatch.md) — CPU fallback 操作指南
- [debugging-workflows/common-debug-commands.md](../debugging-workflows/common-debug-commands.md) — 常用调试命令
- [debugging-workflows/synapse-usage-guide.md](../debugging-workflows/synapse-usage-guide.md) — DLCSynapse 调试

## 来源

- `/work/plan/newraw/如何正确的debug.docx`（已转换为 Markdown）
