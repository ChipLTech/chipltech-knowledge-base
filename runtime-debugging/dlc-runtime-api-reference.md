# DLC Runtime 与 DLCSynapse Core 版本化接口参考

## 适用范围

本文是 DLC Runtime C API、DLCSynapse Core C API 和 `KernelDesc` launch 层在指定版本上的静态接口参考，适用于：

- PyTorch DLC Backend、vLLM DLC Custom Op 和 DLC_CL 的 host 侧集成。
- DLC Runtime 的 device、memory、stream、event、kernel、Graph 和 memory pool 开发。
- 根据接口返回值、异步完成点和对象生命周期定位问题。
- 判断一个兼容命名接口在当前版本中是真实实现、受限实现还是 compatibility stub。

本文不描述 Chipltech-Family Accelerator device kernel ABI，也不承诺 CUDA 行为兼容。DLC Platform 的主路径是 [DLC Kernel Launch Protocol](../CONTEXT.md)：host 侧通过 `KernelDesc` 打包参数，再按名称发射 DLC Custom Kernel。本文是知识库基于一手源码形成的正式版本化参考，但不构成 DLCSynapse 上游发布方的兼容性承诺或硬件验收报告。

## 版本与证据边界

| 项目 | 固定值 |
|---|---|
| 镜像 | `vllmx2:latest` |
| 镜像 ID | `sha256:0994a9bc6191ef3d7a52df2a970540849683ebb933fa16f7315ae31119bec000` |
| DLCSynapse tag | `v19.9.1` |
| DLCSynapse commit | `185592fe87d46a9a1b5a4dfe8cfe4d1db98ab7e7` |
| 审计平台 | Linux LP64 |
| DLC Runtime 声明 | `/work/DLCSynapse/external_includes/dlc_runtime_api.h` |
| DLCSynapse Core 声明 | `/work/DLCSynapse/external_includes/synapse_api.h` |
| 类型定义 | `external_includes/dlc_runtime_api.h`、`synapse_api_types.h`、`synapse_common_types.h` |
| 实现 | `dlc_runtime/dlc_runtime.cpp`、`synapse_core/*.cpp` |
| `dlc_runtime_api.h` SHA-256 | `98173f0c10f4aae0d030a4f9f0f1dc90aa6cb08b8aba9db8c5c94f8eae657765` |
| `synapse_api.h` SHA-256 | `02036d069ec4751dc0f36aacba0a5381d189c87f0db0b9a24ff5c2bfab96032c` |
| `synapse_api_types.h` SHA-256 | `af84be22c3c08c602a5f8fb2c66158853c7da684c28880800626666088503709` |
| `synapse_common_types.h` SHA-256 | `1b81478ff2efc04020f3b5e88bc70aed6727c3397a1921e80435abfe79142ef2` |
| `dlc_profiler_api.h` SHA-256 | `df3e76cf971083a6c97b5cefe1ef35b5b455496ee40820621ae9c98cc120386e` |
| `libdlc_runtime.so` | `/work/DLCSynapse/build/lib/libdlc_runtime.so`; SHA-256 `66b0069341327f4a365cf053634e07b0788a5271a1858b5b3018c634e982db33`; Build ID `f5c3d3a1f565057e9dd1c198925fb7eb557cbe2b` |
| `libsynapse_core.so` | `/work/DLCSynapse/build/lib/libsynapse_core.so`; SHA-256 `1a8fb44d69095e6acee79f216846b51fe52be0c6628d37c222ac4374f636db32`; Build ID `c6803e977e290fd02163192fbc42019992785027` |

本文依据头文件、实现和上述 `.so` 导出进行静态审计，没有完成所有接口的 Real DLC Hardware 动态验收。因此本文使用“静态支持”而不是“硬件已验证”。更换镜像、commit、ABI 或库版本后，应重新核对本参考。

覆盖口径：`dlc_runtime_api.h` 中有 170 条函数声明、169 个唯一函数名；本文对 169 个名称完成机械覆盖，并按功能族给出契约或支持结论。`synapse_api.h` 的 DLCSynapse Core 面更大，本文只完整描述 DLC Runtime 所依赖的主要接口族，并列出静态审计已识别的高风险项；它不是全部 `syn*`/`isyn*` 函数的逐签名 Doxygen 替代品。

## 支持等级

| 等级 | 含义 |
|---|---|
| 静态支持 | 声明和可链接实现存在，参数进入实际后端逻辑。仍需目标环境验证。 |
| 受限支持 | 只有部分 device type、flag、方向、执行模式或参数生效。 |
| 固定值/部分实现 | API 有行为，但部分输出固定、部分参数被忽略或语义弱于接口名称。 |
| Success stub | 返回 success，但没有执行承诺的操作，或没有写必要输出。不得作为能力证明。 |
| Unsupported | 实现明确返回 `synUnsupported` 或等价错误。 |
| ABI 缺失 | 头文件有声明，但当前库没有与声明匹配的可链接 C ABI 符号。 |
| C++ 扩展 | 使用 C++ linkage，不属于稳定 C ABI。 |
| 内部接口 | 有导出或实现，但不在目标公开头文件中，不应作为稳定公共契约。 |

接口存在、编译通过或返回 success，均不自动代表“静态支持”。

