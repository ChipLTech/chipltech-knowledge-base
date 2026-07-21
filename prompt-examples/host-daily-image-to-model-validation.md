---
prompt_schema: host-daily-image-model-validation/v1
stage_0_identity: host-container-bootstrap
stage_1_skill_identity: dlc-env-setup
stage_2_skill_identity: model-adaptation
shared_contract: vllm-dlc-contract/v1
claim_boundary: operational_or_not_verified_only
---

# Host 每日镜像环境适配与模型功能验证 Prompt

## 用途

用于从 Host 选择或拉取一个明确的每日镜像，创建可恢复的持久容器，补齐 Git/SSH 和 DLC Ecosystem 环境，再对指定模型完成分层 serving smoke。该流程覆盖正常路径、常见故障诊断、阶段恢复和 evidence 保存，不把最小 operational smoke 提升为权威 acceptance。

本模板与以下模板配合使用：

- [从已配置容器为 Host 和其他容器赋能 Git/SSH](bootstrap-git-from-configured-container.md)
- [每日空镜像到新模型 vLLM-DLC 适配](vllm-dlc-fresh-image-to-model-adaptation.md)
- [DLC Ecosystem 环境 bootstrap](dlc-env-setup-environment-bootstrap.md)

## 核心原则

- 每日镜像必须记录 immutable image ID 和 repo digest；不能只记录可移动 tag。
- 默认从每日镜像创建新容器，不在验证后用 `docker commit` 把运行态凭据、模型缓存或临时产物固化回镜像。
- 源码、构建产物、报告和 serving 日志放在 Host 持久目录，通过 bind mount 进入容器；容器删除后 evidence 仍可审计。
- Git/SSH 缺失时引用 Git bootstrap 模板执行，禁止把私钥放入 Dockerfile、image layer、Git 仓库或报告。
- Host 驱动、设备初始化和 LYP 操作不属于普通环境适配。`cltech-init -i <日期>` 是包含镜像拉取、硬件检测、驱动安装、`fix_hbm` 和 LYP 内环初始化的 All-in-One 操作，不得把它当作普通 `docker pull` 自动执行。
- CLTech-Init 是当前工具名，`dlc-init` 只作为旧名称识别线索。执行前必须以当前主机的 `cltech-init -v`、`cltech-init -h` 和 `/etc/chipltech/cltech-init.conf` 为准，不能只依赖历史示例。
- 驱动安装/更新、`fix_hbm`、LYP4/LYP8 初始化、质量测试、掉电上电、软/硬重置、LYP repair、kill 非本任务进程和 reboot 必须按操作类别另获明确授权。
- 环境通过、模型加载通过、短 prompt 通过和长上下文通过是不同结论，必须分别报告。

## 可复制 Prompt

