# pytorch_test Framework 指南

## 适用场景

- 为 DLC Custom Kernel 编写 pytorch_test 测例。
- 从真实 Model-Site Dump 构建 pytorch_test 复现。
- 理解 Static Shape Test、Dynamic Fuzz Test、Variant 和 SynShape 的概念。

## 核心结论

pytorch_test Framework 是 DLC_Custom_Kernel Repository 中的 PyTorch 测试框架，位于 `/work/DLC_Custom_Kernel/pytorch_test/`。它用同一组输入在 CPU 和 DLC 上执行并比较输出，支持 Static Shape Test、Dynamic Fuzz Test 和 Model-Site Dump replay。

## 框架位置和结构

```
/work/DLC_Custom_Kernel/pytorch_test/
├── test_kernel/          # 按模块组织的测试文件
│   ├── copy/
│   ├── embed/
│   ├── norm/
│   ├── reduce/
│   ├── rsthinker/        # RSThinker 相关 repro 测试
│   └── ...
├── README.md
├── TEST_GUIDE.md
├── ARCHITECTURE.md
├── main.py               # 当前测试运行入口之一
├── web_server.py         # 测试结果/分析 Web 服务
├── query_test_result.py  # 查询测试结果
├── frontend/             # Web 前端
└── ...
```

## 常用运行命令

```bash
# 运行全部测试
python pytorch_test/run.py

# 运行特定 variant
python pytorch_test/run.py <variant_name>

# 运行特定文件
python -m pytest pytorch_test/test_kernel/matmul/test_addmm.py -v

# 运行特定测试方法
python -m pytest pytorch_test/test_kernel/matmul/test_addmm.py::TestAddmm::test_static -v
```

## 测试类基本结构

```python
from TestBase import TestBase
from test_exec import test_exec

class TestMyOp(TestBase):
    @test_exec(for_variant=["my_op_bf16"])
    def my_op_static_shape_test(self, ctx):
        # 创建输入
        input1 = ctx.get_random_tensor(shape=(2, 3, 4), dtype=torch.bfloat16)
        input2 = ctx.get_random_tensor(shape=(2, 4, 5), dtype=torch.bfloat16)

        # 执行
        result = ctx.dlc.exec("my_kernel", input1, input2)

        # CPU reference（框架自动执行）
        # 框架自动比较 CPU 和 DLC 结果
```

## Variant

Variant 是全局唯一的测试选择器，通过 `@test_exec(for_variant=[...])` 注册。Variant 不一定是 runtime launch kernel name。

**规则**：
- Variant 必须全局唯一。
- 不要假设 variant 名字等于 kernel launch name。

## SynShape vs PyTorch Shape

SynShape 是 kernel/测试侧的 shape 表示，维度顺序通常与 PyTorch shape **相反**：

```
SynShape([96, 96, 768, 1])
对应 PyTorch shape (1, 768, 96, 96)
```

## 常用 ctx 方法

| 方法 | 用途 |
|------|------|
| `ctx.get_random_tensor(shape, dtype)` | 创建随机输入 tensor |
| `ctx.dlc.exec("kernel_name", *args)` | 在 DLC 上执行 |
| `ctx.get_static_input("name")` | 获取静态注册的输入 |

## 读取测试结果

- `Total errors: a/b` — 不满足容差的元素数 / 总元素数。
- 如果比较的是 `torch.isfinite(result).float()`，错误数表示 Finite Mask Mismatch。
- optimizer/foreach/multi-tensor 问题可能依赖完整 tensor list，单参数 replay 不一定复现。

## Static Shape Test vs Dynamic Fuzz Test

| 类型 | 用途 | 何时使用 |
|------|------|---------|
| Static Shape Test | 固定 shape 回归 | 确认已知 bug 不会回归，或复现特定 shape 的已知问题 |
| Dynamic Fuzz Test | 随机输入/shape | 发现边界、layout、dtype 问题 |

## 从 Model-Site Dump 构建 pytorch_test 复现

### 标准流程

1. 用 `dump_transformers_multimodal_precision.py` 或类似脚本在模型 failure 点保存 .pt dump。
2. 在 pytorch_test 中加载 .pt 文件：

```python
import torch
tensors = torch.load("/path/to/dump.pt", map_location="cpu")
input_tensor = tensors["input"]
weight = tensors["weight"]
```

3. 编写测试，用 dump 中的 tensor 作为输入。
4. 先验证 CPU replay 与 dump 中 saved CPU 一致，DLC replay 与 dump 中 saved DLC 一致。
5. 再比较 CPU vs DLC，确认可复现。
6. 尝试收敛到最小可复现 case（小 shape、随机输入、code-only）。

### 收敛阶段

| 阶段 | 输入 | 目标 |
|------|------|------|
| Phase 1 | 真实 .pt dump | 确认问题可复现 |
| Phase 2 | 真实 dump 缩小到最小 shape | 交付算子团队的最小 case |
| Phase 3 | 全随机合成数据 | code-only repro，无外部依赖 |

### AdamW foreach/multi-tensor 复现

optimizer/foreach/multi-tensor 问题特殊之处：
- 需要保留完整参数组和 tensor list 顺序。
- 两种复现模式：
  1. Align with real after snapshot：用 Jarvis 真实 dump 做基线验证。
  2. CPU/DLC both re-execute：CPU 和 DLC 同时重新执行。

## 常见环境变量

| 变量 | 用途 |
|------|------|
| `DLC_SYN_DEBUG=1` | 启用 Synapse 调试输出 |
| `DLC_SYN_VERBOSE=1` | 详细 Synapse 日志 |
| `DLC_SYN_BLOCKING=1` | 阻塞执行模式 |
| `DLC_SYN_USE_SIM=1` | 使用模拟器 |

## 常见坑

1. **Variant 冲突**：不同测试文件使用了相同的 variant 名。
2. **SynShape 反向**：直接传 PyTorch shape 给 kernel 而不做维度反转。
3. **输入不一致**：CPU 和 DLC 两侧输入 tensor 不是完全相同的对象。
4. **.pt 加载位置**：dump 的 .pt 文件路径不对或格式不兼容。
5. **Full dump 遮盖 bug**：过重的 dump（同步、拷贝）可能改变异步执行顺序，掩盖异步 bug。必要时使用 Lazy Dump。

## 相关资料

- [testing/pytorch-test-framework-guide.md](pytorch-test-framework-guide.md)
- [precision-debugging/model-site-dump-to-repro.md](../precision-debugging/model-site-dump-to-repro.md)
- [case-studies/](../case-studies/)

## 来源

- `/work/plan/dlc基础/DLC_kernel测试框架新人使用指南.md`
