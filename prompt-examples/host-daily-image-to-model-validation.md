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
- Host 驱动、设备初始化和 LYP 操作不属于普通环境适配。驱动重载、`cltech-init`/`dlc-init`、软重置、LYP repair、kill 非本任务进程和 reboot 必须另获明确授权。
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
【期望镜像 digest】<sha256:...；没有则写“拉取后记录实际 digest，不做预声明校验”>
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
【容器资源】<shm-size、ipc、pid、memlock/stack ulimit、CPU、内存>
【允许安装系统包】<是/否>
【允许修改 /usr/local】<是/否>
【允许的宿主机/设备操作】<默认“无”；逐项列出明确授权>
【artifact 目录】<Host 工作根目录下且位于 vllm-dlc 源码树外>
【容器结束策略】<保持运行 / 正常停止；默认保持运行便于复查>

按以下阶段执行。每个阶段先写 evidence，再进入下一阶段；失败后从最近通过的 checkpoint 恢复，不要从头盲目重做。

阶段 0：Host preflight、镜像固定和容器创建

1. 只读检查 Host：
   - 记录 OS、kernel、architecture、当前用户、Docker client/server 版本、存储空间和目标目录 owner/mode。
   - 记录 `docker info` 是否健康、目标容器名是否已存在、每日镜像本地是否存在。
   - 只读记录 `/dev/dlc*`、`/mnt/jfs`、`/sys`、`/lib/modules` 和 `/var/log` 的可见性；不要因为缺失而执行驱动初始化。
   - 记录 Host 当前设备观测结果和 HBM 占用。如目标设备正在被其他任务使用，停止并报告，不 kill 或抢占。

2. 准备持久目录：
   - 在【Host 工作根目录】下使用独立目录保存 `src/`、`build/`、`wheels/`、`artifacts/`、`logs/` 和必要 cache。
   - 新建前确认父目录存在且正确；不覆盖同名非本任务目录。
   - artifact 目录必须在所有业务源码树外。创建 run manifest，记录 run ID、开始时间、Host、镜像引用、目标模型和 deployment profile，不写 secret。

3. 固定每日镜像：
   - 使用 `docker pull <每日镜像引用>` 获取用户指定镜像；未授权联网或 pull 时只使用已确认的本地镜像。
   - pull 后记录 image ID、repo digest、created time、architecture 和基础 image metadata。
   - 若提供【期望镜像 digest】，实际 digest 不一致时停止。不要继续使用 tag 当前指向的其他镜像。
   - 不修改或覆盖该 tag，不执行 `docker image prune`。

4. 生成并回显容器运行契约后再创建容器：
   - bind mount 持久 `src/build/wheels/artifacts/logs` 到【容器内工作根目录】下对应路径。
   - 只读挂载模型目录；需要写 tokenizer/cache 时使用单独可写 cache，不修改模型资产。
   - 按用户声明挂载 `/mnt/jfs`、`/dev`、`/sys`、`/lib/modules`、`/var/log`，并设置 ipc/pid/shm/ulimit；不要无理由使用 `--privileged`。
   - 仅暴露 serving 所需端口；端口已占用时选择用户批准的替代端口并记录映射。
   - 设置明确的 `DLC_VISIBLE_DEVICES`，不要把“容器可见全部设备”误当作任务可用全部设备。
   - 使用可保持运行且可 exec 的入口启动容器。容器已存在时先比较 image ID、mount、environment、device、port 和资源配置；一致则复用，不一致则停止，不自动删除或重建。

阶段 0 验收：容器处于 running，image digest 与 run manifest 一致，持久目录可写，模型目录只读可见，目标设备与资源契约符合声明。保存 `docker inspect` 的脱敏结果。

阶段 1：目标容器 Git/SSH 与 DLC Ecosystem 环境适配

5. 先检查容器内 Git/SSH：
   - 如果 Git、SSH identity 和目标私有仓库访问都已通过，记录版本与身份后跳过迁移。
   - 如果缺失，严格使用 `bootstrap-git-from-configured-container.md`，以【源容器】为来源并遵守【Git/SSH 私钥迁移授权】。
   - 不输出 key，不复制 credential store，不把 SSH 文件放进持久源码目录或 image layer。
   - 通过 `git ls-remote`、临时 clone、fetch 和 `git push --dry-run` 验证。只验证读权限但未验证写权限时必须明确标记。

