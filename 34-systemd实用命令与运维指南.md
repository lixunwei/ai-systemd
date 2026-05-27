# systemd 实用命令与运维指南

## 1. 概述

本文档整理 systemd 日常运维中最常用的命令，涵盖服务管理、状态查看、启动分析、cgroup 资源、依赖关系、日志查询等方面，作为源码分析系列的实践补充。

---

## 2. 服务/单元管理 — systemctl

### 2.1 基础操作

```bash
# 启动/停止/重启/重载
systemctl start <unit>
systemctl stop <unit>
systemctl restart <unit>
systemctl reload <unit>          # 不中断进程，重载配置
systemctl reload-or-restart <unit>  # 支持 reload 就 reload，否则 restart

# 启用/禁用开机自启
systemctl enable <unit>          # 创建 [Install] 中声明的符号链接
systemctl disable <unit>         # 删除符号链接
systemctl enable --now <unit>    # 启用并立即启动

# 屏蔽/解除屏蔽
systemctl mask <unit>            # 链接到 /dev/null，彻底禁止启动
systemctl unmask <unit>
```

### 2.2 状态查看

```bash
# 查看单元详细状态
systemctl status <unit>
# 输出：活动状态、PID、内存/CPU、最近日志

# 查看所有单元
systemctl list-units                    # 已加载的活动单元
systemctl list-units --all              # 包含非活动
systemctl list-units --failed           # 仅失败的
systemctl list-units --type=service     # 按类型过滤
systemctl list-units --state=running    # 按状态过滤

# 查看单元文件
systemctl list-unit-files               # 所有单元文件及其 enable 状态
systemctl list-unit-files --type=timer  # 定时器

# 查看单元属性
systemctl show <unit>                   # 所有属性
systemctl show <unit> -p MainPID        # 指定属性
systemctl show <unit> -p CPUUsageNSec -p MemoryCurrent  # 多属性
```

### 2.3 运行时修改

```bash
# 运行时设置属性（不持久化）
systemctl set-property <unit> CPUQuota=20%
systemctl set-property <unit> MemoryMax=512M

# 编辑单元（创建 drop-in 覆盖文件）
systemctl edit <unit>            # 创建 /etc/systemd/system/<unit>.d/override.conf
systemctl edit --full <unit>     # 编辑完整单元文件副本

# 重载配置（修改单元文件后）
systemctl daemon-reload
```

### 2.4 系统级操作

```bash
# 关机/重启
systemctl poweroff
systemctl reboot
systemctl halt
systemctl kexec                  # 热重启内核

# 挂起/休眠
systemctl suspend
systemctl hibernate
systemctl hybrid-sleep

# 切换运行级别
systemctl isolate multi-user.target
systemctl isolate graphical.target
systemctl isolate rescue.target
systemctl isolate emergency.target

# 设置默认启动目标
systemctl set-default graphical.target
systemctl get-default
```

---

## 3. 启动时序分析 — systemd-analyze

### 3.1 启动耗时总览

```bash
systemd-analyze                  # 显示内核/initrd/用户空间耗时
# 输出示例:
# Startup finished in 1.2s (kernel) + 2.1s (initrd) + 5.3s (userspace) = 8.6s

systemd-analyze time             # 同上
```

### 3.2 各服务启动耗时排序

```bash
systemd-analyze blame
# 按耗时从高到低列出所有单元的激活时间
# 输出示例:
#  3.200s NetworkManager-wait-online.service
#  1.850s systemd-udev-settle.service
#  0.950s docker.service
```

### 3.3 关键链分析（最长路径）

```bash
systemd-analyze critical-chain
# 输出从 default.target 向下的时间关键路径
# 标记 @time 表示绝对激活时间，+time 表示单元自身耗时

systemd-analyze critical-chain graphical.target
# 指定起始 target

# 输出示例:
# graphical.target @5.3s
# └─multi-user.target @5.3s
#   └─docker.service @4.3s +0.950s
#     └─network-online.target @4.3s
#       └─NetworkManager-wait-online.service @1.1s +3.2s
```

