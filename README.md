# aodv-uu-0.9.6

## Supported Kernel Versions
- Linux kernel 4.14 - 5.10+ (tested with OpenWrt 22.03.4)
- Raspberry Pi (Raspbian/Linux)
- OpenWrt on MIPS devices (e.g., GL.iNet GL-AR300M with ath79/nand)

## Documentation
- **[CROSS_COMPILE_GUIDE.md](CROSS_COMPILE_GUIDE.md)** - 详细的 OpenWrt 交叉编译指南

## Quick Start

### 1. Load Kernel Module
```bash
insmod kaodv.ko ifname=wlan0
```

### 2. Start AODV Daemon
```bash
./aodvd -i wlan0
```

### 3. Check Status
```bash
cat /proc/kaodv
cat /proc/kaodv_expl
```

## Cross-Compile for OpenWrt (MIPS)

详细说明请参阅 [CROSS_COMPILE_GUIDE.md](CROSS_COMPILE_GUIDE.md)

简要步骤：
1. 下载 OpenWrt SDK for ath79/nand
2. 设置环境变量和交叉编译工具链
3. 编辑 `lnx/Makefile` 设置 `KERNEL_DIR` 和 `CROSS_COMPILE`
4. 运行 `make`

## Target Device Info (GL-AR300M)

| 项目 | 值 |
|------|-----|
| CPU | MIPS 24Kc (QCA9533) |
| OpenWrt | 22.03.4 |
| Kernel | Linux 5.10 |
| Target | ath79/nand |

## Kernel API Changes Supported

| 内核版本 | API 变化 |
|----------|---------|
| 4.15+ | `timer_setup()` 替代 `init_timer()` |
| 5.4+ | `ip_route_me_harder()` 增加 `struct sock *sk` 参数 |
| 5.6+ | `struct proc_ops` 替代 `struct file_operations` |

## Files

| 文件 | 说明 |
|------|------|
| `aodvd` | AODV 用户空间守护进程 |
| `kaodv.ko` | AODV 内核模块 |
| `lnx/` | 内核模块源码 |

## License

GPL v2 - See [aodv-uu-0.9.6/GPL](aodv-uu-0.9.6/GPL)
