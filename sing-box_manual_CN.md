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
### 6.7 高级：当 sing-box TUN 的 `auto_route` 为 `true` 时

本文档前面的部分（特别是6.4节）推荐在 sing-box 的 TUN 入站配置中设置 `"auto_route": false`，并依赖 Surfing 模块的脚本来管理系统级路由。这种方式通常能提供更好的一致性和可预测性。

然而，部分高级用户可能希望利用 sing-box 自身强大的 `auto_route` 功能（当设置为 `true` 时），例如为了使用 sing-box 最新的路由/规则特性，或者在某些特定网络环境下 sing-box 的路由管理更为有效。

如果您选择在 sing-box `config.json` 的 TUN 入站中设置 `"auto_route": true`，则**强烈建议您对 Surfing 模块的 `box_bll/scripts/box.service` 脚本进行相应的修改**，以避免与 sing-box 的路由管理发生冲突。

**1. 理解冲突的根源**

-   **sing-box `auto_route: true`**: sing-box 会主动配置系统路由表，例如将默认路由指向其创建的 TUN 接口，并可能添加其他规则以确保流量正确导入 TUN。它还会管理从 TUN 接口发出的流量如何路由到物理网络。
-   **Surfing `box.service` 脚本**: `tun_forward_enable` 函数及其调用的 `tun_forward_ip_rules` 和 `sing_tun_ip_rules` 等函数，也会尝试配置系统 `ip rule` 和 `iptables` 规则，以引导流量进出 TUN 接口。

当两者都尝试管理相同的路由资源时，可能导致：
-   路由规则重复、冲突，优先级混乱。
-   网络连接不稳定，部分应用无法上网，或出现意外的直连/代理行为。
-   DNS 解析行为异常。

**2. `box.service` 脚本的修改思路**

核心思路是：**让 sing-box 主导路由，脚本只做辅助工作。**

在 `box.service` 的 `tun_forward_enable` 函数中：

-   **应保留的脚本功能：**
    *   `tun_forward_disable()`: 在启动前清理所有旧规则仍然是好的做法。
    *   `/proc/sys/net/ipv4/ip_forward` 的开启：确保系统允许 IP 转发。
    *   `rp_filter` 的配置：这些是通用的系统网络参数。
    *   `probe_tun_device()`: 检查 TUN 接口是否由 sing-box 成功创建。
    *   `tun_forward_iptables_rules()`: 此函数主要配置 `iptables` 的 `FORWARD` 链规则（例如 `iptables -A FORWARD -i Meta -o <phy_if> -j ACCEPT`）。这些规则对于允许数据包在 TUN 接口和物理网络接口之间正确转发是必要的，尤其是在本机作为路由器（如热点）时。sing-box 的 `auto_route` 主要关注IP路由层面，`FORWARD` 链的防火墙规则通常需要单独配置。

-   **应移除或大幅简化的脚本功能：**
    *   **`tun_forward_ip_rules()` 函数内的所有 `ip rule add ...` 命令**:
        这些规则（例如 `ip rule add from 10.0.0.0/8 lookup <some_table>` 或 `ip rule add iif Meta lookup main`）旨在将特定流量导向 TUN。当 `auto_route: true` 时，sing-box 会自行处理这些。脚本的这些规则很可能与之冲突。**建议全部移除 `tun_forward_ip_rules()` 的调用或将其内容清空/注释掉。**
    *   **`sing_tun_ip_rules()` 函数内的所有 `ip rule add ...` 命令**:
        这些规则（例如 `ip rule add from all iif Meta lookup main`）旨在处理从 TUN 接口发出的流量。sing-box 的 `auto_route` 应该已经确保了这些流量能正确到达物理出口。**建议全部移除 `sing_tun_ip_rules()` 的调用或将其内容清空/注释掉。**

**3. 概念性的简化版 `tun_forward_enable` 函数示例 (当 `auto_route: true`)**

```sh
# 这是 box.service 中 tun_forward_enable 函数的一个概念性修改示例
# 仅适用于 sing-box TUN 入站设置了 "auto_route": true 的情况

tun_forward_enable_for_singbox_auto_route_true() {
  tun_forward_disable # 清理旧规则

  sleep 1
  echo 1 > /proc/sys/net/ipv4/ip_forward # 确保 IP 转发开启
  # 根据实际系统路径调整 rp_filter 配置
  [ -e /proc/sys/net/ipv4/conf/all/rp_filter ] && echo 2 > /proc/sys/net/ipv4/conf/all/rp_filter
  [ -e /proc/sys/net/ipv4/conf/default/rp_filter ] && echo 2 > /proc/sys/net/ipv4/conf/default/rp_filter
  # 对于 IPv6 也应考虑相应设置，如果 IPv6 TUN 启用

  if probe_tun_device; then # 检查 sing-box 是否已创建 TUN 设备
    # 只保留必要的 iptables FORWARD 规则
    tun_forward_iptables_rules "-I" # "-I" 表示插入规则到链首
    log Info "TUN forwarding support enabled. Routing primarily managed by sing-box (auto_route=true)."
    log Info "Surfing script only configured essential FORWARD rules."
  else
    log Error "TUN device ($tun_device) not found. Cannot enable TUN forwarding support."
    return 1
  fi
  return 0
}
```
**注意**: 上述脚本仅为示例，实际修改前请务必备份原脚本，并仔细理解每条命令的含义。您需要将 `tun_forward_enable` 的原始调用替换为此修改后的函数调用，或者直接修改原函数内容。

**4. 对 `box.tproxy` 脚本的考量**

在 `start.sh` 脚本中，通常会调用 `${scripts_dir}/box.tproxy enable`。此脚本用于根据 `box.config` 中的 `proxy_method` (如 TPROXY, REDIRECT) 设置 iptables 规则来拦截流量。

如果 sing-box TUN 的 `auto_route: true` 旨在通过路由全面接管流量，那么 `box.tproxy` 中的大部分（甚至全部）基于特定端口的 `DNAT`、`REDIRECT` 或 `TPROXY` 目标规则也应该被禁用或移除。否则，流量可能在到达 TUN 接口（由路由引导）之前就被这些 iptables 规则提前拦截和处理，导致行为混乱。

**5. 重要提示与风险**

-   **高级操作**: 修改这些底层脚本属于高级操作，需要您对 Linux 网络路由、iptables 以及 sing-box 的 `auto_route` 机制有深入的理解。
-   **备份**: 在进行任何修改前，务必备份 `/data/adb/box_bll/scripts/box.service` 和 `/data/adb/box_bll/scripts/box.tproxy` 文件。
-   **测试**: 修改后必须进行彻底测试，包括本机各种应用的联网、热点分享功能、DNS 解析是否符合预期、有无流量泄漏等。
-   **sing-box 配置**: 当 `auto_route: true` 时，务必在 sing-box 的全局 `route` 配置中正确设置 `auto_detect_interface: true` (或 `default_interface` / `default_mark`)，以避免 sing-box 自身出站流量被错误地路由回 TUN 接口导致循环。同时，DNS 配置也更为关键。
-   **模块更新**: Surfing 模块更新时，这些自定义修改可能会被覆盖，需要重新应用。

**结论**:
虽然让 sing-box 的 `auto_route: true` 接管路由可以发挥 sing-box 更全面的路由能力，但这要求用户对整个流量路径和两个系统的交互有清晰的认识，并愿意承担修改和调试脚本的风险。对于大多数用户，遵循文档先前推荐的 `auto_route: false` 并依赖 Surfing 脚本管理路由，可能是更稳妥的选择。
---
