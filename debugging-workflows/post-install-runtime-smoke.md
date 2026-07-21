# 安装后的 Runtime Smoke

## 适用场景

- PyTorch 2.5.0 wheel 已经构建并重装完成，需要确认环境是否真的可用。
- 本地 `vllm` 或 `vllm-dlc` 刚做完 editable install，需要一个短路径判断它们是否健康。
- 想区分“包已经装上”与“DLC Platform 运行时真的健康”这两个不同问题。

## 核心结论

1. post-install smoke 必须在源码树外执行，否则 `import torch` 很容易被源码树污染。
2. Package/import smoke 与 Real DLC Hardware execution smoke 是两个 checkpoint；前者通过不证明设备可以执行。
3. Real DLC Hardware execution smoke 必须在 fresh process 中分层覆盖设备枚举、allocation、H2D、device operation、synchronize、D2H 和 correctness，并设置明确 timeout。
4. 如果 smoke 失败或 hang，按最早失败层级记录边界，不要直接继续 patch 上层仓库或加载模型。

## 操作步骤

### 1. 切到源码树外目录

推荐使用 `/tmp` 或其他明确不在 PyTorch、`vllm`、`vllm-dlc` 源码树内的目录。

原因：站在源码树里 `import torch`，可能命中源码目录，出现假阴性，误判为 wheel 安装失败。

### 2. 运行 Package/Import Smoke

```bash
python3 -c "import torch; print(torch.__version__)"
python3 -c "import torch; print(torch.tensor([0.1], dtype=torch.float32).numpy())"
python3 -c "import torch; print(torch.backends.dlc.is_available() if hasattr(torch.backends, 'dlc') else 'no torch.backends.dlc')"
```

重点不是只看 `import torch` 成功，而是确认：

- 版本是目标版本
- NumPy bridge 正常
- DLC Platform 路线可见

该步骤只创建 `Package/Import Ready` checkpoint。它不验证：

- Real DLC Hardware allocation。
- Host-to-device copy。
- 首个 device operation 是否完成。
- stream synchronize。
- Device-to-host copy 和结果正确性。

Jiutian 75B 恢复案例已经证明：设备可见、allocation 和 H2D 均可成功，而首个 device operation 仍可能 hang。因此不能把 `torch.backends.dlc.is_available()` 或单纯 `.to("dlc")` 当成 `DLC Runtime Execution Ready`。

### 3. 如本次涉及 `vllm`，先按 Platform Contract 分支检查

所有路径先检查：

```bash
python3 -c "import vllm; print(vllm.__version__, vllm.__file__)"
python3 -m pip show vllm
python3 -m pip list | rg '^(vllm|vllm-dlc|vllm_dlc|UNKNOWN|triton|torch|numpy)\b'
```

如果 contract 使用 candidate 内建 `DLCPlatform`：

- 验证 candidate import path 和实际 platform class。
- 记录 platform entry points、`VLLM_PLUGINS` 和 plugin 禁用状态。
- 将 `vllm_dlc` package/import 标记为 `not_applicable`，不要执行预期失败的 plugin import 命令。

如果 contract 使用 `vllm-dlc` plugin，再执行：

```bash
python3 -c "import vllm_dlc; print(vllm_dlc.__file__)"
python3 -m pip show vllm_dlc
```

Plugin import 或 metadata 失败必须使 package checkpoint 失败，不能只打印后继续。两种 contract 都应确认 `pip list` 中不再出现 `UNKNOWN`；如果当前环境策略不需要 `triton`，确认它没有成为残留冲突源。

### 4. 运行 Fresh-Process Layered Execution Smoke

仅在需要 Real DLC Hardware 执行或模型 serving 时运行。先用 `DLC_VISIBLE_DEVICES` 固定物理设备映射，再在 fresh process 中执行以下分层 probe，并用外层 `timeout` 限制总时长：

```python
import torch


def report(name, operation):
    print(f"BEGIN {name}", flush=True)
    value = operation()
    print(f"PASS {name}", flush=True)
    return value


print(f"torch={torch.__version__} path={torch.__file__}", flush=True)
print("BEGIN backend_availability", flush=True)
backend_available = (
    hasattr(torch, "dlc")
    and hasattr(torch.backends, "dlc")
    and torch.backends.dlc.is_available()
)
if not backend_available:
    raise RuntimeError("DLC Platform backend/API surface is unavailable")
print("PASS backend_availability", flush=True)
count = report("device_count", torch.dlc.device_count)
if count <= 0:
    raise RuntimeError("no logical DLC device is available")
report("device_properties", lambda: torch.dlc.get_device_properties(0))
report("set_device", lambda: torch.dlc.set_device(0))
report("mem_get_info", torch.dlc.mem_get_info)
allocated = report("allocation_submit", lambda: torch.empty(1, device="dlc:0"))
print(
    f"META allocation shape={tuple(allocated.shape)} "
    f"dtype={allocated.dtype} device={allocated.device}",
    flush=True,
)
host = report("host_source", lambda: torch.ones(1))
device = report("h2d_submit", lambda: host.to("dlc:0"))
print(
    f"META h2d shape={tuple(device.shape)} dtype={device.dtype} device={device.device}",
    flush=True,
)
result = report("device_op_submit", lambda: device + 1)
print(
    f"META device_op shape={tuple(result.shape)} "
    f"dtype={result.dtype} device={result.device}",
    flush=True,
)
report("device_op_completion_synchronize", torch.dlc.synchronize)
copied = report("d2h", result.cpu)
if copied.item() != 2:
    raise AssertionError(
        f"device correctness mismatch: expected 2, got {copied.item()}"
    )
print("LAYERED_RUNTIME_EXECUTION_PASS", flush=True)
```

