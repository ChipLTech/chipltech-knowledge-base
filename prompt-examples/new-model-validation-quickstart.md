# 新模型适配与验证傻瓜版 Prompt

## 用途

只需要填写模型名称和模型目录，即可让 Agent 从 skills 安装开始，自动完成 Host 每日镜像、容器、Git/SSH、DLC Ecosystem 环境、新模型功能 smoke 和 `vllm bench serve` 服务端到端 benchmark。

本示例不绑定个人 HOME，也不假设仓库一定在 `/work`。Agent 必须先自动发现实际路径，并通过变量保存权威根目录。

## 最少填写

```text
模型名称：<例如 Qwen3-32B>
模型目录：<模型绝对路径，例如 /mnt/jfs/models/Qwen3-32B>
```

## 可直接复制 Prompt

```md
请对下面的新模型执行完整的 DLC Platform 环境适配、功能验证和服务端到端 benchmark。请直接执行，不要只给计划。

【模型名称】<MODEL_NAME>
【模型目录】<MODEL_ABSOLUTE_PATH，例如 /mnt/jfs/models/MODEL_NAME>

除上面两项外，其余路径、镜像、容器、仓库、端口、设备和参数请先自动发现，再按安全规则选择。不得猜测模型 revision、量化类型、设备能力或仓库身份。

第一步：发现或准备 `skills` 和 `chipltech-knowledge-base`

1. 从当前目录、当前 Git worktree、常用工作根目录和 `$HOME` 中自动发现：
   - `skills` 仓库：必须包含 `scripts/link-kilo-skills.sh`、`skills/engineering/dlc-env-setup/SKILL.md` 和 `skills/engineering/model-adaptation/SKILL.md`。
   - `chipltech-knowledge-base` 仓库：必须包含 `CONTEXT.md` 和 `prompt-examples/host-daily-image-to-model-validation.md`。
2. 对每个候选记录 Git root、remote、branch、full HEAD 和 `status --short`。同名候选有多个、remote 不匹配或权威路径不明确时停止，不要随意选择。
3. 将实际路径保存为：
   - `SKILLS_ROOT=<自动发现的 skills Git root>`
   - `KNOWLEDGE_BASE_ROOT=<自动发现的 chipltech-knowledge-base Git root>`
4. 如果仓库缺失：
   - 先确认当前 Host 是否已有 Git/SSH 和 ChipLTech 私有仓库访问能力。
   - Git/SSH 缺失时，按 `$KNOWLEDGE_BASE_ROOT/prompt-examples/bootstrap-git-from-configured-container.md` 的安全规则补齐；如果知识库本身也缺失，则使用用户提供的受信来源和同等安全边界，不输出或提交私钥/token。
   - 只有用户已授权 clone 时，才把缺失仓库 clone 到一个已确认存在、可写且持久的工作根目录。优先使用 SSH remote：
     - `git@github.com:ChipLTech/skills.git`
     - `git@github.com:ChipLTech/chipltech-knowledge-base.git`
   - 不覆盖已有同名目录，不在 remote/ref 不明确时继续。

第二步：安装并验证 Kilo skills

5. 先读取：
   - `$KNOWLEDGE_BASE_ROOT/CONTEXT.md`
   - `$SKILLS_ROOT/README.zh-CN.md`
   - `$SKILLS_ROOT/skills/engineering/dlc-env-setup/SKILL.md`
   - `$SKILLS_ROOT/skills/engineering/model-adaptation/SKILL.md`
   - `$KNOWLEDGE_BASE_ROOT/prompt-examples/host-daily-image-to-model-validation.md`
6. 默认执行当前用户的全局安装，让所有项目可用：

   ```bash
   "$SKILLS_ROOT/scripts/link-kilo-skills.sh" --with-commands
   ```

   不要使用 `--all`；默认 stable 集合已经包含 `dlc-env-setup` 和 `model-adaptation`。
7. 如果用户明确要求只给当前项目安装，则改用：

   ```bash
   "$SKILLS_ROOT/scripts/link-kilo-skills.sh" --project "$KNOWLEDGE_BASE_ROOT" --with-commands
   ```

8. 安装脚本不得覆盖已有普通 skill 目录或用户自定义 command。遇到冲突时报告文件类型、路径和来源，不要删除用户文件。
9. 验证安装结果：
   - 全局安装根目录为 `${XDG_CONFIG_HOME:-$HOME/.config}/kilo`。
   - 项目级安装根目录为 `$KNOWLEDGE_BASE_ROOT/.kilo`。
   - `skills/dlc-env-setup` 和 `skills/model-adaptation` 必须是指向 `$SKILLS_ROOT/skills/engineering/...` 的 symlink。
   - 两个 symlink 下的 `SKILL.md` 必须存在。
   - `command/dlc-env-setup.md` 和 `command/model-adaptation.md` 必须存在，并包含对应 skill 调用和 `$ARGUMENTS`。
10. 如果 skills 是本次新安装的，说明 Kilo 可能缓存 skill 列表。能够在当前 session 直接读取维护源时继续按维护源执行；同时记录“新 Kilo session/重启后 slash command 才保证可见”。不要假装当前 session 已重新加载。

第三步：执行新模型完整验证

11. 严格使用：

   ```text
   $KNOWLEDGE_BASE_ROOT/prompt-examples/host-daily-image-to-model-validation.md
   ```

   作为 Host 到模型验证的执行权威；先使用 `dlc-env-setup` 完成环境阶段，环境 checkpoint 通过后再使用 `model-adaptation` 处理该模型。
12. 自动发现并复用合适的每日镜像、持久容器、Git/SSH、DLC Ecosystem 环境、`vllm`/`vllm-dlc` 源码和 artifact 根目录。复用前必须核对 image digest、容器运行契约、仓库 full SHA、dirty status、Python package import path、设备状态和 HBM。
13. 检查模型目录中的 config、权重、tokenizer、processor、generation config、dtype、quantization、架构、最大上下文和资产完整性。模型目录不存在或资产不完整时返回 `blocked_missing_asset`，不要下载或替换成同名模型。
14. 根据模型配置、权重大小、可用设备和 HBM 选择保守的功能 profile，明确记录：
   - `DLC_VISIBLE_DEVICES`
   - TP/PP/EP
   - dtype/quantization
   - `max_model_len`
   - `max_num_batched_tokens`
   - `max_num_seqs`
   - block size、Chunked Prefill、prefix caching、端口
15. 依次执行并保存 evidence：
   - Host/容器/设备 preflight
   - Git/repo map 和 package identity
   - `pytorch-preflight.sh`
   - `vllm-preflight.sh`
   - `runtime-smoke.sh <源码树外目录>`
   - 模型加载、health、`/v1/models`
   - deterministic 短 prompt
   - deterministic 中 prompt
16. 功能 smoke 通过后，建立独立 benchmark profile。功能 profile 如果使用 `max_num_seqs=1`，不得直接当作饱和并发 benchmark profile；变更 `max_num_seqs` 后必须重启服务并重跑 health 和短 prompt。
17. 默认服务端到端 benchmark contract：
   - `vllm bench serve`
   - backend/endpoint：按当前服务支持的 OpenAI-compatible completions endpoint
   - dataset：random
   - requested input/output length：256/256
   - num prompts：150
   - request rate：`inf`
   - seed：1024
   - temperature：0
   - warm-up：2 requests
   - 正式 attempts：3；资源或时间仅允许一次时必须标记 `single-run observation`，不能称稳定基线
18. 执行前读取当前 `vllm bench serve --help=all`，只使用本版本支持的参数。支持时启用 `--save-result`、`--save-detailed`、`--result-dir` 和 `--result-filename`。
19. artifact 根目录优先复用当前持久 workspace 的 `artifacts/`；无法可靠发现时使用 `<持久工作根目录>/artifacts`，不得写入 `vllm` 或 `vllm-dlc` 源码树。推荐结构：

   ```text
   <ARTIFACT_ROOT>/benchmarks/<date>/<model>/<case>/<attempt>/
   ```

20. 保存：manifest、server/client command、server/client log、原始 JSON、metrics、health-before/after、`summary.md` 和 `metrics.csv`。同时记录 requested token length 与 JSON 中实际 token length。

第四步：失败恢复和安全边界

21. 遇到问题时按 `host-daily-image-to-model-validation.md` 的 checkpoint、诊断和恢复规则处理，保留第一个根因和失败 server epoch。每次只改变一个变量，修复后重跑失败 case 和所有较低层级 smoke。
22. 不覆盖 dirty Git 工作树，不执行未经批准的驱动更新、`fix_hbm`、LYP 初始化、掉电上电、软/硬重置或 reboot。
23. 只处理本任务启动的进程。Host tracked wrapper 退出不代表容器 APIServer/EngineCore 已退出；收尾时交叉检查 Host/容器 PID、监听端口、`cltech_smi` 和 HBM。
24. 残留进程优先按已确认的精确 PID 执行 TERM，再按需 KILL。以下命令不是默认清理命令：

   ```bash
   lsof -t /dev/cltech* | xargs -r kill -9
   cltechpd_clnt -s
   ```

   第一条只有在独占维护窗口、审计全部 PID 归属并明确授权后使用；第二条是设备软重置，仅在任务进程清理后 HBM 仍未释放且明确授权时使用。软重置后必须重跑硬件、LYP、SMI、设备映射和 runtime smoke。

第五步：最终交付

25. 完成后停止本任务模型服务，确认 APIServer、EngineCore、监听端口均退出，并确认目标设备 HBM 已释放。保持用户要求保留的持久容器，不执行 `docker commit` 或 prune。
26. 最终报告必须包含：
   - 自动发现的 `SKILLS_ROOT`、`KNOWLEDGE_BASE_ROOT` 和安装 scope
   - skills 安装/验证结果，以及当前 session 是否需要重启才能看到 slash commands
   - image/container/repo/package/model identity
   - 实际功能 profile 和 benchmark profile
   - 短、中 prompt 结果
   - 每个 benchmark attempt 的成功/失败请求、吞吐、TTFT、TPOT、ITL 和 Peak concurrent requests
   - artifact 根目录和关键文件路径
   - 遇到的问题、根因、单变量修复和重验结果
   - 已验证、`not_verified`、`not_applicable` 和 blocked 范围

结论边界：短 prompt、HTTP 200 或单次 benchmark 通过不能声称 Verified vLLM Alignment、长上下文、权威 Chunked Prefill acceptance、request-correlated DLC Runtime dispatch 或广泛的 Real DLC Hardware model acceptance。
```

