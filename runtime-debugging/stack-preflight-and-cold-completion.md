# 独立 Stack Preflight 与 Cold Completion Gate

## 适用场景

- 不允许修改 PyTorch DLC Backend、DLC_Custom_Kernel Repository、DLCSynapse、DLC Runtime、DLCsim 或 Host kernel Driver，需要在完整 stack 启动前自动发现已知和未知兼容性风险。
- Daily Image、模型服务或安装环境可能通过 import、设备枚举、allocation 和 H2D，但首个 device operation 在 `synchronize()` 卡住。
- 需要在发布镜像或启动模型前阻止 CRT、DLC Custom Kernel、LLVM、Driver API 和 Runtime API 的错误组合进入运行阶段。

## 核心结论

独立启动前资格检查由两个不可互相替代的 checkpoint 组成：

```text
Static Stack Compatible
  AND
Cold First-Compute Completion Ready
  -> 允许进入模型加载或镜像发布
```

1. **Static Stack Preflight** 对 exact immutable image 和 Host stack 做只读身份匹配；只允许显式批准的完整 profile，已撤销和未知 profile 均 fail closed。
2. **Cold First-Compute Gate** 在 fresh process 中执行 H2D、真实 device operation、synchronize、D2H 和 correctness；它发现静态 policy 尚未认识的未来兼容性问题。
3. 静态批准不是 Real DLC Hardware execution evidence；cold C1b 通过也不能自动把未知 profile 写成批准记录。
4. 该机制位于部署、安装验收或镜像发布边界，不要求修改任何运行库。

## 为什么不能只看版本号

Driver API 相等只能证明 Host kernel Driver 与 DLCSynapse/DLC Runtime 的公开 API generation 匹配，不能证明以下 artifact 的执行契约一致：

- 四文件 CRT bundle。
- DLC Custom Kernel binary。
- 生成该 binary 的 LLVM。
- KernelDesc adapter 和 native libraries。
- exact image 中的实际安装文件。

同一 API generation 内仍可能存在 CRT 入口地址、SMEM 字段、XYS 执行路径或编译产物差异（本机制的提出来自一次具体诊断：Driver API 相等、allocation/H2D 均通过，但首个 device operation 在 `synchronize()` 卡住，最终由 CRT-only 替换恢复；该结论属 `runtime observation` + `inference`，不是通用硬件事实）。因此 preflight 必须绑定实际 artifact hash，不能使用 daily tag、分支名或“v20/v21”代替完整身份。

## Static Stack Preflight

### 输入身份

至少绑定：

- immutable Image ID，格式为 `sha256:<64 hex>`。
- target：DLC Chip、TYD Chip 或 HHP Chip。
- Host kernel Driver API。
- exact image 内 DLC Runtime API。
- 四个且仅四个 CRT 文件的 size、SHA-256 和一致 target marker：
  - `dlc_crt.hex`
  - `dlc_crt_xys1.hex`
  - `dlc_crt_cmem.hex`
  - `dlc_crt_cmem_xys1.hex`
- 实际安装的 DLC Custom Kernel library SHA-256。
- LLVM full SHA；无法提取时必须是 `identity_unavailable`，不能从当前 source checkout 推断。

可选但推荐同时保存：PyTorch commit、DLCSynapse/DLC Runtime hashes、Driver module SHA/srcversion、CRT producer manifest 和 DLC Custom Kernel embedded build identity。

### Policy 语义

Policy 只包含经过独立 qualification evidence 批准或明确撤销的 exact profiles：

```text
approved exact profile -> Static Stack Compatible
revoked exact profile  -> known_bad_profile
unknown exact profile  -> unknown_profile
```

Policy 记录必须有唯一 `profile_id`、状态、target、实际 `marker_hex`、Driver/Runtime API、CRT quartet、DLC Custom Kernel library hash 和 LLVM full SHA。approved profile的marker必须匹配target；revoked profile保留实际坏marker，因此无marker的已知坏CRT仍可被精确识别。批准记录应引用 qualification artifact 或 evidence ID；source/tag/version 相似不能创建批准。

### Fail-Closed 规则

以下任一情况必须阻止模型启动或镜像发布：

