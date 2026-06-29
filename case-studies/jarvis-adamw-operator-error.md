# Case Study: Jarvis AdamW 算子错误分析

## 问题现象

绕过 LayerNorm 后，Jarvis 训练在 `optimizer.step` 后出现参数 NaN。单点 NaN 出现在 `time_emb.lin2.weight[2733, 275]`。

## 背景与环境

- **模型**：Jarvis, spatial-parallel 6-card
- **workaround**：`JARVIS_SAFE_LAYER_NORM=1`（绕过 LayerNorm 问题）
- **检查点**：
  - forward: finite ✓
  - backward: finite ✓
  - grad sync: finite ✓
  - optimizer.step: NaN ← 第一个异常

## 定位路径

### 阶段 1：确认 AdamW 路径

默认 AdamW 在 DLC Platform 上实际走 foreach/multi-tensor path。单参数 replay 不一定复现。

### 阶段 2：Lazy Dump

用 Lazy Dump 捕捉真实现场（低扰动，只在检测到 NaN 后写完整 state）。

dump 演进：
1. 单参数 dump → 不完整
2. 完整参数组 dump → 可复现 partial
3. Lazy Dump → 完整复现

注意：重 dump 可能因同步/拷贝扰动掩盖问题。

### 阶段 3：pytorch_test 复现

两种复现模式：
1. **Align with Jarvis real after snapshot**：用 Jarvis 真实 dump 的 78 参数 snapshots 做基线验证。
2. **CPU/DLC both re-execute**：CPU 和 DLC 同时重新执行 AdamW，比较差异。

### 阶段 4：观察到的问题

在大矩阵中出现离散单点误差（单一位置 NaN 或异常更新）。

## 根因或当前结论

DLC AdamW foreach/multi-tensor 路径在默认配置下存在正确性 bug：
- 离散单点误差（不容易出现在小矩阵）
- 单参数 replay 不一定复现（需要完整参数组和 tensorlist 顺序）

## 验证方式

- Lazy Dump 捕捉到真实现场。
- pytorch_test 的两种模式均可复现。
- workaround `JARVIS_ADAMW_FUSED=1` 或 `JARVIS_ADAMW_FOREACH=0` 可防止 NaN。

## 可复用经验

1. **Foreach/Multi-Tensor Path 需要完整 tensorlist**：单参数 replay 可能不触发出错。
2. **Lazy Dump 对 optimizer 问题至关重要**：过重 dump 的同步/拷贝可能改变异步调度。
3. **保存完整参数组和 tensorlist 顺序**：对 optimizer bug 这是必要条件。
4. **两种复现模式各有用途**：align snapshot 验证现场正确，CPU/DLC both replay 验证算子行为。
5. **AdamW foreach 路径 vs 非 foreach 路径**：问题只在 foreach/multi-tensor 路径出现，单一 non-foreach 路径正常。

## 后续动作

- 交付算子团队：`manifest.pt`（78 参数 snapshots）、`test_adamw_default_repro.py`、分析报告。
- 长期：修复 DLC foreach/multi-tensor AdamW kernel。

## 来源

- `/work/plan/dlc基础/Jarvis_DLC_AdamW算子错误分析报告.md`
- `/work/plan/dlc基础/Jarvis_DLC问题定位到pytorch_test复现流程报告.md`
