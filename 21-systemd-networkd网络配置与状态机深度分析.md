# 21. systemd-networkd 网络配置与状态机深度分析

## 概述

systemd-networkd 是 systemd 的网络管理守护进程，负责管理物理和虚拟网络接口的
配置。它通过声明式配置文件（`.network`、`.netdev`、`.link`）描述网络拓扑，
使用 netlink 与内核交互，支持 DHCP、IPv6 RA、LLDP 等动态协议。

| 配置文件 | 处理阶段 | 功能 |
|----------|----------|------|
| `.link` | udev 阶段 | 设备命名、MAC 地址、ethtool、SR-IOV |
| `.netdev` | networkd 启动 | 创建虚拟网络设备（bridge/bond/vlan/...） |
| `.network` | networkd 运行时 | IP 地址、路由、DHCP、DNS、防火墙 |

---

## 一、Manager 架构

### 1.1 Manager 结构体（networkd-manager.h:20-104）

```c
struct Manager {
    // === 通信通道 ===
    sd_netlink *rtnl;              // rtnetlink 连接（内核网络配置）
    sd_netlink *genl;              // Generic Netlink（延迟初始化）
    sd_event *event;               // sd-event 事件循环
    sd_resolve *resolve;           // 异步 DNS 解析
    sd_bus *bus;                   // D-Bus 连接
    sd_device_monitor *device_monitor; // udev 设备监视
    Hashmap *polkit_registry;      // polkit 授权缓存
    int ethtool_fd;                // ethtool ioctl 套接字

    // === 全局配置 ===
    KeepConfiguration keep_configuration; // 保留既有配置模式
    bool test_mode;                // 测试模式（不实际应用）
    bool enumerating;              // 正在枚举接口
    bool manage_foreign_routes;    // 管理非 networkd 创建的路由
    bool manage_foreign_rules;     // 管理非 networkd 创建的策略规则

    // === 全局状态 ===
    Set *dirty_links;              // 待保存状态的接口
    char *state_file;              // 管理器状态文件
    LinkOperationalState operational_state;  // 全局运行状态
    LinkCarrierState carrier_state;          // 全局载波状态
    LinkAddressState address_state;          // 全局地址状态
    LinkOnlineState online_state;            // 全局在线状态

    // === 对象注册表 ===
    Hashmap *links_by_index;       // ifindex → Link*
    Hashmap *links_by_name;        // ifname → Link*
    Hashmap *links_by_hw_addr;     // MAC → Link*
    Hashmap *links_by_dhcp_pd_subnet_prefix; // DHCP-PD 前缀 → Link*
    Hashmap *netdevs;              // name → NetDev*
    OrderedHashmap *networks;      // name → Network*（按优先级排序）
    OrderedSet *address_pools;     // 地址池

    // === DHCP 配置 ===
    DUID dhcp_duid;                // DHCPv4 DUID
    DUID dhcp6_duid;               // DHCPv6 DUID

    // === 动态配置 ===
    char *dynamic_hostname;        // DHCP 获取的主机名
    char *dynamic_timezone;        // DHCP 获取的时区

    // === 全局路由/策略 ===
    Set *rules;                    // 策略路由规则
    Hashmap *nexthops_by_id;       // nexthop ID → NextHop*
    Set *nexthops;                 // 无 OIF 的 nexthop 集合
    Set *routes;                   // 无 OIF 的路由集合
    Set *routes_foreign;           // 外部路由集合

    // === 路由表名映射 ===
    Hashmap *route_table_numbers_by_name;
    Hashmap *route_table_names_by_number;

    // === 无线设备 ===
    Hashmap *wiphy_by_index;       // wiphy 索引 → Wiphy*
    Hashmap *wiphy_by_name;        // wiphy 名称 → Wiphy*

    // === 速度计量 ===
    bool use_speed_meter;
    sd_event_source *speed_meter_event_source;
    usec_t speed_meter_interval_usec;

    // === 请求队列 ===
    OrderedSet *request_queue;     // 异步配置请求队列
    FirewallContext *fw_ctx;       // 防火墙上下文
};
```

### 1.2 启动流程