## 分层模型

```text
PyTorch DLC Backend / vLLM DLC Custom Op / DLC_CL
  -> KernelDesc / dlcKernelDesc* argument packing
  -> dlc_runtime_api.h: dlc* compatibility API
  -> dlc_runtime.cpp: handle/type/status adaptation
  -> synapse_api.h: syn* execution API
  -> synapse_core: Stream/Event/Graph/SynapseState
  -> DLCsim or dlc-thunk -> Real DLC Hardware
```

`dlc_runtime` 通常把句柄、枚举和状态码直接转换为 `syn*` 类型。很多结构使用 `reinterpret_cast` 而不是逐字段转换，因此 DLC 和 DLCSynapse 类型的大小、布局和枚举数值必须匹配。

## 公共类型与通用约定

### 句柄与所有权

`dlcStream_t`、`dlcEvent_t`、`dlcGraph_t`、`dlcGraphExec_t`、`dlcGraphNode_t`、`dlcUserObj_t` 和 `dlcCtx_t` 都是不透明句柄。通用规则：

- `Create`、`Instantiate`、`Open` 和 `Clone` 类 API 通过输出参数交付句柄。
- `Destroy`、`Free`、`Close` 或最后一次 `Release` 成功后，原句柄或地址失效。
- 销毁 API 不会统一把调用方变量置为 `nullptr`。
- `Graph`、`GraphExec`、Graph node、stream 和 event 是不同资源，不能互换句柄类型。
- `nullptr` stream 在部分接口中表示当前设备的 default stream，但具体接口仍应以声明和实现为准。

### 返回值

DLC Runtime 同时暴露两套返回类型：

- `dlcError_t`：Runtime 风格，`dlcSuccess == 0`。
- `DLCresult`：Driver 风格，`DLC_SUCCESS == 0`。

多数 `dlc*` wrapper 直接把 `synStatus` 数值转换成 `dlcError_t`。不要假设所有错误都经过 DLC 层重新映射。

### 同步、提交与完成

- `*Async`、kernel launch、Graph launch、event record 和 host callback 成功，通常只表示已提交或已入队。
- `dlcStreamSynchronize`、`dlcEventSynchronize` 和 `dlcDeviceSynchronize` 是明确的 host 等待入口。
- `dlcStreamQuery` 和 `dlcEventQuery` 是非阻塞查询；未完成通常返回 `dlcErrorNotReady`。
- DLCSynapse 内部 `synSyncLaunch()` 表示推送待处理 kernel queue，不等于等待设备完成。
- DLCsim 或 blocking 模式下部分接口会同步退化或直接成功，不能将其结果外推为 Real DLC Hardware 完成语义。

### Last error

DLCSynapse 使用线程局部 `tls_lastSynError`。`dlcGetLastError()` 和 `dlcPeekAtLastError()` 直接复用 `synGetLastError()` 和 `synPeekAtLastError()`：

- `Get` 返回错误后清空线程局部错误槽。
- `Peek` 不清空线程局部错误槽，但当前实现仍可能读取并清除 device assertion status。
- 历史 `synErrorLaunchFailure` 可能由后续无关 API 报出。
- 并非所有 `dlc*` wrapper 都一致更新 last error。

因此应优先检查当前调用的直接返回值，再用 last-error API 辅助定位异步错误。

## DLC Runtime 接口

### Device、Context 与版本

| API | 契约 | 支持等级与限制 |
|---|---|---|
| `dlcGetDeviceCount(int *count)` | 返回可见设备数量。 | 静态支持；`count` 必须非空。当前 wrapper 在后端返回后仍会解引用 `count` 写日志，传空指针有崩溃风险。 |
| `dlcGetDevice(int *device)` | 返回当前 device ordinal。 | 静态支持。 |
| `dlcSetDevice(int device)` | 设置调用线程当前设备。 | 静态支持。 |
| `dlcDeviceGet(dlcDevice_t *device, int ordinal)` | ordinal 转设备对象。 | 静态支持。 |
| `dlcGetCurrentDeviceId(...)` | 按 device type 返回当前 ID。 | 静态支持。 |
| `dlcSetDeviceWithType(...)` | 按 device type 设置当前设备。 | 受限支持，取决于 DLCSynapse device type。 |
| `dlcGetDeviceMinorById(...)` | device ID 转 driver minor。 | 静态支持。 |
| `dlcGetChipIdByDeviceId(...)` | device ID 转 chip ID。 | 静态支持。 |
| `dlcDeviceGetName(...)` | 写设备名称；第二参数是缓冲区容量。 | 静态支持。 |
| `dlcDeviceGetUuid(...)` | 写 16-byte UUID。 | 静态支持。 |
| `dlcDeviceGetPCIBusId(...)` | 写 PCI bus ID。 | 静态支持。 |
| `dlcDeviceGetAttribute(...)` | 查询单项设备属性。 | 受限支持；部分兼容属性是固定或模拟值。 |
| `dlcGetDeviceProperties(...)` | 填充 `dlcDeviceProp`。 | 受限支持；依赖与 `synDeviceProp` 二进制布局一致，CUDA 兼容字段不代表对应硬件能力。 |
| `dlcDeviceSynchronize()` | 等待当前设备已提交工作。 | 静态支持；DLCsim/blocking 路径可能退化。 |
| `dlcResetDevice()` | 重置当前设备。 | 静态支持；已有 stream、event、allocation 和 GraphExec 应视为失效。 |
| `dlcCtxGetCurrent(dlcCtx_t *ctx)` | 返回当前 context。 | 受限支持；DLCSynapse context 不是完整 CUDA context stack。 |
| `dlcDevicePrimaryCtxGetState(...)` | 返回 primary context flags/active。 | 固定值/部分实现；失败会被压缩为 `DLC_ERROR_NO_DEVICE`。 |
| `dlcDriverGetVersion(int *version)` | 返回 driver 版本。 | 静态支持。 |
| `dlcRuntimeGetVersion(int *version)` | 返回 DLC Runtime 版本。 | 静态支持。 |

