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

## 6. 配置 TUN 模式

TUN 模式允许 sing-box 创建一个虚拟网络接口，并接管设备的全部（或部分）网络流量（包括 TCP 和 UDP），实现系统级的透明代理。

### 6.1 TUN 模式简介
- **功能**: 通过创建名为 `tun_device` (在 `box.config` 中定义，默认为 "Meta") 的虚拟网卡，sing-box 可以捕获所有流向该网卡的网络流量，并根据其内部配置进行处理和转发。
- **优势**: 相比 TProxy 或 Redirect 模式，TUN 模式通常能更全面地处理所有应用的各种协议流量，并且在路由控制上可以更为精细。对于需要代理 UDP 流量或希望全局代理的场景非常有用。
- **Surfing 模块支持**: Surfing 模块的脚本 (`box.service`) 包含用于配置系统路由和防火墙规则以配合 TUN 设备工作的函数 (`tun_forward_enable`, `tun_forward_disable`, `sing_tun_ip_rules`)。这些脚本旨在将系统流量导向由 sing-box 创建的 TUN 接口。

### 6.2 Surfing 模块中的 TUN 配置 (`box.config`)
确保 `box_bll/scripts/box.config` 中的以下参数配置正确：
- `tun_device="Meta"`: 这是 sing-box 创建的 TUN 接口名称，**必须与 sing-box `config.json` 中 TUN 入站设置的 `interface_name` 完全一致**。
- `proxy_method`: 虽然 Surfing 模块提供了 `TPROXY`, `REDIRECT`, `MIXED` 等选项，但在纯 sing-box TUN 模式下，主要依赖 sing-box 自身的 TUN 配置。Surfing 脚本中的 TUN 相关路由规则 (`tun_forward_enable`等) 会在服务启动时应用。

**注意**: `README_CN.md` 中曾提及 TUN 模式 "v7.4.3 弃用"。虽然相关脚本函数依然存在，用户在配置较新版本的 Surfing 模块时应留意此信息，并进行充分测试。

### 6.3 sing-box `config.json` 中的 TUN 入站配置
用户**必须自行配置** sing-box 的 `config.json` 文件以启用 TUN 入站。以下是一个 TUN 入站配置示例：

```json
{
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in", // 自定义标签
      "interface_name": "Meta", // !!关键!! 必须与 box.config 中的 tun_device 一致
      "address": [ // TUN 接口在 sing-box 内的 IP 地址
        "10.0.1.1/24",  // IPv4 地址，避免与局域网冲突
        "fd00:10:0:1::1/64" // IPv6 ULA 地址 (可选)
      ],
      "mtu": 1500, // 通常为 1500
      "stack": "system", // 在 Android/Linux 上通常使用 "system" 或 "mixed"
      "auto_route": false, // 推荐设置为 false，由 Surfing 脚本管理系统路由
                           // 如果设置为 true，需在全局 "route" 配置中设置 "auto_detect_interface": true 或 "default_interface" 避免回环
      "strict_route": true, // 配合 auto_route:true 时，建议开启以防泄漏 (若 auto_route:false，此项影响不大)
      "sniff": true, // 开启流量嗅探以识别协议，配合路由规则使用
      "sniff_override_destination": false, // 根据嗅探结果决定真实目标地址

      // TUN 入站的 DNS 设置 (可选，通常依赖全局 DNS 配置)
      // "dns": {
      //   "servers": [
      //     { "address": "1.1.1.1" }
      //   ]
      // }
    }
    // ... 其他入站配置 (如果有)
  ],

  "dns": { // 全局 DNS 配置 (TUN 模式下至关重要)
    "servers": [
      {
        "tag": "remote-dns",
        "address": "8.8.8.8", // 主要的上游 DNS
        "detour": "direct" // 通过直连出站解析
      },
      {
        "tag": "local-dns", // 可选：本地 DNS，用于特定域名
        "address": "223.5.5.5",
        "detour": "direct"
      }
      // 如果需要 DNS over HTTPS 或其他安全 DNS，请按 sing-box 文档配置
    ],
    "rules": [ // DNS 规则示例
      // { "domain_suffix": [".cn", "aliyuncs.com"], "server": "local-dns" },
      { "outbound": "any", "server": "remote-dns" } // 默认所有查询走 remote-dns
    ],
    "strategy": "prefer_ipv4", // 或 "ipv4_only", "ipv6_only"
    "disable_cache": false,
    // "fakeip": { // FakeIP 模式在 TUN 下可以简化规则，但需仔细配置
    //   "enabled": true,
    //   "inet4_range": "198.18.0.0/15"
    //   // "inet6_range": "fc00::/7" // 如果需要 IPv6 FakeIP
    // }
  },

  "outbounds": [
    {
      "tag": "proxy-out", // 代理出站的标签
      "type": "your_proxy_protocol", // 例如: shadowsocks, vmess, trojan 等
      // ... 具体的代理服务器配置
    },
    {
      "tag": "direct",
      "type": "direct"
    },
    {
      "tag": "block",
      "type": "block"
    }
  ],

  "route": { // 全局路由规则
    // "auto_detect_interface": true, // 若 TUN 入站中 auto_route:true，则必须开启此项或设置 default_interface
    "rules": [
      // 示例规则：阻止 QUIC (可选)
      // { "protocol": "quic", "outbound": "block" },
      // 示例规则：局域网地址直连
      { "ip_cidr": ["192.168.0.0/16", "10.0.0.0/8", "172.16.0.0/12"], "outbound": "direct" },
      { "domain_suffix": ["local", "lan"], "outbound": "direct"},
      // 示例规则：特定应用流量直连 (需要 sniffer 和 Android specific settings in TUN inbound)
      // { "package_name": ["com.android.browser"], "outbound": "direct" },
      // 默认规则：所有其他 TCP 和 UDP 流量都通过代理出站
      { "network": "tcp,udp", "outbound": "proxy-out" }
    ],
    "final": "proxy-out" // 最后的默认出站
  }
}
```

