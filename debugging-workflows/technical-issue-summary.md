# 技术问题简要介绍方法

## 适用场景

本文用于把已经收敛的复杂故障证据压缩成一到两句，让研发、测试、项目和管理同事都能理解的技术说明，适用于：

- OOM、崩溃、卡死、错误输出和性能回退；
- 模型、框架、算子、DLC Runtime、driver 和 Real DLC Hardware 问题；
- Sprint 同步、Issue 标题与摘要、owner 确认和跨 session handoff；
- 跨仓问题的责任边界说明。

本文负责压缩已闭合证据，不替代故障诊断。尚未建立稳定复现、首次异常边界或数值闭合时，先使用 `diagnosing-bugs`；涉及设备、HBM、进程和 cleanup 状态时，使用 `dlc-hardware-observability` 获取 SMI Observation Envelope。

## 核心结论

一条简要介绍保留五个信息槽：

| 信息槽 | 要回答的问题 | 示例 |
|---|---|---|
| 对象 | 谁受影响 | 某模型、某服务、某算子 |
| 场景 | 在做什么时发生 | mask 上采样、模型加载、跨卡通信 |
| 机制 | 哪个组件在什么边界首次异常 | launch 前物化临时输出 |
| 量化 | 异常有多大 | 204 MiB 放大到 50.92 GiB |
| 结果 | 用户看到什么 | OOM、hang、错误输出、性能下降 |

推荐句式：

> `<对象>` 在执行 `<场景>` 时，`<组件>` 在 `<关键边界>` 发生 `<异常机制>`，导致 `<量化变化>`，最终触发 `<用户可见故障>`。

如果解决方向已经验证，可增加第二句：

> 当前可通过 `<已验证规避>` 规避，长期应由 `<owner>` 优化 `<目标机制>`。

目标不是删字，而是在最短表述中保留“对象、场景、首次异常机制、量化影响和用户症状”。

## 摘要前置证据门禁

### 1. 症状与执行身份

先固定用户真正遇到的症状，而不是附近的 warning。证据至少绑定：

- 输入或请求身份；
- 已执行的原始或最小复现命令；
- source full SHA、dirty state、wheel/package、import path、native library、image 和 DLC Custom Kernel identity；
- physical/logical device mapping；
- 精确错误或错误输出。

无法复现时，使用“当前最小边界”或“当前观察到”，不宣称根因。

### 2. 红色反馈闭环

必须有一条已经执行、能稳定触发同一症状的命令。优先使用：

1. 原始输入和原始命令；
2. 去除无关模型逻辑后的最小算子调用；
3. 只保留异常所需的 shape、dtype 和参数；
4. 能通过的正常对照组。

完整的反馈环构造、最小化和修复回归由 `diagnosing-bugs` 负责。

### 3. 首次异常边界

沿执行链查找首次产生错误 shape、字节数、调用次数、参数或状态的层：

```text
业务/模型调用
  -> framework API
  -> PyTorch DLC Backend / vLLM DLC Custom Op
  -> KernelDesc 参数和 layout 适配
  -> DLCSynapse / DLC Runtime allocation and launch
  -> DLC Custom Kernel execution
  -> dlc-thunk / kernel driver completion
  -> Real DLC Hardware
```

最后报告错误的层不自动成为根因：

- DLC Runtime 报 allocation failure，不代表它计算错了请求大小；
- kernel driver 返回 OOM，不代表 kernel driver 有缺陷；
- DLC Custom Kernel 名出现在调用栈，不证明它已经 launch；
- 模型触发特定 shape，不代表模型是底层 owner。

### 4. 单变量实验

每个实验只改变一个变量，例如：

- 模型路径与最小算子路径；
- launch DLC Custom Kernel 与仅分配同 shape tensor；
- contiguous 与 Channels-Last Strided Tensor；
- `C=1` 与 `C>1`；
- bf16 与 fp32；
- 整体处理与分块处理；
- device 执行与 CPU fallback；
- 空闲设备与已有 workload 的设备。

优先设计能排除整个层级的实验。例如“不 launch DLC Custom Kernel，仅创建同 shape 临时 tensor，仍产生同样 HBM 增量”可排除 kernel 数学执行是异常分配量的来源。

HBM 实验应串行执行，并确认每轮前后的设备、进程和 HBM 基线，避免并行 workload 污染结论。

### 5. 数值闭合

摘要中的数字必须能从 shape、dtype、layout 或测量证据复算：

```text
logical tensor bytes = product(logical shape) x dtype bytes
physical tensor bytes = product(physical layout shape) x dtype bytes
amplification = physical bytes / logical bytes
aligned dim = ceil(logical dim / alignment) x alignment
```

只有计算值与 runtime log 或 SMI Observation Envelope 一致时，才把具体放大量写入结论。性能数字必须来自移除诊断插桩后的原始 workload。

### 6. 因果分类

明确区分：

```text
直接根因：首次产生错误规模或错误状态的机制
贡献因素：使该机制存在的 ABI、layout 或设计约束
触发条件：特定模型、输入、shape、阈值或并发状态
```

示例：

```text
直接根因：host 侧一次性物化大型 padded 临时输出
贡献因素：DLC Custom Kernel Entry ABI 要求最内层 channel 对齐
触发条件：模型返回大量 C=1 mask
```

### 7. 红绿差分与语义

