# PyTorch DLC Backend 插件化迁移 prompt

面向“把已经投入生产的 PyTorch DLC Backend 迁移为标准 PyTorch 可加载插件”的日常开发任务。下面两个 prompt 分别覆盖单个迁移切片和跨阶段持续推进；替换 `<>` 中的参数后即可直接使用。

## 适用场景

- 生产 PyTorch DLC Backend 已有可用实现，需要迁移到 PrivateUse1 插件。
- 需要比较生产 Backend、标准 PyTorch 和目标插件三方代码，解决内建接入与插件接入的差异。
- 当前优先目标是批量编译、链接、wheel 和标准 host 加载，批量行为测试可以集中后移。
- 任务跨越 runtime、operator、distributed、compiler、profiler 或 release 多个阶段，需要 multi-agent 持续推进。

## 使用前提

开始前应明确以下三个权威来源：

1. **生产 PyTorch DLC Backend**：硬件语义、DLC Kernel Launch Protocol、kernel name、参数顺序、dtype/layout 和错误行为的权威来源。
2. **标准 PyTorch**：dispatcher、PrivateUse1、codegen、AOTI 和公开 Python API 契约的权威来源。
3. **插件架构参考**：只参考目录分层、注册入口、构建、wheel 和 CI 组织，不继承其他设备后端的实现语义。

共同约束：

- 迁移已有实现，不把任务扩写成开发新的 DLC Custom Kernel。
- 禁止 CPU computation fallback。CPU 只允许作为常量来源或 H2D/D2H transport 端点。
- 不用注册数量、编译成功或单个 smoke 代替完整行为覆盖。
- 测试可以后移，但 source boundary、受影响 target 编译和发布产物检查不能省略。

## 迁移一个组件或纵向切片

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请把生产 PyTorch DLC Backend 中已经投入使用的 <组件或能力> 迁移到标准 PyTorch 可加载的 PrivateUse1 插件。

先读取：
- /work/chipltech-knowledge-base/CONTEXT.md
- /work/chipltech-knowledge-base/README.md
- <迁移计划文档>
- <当前进度文档>

【生产 PyTorch DLC Backend】<绝对路径>
【标准 PyTorch】<仓库或安装根目录，含版本>
【目标插件仓库】<绝对路径>
【插件架构参考】<例如 pytorch_npu 路径；没有则写“无”>
【本次范围】<组件、算子族或纵向能力>
【允许修改的文件】<路径或目录>
【禁止修改的文件】<路径或目录>
【测试策略】<只做编译门禁 / 最小硬件 smoke / 批量行为测试后移>
【是否允许 commit/push】<均不允许 / 允许 commit / 允许 commit 和 push>

请按下面顺序执行：

1. 先用 `/zoom-out` 的方式给出三方代码映射：
   - 生产实现、registration、构建入口在哪里。
   - 标准 PyTorch 对应扩展接口在哪里。
   - 目标插件当前实现、缺口和已有兼容层在哪里。
2. 判断本次工作属于哪一类：
   - 源码可直接搬运，只需替换接入点。
   - 生产实现依赖内建 `DeviceType/DispatchKey`，需要 PrivateUse1 适配。
   - 标准 PyTorch 已提供插件接口，但插件尚未注册。
   - 生产 Backend 本身也没有实现，应显式拒绝并记录，不能自行补语义。
3. 只做插件化所必需的改动，例如：
   - `DeviceType/DispatchKey` 到 `PrivateUse1`。
   - device guard、allocator、stream/event、StorageImpl 的插件注册。
   - KernelDesc 接口适配。
   - YAML/codegen、include、namespace、visibility 和构建 source list。
4. 保持生产语义不变：
   - 不改 kernel name、参数顺序和 registration 逻辑。
   - 不改 dtype/layout、contiguous、rank 和错误边界。
   - 不加入 CPU computation fallback，也不吞掉 unsupported 或异步错误。
5. 先关闭构建链：
   - 生成器或静态 boundary check。
   - 受影响 target 增量编译。
   - 如果影响默认产物，再做 full-core/link/wheel/isolated import。
6. 只运行与本次改动直接相关的最小验证。需要硬件时使用 synthetic/block/one-step，不跑长训练；其余测试写入后移账本。
7. 审查最终 diff，排除临时文件、个人 workspace、生成缓存和其他人的改动。未经授权不要 commit 或 push。

交付物：
- 三方代码映射。
- 搬运内容和插件化改动边界。
- 实际执行的编译、链接或最小 smoke 结果。
- 明确的已完成项、显式限制和后移测试。
- 如果获准提交：commit hash 和 push 结果。
```

### 常见请求写法

- `先不要新增算子，只把生产 BinaryOps 的源码、注册和 CMake 接入迁移到 PrivateUse1。`
- `优先保证 full-core 编译和 wheel 隔离加载，行为测试集中后移。`
- `生产 Backend 没有的语义先显式拒绝，不要自行扩展，也不要 CPU fallback。`
- `对比生产 Backend、标准 PyTorch 和插件三方代码后再改，不要只参考 pytorch_npu。`

## 审计当前阶段并用 multi-agent 继续推进

**▶ 可复制 prompt（替换 `<>` 后直接使用）：**

```md
请先审计 <当前阶段> 的真实完成度，再用 multi-agent 推进 <下一阶段>。不要重复已完成工作，也不要因为窄 smoke 通过就宣布阶段完成。