Graph memory 属性接口 `dlcDeviceGetGraphMemAttribute`、`dlcDeviceSetGraphMemAttribute` 和 `dlcDeviceGraphMemTrim` 有后端实现，但属性范围和 trim 结果取决于 DLCSynapse Graph memory manager，属于受限支持。

### Device Memory 与 Host Memory

| API | 契约 | 支持等级与限制 |
|---|---|---|
| `dlcMalloc(void **ptr, size_t bytes)` | 当前设备 HBM 分配。 | 静态支持；可能额外初始化 kernel library。 |
| `dlcFree(void *ptr)` | 释放 device allocation；`nullptr` 可成功。 | 静态支持。 |
| `dlcMallocAsync(void **ptr, size_t bytes, dlcStream_t stream)` | stream 关联分配。 | 受限支持；底层错误被统一转换为 memory-allocation error。 |
| `dlcFreeAsync(void *ptr, dlcStream_t stream)` | 按 stream 顺序延迟释放。 | 受限支持；固定按 HBM 处理。 |
| `dlcMemGetInfo(size_t *free, size_t *total)` | 查询当前设备 HBM free/total bytes。 | 静态支持。 |
| `dlcMemGetAddressRange(void **base, size_t *size, void *ptr)` | 查询 allocation 基址和范围。 | 静态支持。 |
| `dlcPointerGetAttributes(...)` | 查询 host/device 指针属性。 | 受限支持；部分字段可能未被完整初始化。 |
| `dlcAddrIsDevice(uint64_t addr)` | 根据 DLC 地址编码判断 device address。 | 静态支持，但仅适用于当前地址编码。 |
| `dlc_get_device_id_from_uvm(uint64_t addr)` | 从 UVM 编码地址提取设备 ID。 | 内部兼容接口。 |
| `dlcHostAlloc(void **ptr, size_t bytes, unsigned flags)` | 分配 host memory。 | 静态支持；头文件中重复声明一次，语义不变。 |
| `dlcFreeHost(void *ptr)` | 释放 `dlcHostAlloc` 所得内存。 | 静态支持。 |
| `dlcHostGetFlags(...)` | 查询 host allocation flags。 | 受限支持；DLCsim 路径可能返回 success 但不写输出。 |
| `dlcHostGetDevicePointer(...)` | 返回 host allocation 的 device-visible 地址。 | 受限支持。 |
| `dlcMemPrefetchAsync(...)` | 请求范围预取到目标设备。 | 固定值/部分实现；底层当前主要执行 queue flush，不应依赖真实迁移。 |
| `dlcHostRegister(...)` | 注册已有 host memory。 | **Success stub**，不执行注册。 |
| `dlcHostUnregister(...)` | 注销 host memory。 | **Success stub**，不执行注销。 |

以下 Driver-style virtual memory API 全部是 **Success stub**，当前不会保留地址、创建 allocation、映射、设权限、解除映射或释放：

```text
dlcMemAddressReserve  dlcMemCreate       dlcMemMap
dlcMemSetAccess       dlcMemUnmap        dlcMemRelease
dlcMemAddressFree
```

其中带输出参数的接口可能返回 `DLC_SUCCESS` 但不写输出。调用方不得使用这些输出。

### Memory Copy 与 Memset

```c
dlcError_t dlcMemcpy(void *dst, const void *src,
                     size_t count, dlcMemcpyKind kind);
dlcError_t dlcMemcpyAsync(void *dst, const void *src,
                          size_t count, dlcMemcpyKind kind,
                          dlcStream_t stream);
dlcError_t dlcMemCopySync(uint64_t src, uint64_t size,
                          uint64_t dst, dlcDmaDir direction);
dlcError_t dlcMemCopyAsync(dlcStream_t stream, uint64_t src,
                           uint64_t size, uint64_t dst,
                           dlcDmaDir direction);
```

注意 `dlcMemCopy*` 的参数顺序是 `src, size, dst`，不是常见的 `dst, src, size`。

