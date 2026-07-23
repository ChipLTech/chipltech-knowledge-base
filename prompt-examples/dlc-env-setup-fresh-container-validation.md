# 新容器中验证 `dlc-env-setup` skill 的安装、识别与工作流

面向全新容器或全新工作站环境的验证手册。验证以当前 `/work/skills` 主线中维护的 `dlc-env-setup` skill、安装脚本和稳定 skill 集合为准，并遵循知识库 `CONTEXT.md` 的 DLC Ecosystem 术语；commit `3f04504` 只作为最低兼容基线，不覆盖后续维护源。

验证分为四层：

1. **来源验证**：确认 `/work/skills` 位于团队当前主线；若做最低兼容验证，确认当前 commit 包含 `3f04504`。
2. **安装与暴露验证**：确认全局或项目级 skill symlink 和 slash command wrapper 正确生成，并清理旧别名。
3. **识别与工作流理解验证**：确认 Kilo 识别 `dlc-env-setup`，并依据当前 `SKILL.md` 说明工作流和停止条件。
4. **执行验证**：在真实 DLC Ecosystem 仓库和依赖齐全时，验证 rebuild、安装和源码树外 smoke 闭环。

**阅读说明**：

- shell 代码块可直接在容器终端执行。
- `▶ 可复制 prompt` 代码块是发给 Kilo 的内容。
- 新容器缺少 DLC Ecosystem 仓库时，正确发现缺失项并停止也算通过。
- 当前 skill 的执行流程、停止条件和 description 以 `/work/skills/skills/engineering/dlc-env-setup/SKILL.md` 为唯一权威来源；不要在验证脚本中复制一个可能过时的缩写 description。

---

## 适用场景

- 新容器刚准备安装 `skills` 和 `chipltech-knowledge-base`。
- 需要确认 `dlc-env-setup` 不只是仓库文件，而是已被 Kilo 暴露和识别。
- 需要验证安装脚本已从旧 skill 名称迁移到当前稳定集合，并保留对 `3f04504` 最低兼容基线的检查能力。
- 需要验证知识库 prompt example 能驱动当前 skill 正确发现仓库、执行安全门并在条件不足时停止。

## 目标仓库与固定路径

本手册使用以下目标路径：

```text
/work/skills
/work/chipltech-knowledge-base
```

如果容器中尚无仓库，执行：

```bash
cd /work
git clone git@github.com:ChipLTech/skills.git /work/skills
git clone git@github.com:ChipLTech/chipltech-knowledge-base.git /work/chipltech-knowledge-base
```

确认仓库目标路径和 remote：

```bash
git -C /work/skills rev-parse --show-toplevel
git -C /work/skills remote get-url origin
git -C /work/chipltech-knowledge-base rev-parse --show-toplevel
```

`/work/skills` 的仓库根目录必须输出 `/work/skills`，后续命令不得改用临时测试副本作为 skill 来源。

---

## 第一步：验证 `skills` 来源与基线

先确认 `/work/skills` 是正确 remote 的 clean 工作树，并位于团队当前主线；不要把临时测试副本当作 Kilo skill 来源：

```bash
git -C /work/skills rev-parse --show-toplevel
git -C /work/skills remote get-url origin
git -C /work/skills status --short --branch
```

最低兼容基线是 commit `3f04504` 已包含在当前历史中。先确保本地能解析该 commit；如不能解析，获取远端历史后再验证：

```bash
git -C /work/skills cat-file -e 3f04504^{commit} || git -C /work/skills fetch origin
```

然后同时检查 exact commit 和 ancestor 关系：