```md
请从 Host 开始，使用一个明确的每日镜像创建可恢复容器，完成 Git/SSH 与 DLC Ecosystem 环境适配，再对指定模型做分层功能验证。遇到问题时按本 prompt 的诊断和恢复规则处理，不跳过失败、不伪造结果，也不把 operational smoke 提升为权威 acceptance。

开始前读取并遵循：
- <KNOWLEDGE_BASE_ROOT>/CONTEXT.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/bootstrap-git-from-configured-container.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/dlc-env-setup-environment-bootstrap.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/vllm-dlc-fresh-image-to-model-adaptation.md
- <KNOWLEDGE_BASE_ROOT>/prompt-examples/vllm-dlc-model-adaptation.md
- <SKILLS_ROOT>/skills/engineering/dlc-env-setup/SKILL.md
- <SKILLS_ROOT>/skills/engineering/model-adaptation/SKILL.md

【Host 工作根目录】<例如 /home/user/daily-validation；必须位于持久磁盘>
【每日镜像引用】<registry/repository:tag；必须填写，不得猜测 latest>
【每日镜像日期】<YYYYMMDD；使用 CLTech-Init 时必须与镜像引用一致>
【期望镜像 digest】<sha256:...；没有则写“拉取后记录实际 digest，不做预声明校验”>
【芯片代际】<DLC Chip / TYD Chip；不确定则只读自动发现，不能从镜像名直接猜>
【容器名】<包含日期且唯一，例如 daily-vllm-YYYYMMDD>
【容器内工作根目录】<例如 /work>
【源容器】<已有 Git/SSH 的受信容器；目标容器已有可用 Git/SSH 时写“不需要”>
【Git 私有仓库验证 URL】<用于 ls-remote/clone/push dry-run 的批准仓库>
【Git/SSH 私钥迁移授权】<是/否>
【版本策略】<CI默认最新 / 固定ref / 混合>
【批准的仓库 remote/ref】<逐仓库填写；CI默认最新时声明采用批准的 CI 分支映射>
【模型绝对路径】<优先位于 /mnt/jfs/models；不得猜测或自动替换模型>
【model ID/revision】<明确值>
【tokenizer/processor revision】<明确值或 null>
【served model name】<API 请求使用的 alias>
【设备选择】<DLC_VISIBLE_DEVICES 和所需设备数量>
【deployment profile】<TP/PP/EP、dtype、quantization、max_model_len、max_num_batched_tokens、Chunked Prefill、prefix caching>
【functional profile】<用于 correctness 隔离的完整 serving profile；未指定时基于资产和资源提出保守值并记录依据>
【benchmark profile】<目标 scheduler capacity 的完整 serving profile；必须与 functional profile 做字段级 diff>
【strict functional prompts】<至少一个预期非空且有确定答案/格式的 short prompt；可附 medium prompt>
【DLC Runtime acceptance depth】<单目标设备 / 每张目标设备 / 每张加同时执行 / 加大块 correctness / 加 DLCCL；默认按 deployment profile 选择最小充分范围>
【concurrency/stability gate】<并发级别、每级请求数、稳定性并发/总请求、预验证非空 prompt、timeout 和失败阈值；未声明时写 not_requested，不照搬单个模型规模>
【是否执行服务端到端 benchmark】<是/否；默认是>
【benchmark workload】<dataset、random input/output length、num prompts、request rate、seed、EOS policy、timeout、重复次数；默认建议 random、256/256、150、inf、固定 seed、正式 3 次>
【benchmark 结果根目录】<默认 /home/xuansun/vllm-dlc-workspace/artifacts；必须是 Host 持久目录并挂载进容器>
【benchmark 比较基线】<同模型同 workload 的历史 artifact 路径或“无”；不得用不同配置结果直接判回归>
【容器资源】<shm-size、ipc、pid、memlock/stack ulimit、CPU、内存>
【允许安装系统包】<是/否>
【允许修改 /usr/local】<是/否>
【允许安装或升级 CLTech-Init】<是/否；默认否>
【允许的宿主机/设备操作】<默认“只读检查”；从“驱动安装/更新、fix_hbm、LYP4 内环、LYP8 外环、abs、matmul、hbmcopy、掉电上电、软重置、硬重置、reboot”中逐项批准>
【共享/独占设备模式】<共享 / 独占；未填写时按共享处理，清理以回到 sealed pre-launch baseline 为准>
【恢复动作授权矩阵】<service restart、精确 PID TERM/KILL、core/full soft reset、driver reload、reboot 分别填写；默认仅允许终止已证明属于本任务的精确 PID>
【目标物理设备】<例如 0,1,2；必须与 DLC_VISIBLE_DEVICES 的物理映射核对>
【批准的 HBM 频率】<例如 2.8；未授权 fix_hbm 时写“不适用”>
【artifact 目录】<Host 工作根目录下且位于 vllm-dlc 源码树外>
【容器结束策略】<保持运行 / 正常停止；默认保持运行便于复查>

按以下阶段执行。每个阶段先写 evidence，再进入下一阶段；失败后从最近通过的 checkpoint 恢复，不要从头盲目重做。

阶段 0：Host preflight、镜像固定和容器创建

1. 只读检查 Host：
   - 记录 OS、kernel、architecture、当前用户、Docker client/server 版本、存储空间和目标目录 owner/mode。
   - 记录 `docker info` 是否健康、目标容器名是否已存在、每日镜像本地是否存在。
   - 只读发现 `/dev/cltech*` 和兼容 `/dev/dlc*` 命名，并记录实际设备节点；同时记录 `/mnt/jfs`、`/sys`、`/lib/modules` 和 `/var/log` 的可见性。不要因某个历史节点名缺失而执行驱动初始化。
   - 记录 Host 当前设备观测结果和 HBM 占用。如目标设备正在被其他任务使用，停止并报告，不 kill 或抢占。
   - 检查 `command -v cltech-init`。存在时记录 `cltech-init -v`、`cltech-init -h` 支持的当前参数；不存在时只报告，除非【允许安装或升级 CLTech-Init】为“是”。
   - 只读检查 `/etc/chipltech/cltech-init.conf` 是否存在、owner/mode 和非敏感配置项。记录 `REGION`、DLC/TYD image、image date、`UPDATE_IMAGE`、`UPDATE_DRIVER`、`HBM_CLOCK`、`LYP_INIT` 和质量测试开关；配置可能因版本变化或历史拼写问题存在重复 key，必须结合 `cltech-init -h` 和实际镜像引用判断，不能盲目 source 后执行。
   - CLTech-Init 的命令行参数优先于配置文件。最终报告必须同时记录配置值和本次批准的命令行 override，防止“配置写 no、命令实际强制执行”的审计遗漏。
   - 只读检查 `dlc-driver.service`、`persisd.service`（存在时）的状态和日志位置。服务不存在不等于可以自动安装；服务异常也不等于可以自动 restart。

2. 准备持久目录：
   - 在【Host 工作根目录】下使用独立目录保存 `src/`、`build/`、`wheels/`、`artifacts/`、`logs/` 和必要 cache。
   - 新建前确认父目录存在且正确；不覆盖同名非本任务目录。
   - artifact 目录必须在所有业务源码树外。创建 run manifest，记录 run ID、开始时间、Host、镜像引用、目标模型和 deployment profile，不写 secret。

3. 固定每日镜像：
   - 使用 `docker pull <每日镜像引用>` 获取用户指定镜像；未授权联网或 pull 时只使用已确认的本地镜像。
   - pull 后记录 image ID、repo digest、created time、architecture 和基础 image metadata。
   - 若提供【期望镜像 digest】，实际 digest 不一致时停止。不要继续使用 tag 当前指向的其他镜像。
   - 不修改或覆盖该 tag，不执行 `docker image prune`。
   - 默认使用显式 `docker pull <每日镜像引用>` 完成纯镜像拉取。不要用 `cltech-init -i <每日镜像日期>` 代替，因为后者默认还可能安装驱动、执行 `fix_hbm` 和初始化 LYP。
   - 如果用户明确批准使用 CLTech-Init All-in-One，先验证镜像日期格式和【芯片代际】，回显它将触发的全部子操作，并逐项核对【允许的宿主机/设备操作】；任一隐含子操作未批准时不得执行 All-in-One，应拆分为纯 pull 和已批准的单项操作。

4. 生成并回显容器运行契约后再创建容器：
   - bind mount 持久 `src/build/wheels/artifacts/logs` 到【容器内工作根目录】下对应路径。
   - 只读挂载模型目录；需要写 tokenizer/cache 时使用单独可写 cache，不修改模型资产。
   - 按用户声明挂载 `/mnt/jfs`、`/dev`、`/sys`、`/lib/modules`、`/var/log`，并设置 ipc/pid/shm/ulimit；不要无理由使用 `--privileged`。
   - 仅暴露 serving 所需端口；端口已占用时选择用户批准的替代端口并记录映射。
   - 设置明确的 `DLC_VISIBLE_DEVICES`，不要把“容器可见全部设备”误当作任务可用全部设备。
   - 使用可保持运行且可 exec 的入口启动容器。容器已存在时先比较 image ID、mount、environment、device、port 和资源配置；一致则复用，不一致则停止，不自动删除或重建。

阶段 0 验收：容器处于 running，image digest 与 run manifest 一致，持久目录可写，模型目录只读可见，目标设备与资源契约符合声明。保存 `docker inspect` 的脱敏结果。

阶段 0.5：Host 设备健康与 CLTech-Init 安全门

5. 在模型或 DLC Runtime 验证需要 Real DLC Hardware 时，先执行低风险健康检查：
   - `cltech-init --check`：仅做 Chipltech-Family Accelerator 硬件检测。
   - `cltech-init --check-lyp`：仅做 LYP 状态检查。
   - `cltech-init --smi`：仅在驱动已安装时采集设备观测。
   - 上述命令以当前 `cltech-init -h` 确认支持为前提。当前版本不支持时记录 `unsupported_by_installed_cltech_init`，不要退回旧 `dlc-init` 猜命令。
   - 不要把 `--check`/`--check-lyp` 一概描述为无副作用只读命令。CLTech-Init v1.1.2 的实测行为包括创建临时检测容器，并尝试停止/启动 Cltech-Exporter；执行前后必须比较容器列表和 exporter 状态，保存工具日志，并清理由本次检查成功创建且确认不再使用的临时资源。
    - 将退出状态、开始/结束时间和 `/var/log/cltech-init/<hostname>.log` 对应片段保存到 artifact。只记录相关时间窗并脱敏，不使用无限跟踪命令阻塞流程。
    - 建立设备到 PCI BDF 的映射，并在 pre-launch 保存 current/max link speed 和 width。Current speed 低于 maximum 但 width 未降级、无 AER/设备错误时只记录为未定性 observation，不自动归类为 `BLOCKED_DEVICE`。

6. 根据检查结果分类，不直接升级为破坏性修复：
   - `READY`：目标设备数量、驱动状态、HBM 和请求范围内的 LYP 状态满足 deployment profile，可继续。
   - `BLOCKED_DRIVER`：驱动缺失、版本不匹配或 service 异常。报告 image/driver 版本、`dlc-driver.service` 和 `/var/log/dlc-driver/service.log`（存在时），等待“驱动安装/更新”授权。
    - `BLOCKED_DEVICE`：设备缺失、link down/width 降级、可关联 AER/硬件检查失败或 HBM 异常。保留日志并停止，不执行掉电上电、reset 或 reboot。只有 current PCIe speed 低于 maximum、但 width 和 workload 均正常时标记 `UNCLASSIFIED_PCIE_OBSERVATION`，不阻塞也不建立因果。
   - `BLOCKED_LYP`：请求多卡/跨卡且 LYP 检查失败。单卡 profile 不应因未请求的外环失败而自动做 LYP repair；多卡 profile 必须停止等待对应 LYP4/LYP8 授权。
   - `BLOCKED_BUSY`：目标物理设备被其他任务占用。不要通过 reset、驱动重载或 kill 其他进程抢占。
   - `BLOCKED_TOOLING`：CLTech-Init 缺失、版本不支持所需检查或配置有歧义。不得把工具缺失误报为硬件故障。

7. 只有获得逐项授权后，才可执行对应 Host 变更：
   - 安装/升级工具：优先使用已批准的 `/mnt/jfs/software/cltech-init/install.sh`；`install.sh -f` 会重新初始化配置，仅在明确批准覆盖配置且已保存脱敏配置快照时使用。无 JFS 时下载来源、版本和完整性校验必须先获批准。
   - 驱动更新：`cltech-init -i <日期> -f` 会强制更新指定镜像版本的驱动，但它仍属于 All-in-One 路径；只有驱动更新、`fix_hbm` 和对应 LYP 子操作均逐项获批时才允许执行。若只批准驱动更新，应使用当前 `cltech-init -h` 明确提供的单项方式；当前版本没有单项方式时停止并报告，不能用 All-in-One 代替。执行前停止本任务 serving、确认目标卡无人使用，并说明可能影响整机其他容器。
   - `fix_hbm`：仅作用于【目标物理设备】，使用批准的 `-t` 和 `-g`；不需要 LYP 时必须显式 `--skip-lyp`，避免隐式初始化。不得把容器内逻辑设备编号未经映射直接传给 Host `-t`。
   - LYP 内环使用当前帮助确认的 `--lyp4`；外环使用 `--lyp8`，且需要完整拓扑和所有相关主机的变更窗口。不得把 LYP4 通过解释为 LYP8 通过。
   - `abs`、`matmul`、`hbmcopy` 是手动、可能耗时的质量测试，分别只在对应授权后使用 `--test-abs`、`--test-matmul`、`--test-hbm`，并使用 `-t` 限定目标设备。
   - 掉电上电/LYP 修复目前存在芯片代际能力差异，必须以当前 `cltech-init -h` 为准；不支持的 TYD Chip 操作不得尝试 DLC Chip 路径。

8. 变更后必须重新验证而不是只看命令退出码：
   - 重新运行 `--check`、请求范围内的 `--check-lyp` 和 `--smi`。
   - 核对当前 Host 实际 `/dev/cltech*`/兼容 `/dev/dlc*` 节点、目标设备映射、驱动/service 状态、HBM 频率/容量和错误日志。
   - LYP4 日志读取 `/var/log/cltech-init/lyp4_init.log`，LYP8 日志读取 `/var/log/cltech-init/lyp8_init.log`；主流程日志读取 `/var/log/cltech-init/<hostname>.log`。
   - 如果变更命令返回成功但复检失败，状态仍为 BLOCKED。禁止继续创建“环境健康”checkpoint。
   - Host 驱动或设备状态变化后，重启或重建目标容器前先比较原运行契约；只处理本任务容器，不批量 restart 其他容器。

阶段 0.5 验收：只读检查或获批修复后的复检证明目标物理设备、驱动和请求范围内的 LYP 健康；物理设备到容器 `DLC_VISIBLE_DEVICES` 的映射已记录。单卡任务可明确标记 LYP8 `not_applicable`，但不能标记 PASS。

阶段 1：目标容器 Git/SSH 与 DLC Ecosystem 环境适配

9. 先检查容器内 Git/SSH：
   - 如果 Git、SSH identity 和目标私有仓库访问都已通过，记录版本与身份后跳过迁移。
   - 如果缺失，严格使用 `bootstrap-git-from-configured-container.md`，以【源容器】为来源并遵守【Git/SSH 私钥迁移授权】。
   - 不输出 key，不复制 credential store，不把 SSH 文件放进持久源码目录或 image layer。
   - 通过 `git ls-remote`、临时 clone、fetch 和 `git push --dry-run` 验证。只验证读权限但未验证写权限时必须明确标记。

10. 使用 `dlc-env-setup` 做 repo discovery 和 bootstrap：
   - 不假设固定源码路径；在持久 `src/` 中发现或 clone `dlc-thunk`、`DLCsim`、`DLCSynapse`、`DLC_CL`、`LLVM`、DLC_Custom_Kernel Repository、`pytorch`、`vllm` 和 `vllm-dlc`。
   - 对每个仓库记录 Git root、remote、branch/tag、full HEAD 和 `status --short`。
   - 已有 dirty 工作树时停止，不 stash、不 reset、不 clean、不覆盖。
   - 只按【版本策略】和【批准的仓库 remote/ref】同步。remote/ref 不明确或多个候选路径冲突时停止。
   - 检查 CMake 必须严格大于 `3.27.0`，并检查 compiler、Python、pip、wheel、NumPy bridge 和构建入口。
   - 按当前 `dlc-env-setup` skill 的依赖顺序进行必要的最小 rebuild/reinstall，不因单项失败跳到后续组件。
   - wheel、build log 和环境报告写入 Host 持久目录，不只保存在容器 writable layer。

11. 运行阶段 1a Package/Import 验证：
   - `pytorch-preflight.sh`
   - `vllm-preflight.sh`
   - `runtime-smoke.sh <源码树外临时目录>`；执行前读取脚本内容，确认其实际覆盖范围。
   - PyTorch 2.5.0、NumPy bridge、DLC Platform 和 `vllm` import/package metadata 检查。
   - 记录当前源码 checkout 与实际 import 的 package 路径，防止误用旧 wheel 或其他 editable install。
   - 如果脚本只检查 package/import、NumPy bridge 和 backend availability，它只能创建 `Package/Import Ready`，不能因为文件名含 `runtime-smoke` 就声称 device execution 健康。
   - 先按 deployment contract 分支：candidate 内建 `DLCPlatform` 时验证 platform class/import path/entry points 和 plugin 禁用状态，将 `vllm_dlc` 标记 `not_applicable`；plugin 路径才执行 `vllm_dlc` import/metadata 检查，失败必须阻断。

阶段 1a 验收：repo map、preflight、package/import smoke 和实际 import identity 可审计，创建 `C1a Package/Import Ready`。该 checkpoint 不允许加载模型。

12. 运行阶段 1b DLC Runtime Execution 验证：
   - 严格使用 [安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md) 中的 fresh-process layered probe。
   - 当前 skill 支持时，内建 `DLCPlatform` 使用 `runtime-smoke.sh <源码树外目录> --require-vllm --skip-vllm-dlc --require-device-execution --device-index <逻辑设备>`；plugin deployment 将 `--skip-vllm-dlc` 替换为 `--require-vllm-dlc`。
   - 用 `DLC_VISIBLE_DEVICES` 固定物理设备映射，在源码树外运行，并设置明确的外层 timeout。
   - Probe 必须逐层输出并 flush：backend availability、device count、device properties、memory info、allocation submit、CPU source、H2D submit、nontrivial device operation submit、synchronize completion、D2H、correctness。
   - 不打印 DLC tensor 内容或 `repr`，只记录 shape/dtype/device metadata，避免日志触发隐式同步或 D2H。
   - `torch.backends.dlc.is_available()`、allocation 或 `.to("dlc")` 单独成功均不充分。任何阶段 error/hang 都不得加载模型。
   - 按【DLC Runtime acceptance depth】执行；多设备 serving 至少逐张目标设备独立 probe，再同时 probe。TP/多卡 deployment 还需 DLCCL collective correctness；大模型前建议增加有界的大块 H2D/device-op/D2H correctness。
   - 每个 probe 保存脚本、命令、cwd、environment allowlist、开始/结束时间、timeout、exit code、signal、完整 stdout/stderr、PID identity、设备映射、HBM 和 handles。

阶段 1b 验收：Fresh process 完整通过 device operation、synchronize、D2H 和 correctness，请求范围内的设备与 DLCCL acceptance 完成，创建 `C1b DLC Runtime Execution Ready`。只有该 checkpoint 通过后才能加载模型。

阶段 2：模型身份检查、服务启动和功能验证

13. 模型 preflight：
   - 验证【模型绝对路径】存在且是批准资产，记录 config、tokenizer/processor 文件和 revision；不输出模型敏感内容。
   - 检查模型架构、dtype、quantization metadata、所需设备数、估算 HBM 和 deployment profile 是否匹配。
   - 量化模型核对 `quant_method`、`bits`、`group_size`、`zero_point` 与可用 kernel 路由；不能根据目录名猜测 AWQ、AWQ-Marlin、W8A16 或 compressed-tensors 兼容性。
   - 资源、revision、processor 或 manifest 身份缺失时，在模型加载前报告 `blocked_missing_asset` 或 `blocked_missing_hardware`。

14. 启动 serving：
   - 使用 tracked background process 或容器内可追踪进程启动，不使用失管的 `nohup`/`&`。
    - 将完整启动命令的脱敏版本、environment、Host wrapper、container exec、APIServer、EngineCore 和 worker 身份、端口和日志路径写入 run manifest。
    - 每个 server epoch 使用不可复用的 ID。进程身份至少记录 container ID、Host PID、container PID、`NSpid`、PID namespace inode、`/proc/<pid>/stat` starttime、cmdline、executable、PPID 和 cgroup，避免 PID 重用或 namespace 误判。
   - tracked wrapper 结束不等于容器内子进程已经退出。停止服务后必须同时检查 Host 和容器 process table、端口、`cltech_smi` 进程信息和 HBM；任一仍占用时不能启动下一 server epoch。
   - 请求中的 `model` 必须等于【served model name】；启动未设置 alias 时才使用启动契约规定的模型名。
   - 等待明确 readiness 日志并做 health/model-list 检查。超时后保存日志，不重复启动第二个竞争实例。

15. 按阶梯验证，每层通过后才进入下一层：
   - L0 liveness：health endpoint、model list、server PID 和设备可见性正常。
    - L1 strict short：使用【strict functional prompts】中预验证会产生确定非空结果的 prompt，设置 deterministic 参数，分别判定 HTTP success、request completion、non-empty generation、strict/semantic correctness、finish reason 和 server liveness。
   - L2 中 prompt：保持 deterministic 参数，只增加 prompt 长度，记录输入 token、输出 token、首 token latency、总 latency 和输出摘要。
   - L3 长 prompt/one-shot：每轮只改变一个变量，记录 prompt token 数、decode 参数、Chunked Prefill 选择、输出、finish reason、latency 和日志位置。
   - L4 可选扩展：功能并发、多卡、TP/PP/EP、MoE、量化、vision/multimodal 和长上下文必须按 contract 单独记录，不能由 L1/L2 推断通过。Benchmark 前置 concurrency/stability gate 只有【concurrency/stability gate】已声明时才执行。

16. 每层做前后健康检查：
   - 记录 server 是否仍存活、目标设备 HBM、关键错误日志和请求关联 ID。
    - Strict prompt 空输出、答案错误、重复符号、timeout、异常截断、NaN/Inf、进程退出或明显降速时，停止提升层级，保存最小失败输入。
    - 任意或并发 probe 出现空 text 时，不立即定性为 scheduler request loss。保存原始 JSON、HTTP status、finish reason、completion token 数、stop 字段、server request ID；支持时保存 output token IDs/logprobs。用完全相同 prompt 串行 replay，再用预验证非空 prompt 保持原并发重跑。
    - 相同 prompt 在串行与并发下均为 HTTP 200、正常 stop 且无服务错误时，可记录为“与 prompt-specific immediate stop/EOS 一致”；没有 token ID 证据时不要把具体 EOS token 身份写成已唯一确认。
   - 不用一个失败层级覆盖前面已通过的 bounded 结论，也不把前面通过解释为失败层级已通过。

阶段 3：服务端到端性能 benchmark

17. 只有 C1b、阶段 2 的 L0、L1 和请求范围内功能 smoke 通过，且【是否执行服务端到端 benchmark】为“是”时，才能执行 `vllm bench serve`。本阶段只做 serving 端到端 benchmark，不默认启用 DLC profiler，不采集算子 cycle 占比，也不根据单次结果推导理论性能上限。

18. 建立不可歧义的 benchmark contract：
   - 先保存 `vllm --version` 和当前版本的 `vllm bench serve --help`。不同 vLLM 版本参数可能变化，只使用当前帮助明确支持的参数；不要照抄历史命令后忽略 unknown option。
   - 固定 image digest、`vllm`/`vllm-dlc` full HEAD、实际 import path、模型和 tokenizer revision、served model name、endpoint、host/port、TP/PP/EP、dtype、quantization、block size、Chunked Prefill、`max_model_len`、`max_num_batched_tokens`、`max_num_seqs`、prefix caching、设备映射和 HBM 频率。
   - 固定 dataset、random input length、random output length、`num_prompts`、request rate、seed、temperature/top-p、ignore-EOS 选择和 client 所在位置。`--num-prompts` 是总请求数，不等同于并发数；实际并发语义要结合 request rate、client 调度方式、server `max_num_seqs` 和结果中的 `Peak concurrent requests` 报告。
    - 功能 profile 不应直接用于饱和并发 benchmark。保存 functional 与 benchmark profile 的完整字段级 diff，创建新 server epoch，重跑 health、model list/alias/context 和 strict short；不能只检查 `max_num_seqs`，也不能把两个 profile 的 evidence 混为一个 contract。
   - `request-rate=inf` 是饱和压力/吞吐 workload，不代表真实业务到达率。需要业务流量结论时，另建有限 request-rate case，不能复用饱和结果。
   - 比较两个结果时，上述 contract 必须一致。任何字段变化都视为新的 benchmark case，不直接计算“性能提升/回退”。

19. 在【benchmark 结果根目录】中为每次 run 创建独立目录，推荐：

   ```text
   /home/xuansun/vllm-dlc-workspace/artifacts/benchmarks/<date>/<model>/<case>/<attempt>/
   ```

    每个 attempt 至少保存：
   - `manifest.md`：完整 benchmark contract、image/repo/model identity、开始结束时间和结论。
   - `server-command.txt`：脱敏后的 serving 命令和环境变量。
   - `client-command.txt`：脱敏后的 `vllm bench serve` 命令。
   - `server.log`：覆盖 warm-up 和正式测量时间窗的服务日志。
   - `client.log`：benchmark 原始 stdout/stderr，不只保存人工摘录。
   - `metrics.md`：从原始结果抄录的结构化指标和单位。
   - `health-before.txt`、`health-after.txt`：服务、进程和设备状态。
   - 当前 vLLM 支持结构化 `--save-result`/JSON 输出时额外保存原始 JSON；不支持时保留完整 `client.log`，不要发明参数。
    - case 目录额外保存 `summary.md` 和 `metrics.csv`：逐 attempt 保留原始指标，并汇总成功次数、median、min/max；不能只保存最佳 attempt。
    - 每条命令的 evidence 记录 run/case/attempt/server epoch ID、executable 绝对路径、argv、cwd、environment allowlist、开始/结束时间、timeout、exit code、signal 和 stdout/stderr。使用管道或 `tee` 时必须保留真实 client exit code。
    - Failure epoch、失败 prompt/raw response、process/device snapshot 和日志采用 append-only 新目录或唯一文件名；后续成功 attempt 不得覆盖或删除失败证据。

20. 先完成 benchmark-profile readiness；如果【concurrency/stability gate】已声明则执行该 gate，否则标记 `not_requested`；然后做 benchmark-shaped warm-up，最后正式测量：
   - 使用与正式 workload 相同的模型、endpoint 和长度做小规模 warm-up，但 warm-up 请求数单独记录且不得计入正式结果。
   - warm-up 后确认服务存活、模型 alias 正确、无失败请求、目标设备 HBM 稳定且没有其他客户端流量。
   - 正式 benchmark 默认重复 3 次，每次使用相同 contract 和独立 attempt 目录。重复次数由【benchmark workload】覆盖时按声明执行。
    - attempt 之间确认前一个 client 已退出、server 仍健康且没有遗留请求；不重启 server 来掩盖某一次失败。若确需重启，记录为新的 server epoch，不能与未重启 attempt 淆在一起。
    - 如果合同另有与正式 workload 同规模的 acceptance load（例如 Gate 6），必须单独命名、说明执行顺序和是否计入正式结果。它不能替代后续 formal attempts；其后的小规模 warm-up 应称为 formal measurement conditioning，不应描述为首次冷启动加热。

21. 参考命令形态如下，实际参数必须以当前 `--help` 和【benchmark workload】为准：

   ```bash
   vllm bench serve \
     --model <MODEL_PATH> \
     --served-model-name <SERVED_MODEL_NAME> \
     --dataset-name random \
     --random-input-len 256 \
     --random-output-len 256 \
     --num-prompts 150 \
     --request-rate inf \
     --seed 1024 \
     --temperature 0 \
     --port <PORT>
   ```

   - 如果当前版本要求 `--backend`、`--base-url`、`--endpoint`、`--request-rate` 或 seed，显式填写并写入 contract。
   - 当前 vLLM 可能不再默认发送 `temperature=0`。需要 deterministic workload 时必须显式传入 `--temperature 0`；否则记录服务端实际 sampling default，不能假定 greedy。
   - `<MODEL_PATH>` 用于 tokenizer/workload 构造，`<SERVED_MODEL_NAME>` 必须匹配服务注册的 alias；二者不能因字符串不同就随意改成相同值。
   - 使用 tracked foreground command 执行 client，并把完整 stdout/stderr 同步保存到 `client.log`。benchmark 超时必须有明确上限，不能无限等待。

22. 每个正式 attempt 必须提取并报告：
   - Successful requests、失败请求数/失败率和总请求数。
   - Benchmark duration、Total input tokens、Total generated tokens。
   - Requested input/output length 与 JSON 中实际 `input_lens`/`output_lens` 分布。至少记录 min/max/median、full-length output 数和 `short_of_requested_output_count`；tokenizer 或 endpoint 可能使实际长度变化，比较时使用实际 token 总量并保留请求参数。
   - 只有 raw response/result 明确提供 `finish_reason`、stop 字段或 token ID 时才报告对应 stop/EOS 分布；否则标记 `not_emitted_by_this_vllm_version` 或 `not_observable_from_saved_result`，不得仅凭短于 requested output 推断 EOS。
    - EOS policy 和当前 CLI 中 `ignore_eos` 的实际值。若没有显式反向参数，保存当前 `--help` 和默认值证据，不把历史默认当作永久 contract。
   - Request throughput、Output token throughput、Peak output token throughput、Total token throughput。
   - Peak concurrent requests；它是观测值，不是 `num_prompts` 的同义词。
   - Mean/Median/P99 TTFT。
   - Mean/Median/P99 TPOT。
   - Mean/Median/P99 ITL。
   - 测量前后 server liveness、PID、目标设备 HBM 和关键服务错误。
   - 若当前 vLLM 版本没有输出某项指标，标记 `not_emitted_by_this_vllm_version`，不得填 0 或从不等价指标推算。

23. benchmark 通过标准和比较规则：
   - 所有计划请求成功，或失败率不高于用户明确批准阈值；没有阈值时任何失败请求都使该 attempt 为 FAIL。
   - client 正常结束，server 在 benchmark 后仍健康，没有 OOM、timeout、进程退出、NaN/Inf 或明显错误输出。
   - 至少达到用户声明的重复次数；仅一次成功只能报告 single-run observation，不能称稳定性能基线。
   - 多次结果分别保留，摘要报告 median，并同时报告 min/max 或离散程度；不得只挑最好一次。
   - 只有 benchmark contract 完全一致且基线来源可审计时才计算相对变化。没有批准阈值时只报告数值变化，不自行判定 regression PASS/FAIL。
    - 用户示例中的 150 requests、约 2.94 req/s、682 output tok/s、TTFT/TPOT/ITL 等只说明该特定运行结果格式，不作为其他 image、模型或 deployment profile 的默认门槛。
    - Successful requests 只证明客户端 workload 完成，不证明每个输出非空、生成满 requested output length 或语义正确。Strict functional correctness、actual token/stop 分布和 benchmark completion 分别报告。

24. benchmark 故障处理：
   - client 连接失败：先核对 host/port、endpoint、server readiness 和容器端口映射；不要立即重启服务。
   - 全部或部分请求失败：保存错误类别和失败样本，核对 alias、tokenizer、最大长度、timeout 和 server 日志；功能失败未解决前不继续加压。
   - TTFT 很高但 output throughput 正常：检查请求突发程度、Peak concurrent requests、prefill 长度、Chunked Prefill 和 scheduler queue，不只归因于单个 kernel。
   - TPOT/ITL 退化：检查 decode 并发、TP/LYP 状态、输出长度分布、其他设备负载和服务日志；保持 workload 不变后再重测。
   - 吞吐波动大：检查是否存在其他客户端、设备占用、server epoch、热身不足、频率/温度状态和请求失败；不要增加重复次数来掩盖不稳定。
   - OOM/hang/server crash：该 attempt 为 FAIL，保存 client/server 日志和设备状态，返回最近通过的功能 checkpoint。未经授权不降低 deployment profile 后覆盖原 case。
   - client 卡住：在超时后记录客户端和服务进程、已完成请求数、最后日志和设备状态；只终止本任务 client。服务端终止按已有授权边界处理。

阶段 3 验收：声明 workload 的正式 attempts 全部完成，原始日志和结构化指标已保存到【benchmark 结果根目录】，服务测后健康，重复测量摘要可审计。该结论只称为对应 contract 的 serving end-to-end benchmark，不证明模型正确性、Verified vLLM Alignment 或 Real DLC Hardware 权威 acceptance。

问题诊断与恢复规则

25. 镜像/容器问题：
   - pull 失败：区分 registry auth、DNS/TLS、tag 不存在和磁盘空间；不关闭 TLS，不切换到未经批准的镜像。
   - 容器立即退出：检查 entrypoint、command、architecture 和动态库，不用无限 restart loop 掩盖错误。
   - mount 缺失或权限错误：比较 Host 路径 owner/mode、容器 UID/GID 和 inspect 配置；不要递归 chmod 777。
   - 配置契约错误时保留失败容器和 manifest，使用新名称创建修正容器；未确认持久数据前不删除旧容器。

26. Git/源码问题：
   - SSH 失败按 key owner/mode、有效 `ssh -G`、host key、port/proxy 和 GitHub identity 顺序定位，禁止打印私钥。
   - remote/ref 不匹配、dirty tree、detached HEAD 未批准或 submodule 失败时停止，不 reset/clean 强行通过。
   - 网络暂时失败只重试幂等 fetch/pull，限制次数并保留原始错误；认证失败不循环重试。

27. 构建/import 问题：
   - 保留首个根因错误和完整 log；不要只报告最后一行。
   - 比较 build 使用的 Python/pip/CMake/compiler 与 runtime import 环境，检查旧 wheel、editable install、`PYTHONPATH` 和动态库路径污染。
   - 只重建失败组件及当前 skill 要求的下游组件；上游健康证据不足时回到阶段 1 safety gate。

28. serving/模型与 DLC Runtime 问题：
   - OOM：先核对实际 HBM、并发、TP、`max_model_len`、batching 和 cache 配置；未经批准不自动缩小 deployment profile 后声称原 profile 通过。
   - API 404/model mismatch：核对 `--served-model-name`、请求 endpoint 和 model 字段。
    - timeout/hang：保存进程状态、请求、日志、设备观测和最后通过 checkpoint。先按“小模型 -> 纯 PyTorch -> layered DLC Runtime probe”切层；allocation/H2D PASS 不等于 device operation PASS。`peek_stuck.sh`、软重置、LYP repair、kill 非本任务进程或 reboot 需明确授权。
   - 输出异常：建立相同模型、tokenizer、prompt、decode 参数和 token budget 的等价对照；多步分叉可缩成“分叉前 token + 单步 decode”以隔离 KV cache/scheduler 变量。
    - 每次修复只改变一个变量，生成新的 attempt ID；成功后重跑导致失败的最小 case 和所有较低层级 smoke。
    - 每个 service restart、进程清理、core/full reset 或 driver action 后、下一动作前必须由 fresh process 重跑最小失败 case。没有中间 probe 时，只能报告 `recovered_after_action_sequence`，不得把恢复唯一归因于最后一个动作。
   - 服务 wrapper 已停止但 HBM 仍占用：先比较 Host/容器 PID namespace、端口、`cltech_smi` 和 `lsof /dev/cltech*`。`lsof` 受权限、tracefs/ZFS 和 mount namespace 影响，空输出不能单独证明无设备占用。
    - 已确认是本任务残留进程时，TERM 前重新核对 container ID、Host/container PID、`NSpid`、namespace inode、starttime、cmdline、cgroup 和 server epoch；等待后只有身份仍匹配且未退出时才按需 KILL。Host 无权限 signal 容器 root PID 时，从 owning container 使用对应 container PID 精确终止，不假设两侧 PID 相同。
   - `lsof -t /dev/cltech* | xargs -r kill -9` 是全设备残留进程强制清理命令，可能终止其他用户或其他容器任务。只有独占维护窗口、已审计 `lsof /dev/cltech*` 全部 PID/命令归属且明确授权强制清理时才允许执行，不能作为普通 benchmark 收尾默认命令。
    - 进程清理后本任务 HBM 增量仍未释放时，软重置属于 Host/设备变更。不要使用历史工具名猜命令；先在当前 Host 核对 `command -v`、version、help、目标物理设备语义和授权。执行后重跑硬件/LYP/SMI、设备映射和 fresh-process layered execution smoke。进程清理已回到 baseline 时不得额外 reset。

29. CLTech-Init/驱动/LYP 问题：
   - `cltech-init` 命令不存在：先判断是工具未安装、PATH 问题还是旧 `dlc-init` 残留；不要自动建立 alias。只有安装授权存在时才安装当前 CLTech-Init。
   - `-i` 拉镜像后发生意外驱动或 LYP 变更：立即停止后续模型验证，保存主日志、当前配置、image/driver 身份和设备复检结果；不要再次执行 All-in-One 尝试“修好”。
   - `--check` 失败但 `--smi` 有输出：分别报告硬件检测与观测结果，不能以 SMI 可读覆盖硬件检测 failure。
   - `--check-lyp` 失败：确认 deployment profile 是否真的需要 LYP、目标是内环还是外环、物理拓扑和目标卡选择；没有授权时只保存 `lyp4_init.log`/`lyp8_init.log` 和主日志。
   - 驱动 service 与设备节点不一致：同时检查 `dlc-driver.service`、`persisd.service`、相关日志和当前 Host 的 `/dev/cltech*`/兼容 `/dev/dlc*` 节点，不要仅 restart service。任何 restart 都属于 Host 变更，需要授权和占用检查。
   - HBM 频率或质量测试失败：隔离到明确的物理设备，记录 `-t`、`-g`、测试类型和日志；不要扩大到所有卡，不自动降低频率后声称原频率通过。
   - CLTech-Init 卡住或超时：记录 PID、子进程、当前日志增长、设备占用和最后完成阶段；终止本任务进程也要先确认不会留下半完成驱动/LYP 状态，随后必须完整复检。

阶段 4：收尾和交付

30. 正常停止本任务启动的 serving 进程，确认本任务 process group、容器 APIServer/EngineCore/workers、clients、监听端口和 device handles 均退出，HBM 回到 sealed pre-launch baseline 或批准容差。共享 Host 不要求全机 HBM 为 0，也不清理原有占用；只有满足第 28 步安全门时才使用强制清理或软重置。
31. 按【容器结束策略】保持容器运行或正常停止；不默认删除容器，不执行 `docker commit` 或 prune。
32. 最终报告必须包含：
   - image tag、image ID、repo digest、容器名和脱敏后的运行契约。
   - Host 持久目录、mount、设备选择和资源配置。
   - CLTech-Init version/help 能力、配置与 CLI override、芯片代际、image/driver 身份、物理卡映射、`--check`/`--check-lyp`/`--smi` 结果及相关日志路径。
   - 所有获批 Host 变更的命令类别、目标设备、HBM 频率、LYP4/LYP8 范围、前后状态和复检结果；未授权操作明确列为未执行。
   - Git/SSH 验证状态及所引用的 bootstrap 报告，不包含 secret。
    - repo map、full HEAD、环境构建/reinstall 和三个 preflight/smoke 结果。
    - C1a package/import 与 C1b layered execution 的独立结果、每张目标设备/DLCCL acceptance 和最早失败边界。
   - 模型身份、deployment profile、每个验证层级的 PASS/FAIL/BLOCKED、请求摘要和日志路径。
   - benchmark contract、每个 attempt 的成功/失败请求、吞吐、TTFT、TPOT、ITL、Peak concurrent requests、server health、artifact 路径，以及 `summary.md`/`metrics.csv` 跨 attempt 摘要。
   - 与历史基线比较时，列出 contract 等价性检查、基线来源、绝对值和相对变化；不同 contract 不输出回归结论。
    - 每次 failure 的根因或当前最小边界、采取的单变量修复、重验结果和最后通过 checkpoint。
    - 每个恢复动作后的 fresh-process 结果；缺少中间 probe 时明确记录无法唯一归因。
   - Requested/actual token、short-of-request output，以及可观察时的 finish-reason/stop 分布、EOS policy；同时说明 `request_rate=inf` 不是业务 QPS。
    - PCIe pre-launch/active/post-cleanup 观察；未定性 speed 变化不得写成已证实故障或与 benchmark 建立无证据因果。
   - 未验证范围：未实际执行的长上下文、并发、多卡、量化、Chunked Prefill runtime、DLC Runtime dispatch 或 Real DLC Hardware acceptance 必须标记 `not_verified`。

结论用语限制：
- L1/L2 通过只能称为对应 image、checkout、模型和 deployment profile 的 operational smoke 通过。
- 不得仅凭 HTTP 200、短 prompt、Dummy、DLCsim 或静态检查声称 Verified vLLM Alignment、权威 Chunked Prefill acceptance 或 Real DLC Hardware acceptance。
```

