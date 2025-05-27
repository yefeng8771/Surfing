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

## 6. 配置 TUN 模式 (重要更新)

TUN 模式允许 sing-box 创建一个虚拟网络接口，并接管设备的全部（或部分）网络流量（包括 TCP 和 UDP），实现系统级的透明代理。**此部分已更新，引入了一个新的配置选项来更好地管理 Surfing 脚本与 sing-box 自身路由功能的交互。**

### 6.1 TUN 模式简介
- **功能**: 通过创建名为 `tun_device` (在 `box.config` 中定义，默认为 "Meta") 的虚拟网卡，sing-box 可以捕获所有流向该网卡的网络流量，并根据其内部配置进行处理和转发。
- **优势**: 相比 TProxy 或 Redirect 模式，TUN 模式通常能更全面地处理所有应用的各种协议流量，并且在路由控制上可以更为精细。
- **Surfing 模块支持**: `box.service` 脚本包含用于配置系统路由和防火墙规则以配合 TUN 设备工作的函数。通过新的配置变量，您可以选择是由 Surfing 脚本主要管理路由，还是让 sing-box (通过其 `auto_route: true` 设置) 主要管理路由。

### 6.2 Surfing 模块中的 TUN 配置 (`box.config`)

#### 6.2.1 `tun_device`
- `tun_device="Meta"`: 定义了 sing-box 创建的 TUN 接口名称。**此名称必须与您的 sing-box `config.json` 中 TUN 入站设置的 `interface_name` 完全一致。**

#### 6.2.2 (新增) `SURFING_TUN_MODE_PREFERS_SINGBOX_ROUTING` (手动添加)
为了更灵活地控制 TUN 模式下的路由管理方式，引入了一个新的配置变量。由于工具限制，**您需要手动将以下代码块添加到您的 `/data/adb/box_bll/scripts/box.config` 文件中**，通常可以放在文件末尾：

```sh
# -------------------------
# TUN Mode Advanced Settings
# -------------------------
# SURFING_TUN_MODE_PREFERS_SINGBOX_ROUTING:
# This setting determines how Surfing scripts interact with sing-box's own
# TUN routing capabilities (specifically the "auto_route" option in sing-box's
# TUN inbound configuration).
#
# Set to "true" IF:
#   1. You are using sing-box as the core.
#   2. You have configured "auto_route": true in your sing-box
#      TUN inbound settings (e.g., in /data/adb/box_bll/sing-box/config.json).
#   3. You want sing-box to primarily manage system routes for the TUN interface.
#   In this "true" case, Surfing scripts will apply minimal system routing rules
#   (IP forwarding, rp_filter, essential iptables FORWARD rules)
#   to avoid conflicts with sing-box.
#
# Set to "false" (or leave undefined/commented out) IF:
#   1. You prefer Surfing scripts to manage system routes for the TUN interface.
#   2. You have set "auto_route": false in your sing-box TUN inbound configuration.
#   This is the default behavior and is generally recommended unless you are an
#   advanced user and understand the implications of sing-box's auto_route.
#
SURFING_TUN_MODE_PREFERS_SINGBOX_ROUTING="false"
```
**`box.service` 脚本会读取此变量的值来决定其 TUN 路由配置策略。**

#### 6.2.3 `proxy_method` 与 `box.tproxy` 的影响
- `proxy_method` (例如 TPROXY, REDIRECT): 此设置主要影响 `box.tproxy` 脚本的行为，该脚本负责设置 iptables 规则来拦截流量。
- **重要**: 如果您将 `SURFING_TUN_MODE_PREFERS_SINGBOX_ROUTING` 设置为 `"true"` (即 sing-box 主导路由)，强烈建议您检查 `box.tproxy` 的行为。理想情况下，当 sing-box TUN 通过路由全面接管流量时，`box.tproxy` 不应再添加可能冲突的 iptables 拦截规则 (如 DNAT 或 TPROXY 规则)。
    - 您可能需要将 `proxy_method` 设置为一个能让 `box.tproxy` 添加最少规则的模式。
    - 高级用户可能需要手动注释掉 `box.tproxy` 脚本中的部分规则生成代码，以避免冲突。未来版本的 Surfing 模块可能会更好地集成此场景。

**注意**: `README_CN.md` 中曾提及 TUN 模式 "v7.4.3 弃用"。虽然相关脚本函数依然存在并已更新，用户在配置时仍应留意此信息，并进行充分测试。

### 6.3 sing-box `config.json` 中的 TUN 配置

用户**必须自行配置** sing-box 的 `config.json` 文件以启用 TUN 入站。以下是配置示例，展示了两种主要场景。

#### 6.3.1 场景 A: Surfing 脚本主导路由 (推荐，默认)
   - 在 `box.config` 中设置 `SURFING_TUN_MODE_PREFERS_SINGBOX_ROUTING="false"` (或未设置此变量)。
   - 在 sing-box `config.json` 的 TUN inbound 中设置 `"auto_route": false`。

```json
{
  // ... (全局 log, dns, outbounds, route 等配置保持不变，参考 6.3.3)
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "Meta", // !!与 box.config 中的 tun_device 一致
      "address": [ "10.0.1.1/24", "fd00:10:0:1::1/64" ],
      "mtu": 1500,
      "stack": "system",
      "auto_route": false, // !!关键!! Surfing 脚本负责路由
      "strict_route": false, // auto_route 为 false 时，此项通常也为 false
      "sniff": true,
      "sniff_override_destination": false
      // TUN 入站的 DNS 设置通常可以省略，依赖全局 DNS
    }
  ]
  // ... (全局 dns, outbounds, route 等配置，确保它们能正确处理来自 TUN 的流量)
}
```

