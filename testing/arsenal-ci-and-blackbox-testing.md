# Arsenal CI 与黑盒测试工具入口

## 适用场景

- 需要理解 Arsenal 仓库在 DLC Ecosystem 中承担的 CI/CD、模型测试、DLCCL 排障、Syzkaller fuzzing 和 benchmark 职责。
- 需要让 agent 判断某类改动应该触发哪些 CI 或使用哪些测试工具。
- 需要把 vLLM OpenAI-compatible serving 的黑盒测试和 benchmark 结果纳入模型适配报告。

## 核心结论

Arsenal 是 ChipLTech 软件链 CI 系统与工具集成仓库。它不是单个业务组件，而是多仓库协同测试、硬件/模型测试、驱动 fuzzing、镜像构建、vLLM benchmark 和辅助版本检查工具集合。

知识库使用时应把 Arsenal 视为 **测试与诊断工具入口**：引用它的目录和能力边界，不复制 CI 实现，不把 Arsenal benchmark 或黑盒 HTTP success 提升成 Real DLC Hardware acceptance。

## 主要能力映射

| 能力 | Arsenal 入口 | 说明 |
|---|---|---|
| CI 路由 | `.github/workflows/ci_router.yml`、`arsenal_CI/determine_workflows.sh` | 根据文件变更决定触发哪些 workflow |
| 多仓库测试 | `CI_Split/` | 联合 LLVM、DLCSynapse、DLC_Custom_Kernel Repository、DLCsim、dlc-thunk、DLC_CL、PyTorch、vLLM 等 |
| Kernel 测试 | `kernel_test/` | 单次、循环、PyTorch kernel 测试，测试用例来自 Excel 配置 |
| 模型自动测试 | `model_auto_tests/` | 批量模型性能基准测试 |
| vLLM benchmark | `vllm_benchmark_serving_script/` | 启动 serving、运行 benchmark_serving、抽取指标 CSV |
| 黑盒 HTTP 测试 | `tpu_blackbox_tests/` | 对 vLLM OpenAI-compatible API 做 stress、boundary、fuzz |
| DLCCL hang 分析 | `dlccl_check.py` | 读取 XYS/SMEM/sync 状态，推断 RSYNC/RDMA 等等待关系 |
| Syzkaller fuzzing | `arc_syzkaller_fuzzing/` | 驱动和真实 PCI 场景 fuzzing、crash 收集、Issue 创建 |
| QEMU passthrough | `qemu_tpu_pt_test/` | QEMU 直通测试、VFIO、容器启动/监控 |
| 镜像构建 | `Images_build/` | CI base image、ARC image、PyTorch CI image |
| 版本检查 | `check_version.py` | 比较本地安装版本和 public version manifest |

## CI 路由知识

`ci_router.yml` 是 PR CI 入口，核心职责：

- 检查 commit message 是否符合 `<type>: #<issue_number> <description>`。
- 检查关联 Issue 是否 open。
- 调用 `determine_workflows.sh` 根据变更路径触发工作流。

常见路由：

| 变更范围 | 触发方向 |
|---|---|
| `CI_Split/test_on_tpu/*`、`CI_Split/utils/*` | 传统多仓库测试 |
| `CI_Split/ARC_CI_singlecard/*` | ARC 单卡测试 |
| `CI_Split/ARC_CI_multicard/*` | ARC 多卡测试 |
| `kernel_test/*` | Kernel 测试 |
| `kernel_test/loop/*` | 循环 Kernel 测试 |
| `kernel_test/once/python_script/run_pytorch*` | PyTorch Kernel 测试 |
| `model_auto_tests/*` | 模型测试 |
| `arc_syzkaller_fuzzing/*` | ARC Syzkaller fuzzing |
| `Images_build/*` | 镜像构建 |

## ARC_Debug 手动验证

`ARC_Debug` 是手动触发工作流，用于跨仓分支组合和按需测试。它支持：

- 对多个上游仓库独立指定 branch 或 commit SHA。
- 按需选择单卡、多卡、PyTorch、vLLM infer、vLLM benchmark。
- 设置 `test_repeat_count` 做偶现问题压力测试。
- 注入临时 `extra_env`。
- 在 debug mode 下失败后保留现场一段时间，便于人工 SSH 排查。

使用边界：

- 适合验证跨仓 fix 组合，不适合替代本地最小复现。
- 手动填入的 branch/SHA 必须记录在报告里。
- 自动设备修复或 Runner 标签恢复属于 CI 基础设施行为，不应写成业务代码结论。

## vLLM benchmark serving pipeline

`vllm_benchmark_serving_script/` 的典型流程：

```text
config.json
  -> run_pipeline.py
  -> autotest.py 启动 vLLM server
  -> benchmark_serving.py 按输入/输出长度和 num_prompts 运行 client
  -> extract_logs.py 提取 CSV 指标
```

关键配置字段：