| API | 契约 | 支持等级与限制 |
|---|---|---|
| `dlcMemcpy` | H2H/H2D/D2H/D2D 同步 copy。 | 静态支持；D2D 可能经 host staging。 |
| `dlcMemcpyAsync` | 按 stream copy。 | 受限支持；H2H 在调用线程立即执行，不具备 stream 异步语义。 |
| `dlcMemCopySync` | 使用显式 `dlcDmaDir` 同步 copy。 | 静态支持。 |
| `dlcMemCopyAsync` | 使用显式 `dlcDmaDir` 异步 copy。 | 静态支持。 |
| `dlcMemcpyCmaSync` | 通过 CMA buffer 分块 H2D/D2H。 | 受限支持，仅 H2D/D2H。 |
| `dlcMemcpyPeerAsync` | 跨设备 copy。 | 受限支持；忽略显式 src/dst device 参数，实际从地址解析，可能使用 magic RDMA 或 host staging。 |
| `dlcPassThroughMemcpySync` | 专用 passthrough copy。 | 受限内部接口。 |
| `dlcRDMASYNC`、`dlcRDMAD2H` | 专用 RDMA 数据路径。 | 受限内部接口，底层错误传播不完整。 |
| `dlcMemset` | 同步填充 `count` bytes。 | 静态支持。 |
| `dlcMemsetAsync` | 名义上的异步 byte memset。 | **存在缺陷，不建议用于非 4-byte 对齐范围**：`DLC_MEMSET_MODE=1`、默认模式且小于 256 KiB，或 stream capture active 时走 D32 路径并忽略 `count % 4`；其他路径发射 `custom_full`，仍以 `count / 4` 构造 int32 输出。value 不是严格逐字节语义。 |
| `dlcMemsetD8Async` | 填充 `count` 个 8-bit 元素。 | 受限支持；底层 callback size 计算存在缺陷。 |
| `dlcMemsetD16Async` | 填充 `count` 个 16-bit 元素。 | 受限支持；底层 callback size 计算存在缺陷。 |
| `dlcMemsetD32Async` | 填充 `count` 个 32-bit 元素。 | 静态支持。 |

`dlcDeviceEnablePeerAccess` 和 `dlcDeviceCanAccessPeer` 都是 **Success stub**；后者不会写 `*canAccessPeer`。它们不能用于证明 peer access 已建立。`dlcMemcpyPeerAsync` 的实现也不依赖这两个接口建立真实 peer mapping。

### Stream

| API | 契约 | 支持等级与限制 |
|---|---|---|
| `dlcStreamCreate` | 在当前设备创建 stream。 | 静态支持。 |
| `dlcStreamCreateWithFlags` | 使用 flags 创建 stream。 | 静态支持。 |
| `dlcStreamCreateWithPriority` | 创建 priority stream。 | 受限支持；当前只接受 `dlcStreamNonBlocking` flags。 |
| `dlcStreamGetPriority` | 查询 stream priority。 | 受限支持；DLC Runtime/DLCSynapse 声明存在一级指针类型异常。 |
| `dlcDeviceGetStreamPriorityRange` | 返回 least/greatest priority。 | 固定值实现：least `1`、greatest `-1`。 |
| `dlcStreamDestroy` | 销毁 stream。 | 静态支持；句柄随后失效。 |
| `dlcStreamSynchronize` | 等待 stream 完成。 | 静态支持。 |
| `dlcStreamQuery` | 非阻塞完成查询。 | 静态支持。 |
| `dlcStreamWaitEvent` | 令 stream 后续工作等待 event。 | 静态支持。 |
| `dlcLaunchHostFunc` | 在 stream 中排入 host callback。 | 静态支持；`userData` 必须存活到 callback，callback 不应等待同一 stream。 |

### Event

| API | 契约 | 支持等级与限制 |
|---|---|---|
| `dlcEventCreate` | 创建默认 event。 | 静态支持。 |
| `dlcEventCreateWithFlags` | 使用 flags 创建 event。 | 静态支持；由 wrapper 调用显式 device 版创建。 |
| `dlcEventDestroy` | 销毁 event。 | 静态支持。 |
| `dlcEventRecord` | 将 event 记录在 stream 时间线上。 | 静态支持。 |
| `dlcEventQuery` | 非阻塞查询。 | 静态支持。 |
| `dlcEventSynchronize` | host 等待 event。 | 静态支持。 |
| `dlcEventElapsedTime(float *ms, start, end)` | 返回两个 event 间隔，单位毫秒。 | 静态支持；event 必须支持 timing 且已完成。 |

### Kernel Launch

```c
dlcError_t dlcLaunchKernelName(
    dlcStream_t stream,
    const char *name,
    unsigned argc,
    unsigned char *argv,
    unsigned *offsets,
    uint64_t ops,
    unsigned output_count,
    unsigned *output_ranges,
    uint64_t nbytes,
    const char *msg);
```

| 参数 | 语义 |
|---|---|
| `stream` | 提交目标；`nullptr` 表示当前设备 default stream。 |
| `name` | 已注册 DLC Custom Kernel 名称。 |
| `argc` | 参数数量，包含协议保留的 `DLCMem` 参数。 |
| `argv` | 序列化参数缓冲区。 |
| `offsets` | 长度 `argc + 1`，最后一项是缓冲区末尾偏移。 |
| `ops` | profiler/统计使用的操作数，不影响计算语义。 |
| `output_count`、`output_ranges` | 输出区域描述。 |
| `nbytes` | 业务数据量统计，不是 `argv` 长度。 |
| `msg` | 可选诊断文本。 |

