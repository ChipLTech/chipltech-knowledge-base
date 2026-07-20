# 从已配置容器为 Host 和其他容器赋能 Git/SSH

面向 Host 或新容器缺少 Git、GitHub SSH 配置或私有仓库访问能力时的可复用 prompt 模板。它以一个已经能够访问目标私有仓库的容器为受信来源，在用户明确授权后，安全发现并迁移必要配置，再分别验证 Host 和目标容器的 clone、fetch 与 push 权限。

**阅读说明**：

- `▶ 可复制 prompt` 后的代码块可直接发给 AI，使用前替换 `<>` 占位符。
- SSH 私钥属于高敏感凭据。只有在凭据所有者明确授权、源和目标均受信且目标确实需要同一身份时才允许迁移。
- 默认不输出私钥、公钥全文、passphrase、token、密码、credential store 内容或带凭据的 URL。
- “Git 功能可用”必须分别验证 Git 命令、SSH 身份、私有仓库读取和 push dry-run；只看到 `git --version` 不算完成。

---

## 适用场景

- 已配置容器能通过 SSH 访问 GitHub 私有仓库，但 Host 尚未安装或配置 Git。
- 需要让同一台受信机器上的新容器复用已批准的 GitHub SSH 身份。
- Host 没有无密码 sudo，需要使用用户级 Git 安装。
- Bash 和 Fish 的 PATH 行为不同，需要分别持久化并验证。

## 安全边界

- 先验证源容器身份和访问能力，再把它视为配置来源；不要只因目录中存在 `~/.ssh` 就信任它。
- 迁移前必须确认用户有权复制该 SSH 身份，并列出目标 Host、目标容器和目标仓库范围。
- 不覆盖目标已有的 `~/.ssh`、私钥或 Git 配置。发现冲突时停止并报告，由用户决定合并策略。
- 只迁移 SSH client 所需文件。默认排除 `authorized_keys`、socket、agent 状态、历史记录和其他无关文件。
- Git 配置采用白名单迁移。可考虑 `user.name`、`user.email`、`init.defaultBranch`、`pull.rebase`、`core.autocrlf`；不得盲目复制 `credential.*`、`http.*extraHeader`、token、密码、credential store 或含凭据的 remote URL。
- 不通过聊天、日志、命令参数、临时文本文件或 Git commit 暴露私钥。优先使用 tar stream 或 `docker cp` 的受控路径，并在操作后检查权限和临时文件。
- 不实际 push 测试 commit。使用 `git push --dry-run` 验证写权限，避免修改远程仓库。
- 不为绕过 Git 安全检查而永久添加宽泛的 `safe.directory '*'`。确需处理仓库所有权时，修正所有权或仅对已确认路径使用最小范围配置。

## 完整赋能模板

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请从一个已经配置并验证 GitHub SSH 私有仓库访问能力的容器中，安全获取必要的 Git 配置信息和 SSH client key，为指定 Host 用户及目标容器补齐 Git clone、fetch 和 push 能力，并完成非破坏性验证。

【源容器】<已配置 Git/SSH 的容器名或 ID>
【源容器用户】<例如 root 或 app>
【Host 目标用户】<例如当前登录用户>
【Host HOME】<例如 /home/example；请用 getent/id 验证，不要猜>
【目标容器】<容器名或 ID 列表；没有则写“无”>
【目标容器用户及 HOME】<逐容器填写；不确定则要求自动发现并在歧义时停止>
【验证私有仓库 SSH URL】<例如 git@github.com:ORG/REPO.git>
【验证分支】<例如 main；可写“自动读取远程默认分支”>
【是否明确授权复制 SSH 私钥】<是/否>
【是否允许安装 Git】<是/否>
【是否允许系统级安装】<是/否；需要 sudo 或容器 root>
【是否允许用户级安装】<是/否>
【需要支持的 Shell】<Bash/Fish/其他>
【是否保留验证 clone】<是/否；默认否>

严格按以下流程执行：

1. 做只读 discovery，记录但不要输出任何 secret。
   - 确认当前 Host 用户、UID/GID、HOME、默认 shell、OS/architecture、可用 package manager、sudo 是否需要交互密码，以及 Docker 可用性。
   - 确认源容器正在运行，解析【源容器用户】的真实 HOME，不要假设一定是 `/root`。
   - 在源容器中检查 `command -v git ssh`、`git --version`、SSH client 配置文件名、文件类型、owner 和 mode；只列文件元数据，不读取或回显私钥正文。
   - 检查目标 Host 和每个目标容器是否已有 Git、`~/.ssh`、Git config、PATH 配置和同名文件。
   - 如果任一目标已有 SSH 私钥或配置，不覆盖、不删除、不自动合并，立即报告冲突。

