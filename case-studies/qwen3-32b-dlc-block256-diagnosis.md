# Qwen3-32B DLC Platform Block-256 适配诊断

## 问题现象

- Qwen3-32B, BF16, TP4, `--block-size 256`, `DLC_SYN_COPY_ASYNC=O0`, Real DLC Hardware 物理设备 4-7
- `2 + 2 = ` (trailing space), `temperature=0`, `max_tokens=1` → 10/10 稳定输出 `5` (token ID 19)，而非期望的 `4` (token ID 20)
- Sampler 忠实选择模型 top-1 (greedy decoder 已验证)
- 错误在 sampler 上游——model forward/logits 层面
- 无乱码、无 random 波动、无 server crash

## 背景与环境

### 固定资产

| 资产 | 身份 |
|---|---|
| vLLM | `a208f41eee15d15b0da619ded9384fda5efd2e7f` |
| vLLM-CL | `ce14cbc726c73df65a3ec6e970da523c6ed22ea8` |
| Native extension | SHA-256 `66212445c4dfc03716c61818db98e8fe126d570a99c4abb5d10db5ed48a105da` |
| Base image | `hangzhou-harbor.infraai.top/ci/dlc_base@sha256:0a2dec9c34e87530ad77d9c25967e2e2937fd4c00d6ca648fa1eb8ea7bd42735` |
| 权威模型 | `/mnt/jfs/models/Qwen3-32B`, 17 shards, BF16 |

### Qwen3-32B TP4 几何

- 全局 Q heads 64, local 16, explicit `head_dim=128`, local Q width 2048
- 全局 KV heads 8, local 2, local K width 256, local V width 256
- Local packed QKV width 2560
- 28 layers → 实际 64 layers (Qwen3-32B)
- Vocab size 151936, local LM-head rows 37984

### 关键已知事实 (来自前人实验)

- Extension package visibility 已修复
- Tokenizer 映射正常
- Greedy sampler 忠实选择模型 top-1
- Fused-QKV 不是唯一根因
- Async scheduling / prefix cache / chunked prefill 不是主要根因
- `DLC_SYN_COPY_ASYNC=O1` 在 fresh C1b logical0 挂起; `O0` 稳定
- 即使 `USE_CPU_FOR_COLLECT_OP=true` (全部 collective CPU fallback)，仍 13/13 输出 `5`

## 定位路径 (ordered seam diagnostic)

### H001: Loaded-weight integrity → FALSIFIED

逐 rank 比对 Layer 0 全部 key tensor 与 safetensors 权威切片: Q/K/V packed segments、gate/up packed segments、o_proj、down_proj、q_norm/k_norm、final RMSNorm、embedding、lm_head。

**结果**: 四 rank 全部 BF16 exact、finite mask 相同、diff=0、SHA-256 一致。Loader 无错误。

### H002: Fused QKV/QK RMSNorm/RoPE → FALSIFIED (synthetic)

对 deterministic 输入的 DLC fused `dlc_fused_rms_norm_qkv_rotary_emb` 与显式 `[2048,256,256]` split + float32 per-head RMSNorm + native RoPE 比对。

**结果**: Q/K/V 全部 BF16 exact。但这是 code-only synthetic 输入，不是真实 layer-0 输出。

**关键检查**: 所有实现正确使用 `head_dim=128`，未误用 `hidden_size / num_attention_heads = 80`。

### H003: RMSNorm → FALSIFIED (within tolerance)

Synthetic 输入下比对 DLC RMSNorm 与 CPU float32 reference。

**结果**: Fused-add updated residual exact。Normalized diff ≤ 3.05e-05。无 catastrophic drift。

### H004: Rank-local GEMMs → FALSIFIED (within tolerance)

用 `F.linear` 比对 DLC BF16 linear 与 CPU float32→BF16 round: lm_head、Layer 0 QKV、gate/up、o_proj、down_proj。

**结果**: 全部 cosine > 0.99999，max_abs_diff ≤ 3.9e-04。无 material divergence。

### H005: TP collectives → FIRST DIVERGENCE FOUND

逐一比对 CPU rank-order sum/concat 与 DLCCL all-reduce/all-gather:

- **LM-head all-gather: BF16 exact, diff=0**
- **o_proj all-reduce: max_abs_diff 0.015625** ← 第一个数值分歧
- SiluAndMul: exact
- Embedding boundary: exact

**后续**: 构造了 code-only repro (不依赖模型权重或外部 dump 的固定 seed 最小复现):
- `cpu_replay == saved_cpu` ✓
- `dlc_replay == saved_dlc` ✓
- Owner: read-only DLCCL/native collective binary
- API: `torch.distributed all_reduce backend=dlccl`, shape `[5120]`, dtype bf16

### H007: CPU all-reduce diagnostic probe → FALSIFIED

将 `tensor_model_parallel_all_reduce` 全部路由到 CPU `torch.distributed.all_reduce` (monkey-patch overlay)。

**结果**: 5/5 仍输出 `5` → DLCCL all-reduce 不是唯一根因。

### H008: Real model forward Model-Site Dump → 与 synthetic 一致

用真实 embedding 输出作为 Layer 0 输入，逐模块 DLC output vs CPU float32 reference。

**结果**: Rank 0 diff ≤ 0.002 per-op，与 H002-H004 一致。无 catastrophic single-op divergence。

### Qwen3-1.7B 对比验证

在相同环境 (TP4, block-256, O0) 下:

| 测试 | Qwen3-32B (64层) | Qwen3-1.7B (28层) |
|---|---|---|
| `2 + 2 = ` 算术 | `5` (错误) | `4` (正确) |
| France 短回答 | 正确连贯 | 重复循环 (模型容量限制) |
| France 长回答 | 正确连贯 | 完全退化 (无限循环) |
| 乱码 | 无 | 无 |

**结论**: DLC 平台在 Qwen3-1.7B (28层) 上可正确完成算术，证明不是平台全局故障。32B 的错误与模型深度强相关。

## 根因或当前结论

当前证据确认一个独立问题，并支持一个尚未证实的模型深度相关假设：

### 第一问题 (code-only repro 已存在): DLCCL o_proj all-reduce 数值偏差

- max_abs_diff 0.015625 (BF16)
- Code-only repro 已交付
- 修复条件: DLCCL/native collective binary 编译或替换授权

### 待验证假设: 64 层 BF16 累积误差

- H001-H004、H008 均未发现单个 op 的 catastrophic divergence
- 每个 op diff ≤ 0.002，经 64 层叠加后改变 final logits 的 top-1 排序
- Qwen3-1.7B (28层) 在相同平台下正确，佐证“层数相关”假说，但尚不足以证明因果关系

## 验证方式

- 全部实验证据保存在 campaign artifact root (8 个 epoch，每个含 raw JSON/SMI/hashes)
- 假设 ledger 维护了 H001-H008 的完整 falsification/support 证据链
- Code-only repro 不依赖 dump 文件、模型权重或 safetensors

## 可复用经验

### 诊断方法论

1. **Ordered seam diagnostic**: loaded weights → fused ops → RMSNorm → GEMMs → collectives → real forward dump。每层只变一个变量。
2. 找到第一个 true divergence 后停下游检查。先做 replay equivalence 验证 (CPU replay = saved CPU, DLC replay = saved DLC)，再逐步缩小到 code-only repro。
3. Synthetic input 可证伪单算子假设，但不能替代真实模型中间张量的 Model-Site Dump。
4. Cross-model comparison (32B vs 1.7B) 是区分"平台全局故障"与"模型规模相关"的关键技巧。

### 容器环境

1. Docker run 必需参数: `--runtime chipltech --privileged --pid host --ipc host --shm-size 50g`
2. 必需 mount: `/dev`, `/sys`, `/run`, `/lib/modules`, `/var/log`, workspace (`/workspace`)、site-packages、模型目录
3. `-e PYTHONPATH` 在 `docker exec` 中应放在 `bash -lc` 内部 export，不要用 `-e` flag (会覆盖容器默认环境)
4. sealed site-packages 的 `.pth` editable install 不会生效 (PYTHONPATH 添加的目录不处理 `.pth`)；需直接在 PYTHONPATH 中添加实际源码路径
5. torchvision 依赖在 vLLM-CL 的 import chain 中是必须的 (transformers → image_utils → torchvision.transforms.InterpolationMode)

### Hermes 可靠性

- Hermes `-z` oneshot 模式在 DLC 环境下 tool-calling 不可靠 (5/5 次 invocation 无工具执行)
- 建议: 对关键 Real DLC Hardware 实验，主 Agent 直接通过 `docker exec` 驱动容器，而非委托 Hermes

### 关键 Python 能力边界

- vLLM worker extension 中可用 `self.get_model()` 但不是总能 get_tokenizer
- VocabParallelEmbedding 在非 rank-0 上对 out-of-range token 返回 zero (需 all-reduce)
- DLC tensor 直接索引 `tensor[large_index]` 可能报 `index out of range in self`，需注意 local vocab 分片范围

## 剩余的未验证项

- 本案例只在 DLC Chip 上测试 (未在 TYD Chip 上验证)
- 仅在 `DLC_SYN_COPY_ASYNC=O0` 下测试
- 仅 block-size 256 测试
- Formal benchmark 未运行 (因 functional gate 未通过)
- DLCCL all-reduce 修复后回归未执行

## 相关资料

- Campaign artifact root: `/home/xuansun/modelzoo-image-validation-artifacts/2026-07-28/qwen3-32b-adaptation-campaign-20260728T151421+0800/`
- Terminal state doc: `/home/xuansun/rel/Qwen3-32B-DLC-block256-campaign-terminal-state-20260728.md`
- 完整流程记录: `/home/xuansun/rel/Qwen3-32B-DLC-block256-完整诊断记录-20260728.md`
- O0 baseline: `/home/xuansun/modelzoo-image-validation-artifacts/2026-07-28/qwen3-32b-tp4-copy-async-o0/`
- 交接合同: `/tmp/kilo/qwen3-32b-dlc-block256-next-session-handoff.md`
- [precision-debugging/model-site-dump-to-repro.md](../precision-debugging/model-site-dump-to-repro.md)
- [vllm-cl/model-adaptation-and-main-to-main-decisions.md](../vllm-cl/model-adaptation-and-main-to-main-decisions.md)
- [precision-debugging/token-divergence-and-moe-contract-debugging.md](../precision-debugging/token-divergence-and-moe-contract-debugging.md)

## 来源

- 2026-07-28 Qwen3-32B DLC Block-256 适配 campaign (H001-H008)
- O0 baseline run `/home/xuansun/modelzoo-image-validation-artifacts/2026-07-28/qwen3-32b-tp4-copy-async-o0/`
- Qwen3-1.7B cross-model comparison (same environment)