```
main()（networkd.c:23-118）
 ├── 解析命令行参数
 ├── manager_new() → 分配 Manager
 ├── manager_setup()：
 │    ├── sd_event_default() → 创建事件循环
 │    ├── 注册 SIGTERM/SIGINT/SIGUSR2 信号处理
 │    ├── manager_connect_rtnl()：
 │    │    ├── sd_netlink_open() → 打开 rtnetlink
 │    │    ├── 注册 7 类 netlink 消息回调：
 │    │    │    ├── RTM_NEWLINK/DELLINK → manager_rtnl_process_link
 │    │    │    ├── RTM_NEWADDR/DELADDR → manager_rtnl_process_address
 │    │    │    ├── RTM_NEWROUTE/DELROUTE → manager_rtnl_process_route
 │    │    │    ├── RTM_NEWRULE/DELRULE → manager_rtnl_process_rule
 │    │    │    ├── RTM_NEWNEXTHOP/DELNEXTHOP → manager_rtnl_process_nexthop
 │    │    │    ├── RTM_NEWQDISC/DELQDISC → manager_process_tc
 │    │    │    └── RTM_NEWTCLASS/DELTCLASS → manager_process_tc
 │    │    └── sd_netlink_attach_event()
 │    ├── manager_connect_bus() → D-Bus 连接
 │    ├── udev 设备监视器注册
 │    └── resolve 连接
 ├── 配置加载：
 │    ├── netdev_load() → 加载所有 .netdev 文件
 │    ├── network_load() → 加载所有 .network 文件
 │    └── network_verify() → 校验配置
 ├── manager_enumerate()：
 │    ├── manager_enumerate_links() → RTM_GETLINK dump
 │    ├── manager_enumerate_addresses() → RTM_GETADDR dump
 │    ├── manager_enumerate_routes() → RTM_GETROUTE dump
 │    ├── manager_enumerate_rules() → RTM_GETRULE dump
 │    └── manager_enumerate_nexthops() → RTM_GETNEXTHOP dump
 ├── manager_start()
 │    ├── 启动速度计量器
 │    └── 保存管理器状态
 └── sd_event_loop() → 进入主循环
```

### 1.3 Netlink 事件驱动

networkd 的核心是**纯事件驱动**的 rtnetlink 处理：

```
内核网络子系统 ──netlink──→ sd_netlink
                              │
    ┌─────────────────────────┼─────────────────────────┐
    │                         │                         │
    ▼                         ▼                         ▼
RTM_NEWLINK               RTM_NEWADDR             RTM_NEWROUTE
RTM_DELLINK               RTM_DELADDR             RTM_DELROUTE
    │                         │                         │
    ▼                         ▼                         ▼
manager_rtnl_         manager_rtnl_           manager_rtnl_
process_link()        process_address()       process_route()
    │
    ├── NEWLINK + 新接口 → link_new() → link_update()
    ├── NEWLINK + 已知接口 → link_update()
    └── DELLINK → link_drop() → LINGER
```

---

## 二、Link 状态机

### 2.1 Link 结构体（networkd-link.h:47-194）