```bash
required_commit=3f04504
actual_commit="$(git -C /work/skills rev-parse HEAD)"
required_full="$(git -C /work/skills rev-parse "$required_commit^{commit}")"

if [ "$actual_commit" = "$required_full" ]; then
  printf 'PASS: exact skills commit %s\n' "$required_full"
elif git -C /work/skills merge-base --is-ancestor "$required_full" "$actual_commit"; then
  printf 'PASS: skills HEAD %s contains required ancestor %s\n' "$actual_commit" "$required_full"
else
  printf 'FAIL: skills HEAD %s does not contain required commit %s\n' "$actual_commit" "$required_full" >&2
  exit 1
fi
```

通过标准：`/work/skills` remote 指向 ChipLTech skills 仓库，工作树状态已记录，HEAD 处于团队当前主线；最低兼容验证中，HEAD 等于 `3f0450464f3ab8819e12a7575d56b586997bca8c` 或 `git merge-base --is-ancestor` 明确成功。仅比较短 hash 字符串、commit 日期或文件是否存在都不够。

---

## 第二步：安装到 Kilo

### 2.1 全局安装

适合让当前用户的所有项目都能使用这些 skills：

```bash
cd /work/skills
./scripts/link-kilo-skills.sh --with-commands
```

目标目录：

```text
~/.config/kilo/skills/
~/.config/kilo/command/
```

### 2.2 项目级安装

适合只在知识库项目中验证，不修改全局 Kilo skill 集合：

```bash
cd /work/skills
./scripts/link-kilo-skills.sh --project /work/chipltech-knowledge-base --with-commands
```

目标目录：

```text
/work/chipltech-knowledge-base/.kilo/skills/
/work/chipltech-knowledge-base/.kilo/command/
```

项目级安装后，应从 `/work/chipltech-knowledge-base` 打开 Kilo。以下检查默认展示全局路径；验证项目级安装时，先执行：

```bash
export KILO_CONFIG_ROOT=/work/chipltech-knowledge-base/.kilo
```

验证全局安装时执行：

```bash
export KILO_CONFIG_ROOT="$HOME/.config/kilo"
```

---

## 第三步：验证 `dlc-env-setup` 暴露结果

### 3.1 验证 skill symlink 和维护源

```bash
test -L "$KILO_CONFIG_ROOT/skills/dlc-env-setup"
test "$(readlink -f "$KILO_CONFIG_ROOT/skills/dlc-env-setup")" = "/work/skills/skills/engineering/dlc-env-setup"
test -f "$KILO_CONFIG_ROOT/skills/dlc-env-setup/SKILL.md"
test -x "$KILO_CONFIG_ROOT/skills/dlc-env-setup/scripts/pytorch-preflight.sh"
test -x "$KILO_CONFIG_ROOT/skills/dlc-env-setup/scripts/vllm-preflight.sh"
test -x "$KILO_CONFIG_ROOT/skills/dlc-env-setup/scripts/runtime-smoke.sh"
```

维护源和脚本应为：

```text
/work/skills/skills/engineering/dlc-env-setup/SKILL.md
/work/skills/skills/engineering/dlc-env-setup/scripts/pytorch-preflight.sh
/work/skills/skills/engineering/dlc-env-setup/scripts/vllm-preflight.sh
/work/skills/skills/engineering/dlc-env-setup/scripts/runtime-smoke.sh
```

### 3.2 以当前 `SKILL.md` 验证 command wrapper

不要硬编码 description 的缩写文本。下面的命令直接比较当前 `SKILL.md` frontmatter 与生成 wrapper：

```bash
skill_md=/work/skills/skills/engineering/dlc-env-setup/SKILL.md
command_md="$KILO_CONFIG_ROOT/command/dlc-env-setup.md"

test -f "$command_md"
skill_description="$(awk 'NR == 1 && $0 == "---" { frontmatter=1; next } frontmatter && $0 == "---" { exit } frontmatter && /^description:/ { sub(/^description:[[:space:]]*/, ""); print; exit }' "$skill_md")"
command_description="$(awk 'NR == 1 && $0 == "---" { frontmatter=1; next } frontmatter && $0 == "---" { exit } frontmatter && /^description:/ { sub(/^description:[[:space:]]*/, ""); print; exit }' "$command_md")"
test -n "$skill_description"
test "$command_description" = "$skill_description"
grep -Fq '请使用 `dlc-env-setup` skill，严格按它的流程处理下面的问题：' "$command_md"
grep -Fq '$ARGUMENTS' "$command_md"
```