- `workspace_dir`：vLLM 工作目录。
- `model_dir_path` 和 `model_name`：组合出模型路径，通常可指向 `/mnt/jfs/models/<model>`。
- `server_cmd_template`：serving 启动命令。
- `client_cmd_template`：benchmark client 命令。
- `num_prompts_list`：请求数量梯度。
- `input_output_pairs`：输入/输出 token 长度组合。
- `log_base_dir`：日志和 CSV 输出根目录。
- `server_start_timeout`：server ready 等待时间。

抽取指标包括：

- Successful requests。
- Benchmark duration。
- Total input/generated tokens。
- Request throughput。
- Output token throughput。
- Total token throughput。
- Mean/Median/P99 TTFT。
- Mean/Median/P99 TPOT。
- Mean/Median/P99 ITL。

使用边界：

- 该 pipeline 会启动并终止自己创建的 server process group；不要用于管理非本任务进程。
- benchmark 结果应和 server command、模型路径、TP/PP、dtype、quantization、Chunked Prefill、prefix caching、input/output length 一起保存。
- `request-rate inf` 更偏压力/吞吐测试，不等价于真实业务流量。
- benchmark success 不等于模型正确性或 Real DLC Hardware acceptance。

## 黑盒 HTTP 测试

`tpu_blackbox_tests/` 面向 vLLM OpenAI-compatible HTTP API，包含三类：

| 类型 | 工具 | 目标 |
|---|---|---|
| stress | Locust | 并发吞吐、延迟稳定性 |
| boundary | pytest | 参数极限值、非法参数处理 |
| fuzz | schemathesis + pytest | 畸形输入下服务不崩溃 |

典型环境变量：

```bash
export TPU_API_URL=http://localhost:8003
export TPU_MODEL=/mnt/jfs/models/Llama-2-7b-hf
```

知识库中保留该变量名作为 Arsenal 原工具接口；报告中应使用正式术语描述为 vLLM OpenAI-compatible API 与 Chipltech-Family Accelerator serving，不要用该变量名泛化硬件术语。

判断边界：

- 200 且延迟稳定：说明该 API 场景下服务可用。
- 500/502/503：服务崩溃或适配异常线索。
- P99 持续飙升：内存泄漏、队列堆积或性能退化线索。
- 400/422：非法参数被拒绝，通常是正常行为。
- schemathesis `not_a_server_error` failure：畸形输入可能触发服务错误。

黑盒测试不证明底层 DLC Custom Kernel 正确性、DLC Runtime dispatch 或 Verified vLLM Alignment；它是 serving 层稳定性证据。

## DLCCL hang 分析工具

`dlccl_check.py` 通过 `/usr/local/chipltech/cl_tools` 下的 CSR/SMEM/sync 读取工具，分析多卡通信等待关系。它会观察：

- XYS runstate。
- PC/NPC。
- 当前 DLC Custom Kernel 名称。
- sync entry。
- ring order / chip id。
- RSYNC recv/send 等待。
- RDMA send 等待。
- 双方互等或未知原因卡死。

典型使用：

```bash
python3 dlccl_check.py 0 1 2 3
```

使用边界：

- 这是设备现场诊断工具，需要目标设备和底层 cl_tools 可用。
- 输出可作为“谁在等谁”的排障线索，不等于根因证明。
- 读取底层寄存器/SMEM 前确认当前任务拥有目标设备排障授权。

## 版本检查工具

`check_version.py` 用于对比本地安装版本和 public `ChipLTech/version-manifest`：

```bash
python3 check_version.py
python3 check_version.py offline
python3 check_version.py --summary
```

它会尝试读取：

- DLCsim、DLCSynapse、DLC_CL 等安装产物版本文件。
- DLC-Kernel-Driver sysfs 版本。
- `libcustom_dlc_perf_lib.so` 中的 LLVM / DLC_Custom_Kernel Repository SHA。
- PyTorch installed package、Git repo 或 wheel 中的 git version。
- vLLM installed package、Git repo 或 wheel 中的 commit id。

使用边界：

- 版本检查只能证明本地安装/文件中暴露的版本线索。
- 无法读取时应记录 `N/A` 或 `Unknown`，不要猜测。
- 与 remote manifest 对齐不等于当前 checkout clean、构建产物 fresh 或 Verified vLLM Alignment。

## 相关资料

- [pytorch-test-framework-guide.md](pytorch-test-framework-guide.md)
- [dlc-kernel-test-framework-guide.md](dlc-kernel-test-framework-guide.md)
- [../runtime-debugging/performance-profiling.md](../runtime-debugging/performance-profiling.md)
- [../vllm-dlc/model-adaptation-and-main-to-main-decisions.md](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)

## 来源

- `/work/arsenal/README.md`
- `/work/arsenal/tpu_blackbox_tests/README.md`
- `/work/arsenal/vllm_benchmark_serving_script/`
- `/work/arsenal/dlccl_check.py`
- `/work/arsenal/check_version.py`
