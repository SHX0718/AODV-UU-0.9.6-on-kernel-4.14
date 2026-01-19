# AODV-UU 0.9.6 for Linux Kernel 4.x

AODV-UU 是瑞典乌普萨拉大学开发的 AODV（Ad-hoc On-demand Distance Vector）路由协议实现，基于 RFC 3561 标准。本项目已适配 Linux 4.x 内核（4.14 - 4.15+）。

## 项目简介

AODV 是一种用于移动自组织网络（MANET）的按需路由协议。本实现包含：

- **aodvd**: 用户态守护进程，负责管理路由表
- **kaodv.ko**: 内核模块，负责数据包捕获和转发

## 支持的平台

| 平台 | 内核版本 | 架构 | 状态 |
|------|----------|------|------|
| Ubuntu/Debian | 4.14 - 4.15+ | x86, x86_64 | ✅ 已测试 |
| Raspberry Pi | 4.14.93+ | ARM | ✅ 已测试 |

## 系统要求

- Linux 内核 4.x（需要 Netfilter 支持）
- GCC 编译器
- 内核头文件
- 无线网卡（支持 Ad-hoc/IBSS 模式）

## 快速开始

### 1. 安装依赖

**Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install -y build-essential linux-headers-$(uname -r)
```

**CentOS/RHEL:**
```bash
sudo yum install -y gcc make kernel-devel-$(uname -r)
```

### 2. 编译

```bash
cd aodv-uu-0.9.6

# 清理旧文件
make clean

# 编译（自动检测内核版本）
make
```

编译成功后生成：
- `aodvd` - 用户态守护进程
- `kaodv.ko` - 内核模块

### 3. 安装

```bash
sudo make install
```

这将：
- 安装 `aodvd` 到 `/usr/sbin/`
- 安装 `kaodv.ko` 到 `/lib/modules/$(uname -r)/aodv/`

### 4. 卸载

```bash
sudo make uninstall
```

## 使用指南

### 配置无线网卡为 Ad-hoc 模式

```bash
# 设置接口名称（根据实际情况修改）
IFACE=wlo1

# 关闭接口
sudo ip link set $IFACE down

# 设置 Ad-hoc 模式
sudo iw $IFACE set type ibss

# 启动接口
sudo ip link set $IFACE up

# 加入 Ad-hoc 网络（SSID 和频率）
sudo iw $IFACE ibss join AODV-MESH 2412

# 配置 IP 地址（每个节点使用不同IP）
sudo ip addr add 192.168.1.1/24 dev $IFACE
```

### 加载内核模块

```bash
# 方式1：使用 insmod（指定接口）
sudo insmod kaodv.ko ifname=wlo1

# 方式2：使用 modprobe（安装后）
sudo modprobe kaodv ifname=wlo1

# 验证模块已加载
lsmod | grep kaodv
```

### 运行守护进程

```bash
# 基本运行
sudo aodvd -i wlo1

# 调试模式（推荐用于测试）
sudo aodvd -l -r 3 -i wlo1

# 网关模式
sudo aodvd -l -r 3 -i wlo1 -w
```

**命令行参数：**

| 参数 | 说明 |
|------|------|
| `-i <interface>` | 指定网络接口 |
| `-l` | 启用日志（写入 /var/log/aodvd.log） |
| `-r <seconds>` | 路由表打印间隔（秒） |
| `-w` | 启用网关模式 |
| `-u` | 启用单向链路检测 |
| `-d` | 禁用 HELLO 消息 |
| `-g` | 强制使用本机网关 |
| `--help` | 显示帮助信息 |

### 停止 AODV

```bash
# 停止守护进程
sudo killall aodvd

# 卸载内核模块
sudo rmmod kaodv
```

## 监控和调试

### 查看状态信息

```bash
# 查看路由表
cat /proc/net/kaodv_expl

# 查看数据包队列
cat /proc/net/kaodv_queue

# 查看模块信息
cat /proc/net/kaodv

# 查看日志
tail -f /var/log/aodvd.log

# 查看路由表日志
tail -f /var/log/aodvd.rtlog
```

### 内核日志

```bash
# 查看内核消息
dmesg | grep -i kaodv

# 实时监控
dmesg -w | grep -i kaodv
```

## 交叉编译（ARM/树莓派）

### 编译 ARM 版本

1. 修改 `Makefile` 中的 `bar=yes`
2. 配置交叉编译工具链路径
3. 执行：
```bash
make arm
```

生成的 `aodvd` 和 `kaodv.ko` 可复制到目标系统运行。

## 网关功能

启用网关模式可让 AODV 网络访问外部网络：

```bash
# 在网关节点上运行
sudo aodvd -l -w -i wlo1

# 配置 NAT（将 eth0 替换为连接外网的接口）
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# 在其他节点上添加默认路由
sudo route add default dev wlo1
```

## 测试拓扑示例

```
节点A (192.168.1.1) <---> 节点B (192.168.1.2) <---> 节点C (192.168.1.3)
```

在节点A上测试连通性：
```bash
# 带路由记录的 ping
ping -R 192.168.1.3
```

## 文件结构

```
aodv-uu-0.9.6/
├── *.c, *.h          # 用户态守护进程源码
├── lnx/              # 内核模块源码
│   ├── kaodv-mod.c   # 主模块
│   ├── kaodv-queue.c # 数据包队列
│   ├── kaodv-netlink.c # Netlink 通信
│   ├── kaodv-expl.c  # 路由表管理
│   └── kaodv-ipenc.c # IP 封装
├── Makefile          # 主 Makefile
└── README            # 原始文档
```

## 常见问题

### Q: 模块加载失败
```bash
# 检查内核头文件是否安装
ls /lib/modules/$(uname -r)/build

# 查看详细错误
dmesg | tail -20
```

### Q: 编译错误 "No linux source found"
```bash
# 安装内核头文件
sudo apt-get install linux-headers-$(uname -r)
```

### Q: aodvd 无法启动
```bash
# 确保模块已加载
lsmod | grep kaodv

# 检查接口是否存在
ip link show
```

### Q: 无线网卡不支持 Ad-hoc 模式
```bash
# 检查支持的模式
iw list | grep -A 10 "Supported interface modes"
```

## API 兼容性说明

本版本已适配以下内核 API：

| API | 版本要求 | 说明 |
|-----|----------|------|
| `nf_register_net_hook()` | 4.3+ | Netfilter 钩子注册 |
| `ip_route_me_harder()` | 4.4+ | 带 net 命名空间参数 |
| `netlink_ack()` | 4.12+ | 带 extack 参数 |
| `proc_create()` | 3.10+ | proc 文件系统 |

## 参考资料

- [RFC 3561 - AODV Routing Protocol](https://tools.ietf.org/html/rfc3561)
- [APE Testbed](http://apetestbed.sourceforge.net)
- 原始文档：`aodv-uu-0.9.6/README`

## 许可证

本项目基于 GNU General Public License (GPL) 发布。

## 作者

- **原作者**: Erik Nordström (Uppsala University)
- **移植适配**: 4.x 内核兼容性修改

---

**注意**: 本软件按"原样"提供，不提供任何形式的保证。在生产环境使用前请充分测试。