#### 6.3.2 场景 B: sing-box 主导路由 (高级)
   - 在 `box.config` 中**手动添加并设置** `SURFING_TUN_MODE_PREFERS_SINGBOX_ROUTING="true"`。
   - 在 sing-box `config.json` 的 TUN inbound 中设置 `"auto_route": true`。
   - **同时，必须在 sing-box 全局 `route` 配置中设置 `"auto_detect_interface": true` 或 `"default_interface"`** (指向实际物理网卡的出站) 以避免流量回环。

```json
{
  // ... (全局 log, dns, outbounds 等配置保持不变，参考 6.3.3)
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "Meta", // !!与 box.config 中的 tun_device 一致
      "address": [ "10.0.1.1/24", "fd00:10:0:1::1/64" ],
      "mtu": 1500,
      "stack": "system",
      "auto_route": true, // !!关键!! sing-box 负责主要路由
      "strict_route": true, // 配合 auto_route:true 时，建议开启
      "sniff": true,
      "sniff_override_destination": false
      // TUN 入站的 DNS 设置通常可以省略，依赖全局 DNS
    }
  ],
  "route": { // 全局路由规则
    "auto_detect_interface": true, // !!关键!! 防止流量回环 当 TUN auto_route:true
    // 或者 "default_interface": "wlan0", // 根据实际物理网卡名称
    "rules": [
      // ...您的路由规则...
      { "ip_cidr": ["192.168.0.0/16", "10.0.0.0/8"], "outbound": "direct" },
      { "network": "tcp,udp", "outbound": "proxy-out" } // "proxy-out" 是您的代理出站tag
    ],
    "final": "proxy-out"
  }
  // ... (其他全局 dns, outbounds 等配置)
}
```

#### 6.3.3 通用的全局 DNS 和 Outbounds 示例 (配合上述任一场景)
```json
{
  // ... (inbounds 配置如上)
  "log": {
    "level": "info",
    "output": "/data/adb/box_bll/sing-box/sing-box.log",
    "timestamp": true
  },
  "dns": {
    "servers": [
      { "tag": "remote-dns", "address": "8.8.8.8", "detour": "direct" },
      { "tag": "local-dns", "address": "223.5.5.5", "detour": "direct" }
    ],
    "rules": [ { "outbound": "any", "server": "remote-dns" } ],
    "strategy": "prefer_ipv4",
    "disable_cache": false
  },
  "outbounds": [
    { "tag": "proxy-out", "type": "your_proxy_protocol", "server": "...", "server_port": 0 /* ... */ },
    { "tag": "direct", "type": "direct" },
    { "tag": "block", "type": "block" }
  ]
  // route 部分已在场景 B 中单独列出，对于场景 A，也需要类似的 route 配置，
  // 但不需要 auto_detect_interface 或 default_interface (因为 TUN auto_route:false)
}
```

### 6.4 Surfing 脚本与 TUN 的交互 (根据新逻辑)

- 当 `box.service start` 被调用且 sing-box 核心启动成功（并创建了名为 `tun_device` 的接口）后，`tun_forward_enable` 函数会被执行。
- **如果 `SURFING_TUN_MODE_PREFERS_SINGBOX_ROUTING` 为 `"false"` (默认):**
    - `box.service` 会执行其传统的、较全面的路由和 iptables FORWARD 规则配置，包括调用 `tun_forward_ip_rules` 和 `sing_tun_ip_rules`。
    - 您应在 sing-box `config.json` 中设置 `auto_route: false`。
- **如果 `SURFING_TUN_MODE_PREFERS_SINGBOX_ROUTING` 为 `"true"`:**
    - `box.service` 会执行简化的路由配置：仅确保 IP 转发开启、`rp_filter` 配置正确，并调用 `tun_forward_iptables_rules` 来添加必要的 `iptables FORWARD` 规则。它不会添加详细的 `ip rule` 规则。
    - 您应在 sing-box `config.json` 中设置 `auto_route: true`，并配置好 `route.auto_detect_interface` 或 `route.default_interface`。

### 6.5 TUN 模式故障排查 (通用)
1. **确认 `SURFING_TUN_MODE_PREFERS_SINGBOX_ROUTING` 设置**: 检查您在 `box.config` 中（手动添加的）此变量的值是否符合您的预期场景。
2. **检查 `interface_name` 与 `tun_device`**: 确保 sing-box `config.json` 中的 `interface_name` 与 `box.config` 中的 `tun_device` 完全一致 (默认为 "Meta")。
3. **检查 sing-box 日志**: 查看 `/data/adb/box_bll/run/check.log`，`/data/adb/box_bll/run/error_sing-box.log`，以及您在 `config.json` 中配置的主日志文件。
4. **检查 TUN 接口状态**: 使用 `ifconfig` 或 `ip addr` 查看 "Meta" (或自定义名称) TUN 接口是否存在及 IP 配置。
5. **检查系统路由和规则**:
    - `ip rule list`
    - `ip route list table all` (查看所有路由表)
    - `iptables -L -v -n` 和 `iptables -t nat -L -v -n`
    - 根据您选择的模式，判断路由和 iptables 规则是否符合预期（是由脚本设置还是由 sing-box 设置）。
6. **DNS 解析**: 使用 `ping` 或 `nslookup` 测试。错误的 DNS 配置是常见问题。确保 sing-box 的 DNS 配置能正常工作。
7. **sing-box `auto_route: true` 时的特定检查**:
    - 确保 `route.auto_detect_interface: true` 或 `route.default_interface` 已在 sing-box `config.json` 中正确设置，以防流量回环。
    - 检查是否有其他程序（如 `box.tproxy` 的残留规则）干扰了 sing-box 的路由。

配置 TUN 模式相对复杂，建议仔细阅读 sing-box 官方文档，并从小处着手，逐步验证配置。
---
