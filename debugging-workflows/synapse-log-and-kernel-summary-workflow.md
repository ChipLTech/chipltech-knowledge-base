# Synapse log 与 kernel 摘要工作流

## 适用场景

- 需要确认一次模型运行实际 launch 了哪些 DLC runtime kernel。
- 需要为 dispatch 验证、replay、挂起或性能热点排查从 `syn_*.ansi` 导出算子 CSV。
- 需要把“开环境变量 -> 跑模型 -> 产生日志 -> 摘要 kernel 列表”做成稳定流程。

## 核心结论

用于 DLC Platform 运行路径分析的最小闭环是：

1. 开 Synapse debug 环境变量
2. 跑模型或最小复现
3. 收集 `syn_*.ansi`
4. 用 `tool.py` parser 和 1400 MHz 口径生成按总 cycles 排序的 `operators.csv`
5. 再基于原始 log 和 CSV 做 dispatch、replay、挂起或热点排查

## 环境变量模板

```bash
export DLC_SYN_BLOCKING=1
export DLC_SYN_DEBUG=1
export DLC_SYN_LOG_DIR=/work/tmpx
export DLC_SYN_PROF_DIR=/work/tmpx
export DLC_SYN_PROF_CYCLE=1
export DLC_SYN_VERBOSE=4
export DLC_VISIBLE_DEVICES=3
```

### 这些变量的作用

| 变量 | 作用 |
|---|---|
| `DLC_SYN_BLOCKING=1` | 让错误更靠近真实失败点暴露 |
| `DLC_SYN_DEBUG=1` | 打印更详细的 Synapse 调试信息 |
| `DLC_SYN_LOG_DIR` | 指定 `syn_*.ansi` 输出目录 |
| `DLC_SYN_PROF_DIR` | 指定 profile 相关文件目录 |
| `DLC_SYN_PROF_CYCLE=1` | 开启 profiling 周期信息 |
| `DLC_SYN_VERBOSE=4` | 打印更详细的 kernel launch 信息 |
| `DLC_VISIBLE_DEVICES` | 选择目标 Real DLC Hardware |

## 标准流程

### 步骤 0：封存 run identity 和最终进程环境

每次运行使用独立绝对日志目录，并保存：

- image digest、package/source identity 和完整启动命令。
- 预期环境变量。
- APIServer、EngineCore、worker 的 PID、PGID、rank 和设备映射。
- 每个最终 worker 的 `/proc/<pid>/environ`。
- 启动前的进程、端口、HBM 和 device-handle baseline。

launcher、容器入口和 multiprocessing 可能过滤或重建环境。只有最终执行 worker 的环境可以证明 `DLC_SYN_BLOCKING`、`DLC_SYN_DEBUG`、日志目录和业务 Runtime 开关实际生效。

### 步骤 1：运行模型或最小复现

以 RSThinker 为例：

```bash
cd /work/RSThinker
DLC_SYN_BLOCKING=1 python3 RSThinker_infer_chat_stream.py \
  --model_path /mnt/jfs/models/RSThinker \
  --image_path /mnt/jfs/dataset/GeoCoT380k/AID/image/SparseResidential/sparseresidential_2.jpg \
  --device dlc \
  --cpu_position_ids \
  --max_tokens 10 \
  --temperature 0 \
  --prompt "Analyze this image."
```

如果不是模型场景，也可以直接运行更小的 `repro.py`。

### 步骤 2：找到 `syn_*.ansi`

运行完成后，在 `DLC_SYN_LOG_DIR` 下找到类似：

```text
/work/tmpx/syn_1123318.ansi
```

这个文件保存的是本次运行的 runtime kernel launch 细节。

### 步骤 3：生成 `operators.csv`

推荐使用已发布的 `diagnosing-bugs` Skill 包装器：

```bash
python3 <SKILL_ROOT>/scripts/export-dlc-kernel-csv.py \
  /work/tmpx/syn_1123318.ansi \
  --tool /home/xuansun/llama2-fine-tune/tool.py \
  --output-dir /work/tmpx/kernel-summary
```