## 填写示例

```text
请使用 `chipltech-knowledge-base/prompt-examples/new-model-validation-quickstart.md` 中的“可直接复制 Prompt”，对下面模型执行完整验证：

模型名称：Qwen3-32B
模型目录：/mnt/jfs/models/Qwen3-32B

除模型名称和目录外，其余路径和参数请自动发现。请直接执行，不要只给计划。
```

## 什么时候需要补充信息

只有以下场景建议额外提供：

- 需要与历史性能严格比较：提供同 contract 的 benchmark artifact 路径和回归阈值。
- 模型需要多卡：提供允许使用的物理设备和 TP/PP/EP 目标。
- 量化或 multimodal 模型：提供批准的 quantization/processor revision。
- 必须固定代码版本：提供 `vllm`、`vllm-dlc` 和模型的批准 full SHA/revision。
- 允许 Host/设备变更：逐项批准驱动、`fix_hbm`、LYP 或软重置，不能只写“都允许”。

## 相关资料

- [host-daily-image-to-model-validation.md](host-daily-image-to-model-validation.md)
- [bootstrap-git-from-configured-container.md](bootstrap-git-from-configured-container.md)
- [dlc-env-setup-fresh-container-validation.md](dlc-env-setup-fresh-container-validation.md)
- [vllm-dlc-model-adaptation.md](vllm-dlc-model-adaptation.md)