### 3.4 启动时序图（SVG）

```bash
systemd-analyze plot > boot.svg
# 生成 SVG 时序图，可在浏览器中查看
# 每个单元显示为水平条：灰色=等待，红色=激活中，绿色=活动
```

### 3.5 依赖关系图（DOT）

```bash
systemd-analyze dot <unit> | dot -Tsvg > deps.svg
# 生成单元依赖关系 DOT 图

systemd-analyze dot --to-pattern='*.target' | dot -Tpng > targets.png
# 仅显示到 target 的依赖

systemd-analyze dot 'systemd-*' | dot -Tsvg > systemd-deps.svg
# 匹配模式
```

### 3.6 其他分析命令

```bash
systemd-analyze verify <unit-file>  # 验证单元文件语法
systemd-analyze security <unit>     # 安全性评分（沙箱/隔离程度）
systemd-analyze calendar "Mon *-*-* 03:00" # 验证 OnCalendar 表达式
systemd-analyze timespan "2h 30min"  # 验证时间跨度
systemd-analyze cat-config systemd/system.conf # 查看配置叠加结果
systemd-analyze unit-paths           # 显示单元搜索路径
systemd-analyze dump                 # 导出 PID1 完整内部状态
```

---

## 4. 依赖关系查看

### 4.1 列出依赖

```bash
# 列出单元的所有依赖（递归）
systemctl list-dependencies <unit>

# 仅显示反向依赖（谁依赖了它）
systemctl list-dependencies <unit> --reverse

# 仅显示特定依赖类型
systemctl list-dependencies <unit> --after   # After= 排序依赖
systemctl list-dependencies <unit> --before  # Before= 排序依赖

# 查看启动目标的完整依赖树
systemctl list-dependencies default.target
systemctl list-dependencies multi-user.target --all
```

### 4.2 查看单元关联

```bash
# 查看谁 Wants/Requires 了某个单元
systemctl show <unit> -p WantedBy
systemctl show <unit> -p RequiredBy
systemctl show <unit> -p Conflicts

# 查看单元的 After/Before 关系
systemctl show <unit> -p After
systemctl show <unit> -p Before
```

---

## 5. cgroup 资源查看与控制

### 5.1 查看 cgroup 层级

```bash
# 树形显示 cgroup 进程
systemd-cgls                     # 完整 cgroup 树
systemd-cgls /system.slice       # 指定切片
systemd-cgls /user.slice         # 用户切片

# 查看特定单元的 cgroup 内容
systemd-cgls -u docker.service
```

### 5.2 资源使用统计

```bash
# 按资源消耗排序
systemd-cgtop                    # 类似 top，按 cgroup 分组
# 显示：CPU%、内存、IO、任务数

# 查看特定单元的资源使用
systemctl show <unit> -p MemoryCurrent -p CPUUsageNSec -p TasksCurrent -p IPIngressBytes
```

### 5.3 资源限制配置

```bash
# 运行时设置（立即生效，重启后失效）
systemctl set-property <unit> MemoryMax=1G
systemctl set-property <unit> CPUQuota=50%
systemctl set-property <unit> TasksMax=100
systemctl set-property <unit> IOWeight=200

# 持久化设置（写入 drop-in 文件）
systemctl set-property <unit> MemoryMax=1G --runtime=false

# 或通过编辑单元文件
systemctl edit <unit>
# 添加:
# [Service]
# MemoryMax=1G
# CPUQuota=50%
# TasksMax=100
```

### 5.4 常用 cgroup 资源指令

| 指令 | 说明 | 示例 |
|------|------|------|
| `CPUQuota=` | CPU 配额百分比 | `200%`（允许 2 核） |
| `CPUWeight=` | CPU 权重(1-10000) | `100`（默认） |
| `MemoryMax=` | 内存硬限制 | `1G` |
| `MemoryHigh=` | 内存软限制（触发回收） | `800M` |
| `MemorySwapMax=` | Swap 限制 | `0`（禁用 swap） |
| `TasksMax=` | 最大任务数 | `4096` |
| `IOWeight=` | IO 权重(1-10000) | `100` |
| `IOReadBandwidthMax=` | 读带宽限制 | `/dev/sda 100M` |
| `IOWriteBandwidthMax=` | 写带宽限制 | `/dev/sda 50M` |
| `IPAddressAllow=` | 允许的 IP 范围 | `192.168.1.0/24` |
| `IPAddressDeny=` | 拒绝的 IP 范围 | `any` |

