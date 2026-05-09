# 系统控制工具分析 — systemctl

## 一、概述

`systemctl` 是 systemd 的主要用户界面命令，用于管理系统和服务。它通过 D-Bus 与 PID 1 通信。

源码位于 `src/systemctl/`。

---

## 二、源码结构

`systemctl` 使用模块化设计，每个子命令对应一个源文件：

| 文件 | 功能 |
|------|------|
| systemctl.c | 主入口、参数解析、命令分发 |
| systemctl-start-unit.c | start/stop/restart/reload 命令 |
| systemctl-enable.c | enable/disable/mask/unmask 命令 |
| systemctl-show.c | show/status 命令 |
| systemctl-list-units.c | list-units 命令 |
| systemctl-list-unit-files.c | list-unit-files 命令 |
| systemctl-list-jobs.c | list-jobs 命令 |
| systemctl-list-machines.c | list-machines 命令 |
| systemctl-list-dependencies.c | list-dependencies 命令 |
| systemctl-daemon-reload.c | daemon-reload/daemon-reexec 命令 |
| systemctl-is-active.c | is-active/is-failed/is-enabled 命令 |
| systemctl-kill.c | kill 命令 |
| systemctl-set-property.c | set-property 命令 |
| systemctl-logind.c | poweroff/reboot/suspend 等电源命令 |
| systemctl-util.c | 通用辅助函数 |
| systemctl-compat-halt.c | halt/poweroff/reboot 兼容 |
| systemctl-compat-runlevel.c | runlevel 兼容 |
| systemctl-compat-telinit.c | telinit 兼容 |

---

## 三、全局配置

`systemctl.c` 定义了大量全局参数变量（`systemctl.c:63-114`）：

| 变量 | 含义 |
|------|------|
| arg_action | 当前操作（systemctl/halt/poweroff/reboot 等） |
| arg_transport | 传输方式（local/ssh/machine） |
| arg_host | 远程主机名 |
| arg_output | 输出模式（short/json/verbose 等） |
| arg_wait | 是否等待操作完成 |
| arg_no_block | 是否异步操作（--no-block） |
| arg_no_pager | 是否禁用分页 |
| arg_no_legend | 是否隐藏表头 |
| arg_type | 过滤单元类型 |
| arg_state | 过滤单元状态 |
| arg_job_mode | Job 模式（fail/replace/isolate 等） |
| arg_scope | 作用域（system/user/global） |
| arg_force | 强制操作级别 |

---

## 四、核心命令流程

### 4.1 start/stop/restart 流程

```
systemctl start foo.service
  ↓
systemctl.c → 解析参数 → 调用 start_unit()
  ↓
systemctl-start-unit.c → start_unit()
  ├── 构造 D-Bus 方法调用
  │     目标: org.freedesktop.systemd1
  │     路径: /org/freedesktop/systemd1
  │     方法: Manager.StartUnit("foo.service", "replace")
  ├── 发送 D-Bus 调用
  ├── 获取返回的 Job 对象路径
  ├── 如果 --wait：
  │     bus_wait_for_jobs() — 等待 Job 完成
  │     检查结果（成功/失败/超时）
  └── 返回退出码
```

### 4.2 status 流程

```
systemctl status foo.service
  ↓
systemctl-show.c → show_one()
  ├── D-Bus 获取单元属性
  │     org.freedesktop.systemd1.Unit.* — 通用属性
  │     org.freedesktop.systemd1.Service.* — 服务特有属性
  ├── 格式化输出
  │     ● foo.service - Foo Service
  │        Loaded: loaded (/etc/systemd/system/foo.service; enabled)
  │        Active: active (running) since ...
  │       Process: 1234 ExecStart=...
  │      Main PID: 1234 (foo)
  │         Tasks: 3 (limit: 4915)
  │        Memory: 12.3M
  │           CPU: 1.234s
  │        CGroup: /system.slice/foo.service
  │                └─1234 /usr/bin/foo
  └── 附加最近日志（从 journald 获取）
```

### 4.3 enable/disable 流程

```
systemctl enable foo.service
  ↓
systemctl-enable.c → enable_unit()
  ├── D-Bus 调用 Manager.EnableUnitFiles()
  │     参数: ["foo.service"], false, true
  │     (文件列表, 运行时模式, 强制)
  ├── PID 1 操作:
  │     创建符号链接
  │     /etc/systemd/system/multi-user.target.wants/foo.service
  │     → /usr/lib/systemd/system/foo.service
  ├── 如果需要，调用 daemon-reload
  └── 输出创建的符号链接信息
```

### 4.4 daemon-reload 流程

```
systemctl daemon-reload
  ↓
systemctl-daemon-reload.c → daemon_reload()
  ├── D-Bus 调用 Manager.Reload()
  ├── PID 1 操作:
  │     重新扫描单元文件目录
  │     重新加载所有已变更的单元
  │     重新计算依赖关系
  └── 等待重新加载完成
```

---

## 五、传输模式

systemctl 支持三种传输模式：

| 模式 | 用途 | 参数 |
|------|------|------|
| local | 本地操作 | （默认） |
| ssh | 远程操作 | -H user@host |
| machine | 容器内操作 | -M container |

远程模式通过 SSH 转发 D-Bus 连接实现。

---

## 六、输出格式

| 格式 | 参数 | 用途 |
|------|------|------|
| short | --output=short | 默认简短格式 |
| verbose | --output=verbose | 详细格式（所有字段） |
| json | --output=json | JSON 输出 |
| json-pretty | --output=json-pretty | 格式化 JSON |
| json-short | --output=json-short | 紧凑 JSON |
| cat | --output=cat | 仅消息内容 |

---

## 七、兼容命令

systemctl 包含对传统 SysV init 命令的兼容：

| 传统命令 | systemctl 等效 | 实现文件 |
|---------|---------------|---------|
| halt | systemctl halt | systemctl-compat-halt.c |
| poweroff | systemctl poweroff | systemctl-compat-halt.c |
| reboot | systemctl reboot | systemctl-compat-halt.c |
| telinit N | systemctl isolate runlevelN.target | systemctl-compat-telinit.c |
| runlevel | systemctl list-units --type=target | systemctl-compat-runlevel.c |

---

## 八、Job 等待机制

当使用 `--wait` 或默认同步模式时，systemctl 使用 `bus-wait-for-jobs.c` 等待 Job 完成：

```
BusWaitForJobs
  ├── 订阅 JobRemoved 信号
  ├── 记录预期的 Job 路径
  ├── sd_bus_wait() 阻塞等待
  ├── 收到 JobRemoved 信号
  │     ├── result=="done" → 成功
  │     ├── result=="failed" → 失败
  │     ├── result=="timeout" → 超时
  │     ├── result=="dependency" → 依赖失败
  │     └── result=="skipped" → 跳过
  └── 返回最终结果
```
