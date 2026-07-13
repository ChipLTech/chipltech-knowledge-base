# 新容器中验证 `dlc-env-setup` skill 的调用与功能

面向“全新容器 / 全新工作站环境”的调用与验证手册。目标不是直接假设 DLC Ecosystem 依赖已经齐全，而是分两层验证：

1. **Kilo 暴露层验证**：确认 `skills` 仓库里的 `dlc-env-setup` 确实能被 Kilo 识别为 skill 或 `/dlc-env-setup` slash command。
2. **能力闭环验证**：确认 `chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md` 能在新环境里正确驱动 AI，做出与原始 `/work/test/dlc-env-setup` 同等级别的 repo discovery、停止判断、分阶段 rebuild 决策和最终验证。

**📖 阅读说明**：
- 下面的 shell 命令是你在容器终端执行的。
- `▶ 可复制 prompt` 代码块是你发给 Kilo 的内容。
- 新容器里如果还没有 DLC 相关仓库，验证结果应该是“正确停下并说明缺什么”，这也算通过，不算失败。

---

## 适用场景

- 你在一个全新的容器里刚拉下 `skills` 和 `chipltech-knowledge-base`。
- 你想验证 `dlc-env-setup` 是否真的能被 Kilo 调用，而不是只存在于仓库文件里。
- 你想验证 `dlc-env-setup-skill-usage.md` 这个 prompt example 在新环境里是否真能工作。

## 目标仓库

本手册默认你会准备两个仓库：

- `skills`
- `chipltech-knowledge-base`

下面示例统一假设它们位于：

```text
/work/skills
/work/chipltech-knowledge-base
```

如果你的新容器路径不同，把命令中的路径替换掉即可。

---

## 第一步：在新容器里拉下两个仓库

如果新容器里还没有仓库，先执行：

```bash
cd /work
git clone git@github.com:ChipLTech/skills.git
git clone git@github.com:ChipLTech/chipltech-knowledge-base.git
```

然后确认两个目录都存在：

```bash
ls /work/skills
ls /work/chipltech-knowledge-base
```

---

## 第二步：把 `dlc-env-setup` 暴露给 Kilo

进入 `skills` 仓库根目录后执行：

```bash
cd /work/skills
./scripts/link-kilo-skills.sh --with-commands
```

这个命令会做两件事：

1. 在 `~/.config/kilo/skills/` 下创建 skill symlink
2. 在 `~/.config/kilo/command/` 下创建 slash command wrapper

对 `dlc-env-setup` 来说，目标结果应该是：

```text
~/.config/kilo/skills/dlc-env-setup
~/.config/kilo/command/dlc-env-setup.md
```

---

## 第三步：验证 Kilo 暴露层是否成功

### 3.1 验证 skill symlink

```bash
test -L ~/.config/kilo/skills/dlc-env-setup && readlink -f ~/.config/kilo/skills/dlc-env-setup
```

期望输出应指向：

```text
/work/skills/skills/engineering/dlc-env-setup
```

### 3.2 验证 `SKILL.md`

```bash
test -f ~/.config/kilo/skills/dlc-env-setup/SKILL.md && sed -n '1,20p' ~/.config/kilo/skills/dlc-env-setup/SKILL.md
```

期望看到 frontmatter 里至少包含：

```text
name: dlc-env-setup
description: Rebuild and verify a workstation DLC toolchain...
```

### 3.3 验证 slash command wrapper

```bash
test -f ~/.config/kilo/command/dlc-env-setup.md && sed -n '1,20p' ~/.config/kilo/command/dlc-env-setup.md
```

期望看到类似：

```md
---
description: Rebuild and verify a workstation DLC toolchain...
---

请使用 `dlc-env-setup` skill，严格按它的流程处理下面的问题：

$ARGUMENTS
```

### 3.4 如果这一步失败，怎么判断问题在哪

- `~/.config/kilo/skills/dlc-env-setup` 不存在：说明链接脚本没成功跑完，或者你没有在 `skills` 仓库根目录执行。
- `~/.config/kilo/command/dlc-env-setup.md` 不存在：说明你没加 `--with-commands`。
- symlink 存在但没指向 `skills/engineering/dlc-env-setup`：说明仓库结构或脚本行为异常，应先停在这里排查，不要继续验证 prompt example。

---

## 第四步：重启 Kilo 或开启新 session

Kilo 在旧 session 中可能仍缓存旧的 skill 列表。完成链接后，建议：