### 6.4 关键配置说明
- **`interface_name`**: **极其重要**。必须与 `box_bll/scripts/box.config` 中定义的 `tun_device`（默认为 "Meta"）完全一致。Surfing 模块的脚本会根据 `tun_device` 的名称来配置系统路由。
- **`address`**: TUN 接口在 sing-box 内部使用的 IP 地址和子网掩码。例如 `10.0.1.1/24`。这个 IP 地址不应与您的物理局域网IP冲突。它是系统路由表将流量导向 sing-box 的目标。
- **`stack`**: 推荐使用 `"system"`。
- **`auto_route` (TUN 入站内)**:
    - 当设置为 `true` 时，sing-box 会尝试自动配置系统路由表，将所有流量导向 TUN 接口。此时，您**必须**在全局 `route` 部分配置 `auto_detect_interface: true` 或 `default_interface` (指向您的实际物理网卡出站) 来防止 sing-box 自身的出站流量被重新导回 TUN 接口，造成循环。
    - 鉴于 Surfing 模块的 `box.service` 脚本本身包含了详细的 `ip rule` 和 `iptables` 配置逻辑 (`tun_forward_enable` 函数)，**建议将 sing-box TUN 入站中的 `auto_route` 设置为 `false`**。这样可以避免与 Surfing 脚本的路由配置发生冲突，由 Surfing 脚本全权负责系统级路由。sing-box 仅负责处理进入 "Meta" 接口的流量。
- **全局 DNS 配置 (`dns` 部分)**:
    - 在 TUN 模式下，所有（或大部分）DNS 请求会被导向 TUN 接口并由 sing-box 处理。因此，必须配置 sing-box 的全局 DNS 解析。
    - 您可以配置 sing-box 将 DNS 请求通过代理服务器转发，或直接请求公共 DNS 服务器。
    - FakeIP (`fakeip` 选项) 是一种高级 DNS 技术，可以将域名解析为特定范围内的假 IP 地址，可以简化透明代理的路由规则，但需要用户更深入的理解。
- **全局路由配置 (`route` 部分)**:
    - 定义了 sing-box 如何处理通过 TUN 接口接收到的流量。例如，哪些流量直连（如局域网流量），哪些通过代理出站。
    - 如果 TUN 入站中的 `auto_route` 设置为 `false`（推荐做法），则全局路由规则主要负责将 TUN 接收到的流量正确导向出站（如 `proxy-out` 或 `direct`）。

### 6.5 Surfing 脚本与 TUN 的交互
- 当 `box.service start` 被调用且 sing-box 核心启动成功（并创建了名为 `tun_device` 的接口）后，`tun_forward_enable` 函数会被执行。
- 此函数会：
    1. 清理旧的 TUN 相关路由和防火墙规则。
    2. 开启 IPv4 转发 (`/proc/sys/net/ipv4/ip_forward`)。
    3. 配置 `rp_filter`。
    4. 使用 `ip rule` 添加多条路由规则，这些规则通常用于将特定流量（如来自局域网的流量或发往特定子网的流量）导向一个特定的路由表，或直接通过 TUN 设备。
    5. 使用 `iptables` (和 `ip6tables`) 添加 `FORWARD` 规则，允许流量在物理接口和 TUN 接口之间转发。
- `tun_forward_disable` 函数则负责在服务停止时移除这些规则。

### 6.6 TUN 模式故障排查
1. **检查 `interface_name`**: 确保 `config.json` 中的 `interface_name` 与 `box.config` 中的 `tun_device` 完全一致。
2. **检查 sing-box 日志**:
    - 查看 `/data/adb/box_bll/run/check.log`（配置检查日志）。
    - 查看 `/data/adb/box_bll/run/error_sing-box.log`（运行时错误）。
    - 查看您在 `config.json` 中为 sing-box 配置的主日志文件，获取详细的 TUN 接口创建和流量处理信息。
3. **检查 TUN 接口状态**:
    - sing-box 启动后，使用 `ifconfig` 或 `ip addr` 命令查看名为 `Meta` (或您自定义的 `tun_device` 名称) 的网络接口是否存在，以及其 IP 地址是否与 `config.json` 中配置的一致。
4. **检查系统路由规则**:
    - 使用 `ip rule list` 查看路由策略。
    - 使用 `ip route list table <table_index>` (如果 Surfing 脚本使用了特定路由表) 查看具体路由表。
    - 重点关注是否有规则将流量导向 `Meta` 设备。
5. **检查 iptables 规则**:
    - 使用 `iptables -L -v -n` 和 `iptables -t nat -L -v -n` 查看是否有相关的 `FORWARD` 或 `NAT` 规则。
6. **DNS 解析**: 确认 DNS 是否能正常解析。可以使用 `ping` 或 `nslookup` 测试。错误的 DNS 配置是 TUN 模式下网络不通的常见原因。
7. **`auto_route` 设置**: 如果遇到路由问题或与 Surfing 脚本冲突，尝试将 sing-box TUN 入站中的 `auto_route` 设置为 `false`，并依赖 Surfing 脚本进行路由管理。

配置 TUN 模式相对复杂，需要对网络和 sing-box 配置有较深入的理解。建议仔细阅读 sing-box 官方文档，并从小处着手，逐步验证配置。
---
