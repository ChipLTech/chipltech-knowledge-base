# cltech_smi 设备观测与诊断证据

## 适用场景

- 需要查看 Real DLC Hardware 或 TYD Chip 设备状态、HBM、功耗、温度、固件版本和进程占用。
- 需要为 vLLM-DLC operational evidence、DLC Runtime 排障或设备健康报告提供 query-only 观测材料。
- 需要区分只读观测、诊断采集、LYP 检查和维护操作的边界。

## 核心结论

`cltech_smi` 是团队当前主要的设备观测入口。它能提供设备列表、HBM 使用、功耗/温度、固件版本、进程占用、LYP/DLCCL 检查和 debug 信息收集上传能力。知识库和 agent 使用时应优先把它作为 **query-only 观测工具**；reset、驱动检查、LYP repair、debug 上传等有副作用或外部通信的 action 需要明确授权。

## 运行前提

典型环境需要：

- `/dev/cltechN` 设备节点存在。
- `/sys/devices/virtual/chipltech/cltechN` sysfs 节点存在。
- `/usr/local/chipltech/cl_tools` 工具目录存在。
- `/usr/local/chipltech/chipltech_smi_lib` 安装目录可写，仅构建/安装时需要。
- host 或 privileged 容器能访问 `/dev`、`/sys`、`/run`、`/lib/modules`、`/var/log`。

最小发现命令：

```bash
lspci | grep -E 'DLC|TYD'
ls /sys/devices/virtual/chipltech
ls /dev/cltech*
```

## 构建版本与功能边界

`chipltech_smi_lib` 提供三个构建形态：

| 版本 | 构建命令 | 主要用途 | 边界 |
|---|---|---|---|
| 默认版 | `./build.sh` | 日常运维、诊断、维护 | 功能最全，包含状态、查询、进程、reset、LYP、DLCCL、debug 上传 |
| Release 版 | `./build.sh -release` | 交付环境和问题收集 | 精简状态页和 debug 上传，适合收集现场材料 |
| Demo 版 | `./build.sh -demo` | 演示环境 | 只展示逻辑设备 HBM 信息，不支持 reset、query、LYP 或 debug 上传 |

默认版和 Release 版会安装：

- `/usr/local/chipltech/chipltech_smi_lib/cltech_smi`
- `/usr/local/chipltech/chipltech_smi_lib/cltech_device_info`
- `/usr/local/chipltech/chipltech_smi_lib/cltech_device_info_release`
- `/usr/local/bin/cltech_smi`
- `/usr/local/bin/cltech_device_info`

`build.sh` 会删除并重建仓库内 `build/`，还可能安装系统依赖。CI 或 agent 执行前必须确认工作树安全和系统包变更授权。

## 常用只读命令

状态总览：

```bash
cltech_smi
```

查看帮助和设备列表：

```bash
cltech_smi -h
cltech_smi -L
cltech_smi --list-tpus
cltech_smi -B
cltech_smi --list-excluded-tpus
```

限制显示范围：

```bash
cltech_smi -i 2
CHIPLTECH_VISIBLE_DEVICES=0,2 cltech_smi
```

循环刷新：

```bash
cltech_smi -l 5
cltech_smi --loop=5
```

查询版本与产品信息：

```bash
cltech_smi -q
cltech_smi -q -i 1
```

脚本友好的 HBM 查询：

```bash
cltech_smi --query-dlc=hbm_memory
cltech_smi --query-dlc=memory.total --format=csv,noheader,nounits
```

`--format` 仅和 `--query-dlc` 一起使用，并应放在 `--query-dlc` 之后。

## 诊断命令与授权边界

以下命令可能有更强诊断含义或副作用，执行前需要确认授权、目标设备和当前 host 是否共享：

| 命令 | 用途 | 风险边界 |
|---|---|---|
| `cltech_smi --lypcheck=dlc_info_agg --verbose` | 聚合 LYP/设备信息 | 诊断采集，不等于模型 acceptance |
| `cltech_smi --loopcheck --verbose` | loopback port error 和链路检查 | 可能较慢，需保存日志 |
| `cltech_smi --lypportcheck` | 根据 loopcheck 输出 LYP topology | 依赖前置检查结果 |
| `cltech_smi --dlcclcheck` | DLCCL 通信检查 | 不等于 request-correlated DLC Runtime dispatch |
| `cltech_smi --versioncheck` | 固件、容器驱动、已安装驱动版本检查 | 版本不一致需人工判断是否可接受 |
| `cltech_smi --readeye -i <id>` | 读取 eye address 信息 | 仅诊断，不默认执行 |
| `cltech_smi --debug -m "description"` | 收集并上传 debug 信息 | 会上传日志到 issue 服务，需确认内容和外部通信授权 |
| `cltech_smi -sr ...` | 软复位 | 维护操作，可能影响其他进程 |
| `cltech_smi -hr` | 硬复位 | 强维护操作，必须明确授权 |

TYD Chip 上，`--lypcheck=dlc_info_agg` 当前可用；其他 lypcheck 脚本可能提示 `Feature coming soon` 并跳过。

