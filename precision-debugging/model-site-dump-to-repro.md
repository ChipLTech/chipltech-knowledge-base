# Model-Site Dump 到 pytorch_test 复现流程

## 适用场景

- 真实模型在 DLC Platform 上出现 NaN/Inf 或输出不一致。
- 需要把端到端模型问题压缩成 pytorch_test Framework 中可复现的最小 case。
- 交付算子团队做进一步分析和修复。

## 核心结论

从 Model-Site Dump 到 pytorch_test 复现的标准路径是：

1. **在模型 failure 点做 dump**。
2. **用 compare report 找第一个 divergent module**。
3. **对该 module 做算子级 replay**。
4. **收敛到最小可复现 case**。
5. **交付 code-only repro**（无需 dump 文件、safetensors 或模型权重）。

## 操作流程

### 阶段 1：原始定位 — 模型级 dump

找到第一个异常张量的方法论：

```
从端到端输出倒推：
1. generated tokens / logits (token 分叉/异常)
2. lm_head / decoder output
3. decoder Linear / RMSNorm
4. decoder attention / MLP (prefill and decode)
5. visual encoder blocks (如果多模态)
6. embeddings / position encoding
7. patch embedding / conv / normalization
8. image processor / input (确认输入一致)
```

dump 脚本示例结构：
```python
# 在关键 module 的 forward 中注入 hook
# 保存 input、output、中间变量
# 格式：tensors = {"input": x, "output": y, ...}
# torch.save(tensors, "dump_path.pt")
```

### 阶段 2：compare report

用 compare report 工具比 CPU 和 DLC dump：

```bash
python scripts/compare_rsthinker_precision_dumps.py \
  --run_dir /path/to/compare_output \
  --cuda_dump /path/to/cpu_dump \
  --dlc_dump /path/to/dlc_dump \
  --output_md /path/to/report.md \
  --output_json /path/to/report.json
```

从 report 中找第一个 `abs_diff_max > 0` 的 module，这就是当前的最小边界。

### 阶段 3：Precision Test Run

标准精度验证流程：

1. 选择同一个 Precision Run Directory。
2. 在 CUDA/CPU 环境采集 baseline dump。
3. 在 DLC 环境采集对比 dump。
4. 确认两个 dump 的 processor inputs 一致（input_ids、attention_mask、pixel_values 等）。
5. 对比每个 module tensor output。

### 阶段 4：单算子 replay

对第一个 divergent module 构造最小 replay：

```python
import torch

# 加载真实输入
dump = torch.load("dump_path.pt", map_location="cpu")
x = dump["module_input"]
weight = dump.get("module_weight")  # 如果有

# CPU 执行
x_cpu = x.cpu()
with torch.no_grad():
    y_cpu = F.linear(x_cpu, weight.cpu())

# DLC 执行
x_dlc = x.to("dlc")
weight_dlc = weight.to("dlc")
with torch.no_grad():
    y_dlc = F.linear(x_dlc, weight_dlc)

# 比较
print(f"CPU vs DLC: equal={torch.equal(y_cpu, y_dlc.to('cpu'))}")
print(f"abs_max={(y_cpu - y_dlc.to('cpu')).abs().max()}")
```

### 阶段 5：收敛到最小边界

1. **确认 replay 复现**：`cpu_replay_vs_saved_cpu` 和 `dlc_replay_vs_saved_dlc` 都等于 exact。
2. **确认真实 kernel 路径**：用 `DLC_SYN_DEBUG=1 DLC_SYN_VERBOSE=1` 查看 runtime kernel name。
3. **单变量 fallback**：将该算子的 dispatch 改为 CPU，验证问题消失。
4. **最小 shape**：逐步缩小 tensor shape，保持复现性。

### 阶段 6：code-only repro

最终交付给算子团队的测例应满足：

- **无依赖**：不依赖 .pt dump 文件、safetensors、模型权重。
- **自包含**：所有输入完全由代码生成（随机 tensor + 固定 seed）。
- **可运行**：`python test_file.py` 直接运行。
- **明确 kernel**：标注 runtime kernel name 和源码路径。

阶段 5 到 6 的收敛示例：
```
阶段 5: load .pt -> input [1764, 1536], weight [4096, 1536]
阶段 6: torch.randn(1764, 1536) * 0.1, torch.randn(4096, 1536) * 0.02
        (固定 seed，模拟真实 input/weight 分布)
```

## Lazy Dump 策略

当同步或拷贝可能遮盖异步 bug 时，使用 Lazy Dump：

- 只在 failure 检测到后才写完整 state。
- 不改变原有的执行调度。
- 尤其适用于 optimizer/foreach 等对异步敏感的路径。

## 精度 tolerance 协商

与 kernel owner 的 tolerance 协商关键点：

- 先对齐 tolerance 标准，再做大量测试。
- 常用 owner 接受标准：`("rpd-filter2sigma-p95", 1e-4, 1e-4)`。
- 过严的 tolerance（如 `atol=1e-6`）可能被 owner 拒绝。
- 不同算子类型有不同的合理 tolerance。

## 常见坑

1. **跳过 replay 验证**：必须先确认 `cpu_replay == saved_cpu` 和 `dlc_replay == saved_dlc`，否则 replay 的 bug 不是原始 bug。
2. **用 CPU fallback 后立即宣称修好了**：CPU fallback 是定位手段，不是修复方案。
3. **一次改多个 dispatch**：单变量定位要求一次只改一个。
4. **tolerance 未协商就写报告**：owner 可能因为 tolerance 标准不一致而拒绝。
5. **交付需要外部文件的 repro**：算子团队需要 code-only repro。

## 相关资料

- [operator-dispatch/enabled-kernels-dispatch.md](../operator-dispatch/enabled-kernels-dispatch.md)
- [testing/dlc-kernel-test-framework-guide.md](../testing/dlc-kernel-test-framework-guide.md)
- [case-studies/](../case-studies/)

## 来源

- `/work/RSThinker/docs/rsthinker_precision_dump_runbook.md`
- `/work/RSThinker/docs/rsthinker_custom_softmax_diagnosis_experience.md`
- `/work/RSThinker/docs/rsthinker_visual_mlp_drift_diagnosis_experience.md`
- `/work/plan/dlc基础/Jarvis_DLC问题定位到pytorch_test复现流程报告.md`
- `/work/plan/dlc基础/Paraformer_QKV_Linear精度问题到pytorch_test复现经验报告.md`