## 阶段 Checkpoint

| Checkpoint | 最小通过条件 | 可恢复入口 |
|------------|--------------|------------|
| C0 Container Ready | tag、digest、image ID、持久目录、mount 和容器资源契约正确 | 从 Host 设备检查继续 |
| C0.5 Hardware Ready | 目标设备、驱动、物理卡映射和请求范围内的 LYP 健康 | 从 Git/SSH 检查继续 |
| C1a Package/Import Ready | repo map、preflight、package/import identity 和 backend availability 通过 | 从 layered execution 继续 |
| C1b DLC Runtime Execution Ready | fresh-process device-op/synchronize/D2H correctness 及请求范围内多卡验收通过 | 从模型 preflight 继续 |
| C2 Server Ready | 模型加载、health 和 model list 通过 | 从 L1 请求继续 |
| C3 Short Smoke | deterministic 短 prompt 非空且服务健康 | 从 L2/L3 继续 |
| C4 Functional Scope Done | 用户要求的最高功能层级完成 | 从 C4.5 benchmark profile 或收尾继续 |
| C4.5 Benchmark Profile Ready | 新 server epoch、完整 profile diff、health/models/strict short 通过 | 按 contract 进入 C4.6 或 C4.7 |
| C4.6 Concurrency/Stability Ready | 声明的 gate 通过；未声明时为 `not_requested` | 从 benchmark-shaped warm-up 继续 |
| C4.7 Warm-up Complete | 同形 warm-up 通过且未计入正式结果 | 从 formal attempt 继续 |
| C5 Benchmark Done | 正式 attempts、原始日志、指标摘要和测后健康检查完成 | 收尾与报告 |

