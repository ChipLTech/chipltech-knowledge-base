# Hermes ModelZoo 单模型验证 Prompt

> 适用场景: Hermes 执行单个模型的 DLC Platform 功能验证
> 依赖: `modelzoo-batch-base` 容器, `run-one-model.py`
> 知识库: `vllm-dlc/modelzoo-startup-params-spec.md`, `case-studies/modelzoo-batch-validation-all-difficulties.md`

## 用途

这是 Hermes 执行模型验证的主 prompt。Kilo Agent 在启动每个模型验证时，将此 prompt 作为 Hermes 的输入。

## 可直接复制的 Prompt

### 方式 A: 最简单指令（首选）

```
立即运行这条命令并返回结果:
python3 /home/xuansun/modelconfig/run-one-model.py <MODEL_NAME> <MODEL_PATH>
```

### 方式 B: 分步检查（A 失败时使用）

```
按步骤执行，每步完成后告诉我结果:

Step 1: 检查 HBM 是否空闲
运行: /usr/local/chipltech/chipltech_smi_lib/cltech_smi --query-dlc=hbm_memory
检查设备 4-7 的 HBM 是否小于 500 MB。如果是，回复 READY。如果不是，回复 BUSY。

Step 2: 检查输入和历史证据
确认 <MODEL_PATH> 存在，读取对应 ModelZoo README.md、metafile.yml，以及 /mnt/jfs/modelzoo/<MODEL_NAME>/difficulties.md（若存在）。

Step 3: 运行单模型 runner
运行: python3 /home/xuansun/modelconfig/run-one-model.py <MODEL_NAME> <MODEL_PATH>

Step 4: 检查结果
读取 /mnt/jfs/modelzoo/<MODEL_NAME>/results.json 和 difficulties.md（若存在），回显实际启动参数、功能输出、Tier、失败分类和 evidence 路径。
```

### 方式 C: 工具能力检查（A/B 都失败时使用）

```
请先尝试运行一个最简单的 bash 命令来确认工具可用:
列出 /home/xuansun 目录下的文件

如果能执行，再回到方式 A，并提供明确的 <MODEL_NAME> 和 <MODEL_PATH>。不要在未指定模型时默认重跑 Qwen2-7B。
```

## 失败回退策略

如果 Hermes 不能执行工具:
1. 记录 failure type: `no_tool_execution`
2. Kilo Agent 切换到直接执行模式
3. 下次尝试更短的指令

如果模型验证 runner 报错:
1. 读 `/mnt/jfs/modelzoo/<MODEL_NAME>/difficulties.md`
2. 按困难记录中标注的 Skill 和 KB 路径执行修复
3. 修复后重试

## 执行时注意事项

- 不要触碰设备 0 DeepSeek (http://127.0.0.1:28086/health)
- 不要联网下载模型
- 每次只验证一个模型
- 完成一个模型后间隔至少 10 分钟再启动下一个
- 如果 Tier 1 镜像 15 分钟未响应，自动降级到 Tier 2

## 模型队列（按顺序取第一个未完成的）

```
QwQ-32B /models/QwQ-32B
DeepSeek-R1-Distill-Qwen-7B /models/DeepSeek-R1-Distill-Qwen-7B
DeepSeek-R1-Distill-Qwen-14B /models/DeepSeek-R1-Distill-Qwen-14B
DeepSeek-R1-Distill-Qwen-32B /models/DeepSeek-R1-Distill-Qwen-32B
DeepSeek-R1-Distill-Llama-8B /models/DeepSeek-R1-Distill-Llama-8B
DeepSeek-R1-Distill-Qwen-1.5B /models/DeepSeek-R1-Distill-Qwen-1.5B
gemma-3-4b-it /models/gemma-3-4b-it
Aquila-7B /models/Aquila-7B
Aquila2-7B /models/Aquila2-7B
Baichuan2-7B-Base /models/Baichuan2-7B-Base
Baichuan2-7B-Chat /models/Baichuan2-7B-Chat
chatglm3-6b /models/chatglm3-6b
Chinese-Mistral-7B /models/Chinese-Mistral-7B-Instruct-v0.1
bloomz-7b1 /models/bloomz-7b1
```

## 已完成模型（跳过）

```
Qwen2-7B      ✅ PASS (Tier 1)
Qwen3-14B     ✅ PASS (Tier 2)
Qwen3-32B     WRONG (unresolved accuracy failure)
```