2. 验证源容器确实是可用的受信来源。
   - 使用 batch mode 和超时执行 GitHub SSH 身份验证，例如 `ssh -T -o BatchMode=yes -o ConnectTimeout=10 git@github.com`；GitHub 返回“认证成功但不提供 shell”时允许其非零 shell 状态，必须根据输出语义判断。
   - 执行 `git ls-remote --symref <验证仓库> HEAD`，记录远程默认分支和 HEAD；不得显示含 token 的 URL。
   - 如需确认写权限，先在源容器已有 clean clone 中执行 `git push --dry-run origin HEAD:refs/heads/<验证分支>`。没有安全 clone 时可暂缓，不得为验证而创建实际远程 commit 或 branch。
   - SSH 身份不匹配、仓库不可读、来源配置含未知代理命令/外部 key provider，或需要交互式 passphrase 且无可用 agent 时停止。

3. 对 Git 配置做白名单提取，不整体复制配置文件。
   - 分别检查 source scope：`git config --system --list --show-origin`、`--global` 和必要的 repo-local 配置。
   - 只迁移目标确实需要的非敏感项，例如 `user.name`、`user.email`、`init.defaultBranch`、`pull.rebase`、`core.autocrlf`。
   - 忽略并报告存在但未迁移的敏感或环境绑定项：`credential.*`、`http.*extraHeader`、`url.*.insteadOf` 中的凭据、代理、签名 key、绝对路径、容器专属 helper 和 repo-local remote。
   - 不输出敏感配置值；发现 token、密码或 credential store 时只报告类别和来源文件，值必须脱敏。

4. 仅在【是否明确授权复制 SSH 私钥】为“是”时迁移 SSH client 文件。
   - 源端只选择当前验证实际使用的 identity file、对应 `.pub`（如存在）、`config` 和必要的 `known_hosts`。不要复制整个 HOME。
   - 默认不复制 `authorized_keys`、`environment`、control socket、agent socket、历史文件或无关 identity。
   - 优先通过内存管道在源容器与目标之间传输，例如受控 tar stream；命令和日志中不得出现文件正文。
   - 如果必须经 Host 临时目录中转，只能使用权限 `0700` 的新目录，文件 `0600`，完成后立即安全删除，并确认没有被纳入 Git 工作树或 shell history。
   - 写入 Host 前确认 `$HOME/.ssh` 不存在，或目标目录为空且用户明确批准。写入后设置目录 `0700`、私钥和 config `0600`、known_hosts 不宽于 `0644`，owner 为目标用户 UID/GID。
   - 写入目标容器时同样验证用户 HOME、owner 和权限；不要假设容器用户是 root。
   - SSH config 中如包含源容器专属绝对路径、ProxyCommand、IdentityAgent 或不可达 hostname，先停止并报告，不要原样复制后声称可用。

5. 安装 Git，优先采用目标环境的标准 package manager。
   - 已有可用 Git 时不重装。
   - Host 允许系统级安装且 sudo 可用时，按已识别 OS 使用官方 package manager；不要混用发行版命令。
   - Host sudo 需要未知交互密码且允许用户级安装时，只从对应发行版官方仓库或用户批准来源获取与 OS/architecture 匹配的包，安装到 `$HOME/.local` 下，不修改系统 package database。
   - 用户级安装必须确认 `git` helper、`git-core`、templates、man page 和运行依赖可定位；必要时创建 `$HOME/.local/bin/git` wrapper，设置 `GIT_EXEC_PATH` 和 `GIT_TEMPLATE_DIR`，但不要覆盖已有 wrapper。
   - 目标容器允许安装且以 root 运行时，使用容器自身 package manager 安装；非 root 且无提权时，使用已批准的用户级方案或报告 blocker。
   - 不下载来源不明的静态 Git binary，不关闭 TLS 校验。

6. 为每个目标持久化 PATH，但保持最小改动和幂等。
   - Bash login shell 检查 `.profile`，交互式 Bash 检查 `.bashrc`；只有 `$HOME/.local/bin` 尚未加入时才添加。
   - Fish 使用 `~/.config/fish/config.fish` 中的 `fish_add_path --prepend "$HOME/.local/bin"` 或与现有配置一致的等效方式；Fish 不读取 `.profile` 或 `.bashrc`。
   - 不覆盖现有 shell 配置。修改前读取上下文，修改后分别在对应 shell 中验证 `command -v git`/`type -p git` 和 `git --version`。
   - 非 TTY 验证出现 job-control 或 `TERM` warning 时，区分无害终端警告与真实 Git failure，不要隐藏错误。