## 常见误区

- 把每日 tag 当作固定版本：tag 会移动，必须记录 digest 和 image ID。
- 把 `cltech-init -i <日期>` 当成纯镜像拉取：它默认是 All-in-One，可能更新 Host 驱动、执行 `fix_hbm` 和初始化 LYP。
- 只看 CLTech-Init 退出码：必须用 `--check`、`--check-lyp`、`--smi` 和对应日志做变更后复检。
- 混淆 LYP4 与 LYP8：内环通过不证明外环通过，单卡不需要的外环应标记 `not_applicable` 而非 PASS。
- 把容器 `DLC_VISIBLE_DEVICES=0` 直接当作 Host 物理卡 0：必须记录物理到逻辑设备映射后才能使用 `cltech-init -t`。
- 只在容器 writable layer 保存源码和日志：容器重建后 evidence 丢失，应使用 Host bind mount。
- 为方便把 Host 整个 `~/.ssh` 挂入容器：扩大凭据暴露面，应使用 Git bootstrap 的最小文件迁移策略。
- 容器启动失败就删除重建：先保留 inspect、日志和 manifest，否则失去根因证据。
- 环境 smoke 通过后直接声明模型通过：必须完成真实模型加载和请求验证。
- 把 package/import `runtime-smoke.sh` 当作设备执行验收：脚本未覆盖 device-op/synchronize/D2H 时只能创建 C1a，不能进入模型加载。
- 把 allocation 或 H2D 成功当作 DLC Runtime 健康：首个 device operation completion 必须独立验证。
- 短 prompt 通过后声明长上下文/并发/Chunked Prefill 通过：每个层级需要独立 evidence。
- OOM 后偷偷降低配置并报告成功：修改后的 deployment profile 是新的验证对象，必须单独记录。
- 只停止 Host 侧 tracked wrapper：`docker exec` 内的 APIServer/EngineCore 可能仍运行并占用 HBM，必须做跨 namespace 的进程和设备复检。
- 看到 `lsof -t /dev/cltech*` 空输出就认定设备空闲：权限和 mount namespace 会导致漏报，应与 `cltech_smi`、容器进程和端口交叉验证。
- 无条件运行 `lsof -t /dev/cltech* | xargs -r kill -9` 或历史软复位命令：前者可能误杀其他任务，后者的工具和作用范围可能已变化，必须通过当前 help、独占性、目标设备和授权安全门。
- 把 `num_prompts` 当作并发数：它是总请求数，实际峰值并发应查看 `Peak concurrent requests` 并结合 request rate。
- 只保存终端摘要：必须保留 client/server 原始日志、命令、contract 和健康检查，否则结果不可复现。
- 只跑一次并选最好结果：正式基线应重复测量并报告 median 与离散范围。
- 将不同输入/输出长度、request rate、TP 或 Chunked Prefill 配置直接比较：contract 不一致时不能判定性能回归。
- 把 `request-rate=inf` 当作业务 QPS：它是饱和压力 workload，不等价于线上流量。
- 把空 text 自动当成并发丢请求：先检查 HTTP/finish reason/token/服务错误并串行 replay；strict prompt correctness 与任意 prompt stop 行为分开判定。
- Recovery action exit 0 就声明恢复：必须由 fresh process 重跑最小失败 case；组合动作之间无 probe 时不能唯一归因。
- Current PCIe speed 低于 maximum 就声明硬件故障：没有 width 降级、AER 或相关 workload failure 时只能记为未定性 observation。

