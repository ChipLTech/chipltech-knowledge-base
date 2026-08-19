# Case Study: vLLM async speculative decode launch failure

## 问题现象

GLM-5.1-Mix AWQ 在 TYD Chip、TP8/EP8 serving 中完成模型加载和 API readiness。首个流式请求完成 prefill、返回约 6 个 token 后，worker 报 `synErrorLaunchFailure`，EngineCore 随后死亡，API 最终返回 HTTP 500。

本案例截至 2026-07-30 只定位到组合执行边界，尚未捕获首个失败 DLC Custom Kernel。

## 背景与环境

- 模型：GLM-5.1-Mix AWQ。
- dtype：bfloat16。
- 并行：TP8 / EP8。
- 量化：AWQ 4-bit、`group_size=128`、`zero_point=true`。
- MoE：256 routed experts，每 token 选择 8 experts。
- decode：async scheduling、`deepseek_mtp`、3 speculative tokens。
- attention：Sparse MLA / IndexCache。
- Graph：`FULL_DECODE_ONLY`。
- PyTorch：2.5.0。
- vLLM：`0.2.8.dev14983+gfc84bce57`。
- vllm-cl：`0.1.dev77+gcee737e46.d20260708`。

## 定位路径

### 1. 先按生命周期排除前置阶段

原日志证明 141 个 checkpoint shard、AWQ 权重、MTP drafter、KV cache 和 Graph capture 均完成，API 已监听，请求也完成 prefill 并返回部分 token。因此最早失败边界不在模型资产、初始 TP/EP、API 启动或 prefill。

第一 fatal 是：

```text
DLC error: synErrorLaunchFailure
```

DLC Runtime 明确提示 kernel error 可能在后续 API 异步报告。`c10::dlc::SetDevice` 只是观察到错误的 API 边界，不是已确认的失败操作。

错误传播链：

```text
较早 DLC Custom Kernel execution failure
-> synErrorLaunchFailure
-> worker abort
-> executor cancellation
-> EngineCore dead
-> HTTP 500
```

### 2. 用 scheduler state 收窄 decode step

fatal step：

```text
num_computed_tokens=[13]
num_output_tokens=[6]
num_scheduled_tokens=4
scheduled_spec_decode_tokens=[-1, -1, -1]
```

安装源码证明 `-1` 是 `AsyncScheduler` 的合法 speculative-token placeholder。worker 应使用 GPU draft-token buffer 回填该槽位，不能据此断言非法 token 进入 embedding。

MTP=3 时 uniform decode query length 为 `1+3=4`。实际 full-decode Graph size 为 4、8、12，fatal step 精确命中最小 4-token Graph。最小窗口收敛为：

```text
async MTP draft-token 回填
-> token / position / slot mapping metadata
-> 4-token FULL_DECODE_ONLY Graph replay
-> MTP / Sparse MLA / IndexCache / AWQ-MoE execution
-> asynchronous launch failure
```

这只证明 Graph 参与 fatal step，不证明 Graph 是必要原因。

### 3. 审计启动环境

操作记录曾把环境变量 assignment 与 `nohup vllm serve` 分为两条命令。未 export 的 shell variable 不会自动进入后续子进程。

源码默认值显示漏传并非中性：

| 变量 | 预期 | 默认 | 变化 |
|---|---:|---:|---|
| `DLC_SYN_COPY_ASYNC` | O2 | O3 | 小 H2D 可额外走异步 copy kernel |
| `DLC_SYN_URING` | 0 | 1 | uring submission 默认启用 |
| `DLC_SYN_GRAPH_CACHE` | 1 | 0 | Graph cache 默认关闭 |
| `VLLM_USE_DLC_COL_MAJOR_MATMUL` | 1 | 0 | 部分 linear、shared expert、MLA/attention 路由变化 |

原失败进程已退出，无法读取其最终环境，因此该项是高置信风险，不是已确认根因。

### 4. 实际 E0 暴露权重加载新边界

E0 尝试正确导出文档 profile，并保持 MTP=3、IndexCache 和 Graph 不变。它没有到达请求：只有 rank 2/3 完成约 54.8 GiB 权重放置，其余 rank 约 130 MB HBM并停在 host completion wait。

最终 worker 环境显示变量传播不完整：Graph cache 和 col-major 到达，copy、uring 和部分 Sparse MLA 变量未到达。由此确认 launcher 外层环境不能替代最终 worker `/proc/<pid>/environ` 证据。

E0 是权重加载 stall，不是原 decode failure 的复现。

### 5. 实际 E1 暴露 blocking 初始化新边界

E1 使用原实际 profile并增加 blocking/debug。最终 worker 只收到 `DLC_LAUNCH_BLOCKING=1`；`DLC_SYN_BLOCKING`、debug 和日志目录没有到达。

8 张 TYD Chip 均停在约 130 MB HBM，服务未进入权重加载，也没有生成有效逐 rank launch trace。E1 是初始化 stall，不能用来命名原 decode 的失败 kernel。

### 6. 安全清理

E1 的 EngineCore 在 TERM 后退出，但 resource tracker 和部分 worker 残留。清理前逐一核验 PID、PGID、starttime 和 cmdline，只对得到授权且仍匹配的任务 PID 使用 KILL。清理后 8 张 TYD Chip HBM 均恢复到实验前基线。

## 当前结论

已确认：

- 原故障发生在 decode，不在模型加载、API readiness 或 prefill。
- fatal scheduler step 是 async MTP=3 的 4-token step。
- 该 step 命中最小 `FULL_DECODE_ONLY` Graph。
- `SetDevice` 是异步错误报告点。
- E0 和 E1 分别是权重加载与初始化的新边界。

高置信故障域：

```text
async MTP token/metadata/buffer update
+ 4-token Graph replay
+ Sparse MLA / IndexCache
+ AWQ-MoE
+ async copy / uring timing
```

尚未验证：

- 首个失败 DLC Custom Kernel。
- Graph、MTP=3、async scheduling 或 IndexCache 是否为必要触发条件。
- 特定 TYD Chip、DLCCL 或 LYP 是否参与请求期故障。

## 下一验证

恢复原可启动 profile，只增加 `--enforce-eager`，固定 MTP=3、async scheduling、IndexCache、AWQ TP8/EP8 和短请求。只有 eager 到达同一请求边界，才能使用结果判断 Graph 是否为必要触发条件。

## 可复用经验

1. 用服务生命周期定位最后成功阶段，HTTP 500 和 Engine death 通常是传播结果。
2. 异步 launch error 的报告 API 不是失败 kernel；没有 blocking completion evidence 时不得命名 kernel。
3. scheduler placeholder 必须按安装源码解释，负数不自动等于非法 token。
4. Graph capture PASS 不证明真实动态 state replay 正确；先做 Graph/eager 单变量对照。
5. 最终 worker 环境才是 Runtime 开关的有效证据。
6. 诊断开关导致更早 stall 时，应建立新边界，不得将其写成原故障前移。
7. 清理仅针对身份密封的任务 PID，并以进程、端口、handles 和 HBM baseline 闭环。

## Claim Boundary

本案例不声明 GLM-5.1 根因已修复，不声明某个候选 W4A16、MLA 或 Graph kernel 已确认失败，也不将 SMI 正常外推为请求执行期硬件健康。结论只适用于记录的 image、模型 profile 和已观察到的三个 server epoch。

## 来源

- `/work/err/glm5-1-error-summary.md`
- `/work/err/glm5-1-error-localization.md`
- `/work/err/glm5-1-debugging-process-asset.md`