### 3.3 验证旧别名已移除

当前安装脚本应清理以下旧 skill symlink 和生成的 command wrapper；`3f04504` 是该迁移行为的最低兼容基线：

```text
diagnose
to-prd
to-issues
write-a-skill
review
```

执行：

```bash
for name in diagnose to-prd to-issues write-a-skill review; do
  test ! -L "$KILO_CONFIG_ROOT/skills/$name"
  test ! -e "$KILO_CONFIG_ROOT/command/$name.md"
done
```

注意：安装脚本不会删除用户自行创建的真实 skill 目录，也不会删除无法识别为脚本生成内容的真实 command 文件。如果上面检查失败，先判断是否存在受保护的用户文件；不要为了通过验证而盲目删除它。

同理，链接脚本不会覆盖已有的普通 command 文件。如果 3.2 的 description 比较失败，先确认该文件是否由旧版链接脚本生成；只有确认是可再生 wrapper 后，才删除该文件并重新运行 `link-kilo-skills.sh --with-commands`。用户维护的 command 文件必须保留并单独迁移。

### 3.4 验证当前 replacements 和六个新 stable skills

旧名称的当前 replacements：

| 旧名称 | 当前名称 |
|--------|----------|
| `diagnose` | `diagnosing-bugs` |
| `to-prd` | `to-spec` |
| `to-issues` | `to-tickets` |
| `write-a-skill` | `writing-great-skills` |
| `review` | `code-review` |

当前稳定集合应暴露以下六个新 skills；`3f04504` 是它们进入默认集合的最低兼容基线：

```text
ask-matt
implement
research
resolving-merge-conflicts
wayfinder
teach
```

一次验证全部十一项：

```bash
for name in diagnosing-bugs to-spec to-tickets writing-great-skills code-review ask-matt implement research resolving-merge-conflicts wayfinder teach; do
  test -L "$KILO_CONFIG_ROOT/skills/$name"
  test -f "$KILO_CONFIG_ROOT/skills/$name/SKILL.md"
  test -f "$KILO_CONFIG_ROOT/command/$name.md"
done
```

这些检查验证默认 stable 安装集合；不需要 `--all`。

---

## 第四步：重启 Kilo 或开启新 session

Kilo 可能在现有 session 中缓存 skill 列表。安装完成后重启 Kilo，或至少打开新 session。项目级安装必须从对应项目目录启动，不能在其他目录用全局配置结果代替项目级验证。

---

## 第五步：识别与工作流理解验证

这一层称为“识别与工作流理解验证”：验证 Kilo 是否识别当前 skill，并能从其权威 `SKILL.md` 理解工作流、安全门和停止条件。它不执行系统修改或真实构建。

### 方式 A：slash command

**▶ 可复制 prompt：**

```text
/dlc-env-setup 这是识别与工作流理解验证。不要修改系统环境，不要切 branch，也不要开始构建。请根据当前 dlc-env-setup SKILL.md，说明自动工作流、partial rebuild safety gate、维护脚本、验证标准和必须停止的条件；使用 DLC Ecosystem、DLC_Custom_Kernel Repository、DLCSynapse、DLC Runtime 等知识库正式术语。
```

### 方式 B：自然语言调用

未生成 command wrapper 时使用：

**▶ 可复制 prompt：**

```text
请使用 `dlc-env-setup` skill。这是识别与工作流理解验证。不要修改系统环境，不要切 branch，也不要开始构建。请根据当前 SKILL.md 说明自动工作流、partial rebuild safety gate、维护脚本、验证标准和必须停止的条件；使用 DLC Ecosystem、DLC_Custom_Kernel Repository、DLCSynapse、DLC Runtime 等知识库正式术语。
```

