# Synapse log 与 kernel 摘要工作流

## 适用场景

- 需要确认一次模型运行实际 launch 了哪些 DLC runtime kernel。
- 需要为 dispatch 验证、replay、挂起排障准备 `syn_*.ansi` 和 `*_kernels.txt`。
- 需要把“开环境变量 -> 跑模型 -> 产生日志 -> 摘要 kernel 列表”做成稳定流程。

## 核心结论

用于 DLC Platform 运行路径分析的最小闭环是：

1. 开 Synapse debug 环境变量
2. 跑模型或最小复现
3. 收集 `syn_*.ansi`
4. 用 `python3 tool.py <syn_*.ansi>` 生成 `*_kernels.txt`
5. 再基于 log 和摘要做 dispatch 判断、replay 或挂起排查

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

### 步骤 3：生成 `*_kernels.txt`

```bash
cd /work/llama2-fine-tune
python3 tool.py /work/tmpx/syn_1123318.ansi
```

输出类似：

```text
/work/tmpx/syn_1123318.ansi -> /work/tmpx/syn_1123318_kernels.txt
```

### 步骤 4：阅读 kernel 摘要

`*_kernels.txt` 的用途：

- 快速浏览本次 run 实际 launch 了哪些 kernel
- 判断某条路径是否仍在走 DLC
- 给 `pytorch_test --replay -f <kernel>` 提供候选 kernel 名

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

## 相关资料

- [operator-dispatch/dispatch-key-and-runtime-kernel-mapping.md](../operator-dispatch/dispatch-key-and-runtime-kernel-mapping.md)
- [testing/pytorch-test-replay-from-synapse-log.md](../testing/pytorch-test-replay-from-synapse-log.md)
- [debugging-workflows/common-debug-commands.md](common-debug-commands.md)

## 来源

- `/work/plans/运行过程.md`