```c
typedef struct Link {
    Manager *manager;
    unsigned n_ref;

    // === 接口标识 ===
    int ifindex;                    // 接口索引
    int master_ifindex;             // 主接口索引（bond slave 等）
    char *ifname;                   // 接口名
    char **alternative_names;       // 备选名
    char *kind;                     // netdev 类型（vlan/bridge/...）
    unsigned short iftype;          // ARPHRD_* 类型
    char *state_file;               // 状态文件路径

    // === 地址信息 ===
    struct hw_addr_data hw_addr;         // 当前 MAC
    struct hw_addr_data permanent_hw_addr;// 出厂 MAC
    struct in6_addr ipv6ll_address;      // IPv6 链路本地地址
    uint32_t mtu;                        // 当前 MTU
    uint32_t original_mtu;               // 原始 MTU

    sd_device *sd_device;           // udev 设备对象
    char *driver;                   // 驱动名

    // === 无线网络 ===
    enum nl80211_iftype wlan_iftype; // 无线接口类型
    char *ssid;                      // 当前 SSID
    struct ether_addr bssid;         // 当前 BSSID

    unsigned flags;                 // IFF_* 标志
    uint8_t kernel_operstate;       // 内核 operstate

    // === 配置绑定 ===
    Network *network;               // 匹配到的 .network 配置
    NetDev *netdev;                 // 关联的 NetDev

    // === 状态 ===
    LinkState state;                // 主状态机
    LinkOperationalState operstate; // 运行状态
    LinkCarrierState carrier_state; // 载波状态
    LinkAddressState address_state; // 地址状态
    LinkOnlineState online_state;   // 在线状态

    // === 静态配置消息计数 ===
    unsigned static_address_messages;     // 待确认的地址配置
    unsigned static_route_messages;       // 待确认的路由配置
    unsigned static_nexthop_messages;     // 待确认的 nexthop 配置
    unsigned static_neighbor_messages;    // 待确认的邻居配置
    // ... 更多计数器 ...

    // === 配置完成标志 ===
    bool static_addresses_configured:1;
    bool static_routes_configured:1;
    bool static_nexthops_configured:1;
    bool static_neighbors_configured:1;
    bool tc_configured:1;                 // 流量控制
    bool sr_iov_configured:1;             // SR-IOV
    bool activated:1;                     // 已激活
    bool stacked_netdevs_created:1;       // 堆叠 netdev 已创建

    // === 网络资源集合 ===
    Set *addresses;                 // 已配置地址
    Set *neighbors;                 // 已配置邻居
    Set *routes;                    // 已配置路由
    Set *nexthops;                  // 已配置 nexthop
    Set *qdiscs;                    // 已配置排队规则
    Set *tclasses;                  // 已配置流量类

    // === DHCP ===
    sd_dhcp_client *dhcp_client;    // DHCPv4 客户端
    sd_dhcp_lease *dhcp_lease;      // DHCPv4 租约
    bool dhcp4_configured:1;        // DHCPv4 配置完成

    sd_ipv4ll *ipv4ll;              // IPv4 链路本地
    bool ipv4ll_address_configured:1;

    sd_dhcp6_client *dhcp6_client;  // DHCPv6 客户端
    sd_dhcp6_lease *dhcp6_lease;    // DHCPv6 租约
    bool dhcp6_configured;          // DHCPv6 配置完成

    // === IPv6 邻居发现 ===
    sd_ndisc *ndisc;                // NDisc 客户端
    bool ndisc_configured:1;        // NDisc 配置完成
    sd_radv *radv;                  // 路由通告守护进程

    // === LLDP ===
    sd_lldp_rx *lldp_rx;            // LLDP 接收
    sd_lldp_tx *lldp_tx;            // LLDP 发送

    // === 绑定关系 ===
    Hashmap *bound_by_links;        // 被哪些接口绑定
    Hashmap *bound_to_links;        // 绑定到哪些接口
    Set *slaves;                    // 从属接口集合

    // === 速度计量 ===
    struct rtnl_link_stats64 stats_old, stats_new;

    // === DNS 配置（D-Bus 动态设置）===
    struct in_addr_full **dns;
    OrderedSet *search_domains;
    ResolveSupport llmnr, mdns;
    DnssecMode dnssec_mode;
    DnsOverTlsMode dns_over_tls_mode;

    char **ntp;                     // NTP 服务器
} Link;
```

### 2.2 状态定义（networkd-link.h:30-40）

```c
typedef enum LinkState {
    LINK_STATE_PENDING,      // udev 尚未初始化接口
    LINK_STATE_INITIALIZED,  // udev 已初始化，等待配置
    LINK_STATE_CONFIGURING,  // 正在配置（地址/路由/DHCP...）
    LINK_STATE_CONFIGURED,   // 所有配置已完成
    LINK_STATE_UNMANAGED,    // Unmanaged=yes 或无匹配 .network
    LINK_STATE_FAILED,       // 配置失败
    LINK_STATE_LINGER,       // 接口已删除（RTM_DELLINK）
} LinkState;
```

### 2.3 状态转换图

