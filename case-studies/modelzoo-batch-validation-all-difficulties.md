# ModelZoo 批量验证 — 全部困难与中断点

生成时间: 2026-07-29 12:40 +08:00
范围: Qwen2-7B DLC 功能验证全链路

本文档记录从 Qwen3-32B 诊断到 Baichuan-7B 验证、再到 Qwen2-7B DLC 镜像交付过程中遇到的**所有问题、困难和中断点**。后续 Hermes/Agent 执行模型验证任务时应先读取本文档。

---

## 1. docker exec + pkill 挂起

**现象**: `docker exec container pkill -f vllm` 在没有匹配进程时无限挂起，bash 命令超时。

**影响**: 每次清理残留进程时阻塞。

**根因**: docker exec 的 pkill 在无目标时等待，而不是立即返回。

**修复**: 始终使用 `timeout 5 docker exec container pkill -9 -f vllm`。

**已持久化**: `run-one-model.py` 所有 docker exec 调用均使用 `timeout 5` 包裹。

---

## 2. docker exec -e PYTHONPATH 覆盖

**现象**: `docker exec -e PYTHONPATH=...` 会覆盖容器默认 PYTHONPATH，导致 import 失败。

**根因**: docker exec `-e` 标志替换而非追加环境变量。

**修复**: 在容器内脚本通过 `os.environ["PYTHONPATH"] = "..."` 设置，或使用 `bash -lc 'export PYTHONPATH=...'`。

**已持久化**: `run-one-model.py` 通过 docker exec -e 传递最终 PYTHONPATH 值。

---

## 3. Hermes oneshot 工具调用不可靠

**现象**: 5/5 次 `chipltech-engineering --yolo -z "..."` 无工具执行，模型推理后直接退出。

**根因**: 模型工具调用机制不稳定（DeepSeek V4 Flash 和 gpt-5.6-luna 均如此）。

**修复**: 不委托 Hermes oneshot 执行 Real DLC Hardware 操作。改用主 Agent 直接 `bash`/`background_process` 工具。

**已持久化**: `.config/kilo/skills/model-adaptation/knowledge.md`

---

## 4. LLM() max_num_seqs=1 导致 Engine Init 失败

**现象**: 15/19 模型在 `LLM(max_num_seqs=1, ...)` 时报 `Engine core initialization failed`。

**根因**: vLLM-CL engine 初始化要求 ≥2 seqs；`max_num_seqs=1` 触发无法支持的代码路径。

**修复**: 始终使用 `max_num_seqs=8`（匹配 vLLM API server 默认值）。

**已持久化**: `run-one-model.py` 默认 max_num_seqs=8。

---

## 5. 同一进程 del LLM 后资源未释放

**现象**: batch v4 (同一 Python 进程内 `LLM()` → `del llm` → 下一个 `LLM()`) 全部后续模型报 Engine Init 失败。

**根因**: `del llm` 后 DLC 内存/会话未完全释放，下一个 `LLM()` 继承脏状态。

**修复**: 每个模型使用**独立进程**（subprocess 或 fresh docker exec）。

**已持久化**: 定时任务规范——每次任务只验证一个模型。

---

## 6. VLLM_USE_DLC_COL_MAJOR_MATMUL=1 不兼容

**现象**: 从 ModelZoo README 提取的 `VLLM_USE_DLC_COL_MAJOR_MATMUL=1` 导致 Engine Init 失败。单独/组合测试其他环境变量（DLC_SYN_URING, VLLM_USE_V1）均通过，排除后立即恢复正常。

**根因**: 当前 sealed vLLM-CL 版本与此变量不兼容。

**修复**: 从 ModelZoo README 提取的环境变量中**永久排除**此变量。

**已持久化**: `modelzoo-startup-params-spec.md` 标记此变量为 ❌ 必须排除。

---

## 7. python3 file.py vs python3 -c 行为差异

**现象**: 同一 Python 代码，`python3 /tmp/script.py` 执行时 `os.environ` 在 `from vllm import LLM` 前不生效，导致 Engine Init 失败；而 `python3 -c "$(cat /tmp/script.py)"` 正常工作。

**根因**: `.py` 文件执行与 `-c` 字符串执行在 Python 编译单元上有未知差异，`os.environ` 修改时机不同。

**修复**: 使用 `bash -c 'python3 -c "$(cat /path/to/script.py)"'` 代替直接执行 `.py` 文件。

**已持久化**: `run-one-model.py` 采用 `cat` + `-c` 方式。

---

## 8. ModelZoo README TP=1 在此硬件不可用

**现象**: Qwen2-7B ModelZoo README 指定 `-tp 1`，但此 4-device DLC 环境需要 TP=4。

