# Case Study: RSThinker 视觉 Attention 边界定位

## 问题现象

RSThinker 多模态模型在 DLC Platform 上运行时，SC_AID-1 样本的 CPU/DLC generate tokens 在 token 2 发生分叉。视觉 encoder 输出差异从 block0 通过 23 个 visual blocks 逐层放大，最终导致 lm_head top-1 翻转。

## 背景与环境

- **模型**：RSThinker (GLM-4.1V-9B-Base + LoRA adapter)
- **任务**：遥感图像场景分类 smoke run
- **对比**：CPU oracle vs DLC
- **定位前已知**：processor inputs 一致，pixel_values、input_ids、attention_mask、image_grid_thw 均 exact

## 定位路径

### 阶段 1：从端到端收敛到第一个 divergent module

1-token CPU vs DLC precision dump 对比报告：
```
model.visual.patch_embed: 0               (exact)
model.visual.post_conv_layernorm: 0        (exact, after CPU fallback)
model.visual.embeddings: 0                 (exact, after CPU fallback)
model.visual.blocks.0.norm1: 0             (exact)
model.visual.blocks.0.attn.qkv: abs_max 0.015625  ← first drift
```

### 阶段 2：qkv Linear 边界

真实 tensor replay：
- `F.linear(norm1, qkv.weight)` on CPU vs DLC: abs_max=0.015625
- CPU saved qkv == CPU replay
- DLC saved qkv == DLC replay
- Runtime kernel: `custom_matmul_t_bf16_pingpong`

Dispatch fallback: `custom_matmul_t_pingpong = CPU` → qkv exact。

### 阶段 3：RoPE（旋转位置编码）

qkv exact 后，下一个 drift：attention score matmul 之前的 RoPE。
- DLC trace: `custom_cos_f32` + `custom_sin_tensor`
- Dispatch fallback: `custom_cos_f32 = CPU` + `custom_sin_tensor = CPU` → RoPE exact。

### 阶段 4：Attention 分解

attention 分为三个独立边界：

**4a. Score matmul (bmm)**
- Runtime: `custom_bmm_f32` lambda, launch `custom_bmm_bf16`
- Fallback 后: score exact

**4b. Softmax**
- Runtime: `custom_softmax` lambda, launch `custom_softmax_bf16`
- Fallback 后: softmax weights exact

**4c. SDPA (scaled dot product attention)**
- Runtime: `custom_scaled_dot_product_efficient_attention` lambda
- 默认 CPU fallback 使用的不是 CPU native SDPA（语义错误）
- 需要 patch fallback 为 `at::scaled_dot_product_attention(...)` 并处理 GQA
- 修复后: attention 全部 exact

### 阶段 5：下一个边界 — SiLU

attention exact 后：
```
model.visual.blocks.0.mlp.gate_proj: exact
model.visual.blocks.0.mlp.silu: abs_max 0.03125  ← new drift
```
- Runtime: `custom_silu_tensor_bf16`
- Dispatch: `custom_silu_tensor = CPU` → SiLU exact → block0 所有 module exact。

## 最小边界

13 个 CPU fallback 达成 block0 完全 exact：
```
custom_conv3d_bf16, custom_conv3d_bf16_weights_T,
custom_mean_dim, custom_rsqrt,
custom_foreach_div, custom_grid_sample_2d,
custom_matmul_t_pingpong, custom_cos_f32, custom_sin_tensor,
custom_bmm_f32, custom_softmax,
custom_scaled_dot_product_efficient_attention, custom_silu_tensor
```

## 根因或当前结论

block0 所有 visual modules 在以上 fallback 后 exact，问题已传递到 block1+。后续需要扩展 dump 到更多 visual blocks 继续定位。

## 验证方式

1. **每次 dispatch 修改后**：跑 attention/MLP replay 确认单个边界归零。
2. **关键 checkpoint**：1-token dump 确认全部 module exact。
3. **50-token smoke**：确认 token 输出变化（但非完全对齐）。

## 可复用经验

1. **dispatch lambda name ≠ launch kernel name**：`custom_bmm_f32` 的 launch 是 `custom_bmm_bf16`，改 dispatch 必须用 lambda name。
2. **CPU fallback 语义检查**：`custom_scaled_dot_product_efficient_attention_cpu` 默认 fallback 不是 CPU native SDPA，需要检查所有 fallback 实现。
3. **单变量定位**：一次只改一个 dispatch，确认边界后再继续。
4. **1-token dump before 50-token**：先做 cheap 验证，再跑长时间 smoke。
5. **replay 先于模型 dump**：每次 dispatch 修改后先跑算子 replay 快速验证。

## 当前状态和后续动作

- block0: exact ✓
- block1+: 待定位（50-token tokens 仍然分叉）
- 下一步：扩展 dump 到 block1+，用 coarse bisection 找到下一个 divergent block。

## 来源

- `/tmp/rsthinker_attention_bmm_softmax_sdpa_handoff_20260625_0919.md`
- `/tmp/rsthinker_visual_mlp_silu_handoff_20260626_0158.md`
- `/tmp/rsthinker_next_session_prompt_after_block0_exact_20260626_0601.md`
- `/work/RSThinker/docs/rsthinker_visual_mlp_drift_diagnosis_experience.md`
