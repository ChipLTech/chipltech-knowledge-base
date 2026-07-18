# Case Study: RSThinker 端到端功能精度基线

## 问题现象

RSThinker Model Under Adaptation 的 7 个 P0/P1 算子已确认属于预期 DLC Precision Difference，但仍需要回答模型级问题：当前算子精度水平下，PyTorch DLC Backend Adaptation Path 的端到端输出是否功能可接受。

## 背景与环境

- **模型**：RSThinker (`minglanga/RSThinker`, GLM-4.1V-9B-Base)
- **模型路径**：`/mnt/jfs/models/RSThinker`
- **样本**：`SC_AID-0`
- **decode budget**：5 tokens
- **CPU Reference**：同一套 processor inputs，device=`cpu`
- **DLC Platform**：device=`dlc`
- **DLC Runtime 约束**：`DLC_SYN_BLOCKING=1`
- **判定指标**：`cosine_similarity`、`allclose(atol=1e-4, rtol=1e-4)`、finite-mask、token top-1 match rate

## 定位路径

### 1. 建立 CPU Reference dump

使用 dump harness 在 CPU 上运行 5-token prefill + decode，并打开模块 hook：

```bash
RUN_STAMP=20260702-1402-e2e-baseline DEVICE=cpu MAX_TOKENS=5 SAMPLE_INDEX=0 MODULE_HOOK_REGEX='Glm4v|Linear|Embedding|RMSNorm|LayerNorm|Attention|Mlp|MLP|DecoderLayer|Vision|PatchMerger|Merger' MODULE_HOOK_MAX=1000 bash scripts/run_rsthinker_cuda_precision_dump.sh
```

注意：脚本名保留历史上的 `cuda_precision_dump`，本次通过 `DEVICE=cpu` 明确使用 CPU Reference。

### 2. 建立 DLC dump

DLC run 复用 CPU dump 的 `tensors/inputs`，避免多模态 processor 输入差异污染模型体精度判断：

```bash
RUN_STAMP=20260702-1402-e2e-baseline MAX_TOKENS=5 SAMPLE_INDEX=0 INPUT_TENSORS_DIR=/work/RSThinker/precision_runs/20260702-1402-e2e-baseline-rsthinker-cuda-reference-geocot-sc-aid-1x1/cuda_reference_dump/tensors/inputs MODULE_HOOK_REGEX='Glm4v|Linear|Embedding|RMSNorm|LayerNorm|Attention|Mlp|MLP|DecoderLayer|Vision|PatchMerger|Merger' MODULE_HOOK_MAX=1000 DLC_SYN_BLOCKING=1 bash scripts/run_rsthinker_dlc_precision_dump.sh
```

### 3. 逐层比较

比较脚本使用 Owner 团队接受的 `1e-4/1e-4` 阈值，并记录 cosine、finite mask、abs metrics：

```bash
python3 scripts/compare_rsthinker_precision_dumps.py --run_dir /work/RSThinker/precision_runs/20260702-1402-e2e-baseline-compare --cuda_dump /work/RSThinker/precision_runs/20260702-1402-e2e-baseline-rsthinker-cuda-reference-geocot-sc-aid-1x1/cuda_reference_dump --dlc_dump /work/RSThinker/precision_runs/20260702-1402-e2e-baseline-rsthinker-dlc-dump-geocot-sc-aid-1x1/dlc_dump --output_json /work/RSThinker/precision_runs/20260702-1402-e2e-baseline-precision_analysis_report.json --output_md /work/RSThinker/precision_runs/20260702-1402-e2e-baseline-precision_analysis_report.md --top_k 25 --allclose_atol 1e-4 --allclose_rtol 1e-4
```

## 根因或当前结论

当前 5-token 单样本基线判定为：功能精度可接受。

关键事实：

- Processor inputs exact/allclose：`input_ids`、`attention_mask`、`pixel_values`、`image_grid_thw` 全部一致。
- Finite-mask mismatches：`0`。
- Shape mismatches：`0`。
- 模块 hook 对比数量：`778`。
- 非零 drift 模块数量：`450`。
- 非零 drift 最低 cosine：`0.9992804678516214`。
- 非零 drift 模块中 cosine `< 0.99` 数量：`0`。
- 5 个 decode step 的 top-1 token match rate：`100%`。
- Generated IDs match：`true`。
- Decoded text match：`true`。

第一处非零 drift：

- Module：`model.visual.blocks.20.attn.proj`
- Class：`Linear`
- Cosine：`0.9999999867945111`
- abs_max：`0.0078125`
- abs_mean：`9.381536187902384e-07`

最差模块 cosine：

- Module：`model.language_model.layers.39.post_mlp_layernorm`
- Class：`Glm4vRMSNorm`
- Cosine：`0.9992804678516214`
- abs_max：`11.5`
- abs_mean：`0.11335347592830658`

## 验证方式

- Compare report：`/work/RSThinker/precision_runs/20260702-1402-e2e-baseline-precision_analysis_report.md`
- Functional acceptance baseline：`/work/RSThinker/precision_runs/20260702-1402-e2e-functional-acceptance-baseline.md`
- `python3 -m py_compile scripts/compare_rsthinker_precision_dumps.py`

## 可复用经验

1. **脚本名不等于 oracle 类型**：历史脚本名可能叫 `cuda_precision_dump`，但模型级精度 oracle 应通过 `DEVICE=cpu` 明确使用 CPU Reference。
2. **DLC run 应 replay CPU processor inputs**：多模态 processor 输入差异会被视觉 encoder 放大；端到端模型体精度比较必须先锁定 inputs。
3. **不要用 bf16 mismatch 数做功能判定**：dense logits 和 late hidden states 可出现大量 exact mismatch，但 token top-1 和 cosine 仍稳定。
4. **全零 tensor 的 cosine 需要特殊处理**：两个全零 tensor 应视作 exact/cosine=1，否则会产生误报。
5. **CPU fallback 只用于定位**：本次没有切换 dispatch，也没有把 CPU fallback 当生产缓解。

## 后续动作

- 扩展到更多 RSTester 样本，建立 token match rate 和 task score 分布。
- 若后续样本出现 cosine 断崖或 token mismatch，再按单变量原则切换一个 dispatch 进行定位。
- 进入 vLLM DLC Serving Path 前，应先保留本 PyTorch DLC Backend Adaptation Path 基线作为回归 oracle。

## 来源

- `/tmp/kilo_handoff_rsthinker_e2e_validation_20260702_134745.md`
- `/work/RSThinker/precision_runs/20260702-1402-e2e-functional-acceptance-baseline.md`
- `/work/RSThinker/precision_runs/20260702-1402-e2e-baseline-precision_analysis_report.json`