`dlcLaunchKernelName` 先按名称查询 kernel ID，再调用 `synPushKernelQueue`。成功返回只代表提交，不代表完成。`dlcInitCustomKernels` 初始化 DLC Custom Kernel registry；`dlcManualPushKernelQueue` flush 当前 queue；`dlcGetLaunchedKernelOps` 返回当前线程累计 `ops`，不按设备或 stream 隔离，也没有清零接口。

`dlcLaunchKernel` 是 function-handle launch 入口：

```c
dlcError_t dlcLaunchKernel(const void *func,
                           dlcDim3 gridDim, dlcDim3 blockDim,
                           void **args, void **args_xys1,
                           size_t sharedMem, dlcStream_t stream);
```

它把 `func` 解释为 DLCSynapse function handle，并固定内部 core selection 参数。它不是 CUDA device kernel ABI 的通用承诺。

### `KernelDesc` 与 C Adapter

`external_includes/kernel_launcher.hpp` 中的 `KernelDesc` 是 PyTorch DLC Backend 和其他框架集成的主要参数打包对象：

- `input(...)`、`output(...)`：序列化 tensor address、shape、stride、dtype、format 和 device 信息。
- `scalar_as<T>(...)`、`scalar_lite_as<T>(...)`：序列化 scalar。
- `set_nbytes(...)`：设置统计字节数。
- `launch(name, stream)`：最终调用 `dlcLaunchKernelName`。

限制：

- `KernelDesc` 不可复制、不可移动。
- `post_fn` 在提交返回后立即执行，不等待 kernel 完成。
- 当前 `KernelDesc::launch` 丢弃 `dlcLaunchKernelName` 返回码，提交失败也可能执行 `post_fn`。
- 高维 tensor 只保存有限 extra dimensions，更高维可能截断。
- kernel name 长度达到 63 bytes 时存在缺少显式 NUL 终止的风险。

`kernel_desc_adapter.cpp` 还导出九个 C adapter：

```text
dlcKernelDescGet                 dlcKernelDescNextArg
dlcKernelDescScalarLiteInt       dlcKernelDescSetNBytes
dlcKernelDescLaunch              dlcKernelDescPushOutputDtype
dlcKernelDescPushOutputRange     dlcKernelDescInputTensor
dlcKernelDescOutputTensor
```

这些符号不在 `dlc_runtime_api.h` 中，属于单独的 adapter contract。`dlcKernelDescGet` 返回 thread-local 对象；调用方不得释放、不得跨线程传递，同一线程下一次调用会重置原对象。

### Stream Capture 与 Graph

Capture 生命周期：

```text
dlcStreamBeginCapture
  -> 在 stream 上提交可捕获操作
  -> dlcStreamEndCapture -> dlcGraph_t
  -> dlcGraphInstantiate* -> dlcGraphExec_t
  -> dlcGraphLaunch(stream)
  -> synchronize/query
  -> dlcGraphExecDestroy + dlcGraphDestroy
```

Capture API：

| API | 支持等级与限制 |
|---|---|
| `dlcStreamBeginCapture`、`dlcStreamEndCapture` | 静态支持。 |
| `dlcStreamIsCapturing` | 静态支持。 |
| `dlcStreamGetCaptureInfo` | 受限支持；返回的 dependencies 是内部借用数组，下一次修改 stream/capture 后可失效；edge data 未完整实现。 |
| `dlcStreamUpdateCaptureDependencies` | 受限支持；节点依赖生效，`dependencyData` 被忽略。 |
| `dlcThreadExchangeStreamCaptureMode` | 静态支持，模式是线程局部状态。 |

Graph API 共 57 个唯一函数，按资源族完整列举如下：

| 功能 | API |
|---|---|
| 创建、实例化、执行、销毁 | `dlcGraphCreate`, `dlcGraphInstantiate`, `dlcGraphInstantiateWithFlags`, `dlcGraphInstantiateWithParams`, `dlcGraphLaunch`, `dlcGraphUpload`, `dlcGraphDestroy`, `dlcGraphExecDestroy` |
| 通用节点 | `dlcGraphAddNode`, `dlcGraphDestroyNode`, `dlcGraphNodeGetType`, `dlcGraphExecNodeSetParams` |
| 空节点 | `dlcGraphAddEmptyNode` |
| Memcpy 节点 | `dlcGraphAddMemcpyNode`, `dlcGraphMemcpyNodeGetParams`, `dlcGraphMemcpyNodeSetParams`, `dlcGraphExecMemcpyNodeSetParams` |
| Memset 节点 | `dlcGraphAddMemsetNode`, `dlcGraphMemsetNodeGetParams`, `dlcGraphMemsetNodeSetParams`, `dlcGraphExecMemsetNodeSetParams` |
| Allocation 节点 | `dlcGraphAddMemAllocNode`, `dlcGraphMemAllocNodeGetParams`, `dlcGraphAddMemFreeNode`, `dlcGraphMemFreeNodeGetParams` |
| Host 节点 | `dlcGraphAddHostNode`, `dlcGraphHostNodeGetParams`, `dlcGraphHostNodeSetParams`, `dlcGraphExecHostNodeSetParams` |
| Kernel 节点 | `dlcGraphAddKernelNode`, `dlcGraphKernelNodeGetParams`, `dlcGraphKernelNodeSetParams`, `dlcGraphExecKernelNodeSetParams` |
| Event 节点 | `dlcGraphAddEventRecordNode`, `dlcGraphEventRecordNodeGetEvent`, `dlcGraphEventRecordNodeSetEvent`, `dlcGraphExecEventRecordNodeSetEvent`, `dlcGraphAddEventWaitNode`, `dlcGraphEventWaitNodeGetEvent`, `dlcGraphEventWaitNodeSetEvent`, `dlcGraphExecEventWaitNodeSetEvent` |
| 依赖与拓扑 | `dlcGraphGetRootNodes`, `dlcGraphAddDependencies`, `dlcGraphRemoveDependencies`, `dlcGraphGetNodes`, `dlcGraphGetEdges`, `dlcGraphNodeGetDependencies`, `dlcGraphNodeGetDependentNodes` |
| Clone 与 child graph | `dlcGraphClone`, `dlcGraphNodeFindInClone`, `dlcGraphAddChildGraphNode`, `dlcGraphChildGraphNodeGetGraph`, `dlcGraphExecChildGraphNodeSetParams` |
| 更新与调试 | `dlcGraphExecUpdate`, `dlcGraphDebugDotPrint` |
| User object | `dlcGraphRetainUserObject`, `dlcGraphReleaseUserObject` |

