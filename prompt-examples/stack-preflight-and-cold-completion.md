# 独立 Stack Preflight 与 Cold Completion Prompt

```markdown
请使用 `chipltech-context` 路由，并由 `dlc-env-setup` 执行完整 stack 启动前资格检查；不要修改 PyTorch DLC Backend、DLC_Custom_Kernel Repository、DLCSynapse、DLC Runtime、DLCsim 或 Host kernel Driver。

目标 image：`<IMMUTABLE_IMAGE_ID_OR_TAG_TO_RESOLVE>`
目标物理设备：`<PHYSICAL_DEVICE>`
兼容 policy：`<ABSOLUTE_POLICY_PATH>`
已有 artifact 或 workspace：`<PATHS_OR_AUTO_DISCOVER>`

先读取：

- `runtime-debugging/stack-preflight-and-cold-completion.md`
- `debugging-workflows/post-install-runtime-smoke.md`

自动发现并绑定 exact immutable Image ID、Driver/Runtime API、四文件 CRT bundle、DLC Custom Kernel library 和 LLVM full SHA。先运行 fail-closed static stack preflight；approved profile 只创建 Static Stack Compatible，revoked 或 unknown profile 必须停止，不得自动写 policy。

只有 static preflight 通过且 device execution 已明确授权时，才在 fresh process 中运行 cold first-compute：allocation、H2D、真实 device operation、synchronize、D2H 和 exact correctness。使用外层 timeout，并委托 `dlc-hardware-observability` 按 `SMI Observation Envelope` 四阶段（`before_launch`、`after_ready`、`during_request`、`after_cleanup`）记录 process、handle、HBM 和 cleanup。

任一 gate 失败即阻止模型加载、benchmark 或镜像发布；输出必须区分 direct repository evidence、runtime observation、inference 和 missing evidence，并包含字面量 `Claim Boundary:`。
```