```
                    RTM_NEWLINK（内核上报新接口）
                           │
                           ▼
                    ┌──────────────┐
                    │   PENDING    │ ← 等待 udev 初始化
                    └──────┬───────┘
                           │ udev 完成初始化 / 设备就绪
                           ▼
                    ┌──────────────┐
              ┌────→│ INITIALIZED  │ ← 匹配 .network 文件
              │     └──────┬───────┘
              │            │ 找到匹配的 .network
              │            │ 开始配置
              │            ▼
              │     ┌──────────────┐
              │     │ CONFIGURING  │ ← 异步配置中
              │     └──────┬───────┘
              │            │ link_check_ready() 所有条件满足
              │            ▼
              │     ┌──────────────┐
 重新配置 ────┘     │ CONFIGURED   │ ← 配置完成
                    └──────────────┘

 无匹配 .network ──→ ┌──────────────┐
                     │  UNMANAGED   │
                     └──────────────┘

 配置错误 ─────────→ ┌──────────────┐
                     │   FAILED     │
                     └──────────────┘

 RTM_DELLINK ──────→ ┌──────────────┐
                     │   LINGER     │ ← 接口从内核移除
                     └──────────────┘
```

### 2.4 CONFIGURING → CONFIGURED 判定（link_check_ready）

`link_check_ready()`（networkd-link.c:393-511）定义了**完整的就绪条件链**：

```
link_check_ready(link)
 ├── 必须处于 CONFIGURING 状态
 ├── 必须有匹配的 .network 配置
 │
 ├── 基础配置完成检查：
 │    ├── tc_configured        → 流量控制已配置
 │    ├── set_link_messages==0 → 链路层配置已确认
 │    ├── activated             → 接口已激活
 │    ├── stacked_netdevs_created → 堆叠 netdev 已创建
 │    └── sr_iov_configured     → SR-IOV 已配置
 │
 ├── 静态配置完成检查：
 │    ├── static_addresses_configured
 │    ├── 所有地址 address_is_ready() → 内核已确认
 │    ├── static_address_labels_configured
 │    ├── static_bridge_fdb_configured
 │    ├── static_bridge_mdb_configured
 │    ├── static_ipv6_proxy_ndp_configured
 │    ├── static_neighbors_configured
 │    ├── static_nexthops_configured
 │    ├── static_routes_configured
 │    └── static_routing_policy_rules_configured
 │
 ├── IPv6 链路本地检查：
 │    └── 如果启用且需要载波 → ipv6ll_address 已分配
 │
 ├── 动态地址检查：
 │    └── 如果启用 DHCP/IPv4LL → 至少有一个动态地址
 │
 ├── 动态协议完成检查（至少一个完成）：
 │    ├── ipv4ll_address_configured
 │    ├── dhcp4_configured
 │    ├── dhcp6_configured
 │    ├── dhcp_pd_configured
 │    └── ndisc_configured
 │
 └── 全部满足 → link_set_state(CONFIGURED) ✓
```

### 2.5 配置应用流程

```
link_configure()（networkd-link.c:1024-1149）
 ├── link_set_state(CONFIGURING)
 │
 ├── 链路层配置：
 │    ├── link_request_to_set_mac() → MAC 地址
 │    ├── link_request_to_set_mtu() → MTU
 │    ├── link_request_to_set_addrgen_mode() → IPv6 地址生成模式
 │    └── link_request_to_activate() → 激活（UP）
 │
 ├── 静态配置请求：
 │    ├── link_request_static_addresses() → 静态地址
 │    ├── link_request_static_address_labels() → 地址标签
 │    ├── link_request_static_bridge_fdb() → 桥接 FDB
 │    ├── link_request_static_bridge_mdb() → 桥接 MDB
 │    ├── link_request_static_ipv6_proxy_ndp() → IPv6 代理 NDP
 │    ├── link_request_static_neighbors() → 邻居
 │    ├── link_request_static_nexthops() → Nexthop
 │    ├── link_request_static_routes() → 路由
 │    └── link_request_static_routing_policy_rules() → 策略规则
 │
 ├── 流量控制：
 │    ├── link_request_traffic_control() → QDisc + TClass
 │    └── link_request_sr_iov() → SR-IOV VF 配置
 │
 ├── 堆叠 NetDev：
 │    └── link_request_stacked_netdevs() → VLAN/MACVLAN/... 创建
 │
 └── 动态协议启动：
      ├── link_request_dhcp4_client() → DHCPv4
      ├── link_request_dhcp6_client() → DHCPv6
      ├── link_request_ndisc() → IPv6 邻居发现
      ├── link_request_radv() → 路由通告
      ├── link_request_ipv4ll_address() → IPv4 链路本地
      ├── link_request_lldp_rx() → LLDP 接收
      └── link_request_lldp_tx() → LLDP 发送
```

