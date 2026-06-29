# VSCode 调试配置

## 适用场景

- 在 VSCode 中远程连接到 DLC 开发服务器。
- 配置 Python + DLC 的调试环境。
- 调试 PyTorch DLC Backend 或 DLC Custom Kernel 相关代码。

## 核心结论

VSCode remote SSH + Python debugger 配置可用于 DLC 开发环境，关键是正确设置 Python 解释器路径和环境变量。

## 操作流程

### SSH 配置

1. 打开 VSCode，使用 Remote-SSH 扩展。
2. 打开 SSH config 文件，添加开发服务器信息：

```
Host dlc-dev
    HostName <server_ip>
    User <username>
    Port <port>
    IdentityFile <path_to_key>
```

3. Connect 到远程服务器。

### Debug 配置

1. 在工作区创建 `.vscode/launch.json`。
2. 添加 Python 调试配置：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: DLC Debug",
            "type": "debugpy",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal",
            "env": {
                "DLC_SYN_BLOCKING": "1",
                "DLC_SYN_DEBUG": "1"
            },
            "python": "/path/to/python/interpreter"
        }
    ]
}
```

3. 关键设置：
   - `python` 字段指向正确的 Python 解释器（安装有 DLC pytorch 的）。
   - `env` 中设置必要的环境变量。
   - 调试时可能需要 `DLC_SYN_BLOCKING=1` 以确保同步执行。

### 开始调试

1. 打开要调试的 Python 文件。
2. 在关键位置设置 breakpoint。
3. 按 F5 或选择对应的 debug 配置开始调试。

## 常见坑

1. **Python 解释器错误**：确保使用安装了 DLC PyTorch 的 Python 环境。
2. **环境变量不生效**：DLC 相关环境变量需要在 launch.json 中显式设置。
3. **异步执行导致 breakpoint 行为异常**：使用 `DLC_SYN_BLOCKING=1` 切换到同步模式。
4. **permission denied**：SSH key 和服务器访问权限需要正确配置。

## 相关资料

- [debugging-workflows/common-debug-commands.md](../debugging-workflows/common-debug-commands.md)

## 来源

- `/work/plan/dlc基础/vscode调试配置debug.md`
