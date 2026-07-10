# 从 Synapse log 进入 pytorch_test replay

## 适用场景

- 手里已经有 `syn_*.ansi`，想直接在 `DLC_Custom_Kernel/pytorch_test` 里 replay。
- 想确认某个 runtime kernel 的 shape / dtype / 参数是否能被 pytorch_test 复现。
- 想从真实运行日志快速进入单 kernel 验证，而不是立刻写新测试。

## 核心结论

`pytorch_test --replay` 的最小使用方式是：

```bash
cd /work/DLC_Custom_Kernel/pytorch_test
python3 main.py --replay /path/to/syn_XXXX.ansi
```

如果只关心单个 kernel：

```bash
python3 main.py --replay /path/to/syn_XXXX.ansi -f custom_layer_norm_bf16
```

如果怀疑 log 参数本身有问题，再加：

```bash
python3 main.py --replay /path/to/syn_XXXX.ansi --verify-log
```

## 标准流程

### 步骤 1：先准备好 `syn_*.ansi`

如果手里还没有 log，先走：

- 开 Synapse debug 环境变量
- 跑模型或最小复现
- 生成 `syn_*.ansi`
- 用 `tool.py` 生成 `*_kernels.txt`

详见：[debugging-workflows/synapse-log-and-kernel-summary-workflow.md](../debugging-workflows/synapse-log-and-kernel-summary-workflow.md)

### 步骤 2：跑全量 replay

```bash
cd /work/DLC_Custom_Kernel/pytorch_test
DLC_SYN_BLOCKING=1 python3 main.py --replay /work/tmpx/syn_1123318.ansi
```

用途：

- 看这个 log 中哪些 kernel 能被 pytorch_test 匹配到
- 初步判断 replay 路径是否打通

### 步骤 3：按目标 kernel 过滤

```bash
cd /work/DLC_Custom_Kernel/pytorch_test
DLC_SYN_BLOCKING=1 python3 main.py --replay /work/tmpx/syn_1123318.ansi -f custom_layer_norm_bf16
```

用途：

- 快速只看单个 kernel
- 避免全量 replay 太慢

注意：`-f` 是按 runtime kernel 名或 variant 过滤，不等于 dispatch key。

### 步骤 4：必要时做 log 校验

```bash
cd /work/DLC_Custom_Kernel/pytorch_test
DLC_SYN_BLOCKING=1 python3 main.py --replay /work/tmpx/syn_1123318.ansi --verify-log
```

适用场景：

- 怀疑 log 参数不完整
- 怀疑输入参数顺序和测试定义不一致
- replay 结果明显异常时

## replay 成功/失败分别说明什么

### replay 成功

说明：

- 当前 log 中这个 runtime kernel 的输入参数，能够被 pytorch_test 框架解析和重建
- 这个 kernel 已经有现成测试定义或可复用路径

不说明：

- 真实模型精度问题已经定位完成
- dispatch key 一定已经确认
- 这个 kernel 就是最终根因

### replay 失败

优先按下面四类排查：

1. **log 路径不对**：文件不存在或拿错 run
2. **kernel 名过滤错了**：`-f` 给的是错误名字
3. **pytorch_test 没有这个 kernel 的用例**：框架里根本没有匹配测试
4. **replay 输入和测试定义不匹配**：log 参数顺序、shape、dtype、scalar 解释方式不一致

## 什么时候应该从 replay 转到手写 real-input repro

满足以下任一情况时，优先改成手写 real-input repro：

1. replay 没有对应测试可匹配
2. replay 路径可运行，但精度结论不足以回答模型问题
3. 你需要复现的是 same-input CPU vs DLC 差异，而不是单纯 log 回放
4. 你已经知道目标算子和真实输入，适合直接写更小的 `F.linear` / `F.layer_norm` / `bmm` replay

## 常见坑

1. **把 replay PASS 当成“模型没问题”**：replay 的容差由对应测试决定。
2. **把 `-f` 名字当 dispatch key**：它通常只是 runtime kernel 名。
3. **跳过 `*_kernels.txt`**：容易不知道该过滤哪个 kernel。
4. **没有先确认当前 log 来自哪个模型场景**：后面很难回溯。

## 相关资料

- [testing/dlc-kernel-test-framework-guide.md](dlc-kernel-test-framework-guide.md)
- [debugging-workflows/synapse-log-and-kernel-summary-workflow.md](../debugging-workflows/synapse-log-and-kernel-summary-workflow.md)
- [operator-dispatch/dispatch-key-and-runtime-kernel-mapping.md](../operator-dispatch/dispatch-key-and-runtime-kernel-mapping.md)

## 来源

- `/work/plans/PyTorch_DLC算子插入_dispatch与runtime_kernel映射说明.md`
- `/work/plans/运行过程.md`