执行要求：

- 使用类似 `timeout --signal=TERM --kill-after=5 45 python3 <probe>` 的外层 timeout；具体秒数写入 manifest。
- 每个阶段立即 `flush`，超时后以最后一个 `BEGIN` 和前一个 `PASS` 确定最早边界。
- 不打印 DLC tensor 的 `repr` 或内容；格式化设备 tensor 可能隐式同步、执行额外算子或 D2H，从而移动失败边界。只记录 shape、dtype 和 device 等不读取数据的 metadata。
- `allocation_submit`、`h2d_submit` 和 `device_op_submit` 只表示调用返回/提交被接受；只有紧随其后的 synchronize 或 D2H 完成才能证明 execution completion。
- Probe 文件、完整 stdout/stderr、开始/结束时间、exit code、signal、设备映射和环境变量写入 artifact。
- Device operation 必须是真正的设备计算；只做 allocation 或 `.to("dlc")` 不充分。
- Probe 必须在 fresh process 中运行。恢复动作或设备状态变化后不能复用旧 Python process。
- 多设备 deployment 先在每张目标物理设备上分别运行，再并发运行目标设备 probe；需要 TP/多卡时还要独立验证 DLCCL collective correctness。

只有该步骤通过，才能创建 `DLC Runtime Execution Ready` checkpoint 并加载模型。

### 5. Skill 脚本能力边界

`dlc-env-setup` skill 自带的 `scripts/runtime-smoke.sh` 默认执行源码树外 package/import smoke。Real DLC Hardware 模型验证使用：

```bash
scripts/runtime-smoke.sh /tmp \
  --require-vllm \
  --skip-vllm-dlc \
  --require-device-execution \
  --device-index 0
```

Plugin deployment 将 `--skip-vllm-dlc` 替换为 `--require-vllm-dlc`；candidate 内建 `DLCPlatform` 使用 `--skip-vllm-dlc`，并单独验证 platform class、import path、entry points 和 plugin 禁用状态。`DLC_PACKAGE_SMOKE_TIMEOUT` 和 `DLC_RUNTIME_SMOKE_TIMEOUT` 可分别覆盖默认 45 秒 package/import 和 execution timeout。

运行前仍应读取实际脚本或 `--help`。如果历史脚本版本没有 `--require-device-execution`，或没有明确输出 allocation、H2D、device operation、synchronize、D2H 和 correctness，它只能创建 `Package/Import Ready` checkpoint；此时仍需额外运行第 4 节的 layered execution smoke。

### 6. 失败时的回退路径

- `import torch` 失败：先确认是否仍在源码树内，再回到 wheel 重装与安装位置检查。
- NumPy bridge 失败：回到 PyTorch build preflight，重点检查 `numpy==1.26.4` 与 `USE_NUMPY`。
- `torch.backends.dlc` 不可用：回到底层依赖安装和 DLC Platform 对接检查。
- Contract 必需的 `vllm` / `vllm_dlc` import 失败：回到 packaging preflight，而不是直接重编 PyTorch。标记为 `not_applicable` 的 plugin 不执行失败式探测。
- device enumeration 失败：检查设备映射、设备节点、权限和 DLC Runtime/Host driver identity。
- allocation 失败：检查 HBM、任务占用和 allocation 日志，不把它和 device operation hang 混为一谈。
- H2D 失败或 hang：边界记录为 copy path，不继续执行上层模型验证。
- device operation 失败或 hang：记录为 PyTorch DLC Backend / DLCSynapse / DLC Runtime / Host driver completion path 的当前最小边界；没有更多证据时不唯一归因。
- synchronize 或 D2H 失败：保留异步 launch 和 completion path 证据，不把后续 API 报错误写成首个失败 kernel。
- correctness 失败：保存 CPU Reference、device result、dtype、shape 和环境身份，不能以“请求完成”覆盖正确性失败。

## 常见坑

1. **在源码树中 smoke**：最常见的误判来源。
2. **只做 `import torch`**：这不足以验证 NumPy bridge 和 DLC Platform 可用性。
3. **把 `vllm` import 失败直接归因给 `vllm-dlc`**：底层 PyTorch wheel 不健康时，上层 import 往往只是受害者。
4. **忽略 `UNKNOWN` metadata**：这说明 editable install 元数据本身还有问题。
5. **用 allocation/H2D 代替执行验证**：设备内存可用不代表首个 device operation 能完成。
6. **恢复命令 exit 0 就声明健康**：service restart、soft reset 或进程清理后必须由 fresh-process layered probe 验收。
7. **连续执行多个恢复动作后唯一归因**：动作之间没有 fresh probe 时，只能报告 action sequence 后恢复，不能说最后一个动作是唯一修复。

## 相关资料

- [python-build-preflight-for-pytorch-and-vllm.md](python-build-preflight-for-pytorch-and-vllm.md)
- [../runtime-debugging/dlc-workstation-env-rebuild.md](../runtime-debugging/dlc-workstation-env-rebuild.md)
- [../runtime-debugging/runtime-troubleshooting.md](../runtime-debugging/runtime-troubleshooting.md)

## 来源

- `/work/test/dlc-env-setup/scripts/runtime-smoke.sh`
- `/work/test/dlc-env-setup/SKILL.md`