---

## 三、Network 配置模型

### 3.1 配置文件搜索路径

```
/etc/systemd/network/           ← 最高优先级（管理员配置）
/run/systemd/network/           ← 运行时覆盖
/usr/lib/systemd/network/       ← 最低优先级（发行版默认）
```

文件按字典序排序，**第一个匹配的 .network 文件胜出**。

### 3.2 匹配条件（[Match] 段）

```ini
[Match]
Name=eth*                  # 接口名（支持通配符）
MACAddress=00:11:22:*      # MAC 地址
PermanentMACAddress=...    # 永久 MAC
Driver=e1000e              # 驱动名
Type=ether                 # ARPHRD_* 类型
Kind=vlan                  # netdev 类型
Path=pci-0000:00:1f.6      # udev 设备路径
WLANInterfaceType=station  # 无线接口类型
SSID=MyNetwork             # 无线网络名
BSSID=...                  # 无线 AP MAC
Host=myhostname            # 主机名条件
Virtualization=yes         # 虚拟化环境条件
Architecture=x86-64        # CPU 架构条件
```

### 3.3 配置段概览

| 段名 | 功能 |
|------|------|
| `[Match]` | 匹配条件 |
| `[Link]` | 链路层参数（MTU、MAC、多播等） |
| `[Network]` | 网络层核心配置 |
| `[Address]` | IPv4/IPv6 地址（可多个） |
| `[Route]` | 路由表项（可多个） |
| `[Neighbor]` | ARP/NDP 静态邻居 |
| `[NextHop]` | Nexthop 对象 |
| `[RoutingPolicyRule]` | 策略路由规则 |
| `[DHCPv4]` | DHCPv4 客户端配置 |
| `[DHCPv6]` | DHCPv6 客户端配置 |
| `[DHCPServer]` | DHCPv4 服务器配置 |
| `[IPv6AcceptRA]` | IPv6 路由通告接受 |
| `[IPv6SendRA]` | IPv6 路由通告发送 |
| `[Bridge]` | 桥接端口参数 |
| `[BridgeFDB]` | 桥接 FDB 表项 |
| `[BridgeMDB]` | 桥接 MDB 表项 |
| `[BridgeVLAN]` | 桥接 VLAN 过滤 |
| `[SR-IOV]` | SR-IOV VF 配置 |

### 3.4 [Network] 段关键指令

```ini
[Network]
DHCP=yes|no|ipv4|ipv6        # DHCP 模式
Address=192.168.1.100/24     # 内联地址（简写）
Gateway=192.168.1.1          # 内联网关
DNS=8.8.8.8                  # DNS 服务器
Domains=example.com          # 搜索域
IPv6AcceptRA=yes             # 接受 IPv6 路由通告
LinkLocalAddressing=yes      # 链路本地寻址
LLDP=yes                     # LLDP 接收
EmitLLDP=yes                 # LLDP 发送
Bridge=br0                   # 加入桥接
Bond=bond0                   # 加入 Bond
VLAN=vlan100                 # 创建 VLAN
IPForward=yes                # IP 转发
IPMasquerade=yes             # IP 伪装
ConfigureWithoutCarrier=yes  # 无载波时也配置
```

### 3.5 .network 文件匹配流程

```
link_initialized() / link_reconfigure()
 └── network_get()
      └── 遍历 manager->networks（按文件名排序）：
           对每个 Network：
             net_match_config(
               match_name,       // Name=
               match_mac,        // MACAddress=
               match_permanent_mac, // PermanentMACAddress=
               match_driver,     // Driver=
               match_iftype,     // Type=
               match_kind,       // Kind=
               match_wlan_iftype,// WLANInterfaceType=
               match_ssid,       // SSID=
               match_bssid,      // BSSID=
               device,           // udev 设备（Path= 等）
               ...
             )
             → 第一个匹配成功的 Network 被选中
```

---

## 四、NetDev 虚拟设备子系统

### 4.1 NetDev 状态机

```c
typedef enum NetDevState {
    NETDEV_STATE_LOADING,   // 配置已加载，等待创建
    NETDEV_STATE_CREATING,  // 正在通过 netlink 创建
    NETDEV_STATE_READY,     // 创建完成，ifindex 已分配
    NETDEV_STATE_FAILED,    // 创建失败
    NETDEV_STATE_LINGER,    // 设备已删除
} NetDevState;
```

