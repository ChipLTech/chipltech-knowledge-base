# Case Study: vLLM-DLC PD transport 与 LYP 恢复

## 问题现象

在 DLC Chip 节点 `node-8-45` 上验证 vLLM-DLC Prefill/Decode Separation 时，monolithic Qwen3-1.7B 可返回确定性结果，但 PD transport 连续遇到三类问题：

- Protocol `tcp` 的 TransferEngine 仍自动安装 RDMA，并因 completion queue / RNIC 初始化失败阻止双 role 启动。
- `lyp_full` 的 Prefill、Decode、Proxy 与 request correlation 均可建立，但 data plane 卡在 `ProcessGroupDLCCL.send().wait()` / `recv().wait()`。
- `cltech-init --check-lyp` 显示前后两个 LYP group 均不可用；后组初始化脚本在临时容器内缺少执行权限，修复权限后仍存在真实链路故障。

## 背景与环境

- vLLM：`a208f41eee15d15b0da619ded9384fda5efd2e7f`
- vllm-dlc：`ce14cbc726c73df65a3ec6e970da523c6ed22ea8`
- mooncake-dlc main：`92381029f07d719ec6a50b4fc698324de168fba3`
- direct transport branch：`572dbf0dcd00337db0cbe82a015d8d824988da97`
- PyTorch：2.5.0
- DLC driver：20.0.3
- 模型：`/mnt/jfs/models/Qwen3-1.7B`
- 最终角色：物理卡 4 Prefill、物理卡 7 Decode、TP=1

节点原有卡 0 DeepSeek workload，因此早期诊断只做 read-only 检查和 task-owned cleanup；LYP state change、power cycle、Bluejay 和 reboot 在单独获得 Host-maintenance 授权后执行。

## 定位路径

### 1. 先建立 monolithic baseline

同一模型、prompt 和 decoding parameters 在单卡返回 `PD-OK`，usage 为 prompt 19、completion 4、total 23。模型、tokenizer、DLC 单卡执行和请求 contract 因此被排除为最早失败点。

### 2. TCP failure 定位到 TransferEngine capability

Connector 选择 `tcp`，但日志仍出现：

```text
Auto-discovering topology...
installTransport, type=rdma
Failed to create completion queue
RdmaTransport: No available RNIC
```

源码使用启用 auto-discovery 的 TransferEngine。0.3.10.post1 和 0.3.11.post1 non-CUDA wheel 都不能证明 TCP-only 双进程初始化。该 failure 属于 exact TransferEngine transport capability，不属于模型或 KV layout。

### 3. 修正 `lyp_full` rendezvous

Prefill 会在 API ready 前等待 TCPStore peer。正确顺序是 Prefill start、store listener、Decode start、两 role ready；若等待 Prefill `/health` 后再启动 Decode，会耗尽 store timeout。

修正后控制面成功，但相同 request 的 Prefill/Decode block 都为 1 时，native send/recv completion 仍不返回。在两组设备上复现后，将 failure 收敛到 LYP/DLCCL data plane。

### 4. 回到 Transport Qualification Gate

独立检查发现 LYP group 不可用。后组限定初始化日志最初包含：

```text
rst_lyp_all_back.sh: Permission denied
```

Host 上修改权限无效，因为 `cltech-init` 每次从 image 创建临时容器。最终在 `/usr/local/cltech-init/cltech-init-dlc` 的容器 patch 阶段幂等执行：

```bash
chmod +x '${lyp_patch_dir}'/rst_lyp_all*.sh
```

权限 failure 消失后，RDMA probe 仍 timeout，证明存在真实链路状态问题。

### 5. 正确映射失败卡并修复

后组日志提取的 local index `0,3` 映射到物理卡 4、7，不能直接当作物理卡 0、3。仅对物理卡 4、7执行 power cycle。卡变为 `rev ff` 后按工具要求 reboot，再继续 Bluejay、HBM 和 driver 恢复。

最终 read-only status 为：

```text
前4卡不可用
后4卡可用
```

该证据只限定后组，不能提升为 whole-Host LYP health。

### 6. 使用 direct DLCCL gate

main 之外的 exact feature branch 提供 torch-free `_dlccl_direct` extension。构建后先运行单测：native transport/connector 21 项、trace 2 项全部通过。

进程设置 `DLC_VISIBLE_DEVICES=<physical-id>` 后只看到 local device 0，因此 benchmark 的 `--device` 必须使用 0。修复后的物理卡 4/7 完成 1 MiB payload：

