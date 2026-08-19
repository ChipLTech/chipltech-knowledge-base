# ModelZoo 批量验证 — Hermes 执行经验

生成时间: 2026-07-29 09:35

本文档记录 DLC Platform 上 ModelZoo 批量模型验证中遇到的各类问题、根因和应对方法。Hermes / Agent 在执行模型批量验证任务时应先读取本文档了解已知陷阱。

---

## 已知问题类型

### 1. multiprocessing freeze_support 错误

**现象**: `An attempt has been made to start a new process before the current process has finished its bootstrapping phase`

**影响模型**: 所有使用 vLLM `LLM()` 的模型（Qwen2-7B, gemma-3-4b-it 等）

**根因**: vLLM 内部使用 `torch.multiprocessing.spawn` 启动 worker 进程，要求脚本必须以 `if __name__ == "__main__":` guard 包裹入口。

**修复**: 在被 `docker exec` 调用的 Python 脚本中添加:
```python
if __name__ == "__main__":
    run()
```

**注意**: 常规脚本必须使用 `if __name__ == "__main__":` guard。当前已验证 runner 会为每个模型创建独立 `docker exec`，并以 `python3 -c` 执行完整脚本内容；不要在交互式 REPL 中直接调用 `LLM()`。

### 2. trust_remote_code 缺失

**现象**: `The repository /models/<name> contains custom code which must be executed... Please pass trust_remote_code=True`

**影响模型**: Aquila-7B, Aquila2-7B, chatglm2-6b, chatglm3-6b, Baichuan2-7B-Base, Baichuan2-7B-Chat, Baichuan2-13B-Chat

**根因**: 这些模型的 `config.json` 中包含 `auto_map` 或自定义 Python 代码，vLLM/HF 要求显式授权才能执行。

**修复**: 在 `LLM()` 初始化时添加 `trust_remote_code=True`。对于 API server 方式，添加 `--trust-remote-code` 标志。

**自动检测**: 读取 `config.json` 检查 `"auto_map"` 字段是否存在。

### 3. Docker exec 子进程生命周期

**现象**: API server 启动后健康检查返回 200，但首个 POST 请求后 server 立即退出，日志显示 "Parent process exited, terminating worker queues"。

**影响**: 所有 API server 启动方式

**根因**: `docker exec bash -lc 'python3 server... &'` 中 bash 进程退出后 Docker
清理整个 exec session。`setsid`、`disown`、`exec` 均无效。

**修复**:
- 避免使用 API server 方式做单模型验证
- 改用 `vllm.LLM()` 直接加载模型并推理
- 当前 `background_process` 工具追踪的 `docker exec` 会随着命令结束而进入 `failed` 状态
- 除非用 `background_process` + `persistent: true` 并以 `exec` 方式将 docker exec
  PID 替换为 server 进程，否则不要依赖此方式做 server serving

**功能验证不使用 server 方式**，而是通过单模型 runner 在独立进程中完成加载和推理。

### 4. PYTHONPATH 未传递

**现象**: `No module named 'torchvision'`

**影响**: 所有 `docker exec` 调用

**根因**: `docker exec` 不自动继承容器 login shell 的 PYTHONPATH。

**修复**: 使用 `docker exec -e PYTHONPATH="..."` 传递，或在容器内脚本中
`os.environ["PYTHONPATH"] = "..."`。

**PYTHONPATH 固定值**: `/opt/qwen-site-packages:/opt/qwen-src/vllm:/opt/qwen-src/vllm-cl:/workspace/src/torchvision`

### 5. 软重启后 DeepSeek 服务中断

**现象**: `cltechpd_clnt -s` 后设备 0 的 DeepSeek 服务 HTTP 连接被拒绝

**影响**: 设备 0 共享服务

**根因**: 软重启 reset 全部 8 个设备，正在运行的设备 0 服务被停止。

**说明**: 仅在必要且用户明确指示时使用 `cltechpd_clnt -s`。执行前应：
1. 确认设备 0 服务可以在 reset 后恢复
2. 优先仅对目标设备 4-7 执行操作

### 6. Hermes oneshot tool-calling 不可靠

**现象**: 5/5 次 `chipltech-engineering --yolo -z "..."` 无工具执行

**根因**: 模型在推理后退出而不触发工具调用

**建议**:
- 不委托 Hermes oneshot 执行 Real DLC Hardware 操作
- 改用主 Agent 直接 `bash`/`background_process` 工具
- Hermes 仅用于知识查询、静态分析、代码解释

---

## 可靠验证模式

```bash
python3 /home/xuansun/modelconfig/run-one-model.py <model_name> <model_path>
```

每次只运行一个模型。runner 为模型创建独立进程，完成 `LLM()` 加载、推理、结果保存和资源清理，无需 API server；不得在同一 Python 进程中循环构造多个 `LLM()` 实例。

---

## 环境固定变量

| 变量 | 值 |
|---|---|
| PYTHONPATH | `/opt/qwen-site-packages:/opt/qwen-src/vllm:/opt/qwen-src/vllm-cl:/workspace/src/torchvision` |
| DLC_VISIBLE_DEVICES | `4,5,6,7` |
| DLC_SYN_COPY_ASYNC | `O0` |
| HF_HUB_OFFLINE | `1` |
| TRANSFORMERS_OFFLINE | `1` |
| 基础容器 | `modelzoo-batch-base` |
| 设备 0 服务 | `http://127.0.0.1:28086/health` (不触碰) |