```
加载 .netdev 文件
      │
      ▼
┌──────────┐    RTM_NEWLINK 请求    ┌──────────┐    内核确认     ┌─────────┐
│ LOADING  │ ──────────────────────→│ CREATING │ ──────────────→│  READY  │
└──────────┘                        └──────────┘                └─────────┘
      │                                  │
      │ 配置错误                         │ 创建失败
      ▼                                  ▼
┌──────────┐                        ┌──────────┐
│  FAILED  │                        │  FAILED  │
└──────────┘                        └──────────┘

RTM_DELLINK → LINGER
```

### 4.2 创建类型

| 类型 | 说明 | 示例 |
|------|------|------|
| `INDEPENDENT` | 独立创建，无需物理接口 | bridge, bond, dummy, veth, wireguard |
| `STACKED` | 堆叠在物理接口上 | vlan, macvlan, ipvlan, vxlan |

### 4.3 支持的 37 种虚拟设备

| 类别 | 设备类型 | 源文件 |
|------|----------|--------|
| **二层桥接** | bridge | `bridge.c` |
| **链路聚合** | bond | `bond.c` |
| **VLAN** | vlan | `vlan.c` |
| **MAC 虚拟化** | macvlan, macvtap | `macvlan.c` |
| **IP 虚拟化** | ipvlan, ipvtap | `ipvlan.c` |
| **隧道** | ipip, sit, gre, gretap, ip6tnl, ip6gre, ip6gretap, erspan, vti, vti6 | `tunnel.c` |
| **Overlay** | vxlan, geneve | `vxlan.c`, `geneve.c` |
| **FOU** | fou | `fou-tunnel.c` |
| **VPN** | wireguard | `wireguard.c` |
| **L2TP** | l2tp | `l2tp-tunnel.c` |
| **MACsec** | macsec | `macsec.c` |
| **虚拟对** | veth, vxcan | `veth.c`, `vxcan.c` |
| **TUN/TAP** | tun, tap | `tuntap.c` |
| **VRF** | vrf | `vrf.c` |
| **虚拟 CAN** | vcan | `vcan.c` |
| **BareUDP** | bareudp | `bareudp.c` |
| **Batman** | batadv | `batadv.c` |
| **IPoIB** | ipoib | `ipoib.c` |
| **仿真** | dummy, ifb, netdevsim, nlmon | 各自 `.c` |
| **无线** | wlan | `wlan.c` |
| **IPsec** | xfrm | `xfrm.c` |

### 4.4 创建流程

```
netdev_create_message()（netdev.c:485-543）
 ├── sd_rtnl_message_new_link(RTM_NEWLINK)
 ├── IFLA_IFNAME → 接口名
 ├── IFLA_HWADDR → MAC 地址（如配置）
 ├── IFLA_MTU → MTU（如配置）
 ├── IFLA_LINKINFO：
 │    ├── IFLA_INFO_KIND → "bridge" / "bond" / "vlan" / ...
 │    └── IFLA_INFO_DATA → 类型特定参数
 │         └── fill_message_create()（vtable 虚函数）
 └── 异步发送到内核

independent_netdev_create()（netdev.c:545-605）：
 ├── netdev_create_message()
 ├── sd_netlink_call_async() → 异步等待响应
 └── 状态 → CREATING

stacked_netdev_create()：
 ├── 同上，但额外设置 IFLA_LINK → 父接口 ifindex
 └── 由 link_configure() 触发
```

---

## 五、DHCP 集成

### 5.1 DHCPv4 流程