---

## 6. 日志查询 — journalctl

### 6.1 基础查询

```bash
# 查看所有日志
journalctl

# 查看特定单元日志
journalctl -u <unit>
journalctl -u nginx.service -u php-fpm.service  # 多个单元

# 实时跟踪
journalctl -f                    # 类似 tail -f
journalctl -f -u <unit>         # 跟踪特定单元

# 最近 N 条
journalctl -n 50 -u <unit>
```

### 6.2 时间过滤

```bash
journalctl --since "2024-01-01 00:00:00"
journalctl --since "1 hour ago"
journalctl --since today
journalctl --since yesterday --until today
journalctl -b                    # 本次启动
journalctl -b -1                 # 上次启动
journalctl --list-boots          # 列出所有启动记录
```

### 6.3 级别过滤

```bash
journalctl -p err                # 仅 error 及以上
journalctl -p warning            # warning 及以上
journalctl -p emerg..err         # 范围
# 级别: emerg, alert, crit, err, warning, notice, info, debug
```

### 6.4 字段过滤

```bash
journalctl _PID=1234             # 指定 PID
journalctl _UID=1000             # 指定用户
journalctl _COMM=sshd            # 指定程序名
journalctl _TRANSPORT=kernel     # 内核消息
journalctl CONTAINER_NAME=myapp  # 自定义字段

# 组合
journalctl _SYSTEMD_UNIT=sshd.service _PID=5678
```

### 6.5 输出格式

```bash
journalctl -o short              # 默认（类 syslog）
journalctl -o short-precise      # 精确时间戳（微秒）
journalctl -o verbose            # 显示所有字段
journalctl -o json               # JSON 格式
journalctl -o json-pretty        # 格式化 JSON
journalctl -o cat                # 仅消息文本
journalctl -o export             # 二进制导出格式
```

### 6.6 磁盘管理

```bash
journalctl --disk-usage          # 查看日志占用空间
journalctl --vacuum-size=500M    # 清理到 500M 以内
journalctl --vacuum-time=30d     # 清理 30 天前的
journalctl --rotate              # 强制轮转
journalctl --verify              # 验证日志完整性
```

---

## 7. 临时服务与调试 — systemd-run

### 7.1 运行临时服务

```bash
# 在独立 scope 中运行命令
systemd-run --unit=mytest /usr/bin/sleep 3600

# 带资源限制
systemd-run --property=MemoryMax=100M --property=CPUQuota=10% /usr/bin/stress-ng --cpu 4

# 临时定时器
systemd-run --on-active=30s /usr/bin/systemctl restart nginx
systemd-run --on-calendar="*:0/5" /usr/bin/backup.sh

# 在指定切片中运行
systemd-run --slice=my.slice /usr/bin/myapp

# 交互式（带 TTY）
systemd-run --pty --shell        # 在新 scope 中启动 shell
systemd-run --pty -u debug bash  # 命名的调试 shell
```

### 7.2 瞬态单元

```bash
# 创建瞬态 service（Type=oneshot）
systemd-run --unit=cleanup --service-type=oneshot /usr/bin/find /tmp -mtime +7 -delete

# 查看其状态
systemctl status cleanup.service
```

---

## 8. 登录/会话管理 — loginctl

```bash
# 查看会话
loginctl list-sessions
loginctl session-status <id>
loginctl show-session <id>

# 查看用户
loginctl list-users
loginctl user-status <user>

# 查看座位
loginctl list-seats
loginctl seat-status seat0

# 终止会话
loginctl terminate-session <id>
loginctl kill-session <id>

# Linger 管理
loginctl enable-linger <user>    # 允许无会话时保持服务
loginctl disable-linger <user>

# 锁定/解锁
loginctl lock-session <id>
loginctl lock-sessions           # 锁定所有会话
```