通过标准必须以当前 `/work/skills/skills/engineering/dlc-env-setup/SKILL.md` 为准。至少确认回答体现：

- rediscover repo locations 并回显 repo map，不假设 `/home/workspace`。
- 验证每个仓库的 build entrypoint。
- 仅在请求切 branch 时检查工作树、当前 branch 和目标 branch 可用性。
- 按依赖顺序 rebuild，并对 partial rebuild 检查前置依赖健康度。
- 在 PyTorch、可选 vLLM 和最终 smoke 阶段使用 skill 自带的三个维护脚本。
- 从 PyTorch 源码树外验证版本和 NumPy bridge。
- 缺仓库、入口、branch 安全条件或 smoke 失败时停止。

如果回答只是泛泛描述“安装环境”，且无法对应当前 `SKILL.md`，则识别与工作流理解验证失败。

---

## 第六步：验证知识库 prompt example 的停止语义

目标是确认 `/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md` 能驱动当前 skill 做 repo discovery，并在新容器缺少关键仓库时停止。

**▶ 可复制 prompt：**

```md
请使用 `dlc-env-setup` skill，按 `/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md` 的“全量或分阶段重建”模板工作。

【模式】全量重建
【搜索根目录】/work,$HOME
【需要切换的批准 ref】无
【版本策略】不适用；本验证不 clone、不同步、不切换仓库
【是否包含 vllm / vllm-dlc】否
【是否允许修改 /usr/local】否

不要假设 `/home/workspace`。先做 repo discovery、回显 repo map，并在缺少关键仓库或 build entrypoint 时停止。不要发明路径或假装构建成功。
```

通过标准：

- 先发现并回显实际路径。
- 明确列出缺失的 DLC Ecosystem 仓库或入口。
- 条件不足时停止，不执行后续 build。
- 使用 `CONTEXT.md` 中的正式术语，不用裸 `DLC` 泛指整个生态。

---

## 第七步：用轻量骨架验证路径与 safety gate 语义

没有真实 DLC Ecosystem 仓库时，可以创建目录骨架来验证 repo discovery、entrypoint 检查、shell 变量映射和 partial rebuild safety gate 的理解。该骨架不是 Git repository，也不含真实构建系统或安装产物，因此：

- 它只验证路径发现和安全门语义。
- 它不验证真实编译、安装、DLC Runtime smoke 或产物正确性。
- 它不验证 Git repository branch safety、工作树状态或 branch 切换行为。

创建包含新版 CMake 两个入口的骨架：

```bash
mkdir -p /tmp/dlc-smoke/{cmake-source,dlc-thunk,DLCsim,DLCSynapse,DLC_CL,LLVM,DLC_Custom_Kernel,pytorch}
touch /tmp/dlc-smoke/cmake-source/bootstrap
touch /tmp/dlc-smoke/cmake-source/CMakeLists.txt
touch /tmp/dlc-smoke/dlc-thunk/compile.sh
touch /tmp/dlc-smoke/DLCsim/build.sh
touch /tmp/dlc-smoke/DLCSynapse/compile.sh
touch /tmp/dlc-smoke/DLC_CL/build.sh
touch /tmp/dlc-smoke/LLVM/build.sh
touch /tmp/dlc-smoke/DLC_Custom_Kernel/CMakeLists.txt
touch /tmp/dlc-smoke/pytorch/setup.py
```

**▶ 可复制 prompt：**