```
dhcp4_configure()（networkd-dhcp4.c:1375-1495）
 ├── sd_dhcp_client_new() → 创建客户端
 ├── 设置参数：
 │    ├── ifindex, MAC, hostname, vendor class
 │    ├── DUID / client ID
 │    ├── 请求选项：DNS, NTP, SIP, MTU, 域名, 路由 ...
 │    └── 回调：dhcp4_handler
 └── sd_dhcp_client_attach_event()

dhcp4_start()：
 └── sd_dhcp_client_start() → 开始 DISCOVER

dhcp4_handler()（networkd-dhcp4.c:1110-1224）：
 ├── DHCP_EVENT_IP_ACQUIRE / RENEW / REBIND：
 │    └── dhcp4_lease_acquired()
 │         ├── sd_dhcp_lease_get_address() → IP 地址
 │         ├── sd_dhcp_lease_get_router() → 网关
 │         ├── sd_dhcp_lease_get_dns() → DNS
 │         ├── sd_dhcp_lease_get_mtu() → MTU
 │         ├── link_request_address() → 请求配置地址
 │         ├── dhcp4_request_routes() → 请求配置路由
 │         └── dhcp4_configured = true → link_check_ready()
 ├── DHCP_EVENT_EXPIRED / STOP：
 │    └── dhcp4_lease_lost()
 │         ├── 删除地址和路由
 │         └── dhcp4_configured = false
 └── DHCP_EVENT_TRANSIENT_FAILURE：
      └── 记录日志，继续重试
```

### 5.2 DHCPv6 流程

```
dhcp6_start()（networkd-dhcp6.c:412-513）
 ├── sd_dhcp6_client_new() → 创建客户端
 ├── 设置参数：
 │    ├── ifindex, DUID, IAID
 │    ├── 请求选项：DNS, 域名, NTP, SNTP
 │    └── 前缀委派参数
 └── sd_dhcp6_client_start()

dhcp6_handler()：
 ├── DHCP6_EVENT_IP_ACQUIRE：
 │    └── dhcp6_lease_ip_acquired()
 │         ├── 应用地址和路由
 │         ├── 处理前缀委派（PD）
 │         └── dhcp6_configured = true → link_check_ready()
 └── DHCP6_EVENT_RETRANS_MAX / STOP：
      └── dhcp6_lease_lost()
```

### 5.3 IPv6 邻居发现（NDisc）

```
ndisc_handler()：
 ├── 接收 Router Advertisement（RA）
 ├── 处理前缀信息 → 生成 SLAAC 地址
 ├── 处理路由信息 → 添加路由
 ├── 处理 RDNSS/DNSSL → DNS 配置
 ├── 处理 MTU
 └── ndisc_configured = true → link_check_ready()
```

---

## 六、.link 文件（udev 阶段）

### 6.1 与 .network 文件的区别

| 特性 | .link | .network |
|------|-------|----------|
| 处理阶段 | udev 规则执行期间 | networkd 运行时 |
| 处理程序 | `udev-builtin-net_setup_link` | `systemd-networkd` |
| 功能 | 命名、MAC、ethtool、SR-IOV | IP 配置、路由、DHCP |
| 时机 | 设备出现时（一次性） | 接口存在期间（持续） |

### 6.2 .link 配置段

```ini
[Match]
MACAddress=...              # 匹配条件（同 .network）
Driver=...
Path=...
Type=...

[Link]
Name=eth0                   # 重命名接口
MACAddressPolicy=persistent # MAC 策略
MTUBytes=9000               # MTU
WakeOnLan=magic             # 网络唤醒
ReceiveChecksumOffload=yes  # 校验和卸载
TransmitChecksumOffload=yes
GenericSegmentationOffload=yes
TCPSegmentationOffload=yes
GenericReceiveOffload=yes
LargeReceiveOffload=yes
```

### 6.3 处理流程

```
udev 规则：SUBSYSTEM=="net", IMPORT{builtin}="net_setup_link"

net_setup_link builtin：
 ├── link_config_load() → 加载所有 .link 文件
 ├── link_get_config() → 匹配设备到 LinkConfig
 └── link_apply_config()：
      ├── ethtool 设置（速率、双工、卸载功能...）
      ├── rtnl 设置（MAC、MTU）
      ├── 接口重命名
      ├── 备选名设置
      └── SR-IOV VF 配置
```

---

## 七、异步请求队列

### 7.1 设计理念

networkd 的所有配置操作都是**异步**的：

```
配置请求入队 → request_queue
     │
     ▼
sd_event POST 回调
     │
     ▼
处理请求 → 发送 netlink 消息
     │
     ▼
内核确认 → 消息计数器减 1
     │
     ▼
计数器归零 → *_configured = true
     │
     ▼
link_check_ready() → 检查所有条件
```

### 7.2 消息计数器模式

每类配置使用消息计数器追踪异步完成：