1. 重启 Kilo
2. 或至少打开一个全新的 session

不要在已经加载过旧 skill 列表的 session 中直接判断失败。

---

## 第五步：做最小调用验证

目标：先验证 `/dlc-env-setup` 或 `dlc-env-setup` 这个名字本身能被 Kilo 正确调起，而不是一上来就做全量环境重建。

### 方式 A：验证 slash command

**▶ 可复制 prompt：**

```text
/dlc-env-setup 先不要改系统环境，也不要开始重建。请先说明这个 skill 会怎么工作：包括 repo discovery、branch 安全检查、分阶段 rebuild、PyTorch wheel、可选 vllm repair、以及 final runtime smoke。只输出流程说明和停止条件。
```

### 方式 B：验证自然语言 skill 调用

如果你没有生成 command wrapper，可以用：

**▶ 可复制 prompt：**

```text
请使用 `dlc-env-setup` skill。先不要改系统环境，也不要开始重建。请先说明这个 skill 会怎么工作：包括 repo discovery、branch 安全检查、分阶段 rebuild、PyTorch wheel、可选 vllm repair、以及 final runtime smoke。只输出流程说明和停止条件。
```

### 这一层的通过标准

如果 Kilo 的回答明显体现了下面这些点，就说明 skill 已经被正确调起：

- 先 rediscover repos，不默认 `/home/workspace`
- branch 切换前先做 `git status --short` 等安全检查
- build 顺序包括 `LLVM`、`DLC_Custom_Kernel`、PyTorch wheel、可选 `vllm`
- 强调 `USE_NUMPY` 和源码树外 smoke
- 知道某些情况必须停下

如果 Kilo 只是泛泛地说“我来帮你安装环境”，没有这些具体结构，说明 skill 可能没有真正被调用。

---

## 第六步：验证 `dlc-env-setup-skill-usage.md` 的调用效果

目标：确认知识库里的 prompt example 在新容器里可直接驱动 Kilo 调用正确的 skill 名称和流程。

先打开这份文件：

```text
/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md
```

### 6.1 先做“缺仓库环境”的停止验证

如果新容器里只有 `skills` 和 `chipltech-knowledge-base`，并没有 `cmake-3.27.0`、`dlc-thunk`、`DLCsim`、`DLCSynapse`、`DLC_CL`、`LLVM`、`DLC_Custom_Kernel`、`pytorch` 等仓库，那么正确行为不是瞎编路径，而是停下来说明缺了哪些仓库。

**▶ 可复制 prompt：**

```md
请使用 `dlc-env-setup` skill，按 `/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md` 的“全量或分阶段重建”模板工作。

【模式】全量重建
【搜索根目录】/work,$HOME
【需要切 branch 的仓库】无
【是否包含 vllm / vllm-dlc】否
【是否允许修改 /usr/local】否

先不要假设 `/home/workspace`。先做 repo discovery、回显 repo map，并在缺仓库时正确停止。
```

### 这一层的通过标准

如果 Kilo：

- 先去做 repo discovery
- 明确回显找到了哪些目录
- 明确指出哪些关键 repo 缺失
- 在缺失关键 repo 时停下来，而不是乱编路径或假装继续 build

那就说明 `dlc-env-setup-skill-usage.md` 的“调用入口 + 停止条件”在新环境里是有效的。

---

## 第七步：在有最小假数据时做“分阶段模式”验证

如果你想进一步验证 prompt example 是否真的能覆盖 partial rebuild 逻辑，但新容器里又没有全套 DLC 仓库，可以造一个轻量级目录骨架，只验证流程判断，不验证真正编译。

例如：

```bash
mkdir -p /tmp/dlc-smoke/{cmake-3.27.0,dlc-thunk,DLCsim,DLCSynapse,DLC_CL,LLVM,DLC_Custom_Kernel,pytorch}
touch /tmp/dlc-smoke/dlc-thunk/compile.sh
touch /tmp/dlc-smoke/DLCsim/build.sh
touch /tmp/dlc-smoke/DLCSynapse/compile.sh
touch /tmp/dlc-smoke/DLC_CL/build.sh
touch /tmp/dlc-smoke/LLVM/build.sh
touch /tmp/dlc-smoke/DLC_Custom_Kernel/CMakeLists.txt
touch /tmp/dlc-smoke/pytorch/setup.py
```

然后发给 Kilo：

**▶ 可复制 prompt：**

