# 安装后的 Runtime Smoke

## 适用场景

- PyTorch 2.5.0 wheel 已经构建并重装完成，需要确认环境是否真的可用。
- 本地 `vllm` 或 `vllm-dlc` 刚做完 editable install，需要一个短路径判断它们是否健康。
- 想区分“包已经装上”与“DLC Platform 运行时真的健康”这两个不同问题。

## 核心结论

1. post-install smoke 必须在源码树外执行，否则 `import torch` 很容易被源码树污染。
2. 最小 smoke 要同时覆盖版本、NumPy bridge、DLC Platform 可见性，以及可选的 `vllm` / `vllm-dlc` import。
3. 如果 smoke 失败，先按失败层级回退，不要直接继续 patch 上层仓库。

## 操作步骤

### 1. 切到源码树外目录

推荐使用 `/tmp` 或其他明确不在 PyTorch、`vllm`、`vllm-dlc` 源码树内的目录。

原因：站在源码树里 `import torch`，可能命中源码目录，出现假阴性，误判为 wheel 安装失败。

### 2. 运行最小 PyTorch smoke

```bash
python3 -c "import torch; print(torch.__version__)"
python3 -c "import torch; print(torch.tensor([0.1], dtype=torch.float32).numpy())"
python3 -c "import torch; print(torch.backends.dlc.is_available() if hasattr(torch.backends, 'dlc') else 'no torch.backends.dlc')"
```

重点不是只看 `import torch` 成功，而是确认：

- 版本是目标版本
- NumPy bridge 正常
- DLC Platform 路线可见

### 3. 如本次涉及 `vllm`，再做可选 smoke

```bash
python3 -c "import vllm; print(vllm.__version__)"
python3 -c "import vllm_dlc; print(vllm_dlc.__file__)"
python3 -m pip show vllm
python3 -m pip show vllm_dlc
python3 -m pip list | rg '^(vllm|vllm-dlc|vllm_dlc|UNKNOWN|triton|torch|numpy)\b'
```

检查点：

- `vllm` 可导入
- `vllm_dlc` 可导入
- `pip show` 返回正常 metadata
- `pip list` 中不再出现 `UNKNOWN`
- 如果按当前环境策略不需要 `triton`，确认它没有成为残留冲突源

### 4. 需要脚本化时可直接复用 skill 脚本

如果希望把上面的 smoke 固化为自动化入口，可使用 `dlc-env-setup` skill 自带的 `scripts/runtime-smoke.sh`。知识库保留检查标准，脚本保留在 skill 资产中。

### 5. 失败时的回退路径

- `import torch` 失败：先确认是否仍在源码树内，再回到 wheel 重装与安装位置检查。
- NumPy bridge 失败：回到 PyTorch build preflight，重点检查 `numpy==1.26.4` 与 `USE_NUMPY`。
- `torch.backends.dlc` 不可用：回到底层依赖安装和 DLC Platform 对接检查。
- `vllm` / `vllm_dlc` import 失败：回到 packaging preflight，而不是直接重编 PyTorch。

## 常见坑

1. **在源码树中 smoke**：最常见的误判来源。
2. **只做 `import torch`**：这不足以验证 NumPy bridge 和 DLC Platform 可用性。
3. **把 `vllm` import 失败直接归因给 `vllm-dlc`**：底层 PyTorch wheel 不健康时，上层 import 往往只是受害者。
4. **忽略 `UNKNOWN` metadata**：这说明 editable install 元数据本身还有问题。

## 相关资料

- [python-build-preflight-for-pytorch-and-vllm.md](python-build-preflight-for-pytorch-and-vllm.md)
- [../runtime-debugging/dlc-workstation-env-rebuild.md](../runtime-debugging/dlc-workstation-env-rebuild.md)
- [../runtime-debugging/runtime-troubleshooting.md](../runtime-debugging/runtime-troubleshooting.md)

## 来源

- `/work/test/dlc-env-setup/scripts/runtime-smoke.sh`
- `/work/test/dlc-env-setup/SKILL.md`