```
// 发送请求
link->static_address_messages++;
sd_netlink_call_async(rtnl, RTM_NEWADDR, ..., callback);

// 收到确认
static int address_configure_handler(...) {
    link->static_address_messages--;
    if (link->static_address_messages == 0)
        link->static_addresses_configured = true;
    link_check_ready(link);
}
```

---

## 八、完整数据流

```
                       ┌────────────────────────────┐
                       │    .netdev 配置文件         │
                       │  Bridge/Bond/VLAN/...      │
                       └─────────────┬──────────────┘
                                     │ netdev_load()
                                     ▼
                       ┌────────────────────────────┐
                       │   NetDev 对象创建           │
                       │   RTM_NEWLINK → 内核       │
                       └─────────────┬──────────────┘
                                     │
    ┌──────────────────┐             │
    │  .link 配置文件   │             │
    │  命名/MAC/ethtool│             │
    └────────┬─────────┘             │
             │ udev 阶段             │
             ▼                       ▼
    ┌──────────────────────────────────────────┐
    │          内核网络接口                      │
    │     RTM_NEWLINK 通知 networkd            │
    └────────────────────┬─────────────────────┘
                         │
                         ▼
    ┌──────────────────────────────────────────┐
    │  Link 对象创建 (PENDING)                  │
    │  等待 udev 初始化                         │
    └────────────────────┬─────────────────────┘
                         │ udev 就绪
                         ▼
    ┌──────────────────────────────────────────┐
    │  INITIALIZED → 匹配 .network 文件         │
    └────────────────────┬─────────────────────┘
                         │ 找到匹配
                         ▼
    ┌──────────────────────────────────────────┐
    │  CONFIGURING                              │
    │  ┌──────────────────────────────────┐    │
    │  │ 静态配置：地址/路由/邻居/nexthop  │    │
    │  │ 链路配置：MTU/MAC/激活           │    │
    │  │ 堆叠 NetDev：VLAN/MACVLAN       │    │
    │  │ 流量控制：QDisc/TClass           │    │
    │  │ SR-IOV：VF 配置                  │    │
    │  ├──────────────────────────────────┤    │
    │  │ 动态协议：                        │    │
    │  │  DHCPv4 → 地址+路由+DNS          │    │
    │  │  DHCPv6 → 地址+前缀委派          │    │
    │  │  NDisc  → SLAAC+路由+DNS         │    │
    │  │  IPv4LL → 链路本地地址            │    │
    │  │  LLDP   → 邻居发现               │    │
    │  └──────────────────────────────────┘    │
    └────────────────────┬─────────────────────┘
                         │ link_check_ready() ✓
                         ▼
    ┌──────────────────────────────────────────┐
    │  CONFIGURED                               │
    │  → 更新管理器全局状态                      │
    │  → 通知 systemd-networkd-wait-online      │
    │  → D-Bus 属性变更通知                      │
    └──────────────────────────────────────────┘
```

---

## 九、设计要点总结

### 9.1 声明式配置

- .network / .netdev / .link 三层配置分离
- 文件名排序决定优先级
- 第一个匹配的文件胜出（停止搜索）
- 配置变更 → SIGUSR2 → 重新加载并重新匹配

### 9.2 纯异步架构

- 所有网络操作通过 rtnetlink 异步执行
- 消息计数器追踪待确认操作
- `link_check_ready()` 聚合所有完成条件
- 单线程事件循环，无锁设计

### 9.3 状态收敛

- 目标状态由 .network 文件声明
- networkd 持续驱动当前状态收敛到目标状态
- 载波丢失/恢复 → 自动重新配置
- DHCP 租约过期 → 自动续约或释放
- 接口消失 → LINGER 清理

### 9.4 与其他组件的协作

| 组件 | 交互方式 | 功能 |
|------|----------|------|
| **udev** | .link 文件 + device_monitor | 设备命名、初始化通知 |
| **resolved** | D-Bus | 推送 DNS/域名配置 |
| **timesyncd** | D-Bus | 推送 NTP 服务器 |
| **hostnamed** | D-Bus | 推送 DHCP 主机名 |
| **timedated** | D-Bus | 推送 DHCP 时区 |
| **firewalld** | D-Bus | 防火墙区域配置 |
| **wait-online** | 状态文件 + D-Bus | 等待网络就绪 |