6. 使用 `dlc-env-setup` 做 repo discovery 和 bootstrap：
   - 不假设固定源码路径；在持久 `src/` 中发现或 clone `dlc-thunk`、`DLCsim`、`DLCSynapse`、`DLC_CL`、`LLVM`、DLC_Custom_Kernel Repository、`pytorch`、`vllm` 和 `vllm-dlc`。
   - 对每个仓库记录 Git root、remote、branch/tag、full HEAD 和 `status --short`。
   - 已有 dirty 工作树时停止，不 stash、不 reset、不 clean、不覆盖。
   - 只按【版本策略】和【批准的仓库 remote/ref】同步。remote/ref 不明确或多个候选路径冲突时停止。
   - 检查 CMake 必须严格大于 `3.27.0`，并检查 compiler、Python、pip、wheel、NumPy bridge 和构建入口。
   - 按当前 `dlc-env-setup` skill 的依赖顺序进行必要的最小 rebuild/reinstall，不因单项失败跳到后续组件。
   - wheel、build log 和环境报告写入 Host 持久目录，不只保存在容器 writable layer。

7. 运行阶段 1 验证：
   - `pytorch-preflight.sh`
   - `vllm-preflight.sh`
   - `runtime-smoke.sh <源码树外临时目录>`
   - PyTorch 2.5.0、NumPy bridge、DLC Platform、`vllm` 和 `vllm-dlc` import/package metadata 检查。
   - 记录当前源码 checkout 与实际 import 的 package 路径，防止误用旧 wheel 或其他 editable install。

阶段 1 验收：所有必需 preflight/import/runtime smoke 通过，repo map 可审计，实际 import 指向预期环境，设备在容器中可见。只有该 checkpoint 通过后才能加载模型。

阶段 2：模型身份检查、服务启动和功能验证

8. 模型 preflight：
   - 验证【模型绝对路径】存在且是批准资产，记录 config、tokenizer/processor 文件和 revision；不输出模型敏感内容。
   - 检查模型架构、dtype、quantization metadata、所需设备数、估算 HBM 和 deployment profile 是否匹配。
   - 量化模型核对 `quant_method`、`bits`、`group_size`、`zero_point` 与可用 kernel 路由；不能根据目录名猜测 AWQ、AWQ-Marlin、W8A16 或 compressed-tensors 兼容性。
   - 资源、revision、processor 或 manifest 身份缺失时，在模型加载前报告 `blocked_missing_asset` 或 `blocked_missing_hardware`。

9. 启动 serving：
   - 使用 tracked background process 或容器内可追踪进程启动，不使用失管的 `nohup`/`&`。
   - 将完整启动命令的脱敏版本、environment、PID、端口和日志路径写入 run manifest。
   - 请求中的 `model` 必须等于【served model name】；启动未设置 alias 时才使用启动契约规定的模型名。
   - 等待明确 readiness 日志并做 health/model-list 检查。超时后保存日志，不重复启动第二个竞争实例。

10. 按阶梯验证，每层通过后才进入下一层：
   - L0 liveness：health endpoint、model list、server PID 和设备可见性正常。
   - L1 短 prompt：`temperature=0`、`top_p=1.0`、`max_tokens=64` 或 `128`，确认 HTTP 成功、非空输出、finish reason 和 server liveness。
   - L2 中 prompt：保持 deterministic 参数，只增加 prompt 长度，记录输入 token、输出 token、首 token latency、总 latency 和输出摘要。
   - L3 长 prompt/one-shot：每轮只改变一个变量，记录 prompt token 数、decode 参数、Chunked Prefill 选择、输出、finish reason、latency 和日志位置。
   - L4 可选扩展：并发、多卡、TP/PP/EP、MoE、量化、vision/multimodal 和长上下文必须单独授权、单独记录，不能由 L1/L2 推断通过。

11. 每层做前后健康检查：
   - 记录 server 是否仍存活、目标设备 HBM、关键错误日志和请求关联 ID。
   - 发现空输出、重复符号、timeout、异常截断、NaN/Inf、进程退出或明显降速时，停止提升层级，保存最小失败输入。
   - 不用一个失败层级覆盖前面已通过的 bounded 结论，也不把前面通过解释为失败层级已通过。

问题诊断与恢复规则

12. 镜像/容器问题：
   - pull 失败：区分 registry auth、DNS/TLS、tag 不存在和磁盘空间；不关闭 TLS，不切换到未经批准的镜像。
   - 容器立即退出：检查 entrypoint、command、architecture 和动态库，不用无限 restart loop 掩盖错误。
   - mount 缺失或权限错误：比较 Host 路径 owner/mode、容器 UID/GID 和 inspect 配置；不要递归 chmod 777。
   - 配置契约错误时保留失败容器和 manifest，使用新名称创建修正容器；未确认持久数据前不删除旧容器。

13. Git/源码问题：
   - SSH 失败按 key owner/mode、有效 `ssh -G`、host key、port/proxy 和 GitHub identity 顺序定位，禁止打印私钥。
   - remote/ref 不匹配、dirty tree、detached HEAD 未批准或 submodule 失败时停止，不 reset/clean 强行通过。
   - 网络暂时失败只重试幂等 fetch/pull，限制次数并保留原始错误；认证失败不循环重试。

