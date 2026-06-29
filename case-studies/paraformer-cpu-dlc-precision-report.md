# Case Study: Paraformer DLC/CPU 精度差异定位

> 本文档内容已合并至 [paraformer-qkv-linear-precision.md](paraformer-qkv-linear-precision.md)，请直接阅读该文档。本文保留为早期调查阶段记录。

## 问题现象

Paraformer 语音识别模型在 DLC Platform 上推理，DLC 生成文本与 CPU oracle 不一致：重复 token、替换 token。

## 背景与环境

- 模型：Paraformer (FunASR)
- 对比：CPU oracle vs DLC，997/997 tensors 对齐（无 shape mismatch）
- 工具：dump/compare scripts

## 定位路径

1. 用 compare report 对所有 tensor 做 CPU vs DLC 对比。
2. 找到第一个显著 divergent module：`encoder.encoders0.0.self_attn.linear_q_k_v` (Linear)。
3. 确认输入 tensor CPU/DLC 一致。
4. 确认真实 kernel 路径：`addmm_impl_dlc_` -> `mm_out_dlc`。
5. 确认问题在 float32 Linear/addmm/mm 路径，suspect 是 `custom_addmm*` kernel。
6. 建议后续：verify Linear input identity、构建单算子复现。

## 根因或当前结论

DLC float32 Linear/addmm/mm 路径在特定 shape 下与 CPU 存在差异。初步 suspect 是 `custom_addmm*` kernel 的 tiling 和精度行为。

## 验证方式

- dump 和 compare 过程可复现。
- 997/997 tensors 的对比报告确认了第一个显著差异。

## 可复用经验

1. 不要只看最终 loss 或输出，要从 encoder 入口逐 module 对比。
2. Paraformer SANM attention 路径是高精度风险区域。
3. 浮点差异可以通过 autocast 和 dtype 分析缩小范围。
4. `custom_nonzero_bool` 的 SMEM 日志是噪音，不是 root cause。

## 后续动作

- 从模型 dump 收敛到单算子复现（已完成，见 [paraformer-qkv-linear-precision.md](paraformer-qkv-linear-precision.md)）。
- 交付算子团队。

## 来源

- `/work/plan/dlc基础/Paraformer_DLC_CPU精度差异定位报告.md`