### Reset 验收边界

执行 `cltech_smi -sr ...` 前必须记录当前工具版本/help、完整参数、目标物理设备、设备到容器逻辑编号映射、共享/独占状态、任务进程、HBM 和 handles。Core soft reset 与 full soft reset 是不同维护动作，不能只写“做了 reset”。

Reset 命令 exit 0 只证明工具接受并完成了命令流程，不证明 DLC Runtime execution 已恢复。每次 reset 后、下一恢复动作前必须在 fresh process 中重跑 allocation、H2D、device operation、synchronize、D2H 和 correctness。多卡 deployment 还应复验：

- 每张目标设备独立执行。
- 目标设备同时执行。
- DLCCL collective correctness。

如果多个恢复动作之间没有 fresh-process probe，只能记录为 `recovered_after_action_sequence`，不能将恢复唯一归因于最后一个 reset、service restart 或进程清理动作。

## cltech_device_info

`cltech_device_info` 用于查看更底层的 XYS head/tail 状态：

```bash
cltech_device_info
watch -n 1 cltech_device_info
CHIPLTECH_VISIBLE_DEVICES=0,2 cltech_device_info
```

典型字段包括：

- `dw tail`
- `dw completed head`
- `outfeed tail`
- `outfeed completed head`

当 vLLM、DLCCL 或 DLC Runtime 卡住时，这些字段可作为设备侧进度观察材料，但不能单独证明根因。

## Debug 上传证据

默认版和 Release 版的 `--debug` 会收集并上传：

- `dlc_info_agg.sh` 输出，上传为 `dlc_info_agg.log`。
- `/var/log/dlc_info_agg_<hostname>.log`，上传为 `dlc_info_agg_var_log.log`。
- 最近的 `dmesg` 日志，上传为 `dmesg_chipltech.log`。

Uploader SDK 的职责边界：

- 业务方只传入一次采集生成的多个文件路径。
- SDK 自动采集机器信息，生成 `manifest.json`。
- SDK 申请 presigned upload URL。
- SDK 通过 HTTP PUT 上传文件和 manifest。
- 每个文件使用 `Content-MD5` 做传输完整性校验。
- 失败时返回清晰 `error_message`。

`manifest.json` 应记录 issue type、collect scene、collect command、hostname、machine id、目标设备编号、文件名、文件角色、文件大小和 Content-MD5。不要在知识库或公开报告中写入 access key、secret key、token、个人账号密码或无关机器敏感信息。

## Operational Evidence 使用边界

用于 vLLM-DLC 或环境验证时，`cltech_smi` query-only 输出可以作为以下材料：

- run-local device reference。
- indexed finite positive HBM capacity。
- 进程占用观察。
- 版本/固件/设备列表线索。

但它不能单独证明：

- Real DLC Hardware acceptance。
- Verified vLLM Alignment。
- request-correlated Chunked Prefill DLC Runtime。
- DLC Runtime dispatch。
- 新模型功能正确性。

短 prompt serving smoke、SMI 查询、LYP 检查和进程占用观察应分别记录，不要合并成更强结论。

SMI 显示设备存在、HBM 正常或无可见进程，也不证明首个 device operation 可以完成。Query-only evidence 必须与 fresh-process layered Runtime execution evidence 分开保存。

## PCIe Link 观察边界

采集 PCIe 状态时至少记录：

- Chipltech-Family Accelerator device ID 与 PCI BDF 映射。
- Current link speed 和 maximum link speed。
- Current link width 和 maximum link width。
- 采样时间、工作负载状态和设备是否 idle。
- Pre-launch、active workload 和 post-cleanup 三个阶段的原始输出。
- PCIe capability、AER 或 kernel log 是否因权限不可读。

Current speed 低于 maximum speed，但 width 未降级、无 AER/设备错误且 workload 正常时，只能记录为未定性的 operational observation。Idle power management 可能影响 current speed，但没有电源状态证据时也不能断言一定是节能降速。

以下证据出现时才考虑升级为明确设备或链路问题：

- Link width 降级或 link down。
- 可关联的 AER/kernel/firmware 错误。
- 设备掉卡、DLC Runtime 错误或相同 workload 的可重复性能退化。
- 经过批准的受控复验建立了时间和设备相关性。

权限不足时标记 `not_observable_due_to_permission`。不得为了验证猜测自动执行 PCIe retrain、remove/rescan、reset 或 reboot，也不得把 post-cleanup link speed 观察与先前 DLC Runtime hang 或 benchmark 建立无证据因果关系。

## 相关资料

- [runtime-troubleshooting.md](runtime-troubleshooting.md)
- [common-error-log.md](common-error-log.md)
- [../vllm-dlc/model-adaptation-and-main-to-main-decisions.md](../vllm-dlc/model-adaptation-and-main-to-main-decisions.md)

## 来源

- `/work/chipltech_smi_lib/README.md`
- `/work/chipltech_smi_lib/uploader/docs/uploader接入说明.md`
- `/work/chipltech_smi_lib/uploader/docs/development_guide.md`