- Driver API 与 Runtime API 不相等。
- Image identity 是 mutable tag 或不可读。
- CRT 缺失、额外、不可读或 marker 不一致。
- DLC Custom Kernel library 或 LLVM identity 不可用。
- Policy malformed。
- exact profile 已撤销。
- exact profile 未知。

unknown 不是“可能兼容”，而是“尚未资格验证”。普通启动流程不得通过现场跑一次设备 smoke 自动修改 policy。

## Cold First-Compute Gate

仅在 Static Stack Preflight 通过、设备执行已授权且目标 device mapping 闭合后运行。

fresh-process layered C1b 必须依次覆盖：

```text
device_count
device_properties
allocation
H2D submit
device operation submit
device operation synchronize
D2H
exact correctness
```

要求：

- 在源码树外执行。
- 使用外层 bounded timeout。
- 每层输出并 flush `BEGIN`/`PASS`。
- device operation 必须是真实计算，例如 `device + 1`。
- 只有 synchronize、D2H 和 correctness 通过才是 completion evidence。
- 保存 stdout/stderr、exit code、signal、device mapping、process identity 和 cleanup evidence。

allocation、H2D 或 submit API 返回不能代替该 gate。具体 probe 和调用方式见[安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md)。

## 推荐执行顺序

```text
解析 immutable Image ID
  -> 提取 exact image artifacts
  -> 读取 Host Driver API
  -> Static Stack Preflight
  -> query-only SMI baseline
  -> fresh-process Cold First-Compute Gate
  -> task-owned cleanup closure
  -> 模型加载、功能验证或镜像发布
```

Static gate 失败时不进入 device execution。Cold gate 失败时保留最早失败边界并路由 `diagnosing-bugs`，不能继续模型加载或 benchmark。

## 退出与公共状态

| Exit | Reason | 含义 |
| ---: | --- | --- |
| 0 | `approved_profile` | exact 静态身份已批准；仍需 C1b |
| 2 | `driver_runtime_api_mismatch` | Driver/Runtime API 不相等 |
| 3 | `crt_bundle_incomplete` | CRT 文件集合缺失或额外 |
| 4 | `crt_bundle_inconsistent` | marker 或 bundle 内部不一致 |
| 5 | `known_bad_profile` | exact profile 已撤销 |
| 6 | `unknown_profile` | exact profile 未资格验证 |
| 10 | `policy_unavailable` / `invalid_policy` | policy 不可用或 malformed |
| 11 | `identity_unavailable` | immutable image、kernel 或 LLVM 身份缺失 |

动态 gate 保留自身 exit code，包括外层 timeout 常见的 `124`。调用者应保留静态和动态两份独立结果，不应折叠成一个模糊的 `healthy=true`。

## Skill 执行入口

`dlc-env-setup` 拥有 Stack Preflight 与 post-install C1b 的执行顺序和当前 CLI；执行细节（参数、退出码、脚本发现）以该 skill 及其脚本 `--help` 为权威来源，本主题文档只提供 rationale、证据分级和 Claim Boundary，不维护会漂移的命令行副本。调用前先加载该 skill 并读取脚本 `--help`。

Real DLC Hardware inventory、HBM、process、handle 和 cleanup 的规范化证据必须遵循 `SMI Observation Envelope` 的四阶段语义（`before_launch`、`after_ready`、`during_request`、`after_cleanup`），由 `dlc-hardware-observability` 提供；它不替代 C1b。

## 证据和治理边界

- Policy 是 qualification decision，不是 runtime truth。
- 新 profile 先以独立 qualification epoch 取得静态身份、cold C1b 和 cleanup 证据，再经过 review 更新 policy。
- Policy 更新与普通 startup 必须分离，防止“未知组合现场自批准”。
- Profile 中不得保存密码、token、模型权重或个人信息。
- Static preflight 只读取 artifact，不执行 reset、Driver 切换、HBM、LYP 或其他 Host 维护。

## Claim Boundary

Static Stack Preflight 可以证明当前提取的 exact stack identity 与一个已批准或撤销的 policy profile 是否完全匹配；它不能证明 Real DLC Hardware execution、模型正确性或性能。Cold First-Compute Gate 可以证明声明设备上的有界基础 execution completion 与正确性；它不能外推模型功能、distributed collective 或 benchmark 稳定性。只有两者分别通过，调用者才可进入后续模型或发布 gate。