Graph 总体为静态支持，但有以下正式限制：

- `dlcGraphUpload` 当前只做状态检查，属于成功占位，不执行真实 upload。
- Graph edge metadata 被忽略。
- instantiate flags 只支持少量单值，不支持任意组合。
- 包含 alloc/free 节点的 Graph 有重复实例化限制。
- `dlcGraphInstantiate` 的第五参数在头文件叫 `flags`、实现叫 `bufferSize`，实际被忽略；`pErrorNode` 和 `pLogBuffer` 也被忽略。
- `dlcGraphInstantiateWithFlags` 在 Linux LP64 上因 `unsigned long long` 与实现 `uint64_t` 类型漂移，当前 `.so` 只有 C++ mangled symbol，公开 C ABI **缺失**。
- Graph GetParams 声明错误地使用 `const` 输出指针；实现通过 `const_cast` 写入。调用方必须提供真实可写对象。
- `dlcGraphDestroyNode` 延续了后端的 node/graph 句柄类型错误，存在缺陷，不建议单独销毁 node。
- `dlcGraphMemFreeNodeGetParams` 的输出类型不是 `void **`，不能可靠回传地址。

### User Object

`dlcUserObjectCreate`、`dlcUserObjectRetain`、`dlcUserObjectRelease` 提供引用计数对象。创建时 `destroy != nullptr`、`initialRefcount > 0`；引用归零时调用 `destroy(objectData)`。Graph 可以 retain/release user object，因此销毁顺序应由引用计数决定。

### IPC

| API | 支持等级与限制 |
|---|---|
| `dlcIpcGetMemHandle` | 受限支持；当前 handle 主要保存 DLC 虚拟地址和范围。 |
| `dlcIpcOpenMemHandle` | 受限支持；恢复地址并增加软件引用，不是完整 OS 级跨进程映射承诺。 |
| `dlcIpcCloseMemHandle` | 受限支持；减少软件引用。 |
| `dlcIpcGetEventHandle` | **Success stub**，不写输出。 |
| `dlcIpcOpenEventHandle` | **Success stub**，不写输出。 |

### Memory Pool

| API | 支持等级与限制 |
|---|---|
| `dlcMemPoolCreate`、`dlcMemPoolDestroy` | 受限支持；create 主要使用 location，其他 properties 被忽略。 |
| `dlcMemPoolSetAttribute`、`dlcMemPoolGetAttribute` | 受限支持；只支持部分 reuse/threshold/stat 属性。 |
| `dlcMemPoolSetAccess` | 固定值/部分实现；主要维护软件 access map。 |
| `dlcMemPoolTrimTo` | 静态支持。 |
| `dlcMemAllocFromPoolAsync` | 受限支持；allocation 并非严格 stream-ordered。 |
| `dlcDeviceGetDefaultMemPool` | **Success stub**，不写输出。 |

### Profiler 与 Callback

| API | 支持等级与限制 |
|---|---|
| `dlcProfilerFlush` | 静态支持，flush 内部 profiler。 |
| `dlcProfilerGetDumpFilename` | 静态支持；返回内部字符串，调用方不得释放，需要长期保存时复制。 |
| `dlcProfilerStart`、`dlcProfilerStop` | 在独立 `dlc_profiler_api.h` 声明，不在 `dlc_runtime_api.h`；实现明确 unsupported 却返回 success。 |
| `dlcRegisterInt2Host2Callback`、`dlcDeregisterInt2Host2Callback` | 受限内部接口；callback 类型是无类型 `void *`，公共契约不足。 |

## DLCSynapse Core 接口

`synapse_api.h` 是 DLC Runtime 的后端接口面，不应默认作为 PyTorch DLC Backend 的首选直接接口。下文是主要接口族和已识别高风险项，不是逐函数穷举；未列出的 Core 接口不能自动视为静态支持。

### 主要接口族与支持状态