```text
Producer: 211.748 us, 168.456 us
Consumer: 208.168 us, 160.418 us
```

benchmark 校验接收内容，因此 Transport Qualification Gate 通过。

### 7. 请求级 PD 闭环

使用 `dlccl_direct` 启动 Prefill、Decode 和 Proxy。外部 request UUID 为：

```text
08da0d40-b06e-4374-aa04-f17a7ba0bf98
```

Prefill request：

```text
...-aee96457
block_ids=1
register_send
```

Decode local request：

```text
...-a1266e53
remote_request_id=...-aee96457
local_blocks=1
successfully received KV
External prefix cache hit rate: 100.0%
```

PD 返回 `PD-OK`，usage 与 monolithic baseline 相同。该 request correlation、block registration、receive completion、external-cache use 和输出等价组成 `pd_validated` 的闭环。

## Site Recovery

`cltech-init` 限定操作仍触发了全局 process cleanup，后续 reboot 也终止了原卡 0 workload。根据事先记录的模型、端口和启动参数恢复时，第一次 engine 卡在初始化；仅对卡 0执行 soft reset 并恢复 2.8 GHz HBM 后，DeepSeek 服务重新通过：

- `/health`
- `/v1/models` model identity
- 实际 `/v1/chat/completions`

测试结束时 8 卡 HBM/PCIe 恢复到预期最终状态，卡 1-7 无 HBM 占用，所有 PD/benchmark 端口释放；原卡 0 workload 则按已记录的模型、端口、启动参数和功能请求完成恢复对比。

## 根因与结论

- TCP failure：exact TransferEngine build 的 protocol 与 transport installation 语义不一致，`tcp` 仍依赖 RDMA auto-discovery。
- `lyp_full` hang：应用控制面已闭合，`ProcessGroupDLCCL` data plane 不完成；随后证实 Host LYP group 不可用，这是未满足的必要前置条件和高概率贡献因素。修复后未重跑旧 `lyp_full`，因此不能排除 connector 或 ProcessGroupDLCCL 的独立缺陷，也不能把 LYP 状态声明为唯一根因。
- LYP 初始化首个工具缺陷：临时容器内 back-group reset script 缺少 executable bit。
- 修复后剩余硬件状态通过定向 power cycle、reboot、Bluejay/HBM 和 group-scoped verification 恢复。
- 最终通过路径：exact feature branch 的 `dlccl_direct` + content-validating native benchmark + request-correlated PD evidence。

## 可复用经验

1. Monolithic baseline 先隔离模型与 PD orchestration。
2. 加载两个模型前必须执行 Transport Qualification Gate。
3. 单卡 tensor allocation、PCIe 16 GT/s 或 role health 不能证明 LYP data plane。
4. `lyp_full` Decode 应在 store listener 出现后启动，不等待 Prefill HTTP ready。
5. Request/block 已对齐但 native completion hang 时，先回到 transport 与 group-scoped LYP status。
6. Host 文件权限不一定影响工具新建容器内的 image 文件。
7. 工具 local failure index 必须映射 physical device 后才能 repair。
8. `DLC_VISIBLE_DEVICES` 的 physical identity 与 native API local device index 必须分别记录。
9. `docker exec` client 停止不证明容器内进程退出。
10. Host maintenance 前必须建立 Site Recovery Contract；恢复 workload 要验证真实 completion。

## Claim Boundary

本案例只证明 exact branch、后组卡 4/7、Qwen3-1.7B、TP=1、单请求 `dlccl_direct` PD 核心功能。它不证明：

- main 的 TCP 或 `lyp_full` 已修复。
- 前 4 卡 LYP 可用。
- TP>1、并发、长上下文或正式性能达标。
- 手工构建 extension 已成为标准发布物。

## 来源

- `/home/xuansun/文档/node-8-45_vllm-dlc_PD分离测试完整过程资产_20260723.md`
- `/home/xuansun/vllm-dlc-workspace/artifacts/pd-baseline-response.json`
- `/home/xuansun/vllm-dlc-workspace/artifacts/pd-dlccl-direct-response.json`
- `/home/xuansun/vllm-dlc-workspace/artifacts/dlccl-direct-producer-after-repair.json`
- `/home/xuansun/vllm-dlc-workspace/artifacts/dlccl-direct-consumer-after-repair.json`