14. 构建/import 问题：
   - 保留首个根因错误和完整 log；不要只报告最后一行。
   - 比较 build 使用的 Python/pip/CMake/compiler 与 runtime import 环境，检查旧 wheel、editable install、`PYTHONPATH` 和动态库路径污染。
   - 只重建失败组件及当前 skill 要求的下游组件；上游健康证据不足时回到阶段 1 safety gate。

15. serving/模型问题：
   - OOM：先核对实际 HBM、并发、TP、`max_model_len`、batching 和 cache 配置；未经批准不自动缩小 deployment profile 后声称原 profile 通过。
   - API 404/model mismatch：核对 `--served-model-name`、请求 endpoint 和 model 字段。
   - timeout/hang：保存进程状态、请求、日志、设备观测和最后通过 checkpoint。`peek_stuck.sh`、软重置、LYP repair、kill 非本任务进程或 reboot 需明确授权。
   - 输出异常：建立相同模型、tokenizer、prompt、decode 参数和 token budget 的等价对照；多步分叉可缩成“分叉前 token + 单步 decode”以隔离 KV cache/scheduler 变量。
   - 每次修复只改变一个变量，生成新的 attempt ID；成功后重跑导致失败的最小 case 和所有较低层级 smoke。

阶段 3：收尾和交付

16. 正常停止本任务启动的 serving 进程，确认释放 HBM。不要停止其他用户或其他任务进程。
17. 按【容器结束策略】保持容器运行或正常停止；不默认删除容器，不执行 `docker commit` 或 prune。
18. 最终报告必须包含：
   - image tag、image ID、repo digest、容器名和脱敏后的运行契约。
   - Host 持久目录、mount、设备选择和资源配置。
   - Git/SSH 验证状态及所引用的 bootstrap 报告，不包含 secret。
   - repo map、full HEAD、环境构建/reinstall 和三个 preflight/smoke 结果。
   - 模型身份、deployment profile、每个验证层级的 PASS/FAIL/BLOCKED、请求摘要和日志路径。
   - 每次 failure 的根因或当前最小边界、采取的单变量修复、重验结果和最后通过 checkpoint。
   - 未验证范围：未实际执行的长上下文、并发、多卡、量化、Chunked Prefill runtime、DLC Runtime dispatch 或 Real DLC Hardware acceptance 必须标记 `not_verified`。

结论用语限制：
- L1/L2 通过只能称为对应 image、checkout、模型和 deployment profile 的 operational smoke 通过。
- 不得仅凭 HTTP 200、短 prompt、Dummy、DLCsim 或静态检查声称 Verified vLLM Alignment、权威 Chunked Prefill acceptance 或 Real DLC Hardware acceptance。
```

## 阶段 Checkpoint

| Checkpoint | 最小通过条件 | 可恢复入口 |
|------------|--------------|------------|
| C0 Container Ready | digest、mount、设备和持久目录正确 | 从 Git/SSH 检查继续 |
| C1 Environment Ready | repo map、preflight、import 和 runtime smoke 通过 | 从模型 preflight 继续 |
| C2 Server Ready | 模型加载、health 和 model list 通过 | 从 L1 请求继续 |
| C3 Short Smoke | deterministic 短 prompt 非空且服务健康 | 从 L2/L3 继续 |
| C4 Requested Scope Done | 用户要求的最高层级完成 | 收尾与报告 |

## 常见误区

- 把每日 tag 当作固定版本：tag 会移动，必须记录 digest 和 image ID。
- 只在容器 writable layer 保存源码和日志：容器重建后 evidence 丢失，应使用 Host bind mount。
- 为方便把 Host 整个 `~/.ssh` 挂入容器：扩大凭据暴露面，应使用 Git bootstrap 的最小文件迁移策略。
- 容器启动失败就删除重建：先保留 inspect、日志和 manifest，否则失去根因证据。
- 环境 smoke 通过后直接声明模型通过：必须完成真实模型加载和请求验证。
- 短 prompt 通过后声明长上下文/并发/Chunked Prefill 通过：每个层级需要独立 evidence。
- OOM 后偷偷降低配置并报告成功：修改后的 deployment profile 是新的验证对象，必须单独记录。

## 相关资料

- [bootstrap-git-from-configured-container.md](bootstrap-git-from-configured-container.md)
- [dlc-env-setup-environment-bootstrap.md](dlc-env-setup-environment-bootstrap.md)
- [vllm-dlc-fresh-image-to-model-adaptation.md](vllm-dlc-fresh-image-to-model-adaptation.md)
- [vllm-dlc-model-adaptation.md](vllm-dlc-model-adaptation.md)
- [../runtime-debugging/dlc-workstation-env-rebuild.md](../runtime-debugging/dlc-workstation-env-rebuild.md)
- [../debugging-workflows/post-install-runtime-smoke.md](../debugging-workflows/post-install-runtime-smoke.md)
