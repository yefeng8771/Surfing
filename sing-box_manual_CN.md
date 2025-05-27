# sing-box 核心运行文档 (Surfing 模块)

本文档旨在说明如何在 Surfing 模块环境下配置和管理 sing-box 核心。

## 1. sing-box 简介

- 简要介绍 sing-box 及其在 Surfing 模块中的作用。
- 提及 sing-box 是一个灵活的、基于规则的代理工具。

## 2. 配置 sing-box

### 2.1 核心切换
要在 Surfing 模块中使用 sing-box，您需要修改核心配置文件：

- **配置文件路径**: `/data/adb/box_bll/scripts/box.config`
- 将 `bin_name` 的值从默认的 `"clash"` 修改为 `"sing-box"`。
  ```sh
  bin_name="sing-box"
  ```

### 2.2 sing-box 自身配置
sing-box 的主要配置依赖其自身的 JSON 格式配置文件。

- **sing-box 配置目录**: `/data/adb/box_bll/sing-box/`
- **主配置文件**: 通常在此目录下的 `config.json` 文件。您需要根据 sing-box 的官方文档创建或修改此文件。
- **重要提示**: Surfing 模块的启动脚本会使用 `-D /data/adb/box_bll/sing-box/` 参数来指定此目录为 sing-box 的工作和配置目录。

### 2.3 脚本配置 (`box.config`)
以下是 `box_bll/scripts/box.config` 文件中与 sing-box 运行相关的一些通用参数：

- `box_path="/data/adb/box_bll"`: 模块的基础路径。
- `bin_path="${box_path}/bin/sing-box"`: sing-box 可执行文件的路径。
- `run_path="${box_path}/run"`: 运行时文件（如 PID 文件和部分日志）的存放路径。
- `pid_file="${run_path}/sing-box.pid"`: sing-box 进程的 PID 文件路径。
- `box_user_group="root:net_admin"`: sing-box 运行时使用的用户和用户组。

## 3. 管理 sing-box 服务

Surfing 模块通过 `box.service` 脚本管理核心服务的生命周期。

### 3.1 启动 sing-box
- **命令**: `sh /data/adb/box_bll/scripts/box.service start`
- **操作**:
    1.  首先，脚本会执行配置检查: `/data/adb/box_bll/bin/sing-box check -D /data/adb/box_bll/sing-box/`
        - 配置检查的日志会输出到 `/data/adb/box_bll/run/check.log`。如果检查失败，sing-box 不会启动。
    2.  如果配置检查通过，脚本会以指定用户组启动 sing-box: `nohup busybox setuidgid "root:net_admin" /data/adb/box_bll/bin/sing-box run -D /data/adb/box_bll/sing-box/ > /dev/null 2> /data/adb/box_bll/run/error_sing-box.log &`
    3.  成功启动后，进程 PID 会被写入 `/data/adb/box_bll/run/sing-box.pid`。

### 3.2 停止 sing-box
- **命令**: `sh /data/adb/box_bll/scripts/box.service stop`
- **操作**:
    1.  脚本会从 PID 文件读取 sing-box 的进程 ID。
    2.  尝试通过 `kill` 命令终止进程。
    3.  如果 `kill` 失败，则尝试使用 `killall sing-box`。
    4.  删除 PID 文件。

### 3.3 查看 sing-box 状态
- **命令**: `sh /data/adb/box_bll/scripts/box.service status`
- **操作**:
    - 脚本会检查 `sing-box` 进程是否存在。
    - 如果存在，会显示以下信息：
        - 运行用户组
        - PID
        - 内存使用情况
        - CPU 使用情况
        - 服务已运行时间
    - 如果进程不存在，则会提示服务已停止。

### 3.4 重启 sing-box
- **命令**: `sh /data/adb/box_bll/scripts/box.service restart`
- **操作**:
    1.  执行停止服务逻辑。
    2.  等待2秒。
    3.  执行启动服务逻辑。
    4.  等待2秒并检查服务监听状态。

## 4. 日志文件

理解和查看日志对于排查问题至关重要。

- **配置检查日志**: `/data/adb/box_bll/run/check.log`
    -  此文件包含 `sing-box check -D ...` 命令的输出。注意：此日志会被不同核心的操作覆盖。
- **运行时错误日志 (stderr)**: `/data/adb/box_bll/run/error_sing-box.log`
    -  sing-box 进程的标准错误输出会重定向到此文件。
- **主要操作日志 (stdout)**:
    -  `box.service` 脚本将 sing-box 的标准输出 (stdout) 重定向到了 `/dev/null`。
    -  **因此，您需要在 sing-box 的 `config.json` 文件中自行配置日志记录选项** (例如，配置 `log` 对象，将其输出到 `/data/adb/box_bll/sing-box/sing-box.log` 或其他希望的位置)。
    -  Surfing 模块会定期清理旧的 `.log` 文件 (位于 `${box_path}` 下，超过3天的文件)。

## 5. 运行建议与注意事项

- **确保二进制文件存在**: sing-box 可执行文件必须位于 `/data/adb/box_bll/bin/sing-box` 且具有执行权限。
- **核心配置文件**: 仔细检查 `/data/adb/box_bll/scripts/box.config` 中的 `bin_name` 是否已正确设置为 `sing-box`。
- **sing-box `config.json`**: 这是 sing-box 运行的核心。请务必根据 sing-box 官方文档正确配置，特别是监听端口、路由规则和出站等。
    -  确保配置文件 `/data/adb/box_bll/sing-box/config.json` 的语法正确。可以在 PC 上使用 sing-box 的 `check` 命令进行预检查。
- **日志配置**: 强烈建议在 sing-box 的 `config.json` 中配置详细的日志输出到特定文件，方便调试。例如：
  ```json
  {
    "log": {
      "level": "info", // 可选: debug, info, warning, error, none
      "output": "/data/adb/box_bll/sing-box/sing-box.log", // 日志输出文件
      "timestamp": true
    },
    // ... 其他配置项
  }
  ```
- **权限问题**: `box.service` 脚本会尝试以 `root:net_admin` 用户组运行 sing-box。通常情况下，这能提供足够的权限。
- **TUN 模式**: `box.service` 脚本中包含 `tun_forward_enable` 和 `tun_forward_disable` 等函数，暗示了对 TUN 模式的支持。如果您的 sing-box 配置使用 TUN 入站，这些脚本逻辑可能会自动处理相关的路由和防火墙规则。请参考 `box.service` 和 `box.config` (`tun_device` 等参数) 获取更多细节。
- **故障排查**:
    -  首先检查 `/data/adb/box_bll/run/check.log` 确认配置是否通过 sing-box 的检查。
    -  然后查看 `/data/adb/box_bll/run/error_sing-box.log` 获取 sing-box 启动或运行时的错误信息。
    -  最后，查看您在 `config.json` 中配置的主日志文件，获取详细的运行日志。
    -  使用 `sh /data/adb/box_bll/scripts/box.service status` 确认进程状态。

本文档基于对 Surfing 模块脚本的分析创建。具体行为可能随模块版本更新而有所变化。建议同时参考模块的官方 README 和 sing-box 官方文档。