```md
请使用 `dlc-env-setup` skill，按 `/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md` 的模板工作。这是轻量骨架的路径与 safety gate 语义验证，不要执行 build、安装、branch 检查或系统修改。

【模式】只重建 PyTorch wheel
【搜索根目录】/tmp/dlc-smoke
【需要切换的批准 ref】无
【版本策略】不适用；本骨架不是 Git repository，不验证 CI 默认最新或固定 ref
【是否包含 vllm / vllm-dlc】否
【是否允许修改 /usr/local】否
【CMake 要求】已安装 `cmake --version` 必须严格大于 `3.27.0`；本骨架只模拟源码入口，不证明系统 CMake 合格

请验证 repo discovery、entrypoint、shell 变量映射和 partial rebuild safety gate，并说明还需要哪些真实安装前置条件。明确指出这些目录不是 Git repository，因此本次不验证 branch safety。不要把 touch 创建的入口当成可构建实现。
```

通过标准：Kilo 能识别所有入口，回显路径映射，并说明“只重建 PyTorch wheel”仍需验证原生依赖安装健康；不能把该骨架报告为完整功能通过，也不能声称已验证 Git branch safety。

清理骨架：

```bash
rm -rf /tmp/dlc-smoke
```

---

## 第八步：在真实 DLC Ecosystem 环境中做完整执行验证

只有真实仓库、构建入口、依赖和所需权限齐全时，才能进行完整执行验证：

- 满足 `cmake --version > 3.27.0` 的可用 CMake
- `dlc-thunk`
- `DLCsim`
- `DLCSynapse`
- `DLC_CL`
- `LLVM`
- `DLC_Custom_Kernel`（文档中称为 DLC_Custom_Kernel Repository）
- `pytorch`（构建 PyTorch DLC Backend）
- 可选：`vllm`、`vllm-dlc`

**▶ 可复制 prompt：**

```text
/dlc-env-setup 请按 /work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md 的“全量或分阶段重建”模板执行完整验证，并以当前 /work/skills/skills/engineering/dlc-env-setup/SKILL.md 及其 scripts/ 为执行权威。

【模式】<全量重建 / 从 LLVM 开始 / 从 DLC_Custom_Kernel 开始 / 只重建 PyTorch wheel / 只重装 wheel / 修 vllm>
【搜索根目录】<真实 repo 根目录>
【版本策略】<CI默认最新 / 固定ref / 混合；已有仓库不需要切换时写“保持当前 checkout”>
【需要切换的批准 ref】<CI默认最新可写“使用 Arsenal CI 默认分支最新 head”；固定ref/混合时每项提供 repo、批准的 remote URL、branch/tag 和可选 SHA；例如 LLVM|git@github.com:ChipLTech/LLVM.git|feature-x|<sha>；没有就写“无”>
【是否包含 vllm / vllm-dlc】<是/否>
【是否允许修改 /usr/local】<是/否>
```

完整执行验证只有在当前 `SKILL.md` 的 workflow、stop conditions 和 verification standard 全部满足时才通过。至少需要保存并检查：

- 实际 repo map、entrypoint 和 shell 变量映射。
- 如请求切换 ref，真实 Git repository 的 remote 身份、工作树、当前 ref、批准 ref 可用性、目标 SHA 和切换结果。
- 实际选择的 rebuild 起点、前置依赖健康证据和依赖顺序。
- `scripts/pytorch-preflight.sh`、可选 `scripts/vllm-preflight.sh` 和 `scripts/runtime-smoke.sh` 的执行结果。
- 如果本次执行 Real DLC Hardware C1b/serving，保存 `dlc-hardware-observability` 的 tool/source identity 与四阶段 raw/normalized evidence；仅 package/import 验证时记录 `not_applicable`，不改变原验证层级。
- CMake、PyTorch 2.5.0、NumPy bridge、DLC_Custom_Kernel Repository 安装工具和可选 vLLM metadata/import 检查。
- 任一失败阶段的命令、首批关键错误和停止位置。

不能用识别回答、轻量骨架、单个文件存在性或未执行的命令计划替代完整执行验证。

---

## 快速判断表

