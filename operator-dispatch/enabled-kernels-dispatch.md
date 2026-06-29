# enabled_kernels Dispatch 与 CPU Fallback

## 适用场景

- 需要把某个 DLC 算子切换为 CPU fallback 以缩小定位范围。
- 需要理解 DispatchType 机制和 enabled_kernels.hpp 的工作方式。
- 做单变量定位（一次只改一个算子）时的参考。

## 核心结论

DLC 算子的 dispatch 模式由 `enabled_kernels.hpp` 中的 `DispatchType` 常量控制。源文件位于 `/work/DLC_Custom_Kernel/dlc_src/enabled_kernels.hpp`，安装后常见路径为 `/usr/local/chipltech/synapse/include/enabled_kernels.hpp`。将某个算子的 `DispatchType::DLC` 改为 `DispatchType::CPU` 后，重新编译 PyTorch，该算子将在 CPU 上执行。

CPU fallback 是**定位手段**，不是生产修复方案。

## DispatchType 三种模式

| 值 | 含义 |
|-----|------|
| `DispatchType::DLC` | 在 DLC 上执行 |
| `DispatchType::CPU` | 在 CPU 上执行 |
| `BOTH` | CPU 和 DLC 同时执行并对结果比较 |

## 操作步骤

### 1. 定位需要 fallback 的算子

通过 verbose trace 确认 kernel launch name：

```bash
env DLC_SYN_DEBUG=1 DLC_SYN_VERBOSE=1 python3 main.py ... --perf --skip-check --dry-run
```

从输出中定位 `launch custom_xxx` 行，确认 runtime kernel name。

### 2. 查找对应的 dispatch 常量

在 PyTorch DLC Backend 源码中找到使用 `DLC_CHECK_RESULT(..., ...)` 的地方：

```
grep -r "DLC_CHECK_RESULT(custom_xxx" /work/pytorch/aten/src/ATen/native/dlc/
```

**关键**：`DLC_CHECK_RESULT` 的第一个参数（lambda name）才是 dispatch 常量，不是 launch kernel name。

### 3. 修改 dispatch 常量

编辑当前环境实际生效的 `enabled_kernels.hpp`，通常是 `/usr/local/chipltech/synapse/include/enabled_kernels.hpp`；如需从源码侧更新，则修改 `/work/DLC_Custom_Kernel/dlc_src/enabled_kernels.hpp` 并重新安装。将对应的常量改为 `DispatchType::CPU`：

```cpp
// 修改前
const DispatchType custom_matmul_t_pingpong = DispatchType::DLC;

// 修改后
const DispatchType custom_matmul_t_pingpong = DispatchType::CPU;
```

### 4. 重新编译 PyTorch

```bash
cd /work/pytorch
USE_CUDA=0 DEBUG=1 MAX_JOBS=32 python3 setup.py develop
```

### 5. 验证

运行原有测试或用例，确认算子已在 CPU 上执行。

## lambda name ≠ launch kernel name 常见案例

| DLC_CHECK_RESULT lambda name | 实际 launch kernel name |
|---|---|
| `custom_bmm_f32` | `custom_bmm_bf16_small` / `custom_bmm_bf16` |
| `custom_softmax` | `custom_softmax_bf16` |
| `custom_scaled_dot_product_efficient_attention` | `custom_scaled_dot_product_efficient_attention_bf16` |
| `custom_matmul_t_pingpong` | `custom_matmul_t_bf16_pingpong` |
| `custom_silu_tensor` | `custom_silu_tensor_bf16` |
| `custom_conv3d_bf16` | 可能实际 launch `custom_conv3d_bf16_weights_T` |

**规则**：始终修改 `DLC_CHECK_RESULT(xxx, ...)` 中第一个参数 `xxx` 对应的 dispatch 常量，不要根据 launch name 猜测 dispatch 常量名。

## 常见坑

1. **改错 dispatch 常量**：改了 launch kernel name 对应的常量，但实际 dispatch 使用的是不同的常量。
2. **忘记重新编译**：修改 `enabled_kernels.hpp` 后必须重新编译 PyTorch 才能生效。
3. **一次改多个算子**：单变量定位要求一次只改一个算子，否则无法确认哪个是边界。
4. **编译缓存**：`python setup.py develop` 可能不检测头文件变化，如果修改不生效，尝试 `ninja -j100` 或清理 build 目录。
5. **CPU fallback 语义**：CPU fallback 路径可能有不同的计算语义（如 SDPA 的 fallback 默认不是 CPU native SDPA），需要检查 fallback 实现。

## 验证方法

修改后，通过以下方式验证 fallback 生效：

1. 用 `DLC_SYN_DEBUG=1 DLC_SYN_VERBOSE=1` 确认该算子不再出现在 DLC launch trace 中。
2. 用精度对比确认 CPU 结果与预期一致。
3. 做单变量实验：改回 `DLC` 确认问题重现，改回 `CPU` 确认问题消失。

## 相关资料

- [CONTEXT.md](../CONTEXT.md) — DispatchType、DLC_CHECK_RESULT 定义
- [pytorch-dlc-backend/operator-integration-guide.md](../pytorch-dlc-backend/operator-integration-guide.md) — 算子接入和 dispatch 源码定位
- [precision-debugging/precision-debugging-overview.md](../precision-debugging/precision-debugging-overview.md) — dispatch fallback 在精度定位中的使用

## 来源

- `/work/plans/算子dispatch.md`
- 多个 RSThinker handoff 中的 dispatch 操作经验
