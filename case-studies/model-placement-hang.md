# Case Study: RSThinker 模型放置卡住

## 问题现象

RSThinker 模型（10.29B bf16）在 DLC Platform 上执行 `model.to("dlc")` 时 hang 了 ~900s。后续无法稳定复现。

## 背景与环境

- **模型**：RSThinker，~10.29B param bf16 model
- **操作**：`model.to("dlc")`
- **正常时间**：~24s

## 定位路径

### 阶段 1：排除确定性原因

逐步排除：
- DLC 设备被占用：检查 `/dev/dlc*` 进程
- 模型参数分配量超出 HBM
- PyTorch/transformers 版本问题
- DLC Runtime/驱动已知 bug

### 阶段 2：增量测试

从最小到最大分配渐进：
1. `torch.arange(4).to("dlc")` — 正常
2. tiny/medium copy — 正常
3. incremental bf16 分配 up to 20 GiB — 正常
4. full model placement — ~24s 正常

### 阶段 3：结论

历史 hang 不是确定性的。最可能原因：残留 DLC process、首次初始化路径瞬态、或 DLC Runtime/驱动状态不正常。环境重置后问题消失。

## 根因或当前结论

非确定性的 DLC Runtime/驱动瞬态状态问题。不可复现。

## 可复用经验

1. **增量分配测试**：tiny → 20 GiB → full model，逐步确认每级分配正常。
2. **hang 排查优先做简单测试**：`torch.arange(4).to("dlc")` 先确认 DLC 基本可用。
3. **检查残留进程**：`lsof -t /dev/dlc*`。
4. **Step 04 因此分析被 unblock**。

## 来源

- `/work/RSThinker/dlc_model_placement_hang_analysis.md`