| 验证层级 | 验证内容 | 缺真实 DLC Ecosystem 仓库时的结果 | 通过依据 |
|----------|----------|-----------------------------------|----------|
| 来源层 | skills revision | 不受影响 | `/work/skills` 位于团队当前主线；最低兼容检查中 `3f04504` 是 exact commit 或 HEAD ancestor |
| 安装层 | 全局或项目级 Kilo 暴露 | 可完成 | symlink、wrapper、旧别名清理和 stable 集合检查通过 |
| 识别与工作流理解层 | Kilo 是否加载并理解当前 skill | 可完成 | 回答对应当前 `SKILL.md`、scripts 和停止条件 |
| 停止语义层 | prompt example 是否安全停止 | 应列出缺失项并停止 | 不发明路径、不继续 build |
| 骨架语义层 | 路径、入口和 partial gate | 可完成有限验证 | 不宣称真实 build 或 Git branch safety |
| 完整执行层 | rebuild、安装和 smoke 闭环 | 不可完成 | 当前 `SKILL.md` 全部 verification standard 满足 |

## 常见失败

- `/work/skills` 无法解析 `3f04504`：先 fetch；仍不存在则来源仓库或 remote 不正确。
- skill symlink 指向其他目录：重新从 `/work/skills` 根目录执行安装脚本。
- 项目级 skill 在 Kilo 中不可见：确认从 `/work/chipltech-knowledge-base` 启动新 session。
- 旧名称仍是 symlink 或生成 wrapper：确认安装脚本来自当前团队主线且至少包含 commit `3f04504`，然后重新执行 `--with-commands`。
- 旧名称是用户真实目录或自定义 command：脚本按设计保护它，不得自动删除。
- wrapper description 不匹配：重新生成 wrapper，并以当前 `SKILL.md` frontmatter 为准。
- 轻量骨架被报告为完整验证通过：结论无效，必须降级为路径与 safety gate 语义验证。

## 相关资料

- [dlc-env-setup-skill-usage.md](dlc-env-setup-skill-usage.md)
- [../runtime-debugging/dlc-workstation-env-rebuild.md](../runtime-debugging/dlc-workstation-env-rebuild.md)
- [../debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md](../debugging-workflows/python-build-preflight-for-pytorch-and-vllm.md)
- [../debugging-workflows/post-install-runtime-smoke.md](../debugging-workflows/post-install-runtime-smoke.md)
- [../runtime-debugging/chipltech-smi-observability.md](../runtime-debugging/chipltech-smi-observability.md)
- `/work/chipltech-knowledge-base/CONTEXT.md`
- `/work/skills/README.zh-CN.md`
- `/work/skills/SKILLHUB.yaml`
- `/work/skills/scripts/link-kilo-skills.sh`
- `/work/skills/skills/engineering/dlc-env-setup/SKILL.md`
- `/work/skills/skills/engineering/dlc-env-setup/scripts/pytorch-preflight.sh`
- `/work/skills/skills/engineering/dlc-env-setup/scripts/vllm-preflight.sh`
- `/work/skills/skills/engineering/dlc-env-setup/scripts/runtime-smoke.sh`

## 来源与版本边界

- stable skill 名称、旧别名清理规则和安装范围：当前 `/work/skills` 的 `scripts/link-kilo-skills.sh`、`README.zh-CN.md` 和 `SKILLHUB.yaml`；commit `3f04504` 仅作为最低兼容基线。
- `dlc-env-setup` 工作流、停止条件、验证标准和脚本资产：当前 `/work/skills/skills/engineering/dlc-env-setup/SKILL.md` 及同目录 `scripts/`。
- DLC Ecosystem 术语：`/work/chipltech-knowledge-base/CONTEXT.md`。
- prompt 调用模板：`/work/chipltech-knowledge-base/prompt-examples/dlc-env-setup-skill-usage.md`。

若 HEAD 是 `3f04504` 的后代，必须重新阅读上述当前文件；commit `3f04504` 只是最低兼容基线，不应覆盖后续维护源中的更新。