固定输出：

```text
/work/tmpx/kernel-summary/operators.csv
```

该流程只落盘 `operators.csv`，不输出 kernel 文本、图片或 manifest。命令 stdout 返回原始 log、`tool.py` 和 CSV 的 SHA-256、launch/kernel 数量及 `clock_mhz: 1400`；需要身份审计时把 stdout 保存到 task-owned evidence。

只需要原始文本摘要时，也可以直接运行：

```bash
cd /work/llama2-fine-tune
python3 /home/xuansun/llama2-fine-tune/tool.py /work/tmpx/syn_1123318.ansi
```

输出类似：

```text
/work/tmpx/syn_1123318.ansi -> /work/tmpx/syn_1123318_kernels.txt
```

### 步骤 4：阅读 CSV

`operators.csv` 按总 cycles 降序，保存 kernel name、调用次数、总 cycles、cycles 占比、总/平均时间、ops、平均/最大 GFLOPS、bytes、平均/最大 bandwidth、crt cycles 和平均 crt 时间。所有时间统一按 `tool.py` 的 `F=1400` MHz 换算。kernel name 可用于快速浏览实际 launch、判断 DLC 路径和提供 replay 候选。

`table.py` 可以作为 CSV 字段和排序方式的参考，但它当前把 `Total Time` 按 1500 MHz 换算，与 `tool.py` 的 1400 MHz 不一致。本流程以 `tool.py` 为准，不直接使用 `table.py` 的时间换算，也不生成 PNG。

## 什么时候需要这个流程

1. **dispatch 验证**：确认修改 `enabled_kernels.hpp` 后是否真的 fallback
2. **replay 准备**：先知道本次 run 里出现了哪些 runtime kernel
3. **挂起排查**：找到最后一个 launch 成功但未正常返回的 kernel
4. **多路径区分**：例如分清 `custom_matmul_t_pingpong` 和 `custom_matmul_st_*_rhsT_bw`

## 常见坑

1. **只开 `DLC_SYN_BLOCKING=1`，不留 log 目录**：跑完没有 `syn_*.ansi` 可分析。
2. **把 `*_kernels.txt` 当 truth source**：它是摘要，最终还是要回看 `syn_*.ansi`。
3. **调试结束不清理环境变量**：高 verbose 对后续运行开销很大。
4. **没记录本次 log 路径**：导致后面 replay 或 dispatch 验证无法复盘。
5. **只保存 launcher 环境**：最终 worker 可能没有收到 blocking/debug/log-dir 变量。
6. **blocking 后出现更早 stall 仍继续归因**：如果新 run 没有到达原失败阶段，它是新的观察边界，不是已经定位的首个失败 kernel。
7. **多 rank 共用不可区分的日志**：必须保存 PID/rank/device 到 `syn_*.ansi` 的映射，否则不能把 worker rank 映射为物理设备故障。
8. **把最后一行 kernel 摘要直接当失败 kernel**：异步模式下只能作为候选；首个失败 kernel 需要 blocking 返回、同步边界或其他明确 completion evidence。
9. **混用 1400/1500 MHz**：会让文本和 CSV 时间不一致；本流程统一使用 1400 MHz，并要求 CSV 明示 `clock_mhz`。
10. **删除原始 log**：CSV 是派生摘要，必须保留原始 `syn_*.ansi`；需要审计时同时保存命令 stdout 中的 `tool.py` 和 CSV digest。
11. **把聚合排名当模型归因**：同名 kernel 可跨 request、layer、rank、device 和 shape；缺独立绑定证据时这些 scope 都保持 `not_verified`。

## 相关资料

- [operator-dispatch/dispatch-key-and-runtime-kernel-mapping.md](../operator-dispatch/dispatch-key-and-runtime-kernel-mapping.md)
- [testing/pytorch-test-replay-from-synapse-log.md](../testing/pytorch-test-replay-from-synapse-log.md)
- [debugging-workflows/common-debug-commands.md](common-debug-commands.md)

## 来源

- `/work/plans/运行过程.md`
