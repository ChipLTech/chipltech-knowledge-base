# dispatch key 与 runtime kernel 名映射

## 适用场景

- 需要确认 `enabled_kernels.hpp` 改动控制的到底是什么。
- 需要理解 `DLC_CHECK_RESULT(...)` 的 dispatch key 和 `syn_*.ansi` 中 runtime kernel 名的关系。
- 遇到“header 看起来改了，但 runtime log 仍然 launch DLC kernel”的情况。

## 核心结论

`enabled_kernels.hpp` 控制的是 **PyTorch DLC Backend 插入点的 dispatch key**，不是 runtime log 中最终 launch 的全部 kernel 名。一个 PyTorch op 进入 DLC Backend 后，通常会经历：

```text
PyTorch op
  -> native_functions.yaml DLC dispatch
  -> /work/pytorch/aten/src/ATen/native/dlc/*.cpp 插入函数
  -> DLC_CHECK_RESULT(dispatch_key, dlc_out)
  -> enabled_kernels.hpp 根据 dispatch key 判断 CPU / DLC / BOTH
  -> lambda 内部选择一个或多个实际 runtime kernel 名
  -> syn_*.ansi 中出现 launch custom_xxx
```

因此：

- header 里的名字不一定等于 runtime launch 名
- runtime launch 名也不一定等于 pytorch_test variant 名
- 只 grep header 不能证明当前 runtime 已经切到 CPU fallback

## 四层名字不要混用

| 层级 | 例子 | 含义 |
|---|---|---|
| PyTorch op | `torch.bmm`、`F.linear`、`F.layer_norm` | Python/ATen 语义入口 |
| Backend 插入函数 | `bmm_out_dlc`、`mm_out_dlc`、`layer_norm_dlc` | 进入 DLC 后端后的 C++ 实现 |
| dispatch key / lambda 名 | `custom_bmm_f32`、`custom_matmul_t_pingpong`、`custom_layer_norm_f32` | `DLC_CHECK_RESULT(...)` 使用的开关名 |
| runtime kernel launch 名 | `custom_bmm_bf16`、`custom_matmul_st_4_rhsT_bw`、`custom_layer_norm_bf16` | 最终 `k.launch(...)` 的名字 |

## 常见例子

### `torch.bmm`

- PyTorch op：`torch.bmm`
- Backend 插入函数：`bmm_out_dlc`
- dispatch key：`custom_bmm_f32`
- 可能的 runtime kernel：
  - `custom_bmm_f32`
  - `custom_bmm_bf16`
  - `custom_bmm_bf16_small`

结论：`custom_bmm_f32 = CPU` 控制的是整个插入点，不是只控制 log 里名字等于 `custom_bmm_f32` 的 launch。

### `F.linear` / `at::mm`

- PyTorch op：`F.linear` / `at::mm`
- Backend 插入函数：`mm_out_dlc`
- dispatch key：`custom_matmul_t_pingpong`
- 可能的 runtime kernel：
  - `custom_matmul_st_1_rhsT_bw`
  - `custom_matmul_st_2_6_rhsT_bw`
  - `custom_matmul_st_4_rhsT_bw`
  - `custom_matmul_st_8_rhsT_bw`
  - `custom_matmul_t_pingpong`
  - `custom_matmul_bf16_rhsT_bw`
  - `custom_matmul_t_bf16_pingpong`

结论：看到 `launch custom_matmul_st_4_rhsT_bw`，不代表 dispatch key 叫这个；真正受 hpp 控制的可能仍是 `custom_matmul_t_pingpong`。

### `F.layer_norm`

- PyTorch op：`F.layer_norm`
- Backend 插入函数：`layer_norm_dlc`
- dispatch key：`custom_layer_norm_f32`
- 可能的 runtime kernel：
  - `custom_layer_norm_f32`
  - `custom_layer_norm_bf16`

结论：bf16 输入时 log 里出现 `launch custom_layer_norm_bf16`，不代表应该去改 `custom_layer_norm_bf16 = CPU`；真正应先核对的是 `DLC_CHECK_RESULT(custom_layer_norm_f32, ...)`。

## 如何可靠验证 dispatch 是否生效

### 不可靠方法

```bash
grep custom_bmm_f32 /usr/local/chipltech/synapse/include/enabled_kernels.hpp
```

这只能说明 header 文本当前长什么样，不能证明 Python runtime 已经用这份 header 重新编译过。

### 可靠方法

1. 在 `/work/pytorch/aten/src/ATen/native/dlc/` 中确认 `DLC_CHECK_RESULT(...)` 的 dispatch key
2. 修改 `/usr/local/chipltech/synapse/include/enabled_kernels.hpp`
3. 重编 `/work/pytorch`
4. 打开 Synapse log 环境变量后复跑最小用例
5. 同时看：
   - `syn_*.ansi`
   - `*_kernels.txt`

判断标准：

| 现象 | 说明 |
|---|---|
| 出现 `is redispatched to cpu` | dispatch key 真正走 CPU fallback |
| 仍出现 `launch custom_xxx on dlc:0` | runtime 仍在走 DLC，改动未生效、未 rebuild 或命中其他路径 |
| 出现不同的 runtime kernel 名 | 先回到 `DLC_CHECK_RESULT(...)`，不要按 launch 名猜 dispatch key |

## 常见坑

1. **把 runtime kernel 名当 dispatch key 改**：最常见误判。
2. **改完 header 不重编 PyTorch**：这不算验证。
3. **只看 grep 结果，不看 runtime log**：无法确认真实路径。
4. **把 pytorch_test variant 当 runtime kernel 名**：两者不保证一致。

## 相关资料

- [operator-dispatch/enabled-kernels-dispatch.md](enabled-kernels-dispatch.md)
- [debugging-workflows/synapse-log-and-kernel-summary-workflow.md](../debugging-workflows/synapse-log-and-kernel-summary-workflow.md)
- [testing/pytorch-test-replay-from-synapse-log.md](../testing/pytorch-test-replay-from-synapse-log.md)

## 来源

- `/work/plans/PyTorch_DLC算子插入_dispatch与runtime_kernel映射说明.md`
