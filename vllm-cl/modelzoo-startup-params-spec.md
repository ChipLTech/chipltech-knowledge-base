# ModelZoo 模型验证——启动参数规范

**生效时间**: 2026-07-29

## 目录约定

| 目录 | 用途 | 内容 |
|---|---|---|
| `/home/xuansun/ModelZoo` | ModelZoo 仓库（只读 reference） | metafile.yml, README.md, benchmark artifacts |
| `/mnt/jfs/modelzoo` | 模型验证 evidence | results.json, difficulties.md, server.log |
| `/home/xuansun/modelconfig` | 模型验证总结分析文档 | summary.md, batch-runner 脚本 |

## 强制规则

在对任何一个 ModelZoo 模型执行验证前，**必须先读取该模型在 ModelZoo 中的 README.md 和 metafile.yml**，提取以下参数后按模型官方参数启动：

### 1. 必须从 ModelZoo README.md 提取的参数

| 参数 | 提取方式 | 默认值（找不到时） |
|---|---|---|
| `tensor_parallel_size` | 搜索 `-tp N` | 4 |
| `block_size` | 搜索 `--block-size N` | 256 |
| `enforce_eager` | 搜索 `--enforce-eager` | True |
| `dtype` | 搜索 `--dtype X` | bfloat16 |
| `gpu_memory_utilization` | 搜索 `--gpu_memory_utilization X` | 0.95 |
| `enable_chunked_prefill` | 存在则为 True，无则为按需 | True |
| `seed` | 搜索 `--seed N` | 0 |
| `max_model_len` | 搜索 `--max-model-len N` | 1024 |
| `trust_remote_code` | 搜索 `--trust-remote-code` | 按 config.json 中 auto_map 判定 |
| `max_num_seqs` | 搜索 `--max-num-seqs N` | 8；当前 vLLM-CL engine 不支持 1 |

### 2. 必须从 ModelZoo README.md 提取的环境变量

| 变量 | 搜索模式 | 当前 host 是否使用 |
|---|---|---|
| `DLC_SYN_URING` | `DLC_SYN_URING=1` | ✅ 可安全使用 |
| `VLLM_USE_DLC_COL_MAJOR_MATMUL` | `VLLM_USE_DLC_COL_MAJOR_MATMUL=1` | **❌ 必须排除** — 与 sealed vLLM-CL 不兼容，导致 Engine Init 失败 |
| `VLLM_USE_V1` | `VLLM_USE_V1=1` | ✅ 可安全使用 |
| `DLC_SYN_COPY_ASYNC` | `DLC_SYN_COPY_ASYNC=O0` | ✅ 必须使用（当前 Host 稳定 profile） |

### 3. 特殊参数（仅在 README 中明确指定时使用）

- `--compilation-config`：QwQ-32B 特有
- `--no-enable-prefix-caching`：QwQ-32B 特有
- `--max-model-len 4096`：QwQ-32B 特有

### 4. 无 README.md 的模型

如果模型仅有 metafile.yml 无 README.md：
- 使用默认参数
- 记录 `# ModelZoo README not found — using defaults`
- 在 difficulties 中标记 `missing_modelzoo_docs`

### 5. 检查顺序

```
1. 读 metafile.yml → 确认 Infer/Train 状态
2. 读 README.md → 提取 TP / env / special params
3. 功能验证通过单模型 runner 启动 `LLM()`；仅在任务明确要求 serving 时启动 API server
4. 执行 functional test
5. 保存结果 + 记录参数来源
```

## 禁止行为

- 禁止忽略 ModelZoo 明确给出的模型专用参数；未指定 `max_num_seqs` 时使用已验证的安全基线 8
- 禁止不读 ModelZoo README 就启动模型
- 禁止假设 Qwen 系列的参数适用于其他模型架构

## 相关代码

批量子进程脚本从 ModelZoo README 提取参数的函数：
`parse_modelzoo_params()` — 位于 `/opt/batch-v7.py` (容器内)

## 定时任务规则（防冲突）

### 强制规则

1. **每次定时任务只验证一个模型**。禁止在同一设备组 (4-7) 上并发启动多个模型。
2. **模型之间必须有间隔**。上一个模型的 server/进程完全退出、HBM 清零后，下一个模型才能开始。
3. **间隔时间**：根据上一个模型大小计算——7B 级 >=5min，14B 级 >=8min，32B 级 >=15min。
4. 定时任务脚本必须：
   - 启动前检查 `cltech_smi` 确认设备 4-7 HBM ≤ 500 MB
   - 若有残留进程则先 `pkill -f vllm`
   - 验证完成后保存结果，然后立即停止 server
   - 停止后等待 HBM 清零再退出

### 禁止行为

- 禁止单次任务验证多个模型（避免 HBM/进程冲突）
- 禁止不管前一个模型是否退出就启动下一个
- 禁止把 batch script 循环当作"定时任务"

### 超时与自动降级规则

| 规则 | 值 |
|---|---|
| Tier 1 server 启动超时 | 15 分钟 (900s) |
| Tier 2 server 启动超时 | 15 分钟 (900s) |
| 超时后动作 | 自动记录 `tier1_server_timeout` difficulty，降级到 Tier 2 |

**降级逻辑**：
```
Tier 1 (daily vLLM + vllm-cl)
  ↓ server 15min 无响应 / engine init 失败 / 输出错误
Tier 2 (chiju_env:0729 O2, vllm-cl 需编译, vllm 需 symlink)
  ↓ 仍失败
记录 both_tiers_failed → 标记 Hermes 下一步操作
```

每次验证完成一个模型后，**必须**保存该模型遇到的所有问题、困难和中断点。

**保存规则**：

1. 如果有任何困难（启动失败、错误输出、超时、架构不兼容等），写入 `difficulties.md`（位于 `RESULT_ROOT/<model_name>/`）
2. 困难分类：
   - `needs_trust_remote_code` — 模型需要 trust_remote_code
   - `engine_init_failed` — LLM engine 初始化失败（架构不兼容）
   - `wrong_output` — 模型加载成功但输出错误
   - `out_of_memory` — HBM 不足
   - `tokenizer_error` — tokenizer 加载失败
   - `missing_config` — config.json 缺失
   - `timeout` — 模型加载/推理超时
   - `unknown_error` — 未知错误
3. 每条困难记录：模型名、困难类型、时间、详细错误信息、尝试的修复方案
4. 无困难时**不记录**（不产生空文件）

**Hermes 学习机制**：

后续 Hermes 在开始新模型验证前，应先读取 `difficulties.md` 文件中同模型的历史困难记录，并按以下策略应对：
- `needs_trust_remote_code` → 自动添加 `--trust-remote-code`
- `engine_init_failed` → 检查 ModelZoo README 中的 TP/架构要求；若标记 Infer=false 则跳过
- `wrong_output` → 检查 DLC 精度差异文档，评估是否为已知 DLC Precision Difference
- `out_of_memory` → 降低 `gpu_memory_utilization` 或 `max_model_len`
- `tokenizer_error` → 标记为架构不兼容

**累计优化**：每轮批量验证完成后，总结新发现的困难模式，更新本规范文档。

## 更新历史

- 2026-07-29: 初始规范 + 定时任务隔离规则 + 困难追踪、Hermes 学习机制和超时降级规则