规避或修复应在同一输入上形成差分：原路径稳定复现，且只改变目标机制后成功。同时验证：

- shape、box、score、mask 或请求结果保持正确；
- CPU Reference 在声明容差内；
- peak memory 或 latency 使用无诊断插桩结果；
- 设备和进程 cleanup 闭合。

“不再报错”不等于“结果正确”。

## 从证据到通俗语言

源码证据：

```text
KernelDesc::output 对 permuted view 调用 contiguous 转换，storage 将 dim0 对齐到 256。
```

Sprint 表述：

```text
PyTorch DLC Backend 在 DLC Custom Kernel launch 前物化布局转换临时输出，并将单通道按 256 通道对齐。
```

保留发生阶段、核心动作、异常机制、量化影响和用户症状；类名、辅助函数名、完整调用栈和源码行号留在 Issue 或 handoff 中。

## 输出模板

### Sprint 同步

> `<对象>` 在执行 `<操作>` 时，`<组件>` 在 `<边界>` 进行了 `<异常处理>`，导致 `<量化影响>`并触发 `<故障>`。

控制在一到两句。

### Owner 确认

> 当前确认 `<逻辑数据量>` 在 `<适配步骤>` 被放大到 `<物理数据量>`，并在 `<launch 前/执行中/同步时>` 失败。需要确认这是必须遵守的 contract，还是可以通过 `<候选方向>` 优化。

尚未获得 owner 确认时，把 owner 标为待确认，不把候选修复写成承诺。

### Issue 标题

```text
[组件/模型] <操作> 在 <阶段> 产生 <异常资源开销> 并导致 <故障>
```

### Issue 摘要

> `<workload>` 在 `<算子和 shape>` 下稳定触发 `<症状>`。已确认 `<异常机制>` 在 `<首次异常层>` 产生 `<量化变化>`，且 `<对照实验>` 证明该峰值不是计算必需，请评估 `<修复方向>`。

### Handoff

Handoff 不压缩成一句话，至少记录：

- 原始症状和红色命令；
- 输入、source、package、binary、image、device identity；
- 最小复现和正常对照；
- 已验证和已证伪假设；
- 最后成功阶段与首次异常边界；
- 数值闭合公式；
- 修复或规避的语义验证；
- artifact 路径、下一 owner 和 Claim Boundary。

## Owner 判断

Issue 交给首次产生错误 contract 或错误状态的组件，而不是最后打印错误的组件：

| 问题面 | 常见 owner |
|---|---|
| 模型调用、阈值或业务后处理 | 模型/应用仓库 |
| Public Operator Schema、dispatch、host wrapper、KernelDesc packing | framework backend / extension 仓库 |
| DLC Custom Kernel 实现、Entry ABI、Variant、metadata | DLC_Custom_Kernel Repository |
| allocation、stream/event、launch API 本身 | DLC Runtime owner |
| 用户态到 kernel driver 的桥接 | dlc-thunk owner |
| kernel driver completion 或设备异常 | kernel driver / hardware owner |

跨仓问题可以有一个主 Issue 和一个关联 Issue。主 Issue 跟踪已确认的首次异常边界；关联 Issue 只跟踪另一个 owner 必须修改的 ABI 或实现。调查过多个仓库不构成创建多个 Issue 的理由。

## 源码引用

优先引用 repository、full SHA、文件路径、函数或符号和关键语句。行号会随 commit 漂移；必须引用时，同时绑定完整 source identity 和行范围。不得将另一个源码快照的行号当作运行二进制的确定身份。

## 不进入一句话的内容

- 完整调用栈、环境变量和尝试历史；
- image、binary 和输入的完整 digest；
- 所有源码文件、行号和规避代码；
- 尚未证实的推测。

这些内容保留在 Issue、诊断报告、artifact 或 handoff 中，并由一句话引用其路径。

## 常见错误

- 从最后一条 log 直接归因，没有定位请求值或状态的产生者。
- 把触发模型、shape 或 workload 写成底层根因和 owner。
- 把硬件或 ABI 约束等同于实现无需优化。
- 只证明 CPU fallback、分块等规避不报错，没有证明结果正确。
- 在 Sprint 摘要中堆叠源码细节，掩盖首次异常机制。
- 使用无法从 shape、dtype、layout 或 benchmark 复算的数字。
- 并行实验共用设备，污染 free HBM 和 OOM 证据。

## 完成标准

- 非参与排查者第一次听也能理解；
- 对象、场景、首次异常机制、量化影响和用户症状齐全；
- 触发条件没有被误写成 owner；
- 错误传播层没有被误写成根因层；
- 数字可由证据复算；
- 不包含未验证断言；
- 可自然扩展为 Issue 摘要；
- 详细证据有独立路径可追溯。

## Claim Boundary

这套方法只能把已闭合的诊断证据压缩成稳定、可读的技术说明。它不能替代复现、exact source 和 runtime identity、Real DLC Hardware 观察、结果正确性验证、性能 benchmark 或 owner 确认。首次异常边界尚未收敛时，使用“当前最小边界”，不使用“根因”。

## 相关资料

- [有效 Debug 方法论](effective-debugging.md)
- [Chipltech SMI 可观测性](../runtime-debugging/chipltech-smi-observability.md)
- [Runtime 排障](../runtime-debugging/runtime-troubleshooting.md)
- [常用调试命令](common-debug-commands.md)