7. 分别验证 Host 和每个目标容器。
   - `git --version` 和 Git exec/helper path 可用。
   - `ssh -G github.com` 或目标 alias 的有效 `hostname`、`user`、`port`、`identityfile` 符合预期；输出不得包含私钥。
   - `ssh -T -o BatchMode=yes -o ConnectTimeout=10 git@github.com` 能确认批准的 GitHub 身份。
   - 在权限 `0700` 的临时目录中 clone【验证私有仓库 SSH URL】，确认 remote、默认分支、HEAD 和 clean status。
   - 执行 `git fetch --dry-run` 或普通 fetch（不得改工作树），再执行 `git push --dry-run origin HEAD:refs/heads/<验证分支>`。
   - push dry-run 必须基于与远程目标分支一致且无本地新增 commit 的 HEAD，避免测试产生误导或触发不必要 hook。
   - 默认删除验证 clone；只有【是否保留验证 clone】为“是”时保留并报告路径。

8. 完成后做 secret 和副作用检查。
   - 检查目标权限、owner、临时目录、Git status 和 shell 配置 diff。
   - 确认没有私钥、token、credential 文件或脱敏前日志进入任何 Git repository。
   - 不打印私钥 fingerprint 以外的 key 内容；如报告 fingerprint，也只用于目标间一致性核对。
   - 不提交 SSH/Git credential 文件。只有用户另行要求提交文档或代码时，才按目标仓库规则提交非敏感变更。

9. 最终交付按目标逐项报告：
   - Git 安装方式、版本、可执行文件和 helper 路径。
   - 生效 shell 与 PATH 配置文件。
   - SSH 配置来源、目标 owner/mode、认证身份；所有敏感值脱敏。
   - 私有仓库 clone、fetch、push dry-run 的结果，以及验证分支和 HEAD。
   - 实际迁移的 Git 配置白名单项名称，不回显敏感值。
   - 跳过的敏感或环境绑定配置类别、冲突、blocker 和未验证范围。
```

## Host 已有 SSH 配置时

不要把“已有配置”当成可覆盖条件。可以改用下面的只读 prompt，让 AI 先给出合并边界：

**▶ 可复制 prompt：**

```md
目标 Host 已存在 `~/.ssh`。请只读比较源容器和 Host 的 SSH client 配置：列出文件名、类型、owner、mode、host alias、有效 hostname/user/port、identity file 路径和公钥 fingerprint，不输出任何 key 正文。不要复制、覆盖、chmod 或修改配置。请指出 alias 冲突、identity 冲突、ProxyCommand/IdentityAgent 依赖和最小安全合并方案，然后停止等待明确授权。
```

## 常见失败

- 直接 `docker cp` 整个 `~/.ssh`：可能复制 `authorized_keys`、socket、无关 key 或错误 owner，应只选择验证实际使用的 client 文件。
- 用 `git config --global --list` 直接贴日志：配置可能含 token、helper 或带凭据 URL，应按 key 名筛选并脱敏。
- 只改 `.profile`：交互式 Bash 或 Fish 可能仍找不到用户级 Git，应分别验证实际 shell。
- 只执行 `ssh -T`：只能证明 GitHub 身份认证，不证明指定私有仓库可读或可写。
- 用实际 commit/push 测写权限：会污染远端历史；应使用与目标分支一致的 `git push --dry-run`。
- 对目标已有 `.ssh` 直接覆盖：可能破坏其他仓库或服务器访问，必须停止并显式决定合并方式。
- 用户级 Git 只有主二进制：`git clone` 等子命令可能因 helper、templates 或运行依赖缺失而失败，必须做端到端验证。
- 为消除 `dubious ownership` 设置 `safe.directory '*'`：扩大信任边界，应修复 owner 或只允许已确认的单一路径。

## 验收标准

每个目标环境都必须独立满足：

1. 当前用户在目标 shell 中能解析预期的 `git`。
2. Git 主程序、helper 和 templates 可用。
3. SSH 使用批准的 identity，文件 owner/mode 正确。
4. 指定私有仓库 clone 和 fetch 成功。
5. `git push --dry-run` 成功且没有远程副作用。
6. 没有泄露或提交私钥、token、密码及 credential store。

## 相关资料

- [dlc-env-setup-environment-bootstrap.md](dlc-env-setup-environment-bootstrap.md)
- [dlc-env-setup-fresh-container-validation.md](dlc-env-setup-fresh-container-validation.md)
- [../README.md](../README.md)