| 接口族 | 代表接口 | 当前结论 |
|---|---|---|
| 初始化 | `synInitialize`, `synDestroy` | 静态支持；多数 API 也会隐式初始化全局状态。 |
| Stream | `synStreamCreate*`, `Destroy`, `WaitEvent`, `Synchronize`, `Query` | 基本支持；stream type 未完整建模，priority 签名有缺陷。 |
| Event | `synEventCreate*`, `Destroy`, `Record`, `Query`, `Synchronize`, `ElapsedTime` | 基本支持；DLCsim/blocking 有同步退化和未写输出风险。 |
| Device | `synDeviceGetCount`, `Set/GetDevice`, properties, version, synchronize/reset | 基本支持；部分 device type 和属性受限。 |
| Memory | `synDeviceMalloc/Free`, `synMalloc/Free`, host alloc/map/register, memcpy/memset | 基本支持但含下述 stub/缺陷。 |
| Memory pool | `synMemPool*`, `synMemAllocFromPoolAsync`, `synFreeAsync`, `synMallocAsync` | 受限支持，stream ordering 和 access 主要为软件语义。 |
| Kernel/module | `synModuleLoadDataEx`, `synModuleGetFunction`, `synLaunchKernel`, `synPushKernelQueue` | 静态支持；`synFuncSetAttribute` 是成功占位。 |
| Graph/capture | Graph node、instantiate、launch、update、clone、capture | 大部分静态支持；upload、edge data 和 flags 受限。 |
| Context | `synCtx*`, primary context | 部分实现；多个配置 API 是 fixed-value/NOP，且存在生命周期缺陷。 |
| User object | `synUserObject*`, Graph retain/release | 静态支持。 |
| Configuration/Section/Profiler | `synConfiguration*`, `synSection*`, `synProfiler*` | 大部分明确 Unsupported。 |

### 已识别的 Unsupported `syn*` 接口

```text
synSectionCreate                 synSectionSetGroup
synSectionGetGroup               synSectionSetPersistent
synSectionGetPersistent          synSectionDestroy
synMemCopyAsyncMultiple          synDeviceAcquireByModuleId
synDeviceAcquire                 synDeviceEnablePeerAccess
synNodeTypeSetPrecision          synNodeDependencySet
synDeviceGetInfo                 synConfigurationSet
synConfigurationGet             synProfilerStart
synProfilerStop                  synProfilerGetTrace
```

`synSectionSetRMW(true)` 也返回 Unsupported；设置 false 成功但不会保存状态。

### 已识别的 Success stub 或固定值接口

- `synMemAllocManaged`：返回 success，但不写 allocation 地址。
- `synMemPrefetchAsync`：主要 flush queue 后返回 success，不执行明确迁移。
- `synMemHostGetDevicePointer`：返回 success，但不写输出。
- `synDevicePrimaryCtxSetFlags`：忽略 flags。
- `synCtxGetCacheConfig`、`synCtxGetSharedMemConfig`：返回固定空配置。
- `synCtxSetCacheConfig`、`synCtxSetSharedMemConfig`：NOP。
- `synCtxSynchronize`、`synCtxResetPersistingL2Cache`：直接 success。
- `synGraphUpload`：只做检查，无实际 upload。
- `synFuncSetAttribute`：直接 success，属性未生效。
- `synSectionGetRMW`：固定写 false 后返回 success。

### 已知实现缺陷

- `synCtxDestroy` 删除 context 后继续读取字段，存在 use-after-free。
- `synDevicePrimaryCtxRelease` 可能无条件 pop 空 TLS stack。
- `synMemSet`、`synMemset` 在检查 `malloc` 结果前使用 buffer。
- `synMemsetD8Async`、`synMemsetD16Async` 的 callback size 使用错误元素宽度。
- `synPointerGetAttributes` 只写部分字段。
- `synDeviceFree`、专用 passthrough/RDMA copy 的底层错误传播不完整。
- `synGraphDestroyNode` 使用 `synGraphHandle` 参数类型而不是 node handle。
- `synGraphMemFreeNodeGetParams` 输出参数类型设计错误。

### C 与 C++ ABI 边界

`synMagicCopyAsync` 和 `synLaunchKernelQueue` 位于 `extern "C"` 之外，在 C++ 分支中还有默认参数。它们是 C++ extension，当前库导出 mangled symbol，不属于稳定 DLCSynapse C ABI。

## 最小 C++ 使用示例

以下示例展示基础生命周期和完成边界，不代表所有错误恢复策略：

```cpp
#include <dlc_runtime_api.h>

#include <cstddef>
#include <cstdint>
#include <cstdio>

#define DLC_CHECK(call)                                                   \
  do {                                                                    \
    dlcError_t status = (call);                                           \
    if (status != dlcSuccess) {                                           \
      std::fprintf(stderr, "%s failed: %s\n", #call,                     \
                   dlcGetErrorString(status));                            \
      return 1;                                                           \
    }                                                                     \
  } while (0)

int main() {
  int count = 0;
  DLC_CHECK(dlcGetDeviceCount(&count));
  if (count == 0) return 0;

  DLC_CHECK(dlcSetDevice(0));

  dlcStream_t stream = nullptr;
  dlcEvent_t done = nullptr;
  void *device = nullptr;

  DLC_CHECK(dlcStreamCreate(&stream));
  DLC_CHECK(dlcEventCreate(&done));
  DLC_CHECK(dlcMalloc(&device, 4096));
  DLC_CHECK(dlcMemsetD32Async(
      reinterpret_cast<DLCdeviceptr>(device), 0, 4096 / sizeof(uint32_t),
      stream));
  DLC_CHECK(dlcEventRecord(done, stream));
  DLC_CHECK(dlcEventSynchronize(done));

  DLC_CHECK(dlcFree(device));
  DLC_CHECK(dlcEventDestroy(done));
  DLC_CHECK(dlcStreamDestroy(stream));
  return 0;
}
```