## 相关资料

- [bootstrap-git-from-configured-container.md](bootstrap-git-from-configured-container.md)
- [dlc-env-setup-environment-bootstrap.md](dlc-env-setup-environment-bootstrap.md)
- [vllm-dlc-fresh-image-to-model-adaptation.md](vllm-dlc-fresh-image-to-model-adaptation.md)
- [vllm-dlc-model-adaptation.md](vllm-dlc-model-adaptation.md)
- [../runtime-debugging/dlc-workstation-env-rebuild.md](../runtime-debugging/dlc-workstation-env-rebuild.md)
- [../debugging-workflows/post-install-runtime-smoke.md](../debugging-workflows/post-install-runtime-smoke.md)
- [../runtime-debugging/chipltech-smi-observability.md](../runtime-debugging/chipltech-smi-observability.md)
- [../testing/arsenal-ci-and-blackbox-testing.md](../testing/arsenal-ci-and-blackbox-testing.md)
- [../runtime-debugging/performance-profiling.md](../runtime-debugging/performance-profiling.md)

## CLTech-Init 来源边界

- 本模板中的 CLTech-Init 命令语义来自《CLTech-Init用户使用说明（原dlc-init）》：工具支持 DLC Chip 和 TYD Chip，可执行每日镜像拉取、硬件检测、驱动安装、`fix_hbm`、LYP 初始化和手动质量测试。
- 文档示例不是版本无关 API。实际执行必须先读取当前主机的 `cltech-init -v`、`cltech-init -h` 和 `/etc/chipltech/cltech-init.conf`。
- 原说明中的 `TPU` 在本知识库统一表述为 Chipltech-Family Accelerator；不能沿用该旧称泛指 DLC Chip 或 TYD Chip。