**根因**: ModelZoo README 是原始 bench 机的单卡配置，不适合当前多卡环境。

**修复**: `TP = max(ModelZoo_TP, 4)`，低于 4 时自动使用 4。

**已持久化**: `run-one-model.py` 解析逻辑。

---

## 9. trust_remote_code 缺失

**现象**: Aquila/Baichuan/chatglm 系列模型需要 `--trust-remote-code`。

**根因**: 这些模型的 `config.json` 包含 `auto_map`，vLLM 要求显式授权。

**修复**: 从 ModelZoo README 提取参数时检测 `--trust-remote-code` 是否存在。

**已持久化**: `run-one-model.py` 的 `parse_modelzoo_params()`。

---

## 10. Docker cp + heredoc 脚本写入不可靠

**现象**: 通过 heredoc (`cat > file << 'EOF' ... EOF`) 写入的 Python 脚本可能被截断或包含格式错误。

**根因**: shell escaping 复杂、多行字符串在 docker exec 中不可靠。

**修复**: 使用 host 侧 Python `write_text()` + `docker cp` 传输文件。

**已持久化**: `run-one-model.py` 使用 `docker cp` 方式。

---

## 11. bash 工具中 nohup+& 后台进程被 Kill

**现象**: `nohup docker exec ... &` 在 bash 工具返回后进程被杀死。

**根因**: bash 工具在退出时清理子进程。

**修复**: 使用 `background_process` 工具（persistent: true）追踪 docker exec 进程。

---

## 12. ModelZoo 路径混淆

**现象**: 混淆 `/home/xuansun/ModelZoo` (ModelZoo 仓库) 与 `/mnt/jfs/ModelZoo` (不存在)。

**根因**: 目录约定未文档化。

**修复**: 明确三条路径约定。

**已持久化**: `modelzoo-startup-params-spec.md` 目录约定表。

| 目录 | 用途 |
|---|---|
| `/home/xuansun/ModelZoo` | ModelZoo 仓库 (只读 reference) |
| `/mnt/jfs/modelzoo` | 验证 evidence + DLC 镜像 |
| `/home/xuansun/modelconfig` | 总结分析文档 + runner 脚本 |

---

## 13. 软重启杀死设备 0 DeepSeek

**现象**: `cltechpd_clnt -s` 重置全部 8 个设备，设备 0 的 DeepSeek 服务中断。

**根因**: 软重启范围是全局的，不限于目标设备。

**修复**: 仅在明确需要且用户指示时使用。使用前停止保护服务。

**已持久化**: `modelzoo-batch-validation-hermes-guide.md`

---

## 14. chunked-prefill 导致 server 首次 POST 崩溃

**现象**: API server 启动后 health 200，但首次 `POST /v1/completions` 返回 404 后 server 立即退出。

**根因**: `--enable-chunked-prefill` 与 `max_model_len=1024`/`block_size=256` 组合在此版本不稳定。

**修复**: 功能验证阶段使用 `LLM()` 代替 API server 方式。

---

## 15. API server 由 docker exec 启动后自动退出

**现象**: 通过 `docker exec -d` / `nohup` / `setsid` 等方式启动 API server，server 在 docker exec 会话结束时被杀死。

**根因**: Docker exec 会话生命周期与 server 进程绑定——父进程退出后子进程被清理。`exec` 替换 bash 无效。

**修复**: 功能验证阶段使用 `LLM()` 代替 API server。如需 serving，使用 `background_process` 工具（persistent: true）。

---

## 16. batch v4/v5/v6 全失败但单次测试通过

**现象**: Qwen2-7B 单独 `LLM()` 测试通过，但放入批量脚本后全失败。

**根因**: 依次为——
- v4: `max_num_seqs=1` 导致 Engine Init
- v5: 同一进程 `del LLM` 后资源未释放
- v6: subprocess 中 `python3 script.py` 的 `os.environ` 不生效

**修复**: v7 当前版本——每模型 fresh `docker exec` + `cat` + `-c` 方式，已验证通过。

---

## Hermes 学习总结

| 类别 | 遇到次数 | 自动处理策略 |
|---|---|---|
| `needs_trust_remote_code` | 5+ 次 | 从 ModelZoo README 自动检测 |
| `engine_init_failed` | 15+ 次 | 多种根因(TP/seqs/env/multiprocessing)→逐层排查 |
| docker exec 挂起 | 10+ 次 | 统一 `timeout 5` 包裹 |
| Hermes 不执行工具 | 5 次 | 不委托 oneshot 执行硬件操作 |
| PYTHONPATH 丢失 | 5 次 | 统一在 run-one-model.py 中传递 |
| 路径混淆 | 3 次 | 已文档化三条路径约定 |
