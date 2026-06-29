# PyTorch DLC 原生测试框架指南

## 适用场景

- 为新接入的 PyTorch DLC Backend 算子添加测试。
- 理解 PyTorch 原生测试的结构和运行方式。
- 从模型问题收敛到 PyTorch 算子测试。

## 核心结论

PyTorch DLC 原生测试位于 `/work/pytorch/test/`，使用 PyTorch 的 `TestCase` 框架和 `@onlyDLC`、`@dtypes` 等装饰器参数化设备与类型。核心测试文件是 `test/dlc_ops/test_dlc_ops.py`。

这套测试与 pytorch_test Framework 的区别：这是 PyTorch 仓库内的测试，验证 ATen dispatch 到 DLC Custom Kernel 的完整性，而不是单独验证 kernel 行为。

## 新人心智模型

```
PyTorch API (torch.nn.functional.xxx)
  -> ATen dispatch
  -> PyTorch DLC Backend (aten/src/ATen/native/dlc/<op>.cxx/.cpp/.cc)
  -> KernelDesc
  -> DLCSynapse
  -> DLC Custom Kernel
```

测试要做的是：给定 PyTorch 接口调用，验证 CPU reference 和 DLC 输出的差异。

## 目录结构

```
/work/pytorch/test/
├── dlc_ops/
│   └── test_dlc_ops.py    # 主要 DLC 算子测试文件
├── test_dlc.py             # DLC 设备基础测试
└── ...
```

## 常用运行命令

```bash
# 运行全部 DLC 算子测试
python test/dlc_ops/test_dlc_ops.py

# 运行特定测试类
python test/dlc_ops/test_dlc_ops.py TestDlcOps

# 运行特定测试方法
python test/dlc_ops/test_dlc_ops.py TestDlcOps.test_add

# 带 pytest 参数
python -m pytest test/dlc_ops/test_dlc_ops.py -k "test_add" -v
```

## 测试文件基本结构

```python
import torch
from torch.testing._internal.common_utils import TestCase, run_tests
from torch.testing._internal.common_device_type import onlyDLC, dtypes

class TestDlcOps(TestCase):
    @onlyDLC
    @dtypes(torch.float32, torch.bfloat16)
    def test_op_name(self, device, dtype):
        # 构造输入
        x = torch.randn(3, 4, device=device, dtype=dtype)

        # CPU reference
        expected = x.abs()  # CPU 计算

        # DLC 执行
        actual = torch.dlc_abs(x)

        # 比较
        self.assertTrue(torch.allclose(actual, expected, atol=1e-5, rtol=1e-5))
```

## 关键装饰器

| 装饰器 | 含义 |
|--------|------|
| `@onlyDLC` | 仅在 DLC 设备上运行 |
| `@dtypes(torch.float32, torch.bfloat16)` | 参数化 dtype |
| `@parametrize("shape", [(3, 4), (1, 2, 3)])` | 参数化 shape |

## 编写 CPU Reference 的正确方式

CPU reference 必须能独立验证 DLC 输出：

```python
# 正确：用相同输入在 CPU 上重新计算
def reference_op(x):
    return x.abs()

# 错误：用 DLC 结果验证 DLC 结果
```

## Shape / Stride / Layout 测试

高风险场景需要特别测试：
- 非连续 tensor（transpose、slice 后）
- Channels-Last Strided Tensor
- 大 shape
- dim 参数的各种合法值（正数、负数、边界值）

## 算子接入后的测试覆盖清单

接入新算子后，至少覆盖：
1. forward 基本正确性（常用 shape、dtype）
2. backward（如果 support）
3. inplace 变体（如果有）
4. out 变体（如果有）
5. 非连续输入
6. 边界 shape（0-dim、1-dim、空 tensor 等）
7. 高风险 dtype（bf16、bool、int64）

## 从模型问题收敛到算子测试

1. 在模型中定位第一个异常 tensor（用 finiteness check 或 dump）。
2. 用 dump 脚本捕捉异常 tensor 的 input、output、shape、stride、dtype。
3. 在 PyTorch 测试中复现：用相同 shape/stride 的随机 tensor 先测试。
4. 如果可以复现，用 Model-Site Dump 的 .pt 文件做精确复现。
5. 收敛到最小可复现 case 后，提交为算子测试。

## 常见坑

1. **device 参数必须使用**：`device=device` 而不是硬编码 `device="dlc"`。
2. **dtype 习惯 bf16**：很多 bug 只在 bf16 下出现。
3. **SynShape 反向**：kernel 期望的维度顺序与 PyTorch 可能相反。
4. **输入一致性**：CPU reference 和 DLC 执行必须使用**完全相同**的输入 tensor。

## 相关资料

- [testing/dlc-kernel-test-framework-guide.md](dlc-kernel-test-framework-guide.md)
- [pytorch-dlc-backend/operator-integration-guide.md](../pytorch-dlc-backend/operator-integration-guide.md)
- [precision-debugging/model-site-dump-to-repro.md](../precision-debugging/model-site-dump-to-repro.md)

## 来源

- `/work/plan/dlc基础/torch的测试框架新人使用指南.md`