实际代码应使用单一 cleanup 路径处理中途失败，避免泄漏已创建资源。

## PyTorch 对应关系

PyTorch 的 `torch.dlc` / `torch.backends.dlc` 是更高层接口，不与每个 `dlc*` C API 一一公开对应：

| PyTorch API/操作 | 主要底层能力 |
|---|---|
| `torch.dlc.device_count()` | device enumeration |
| `torch.dlc.set_device(i)` | current device selection |
| `torch.dlc.get_device_properties(i)` | device properties |
| `torch.dlc.mem_get_info()` | HBM memory info |
| `torch.empty(..., device="dlc:i")` | device allocation |
| `.to("dlc:i")` / `.cpu()` | H2D / D2H copy |
| DLC tensor operator | PyTorch DLC Backend -> `KernelDesc` -> named kernel launch |
| `torch.dlc.synchronize()` | current device synchronization |

PyTorch API 可见、allocation 成功或 H2D 成功，都不能证明 kernel execution 和 completion path 健康。完整 smoke 见 [安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md)。

## 禁止依赖清单

当前版本不得在生产逻辑中依赖以下接口的 success 返回值：

```text
dlcDeviceEnablePeerAccess       dlcDeviceCanAccessPeer
dlcIpcOpenEventHandle           dlcIpcGetEventHandle
dlcDeviceGetDefaultMemPool      dlcHostRegister
dlcHostUnregister               dlcMemAddressReserve
dlcMemCreate                    dlcMemMap
dlcMemSetAccess                 dlcMemUnmap
dlcMemRelease                   dlcMemAddressFree
```

也不得将以下兼容性事实写成能力结论：

- 头文件沿用 CUDA 名称或注释，不等于 CUDA 行为兼容。
- API 返回 success，不等于操作已经执行。
- kernel launch 返回 success，不等于 kernel 已完成。
- DLCsim 通过，不等于 Real DLC Hardware 已验证。
- Graph/peer/IPC/VMM 符号存在，不等于对应高级能力可用。

## 升级时的复核清单

1. 固定新镜像 digest、DLCSynapse commit/tag 和 Linux ABI。
2. 对 `dlc_runtime_api.h`、`synapse_api.h` 和状态类型头重新计算摘要。
3. 比较公开声明与 `libdlc_runtime.so`、`libsynapse_core.so` 导出。
4. 检索重复声明、`TODO`、`Unsupported` 和直接 success 返回。
5. 检查 `dlcGraphInstantiateWithFlags` 是否恢复正确 C ABI。
6. 检查所有 Success stub 是否改为真实实现或明确 unsupported。
7. 在 DLCsim 和 Real DLC Hardware 分别验证最小 lifecycle、copy、stream/event、kernel 和 Graph case。
8. 更新本文的版本表、支持等级和已知缺陷，不用新版本覆盖旧版本事实。

## 相关资料

- [Runtime 排障指南](runtime-troubleshooting.md)
- [DLCSynapse 使用指南](../debugging-workflows/synapse-usage-guide.md)
- [安装后的 Runtime Smoke](../debugging-workflows/post-install-runtime-smoke.md)
- [PyTorch DLC Backend 算子接入指南](../pytorch-dlc-backend/operator-integration-guide.md)
- [DLC 与 CUDA 对比](../foundation/dlc-vs-cuda-comparison.md)
- [CONTEXT.md](../CONTEXT.md)

## 一手来源

- `/work/DLCSynapse/external_includes/dlc_runtime_api.h`
- `/work/DLCSynapse/external_includes/synapse_api.h`
- `/work/DLCSynapse/external_includes/synapse_api_types.h`
- `/work/DLCSynapse/external_includes/synapse_common_types.h`
- `/work/DLCSynapse/external_includes/dlc_profiler_api.h`
- `/work/DLCSynapse/external_includes/kernel_launcher.hpp`
- `/work/DLCSynapse/dlc_runtime/dlc_runtime.cpp`
- `/work/DLCSynapse/dlc_runtime/kernel_desc_adapter.cpp`
- `/work/DLCSynapse/synapse_core/synapse_core.cpp`
- `/work/DLCSynapse/synapse_core/synapse_ctx.cpp`
- `/work/DLCSynapse/synapse_core/synapse_graph.cpp`
- `/work/DLCSynapse/synapse_core/synapse_state.cpp`
- `/work/DLCSynapse/synapse_core/stream.cpp`
- `/work/DLCSynapse/synapse_core/event.hpp`

以上来源均对应本文“版本与证据边界”中的固定 commit。