---

## 9. 网络管理 — networkctl

```bash
networkctl list                  # 列出所有网络接口
networkctl status                # 概览
networkctl status eth0           # 特定接口详情
networkctl up eth0               # 启用接口
networkctl down eth0             # 禁用接口
networkctl reconfigure eth0      # 重新配置
```

---

## 10. DNS 解析 — resolvectl

```bash
resolvectl status                # DNS 配置状态
resolvectl query example.com     # 查询域名
resolvectl statistics            # DNS 统计信息
resolvectl flush-caches          # 清空 DNS 缓存
resolvectl dns eth0 8.8.8.8      # 设置接口 DNS
```

---

## 11. 时间管理 — timedatectl

```bash
timedatectl                      # 查看当前时间设置
timedatectl set-timezone Asia/Shanghai
timedatectl set-ntp true         # 启用 NTP 同步
timedatectl set-time "2024-01-01 12:00:00"
timedatectl list-timezones       # 列出所有时区
```

---

## 12. 主机信息 — hostnamectl

```bash
hostnamectl                      # 查看主机名信息
hostnamectl set-hostname myserver
hostnamectl set-chassis server   # 设置机箱类型
hostnamectl set-deployment production
```

---

## 13. 实用技巧

### 13.1 查找耗时最长的启动瓶颈

```bash
systemd-analyze blame | head -20
systemd-analyze critical-chain
systemd-analyze plot > /tmp/boot.svg && xdg-open /tmp/boot.svg
```

### 13.2 安全审计一个服务

```bash
systemd-analyze security <unit>
# 输出沙箱评分（0-10，10=最不安全）
# 列出每项安全特性的启用状态
```

### 13.3 追踪单元状态变化

```bash
# 订阅单元状态变化
busctl monitor org.freedesktop.systemd1

# 或使用 journalctl 过滤
journalctl -u <unit> -f --output=short-precise
```

### 13.4 查看失败原因

```bash
systemctl --failed                # 列出所有失败单元
systemctl status <failed-unit>    # 查看退出码和最近日志
journalctl -u <unit> -b --no-pager | tail -50
```

### 13.5 临时覆盖单元设置

```bash
# 不修改原文件，创建 override
systemctl edit <unit>
# 添加内容后保存，立即生效（需要 daemon-reload + restart）

# 查看最终生效的配置
systemctl cat <unit>              # 显示所有配置片段
```

### 13.6 导出/恢复 PID1 状态

```bash
systemd-analyze dump              # 导出 PID1 完整内部状态（JSON）
systemctl daemon-reexec           # 重新执行 PID1（升级后）
systemctl daemon-reload           # 重新加载所有单元文件
```

---

## 14. 命令速查表

| 需求 | 命令 |
|------|------|
| 查看服务状态 | `systemctl status <unit>` |
| 查看所有失败 | `systemctl --failed` |
| 查看启动耗时 | `systemd-analyze blame` |
| 查看关键路径 | `systemd-analyze critical-chain` |
| 生成启动图 | `systemd-analyze plot > boot.svg` |
| 查看依赖树 | `systemctl list-dependencies <unit>` |
| 查看 cgroup 树 | `systemd-cgls` |
| 实时资源监控 | `systemd-cgtop` |
| 设置内存限制 | `systemctl set-property <unit> MemoryMax=1G` |
| 查看单元日志 | `journalctl -u <unit>` |
| 本次启动日志 | `journalctl -b` |
| 只看错误 | `journalctl -p err` |
| 清理日志 | `journalctl --vacuum-size=500M` |
| 临时运行任务 | `systemd-run --pty /bin/bash` |
| 安全评估 | `systemd-analyze security <unit>` |
| 验证配置 | `systemd-analyze verify <file>` |
| 查看 drop-in | `systemctl cat <unit>` |
| 查看所有定时器 | `systemctl list-timers` |
| 查看所有 socket | `systemctl list-sockets` |