先读取：
- /work/chipltech-knowledge-base/CONTEXT.md
- <权威计划文档>
- <当前进度或日报>
- 最近相关提交和工作树状态

【目标仓库】<绝对路径>
【当前分支】<分支名>
【生产语义仓库】<绝对路径>
【标准 host】<标准 PyTorch 路径和版本>
【当前阶段】<例如 P1 operator closeout>
【下一阶段】<例如 P2 distributed/compiler/profiler>
【允许后移的测试】<批量算子、长训练、多机等>
【必须保留的门禁】<full compile、wheel import、最小硬件 smoke 等>
【是否允许 commit/push】<均不允许 / 允许 commit / 允许 commit 和 push>

### 先做完成度审计

1. 从计划中列出每个交付物和退出条件。
2. 为每一项寻找直接证据：源码、source list、生成物、ELF/wheel、命令输出或硬件结果。
3. 状态只使用：
   - 已由直接证据证明完成。
   - 实现完成，行为测试后移。
   - 显式限制，生产 Backend 也不支持。
   - 尚未实现。
4. 不把“没有发现问题”、注册数量、编译通过或单一 example 当成更大范围的完成证据。
5. 给出精确统计、当前阶段剩余缺口和下一阶段实施顺序。

### 再拆 multi-agent 工作流

1. 按独立产物和文件所有权拆分，例如：
   - runtime/stream/event/async error。
   - distributed/ProcessGroup/DLCCL。
   - Inductor/AOTI/codegen。
   - profiler 或 release/wheel。
2. 每个 Agent 任务必须写明：
   - 允许和禁止修改的文件。
   - 权威参考源码。
   - 完成判据和最小验证命令。
   - 是否只读审计，是否允许写文件。
3. 多个任务会修改同一文件时，不要并行写；改成一个 Agent 实现、另一个 Agent 只读审计。
4. 子 Agent 不自行 commit。主 Agent 负责共享工作树审查、构建集成、文档和提交。

### 持续推进

1. 优先解决阻塞整体 codegen、compile、link、wheel 或标准 host 加载的缺口。
2. 允许批量行为测试后移，但每个纵向切片至少保留：
   - 生成器或静态 boundary gate。
   - 受影响 target 编译。
   - 影响发布产物时的 full-core/wheel/isolated import。
   - 需要运行时证据时的一个最小 synthetic/block/one-step smoke。
3. unsupported 路径必须确定性失败。禁止 CPU computation fallback、假 activity、空 stub 或吞异常。
4. 每个切片结束时更新计划和日报，分别写清已完成、显式限制、测试后移和下一步。
5. 每次提交前运行 `git diff --check`，只暂存本切片文件。未经授权不要 commit 或 push。
6. 所有计划交付物都取得直接证据前，Goal 保持 active。

交付物：
- 当前阶段逐项审计表和精确统计。
- multi-agent 文件所有权、验收和状态表。
- 本轮完成的纵向切片及验证结果。
- 后移测试账本和下一步顺序。
- 如果获准提交：commit hash 和 push 结果。
```

### 常见请求写法

- `P1 批量行为测试先后移，守住 full-core 编译后直接推进 P2。`
- `把 runtime、distributed、AOTI 拆给不同 Agent，主 Agent 统一审查和提交。`
- `先确认哪些是实现完成但未测试，哪些是真正没实现，再决定下一批。`
- `不要跑真实长训练，只用 synthetic block 或 one-step 证明纵向链路。`

## 什么时候停

1. 生产 Backend、标准 PyTorch 或目标插件的 authoritative root 不明确。
2. 计划文档与当前代码明显冲突，且无法从提交和构建产物确认哪一个是权威状态。
3. 需要修改共享文件，但多个 Agent 已经在并行写同一文件。
4. 发现 CPU computation fallback、kernel ABI 变化或生产语义扩展，而用户没有授权。
5. 工作树包含会与本任务重叠的未提交改动，无法安全隔离。
6. 需要 commit 或 push，但用户尚未明确授权。

## 相关资料

- [实际业务skill使用示例.md](实际业务skill使用示例.md)
- [../CONTEXT.md](../CONTEXT.md)
- [../operator-dispatch/enabled-kernels-dispatch.md](../operator-dispatch/enabled-kernels-dispatch.md)
- [../runtime-debugging/runtime-troubleshooting.md](../runtime-debugging/runtime-troubleshooting.md)