```md
请使用 `dlc-env-setup` skill，按 `/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md` 的模板工作，但这次只做流程级验证，不要真的跑长构建命令。

【模式】只重建 PyTorch wheel
【搜索根目录】/tmp/dlc-smoke
【需要切 branch 的仓库】无
【是否包含 vllm / vllm-dlc】否
【是否允许修改 /usr/local】否

请验证 repo discovery、shell 变量映射、partial rebuild safety gate、以及你会要求哪些前置依赖存在。不要真的开始 build。
```

### 这一层的通过标准

如果 Kilo 能明确说出：

- 这是 `只重建 PyTorch wheel`
- 因此必须先确认更早阶段依赖存在
- 需要回显 shell 变量映射
- 如果前置条件不成立就应回退到更早阶段

那说明 prompt example 已经把原始 skill 的 partial rebuild 语义传达出来了。

---

## 第八步：在真正有 DLC 仓库的环境里做完整功能验证

只有当新容器里真实具备这些仓库与依赖时，才适合验证完整功能：

- `cmake-3.27.0`
- `dlc-thunk`
- `DLCsim`
- `DLCSynapse`
- `DLC_CL`
- `LLVM`
- `DLC_Custom_Kernel`
- `pytorch`
- 可选：`vllm`、`vllm-dlc`

这时可以直接使用 `dlc-env-setup-skill-usage.md` 里的主模板，或者更短地用：

**▶ 可复制 prompt：**

```text
/dlc-env-setup 请按 /work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md 的“全量或分阶段重建”模板工作。

【模式】<全量重建 / 从 LLVM 开始 / 从 DLC_Custom_Kernel 开始 / 只重建 PyTorch wheel / 只重装 wheel / 修 vllm>
【搜索根目录】<真实 repo 根目录>
【需要切 branch 的仓库】<例如 LLVM:feature-x, pytorch:release/2.5-dlc；没有就写“无”>
【是否包含 vllm / vllm-dlc】<是/否>
【是否允许修改 /usr/local】<是/否>
```

### 完整功能验证的通过标准

如果 Kilo 最终能够：

- 回显正确 repo map
- 正确构造 shell 变量映射
- 在 branch 切换前做 git 安全检查
- 按依赖顺序推进
- 在 PyTorch 阶段检查 `USE_NUMPY`
- 在源码树外做 runtime smoke
- 在 `vllm` 路线处理 `UNKNOWN` / `triton` / wheel path

那就说明这份 prompt example 在新环境里已经实现了与 `/work/test/dlc-env-setup` 高度等价的功能闭环。

---

## 快速判断表

| 验证层级 | 你在验证什么 | 新容器里缺 DLC 仓库时应有的结果 | 通过标准 |
|----------|--------------|----------------------------------|----------|
| 暴露层 | Kilo 能否看到 `dlc-env-setup` | skill / command 文件能被创建 | `~/.config/kilo/skills/dlc-env-setup` 和 `~/.config/kilo/command/dlc-env-setup.md` 存在 |
| 调用层 | `/dlc-env-setup` 或 `dlc-env-setup` 是否真正触发该 skill | Kilo 按 skill 结构回答流程 | 回答里出现 repo discovery、branch 检查、PyTorch / vLLM / smoke 结构 |
| 停止层 | prompt example 是否会在缺仓库时正确停下 | 正确报缺哪些 repo | 不乱编路径，不假装继续 |
| 语义层 | partial rebuild 安全门是否生效 | 即使不 build，也能说对前置依赖规则 | 能明确解释 `只重建 wheel` / `只重建 PyTorch` 的 gate |
| 完整功能层 | 全量环境重建是否可执行 | 只有在真实 DLC 环境中才能测 | 能走完 repo discovery -> rebuild -> smoke 闭环 |

## 相关资料

- [dlc-env-setup-skill-usage.md](dlc-env-setup-skill-usage.md)
- [../runtime-debugging/dlc-workstation-env-rebuild.md](../runtime-debugging/dlc-workstation-env-rebuild.md)
- [../debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md](../debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md)
- [../debugging-workflows/post-install-runtime-smoke.md](../debugging-workflows/post-install-runtime-smoke.md)
- `/work/skills/README.zh-CN.md`
- `/work/skills/scripts/link-kilo-skills.sh`

## 来源

- `/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md`
- `/work/skills/README.zh-CN.md`
- `/work/skills/scripts/link-kilo-skills.sh`
- `/work/test/dlc-env-setup/SKILL.md`
