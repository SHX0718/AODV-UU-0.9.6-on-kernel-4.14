# AODV-UU 交叉编译指南

## 目标设备信息

| 项目 | 值 |
|------|-----|
| 设备型号 | GL.iNet GL-AR300M (NOR) |
| CPU | Qualcomm Atheros QCA9533 ver 2 rev 0 |
| CPU架构 | MIPS 24Kc V7.4 |
| OpenWrt版本 | 22.03.4 |
| Linux内核版本 | 5.10.x |
| 目标平台 | ath79/nand |
| 架构标识 | mips_24kc |

### 网络接口

| 接口 | MAC地址 | 说明 |
|------|---------|------|
| eth0 | 94:83:C4:6A:2F:35 | 以太网端口 |
| eth1 | 94:83:C4:6A:2F:36 | 以太网端口（连接到br-lan） |
| wlan0 | 94:83:C4:6A:2F:35 | 无线网卡 |
| br-lan | 94:83:C4:6A:2F:36 | 网桥接口 (192.168.8.1) |

---

## 一、环境准备

### 1.1 主机系统要求

推荐使用 Ubuntu 20.04/22.04 LTS 或其他 Linux 发行版。

```bash
# 安装必要的依赖包
sudo apt update
sudo apt install -y build-essential ccache ecj fastjar file g++ gawk \
    gettext git java-propose-classpath libelf-dev libncurses5-dev \
    libncursesw5-dev libssl-dev python3 python3-distutils python3-setuptools \
    rsync subversion swig time unzip wget xsltproc zlib1g-dev
```

### 1.2 创建工作目录

```bash
# 创建工作目录
mkdir -p ~/openwrt-build
cd ~/openwrt-build
```

---

## 二、获取 OpenWrt SDK

有两种方式获取编译环境：使用预编译的 SDK 或从源码编译。

### 方式一：下载预编译 SDK（推荐）

```bash
# 下载 OpenWrt 22.03.4 SDK for ath79/nand
cd ~/openwrt-build
wget https://downloads.openwrt.org/releases/22.03.4/targets/ath79/nand/openwrt-sdk-22.03.4-ath79-nand_gcc-11.2.0_musl.Linux-x86_64.tar.xz

# 解压
tar -xJf openwrt-sdk-22.03.4-ath79-nand_gcc-11.2.0_musl.Linux-x86_64.tar.xz
mv openwrt-sdk-22.03.4-ath79-nand_gcc-11.2.0_musl.Linux-x86_64 openwrt-sdk
```

### 方式二：从源码编译（完整控制）

```bash
# 克隆 OpenWrt 源码
git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt
git checkout v22.03.4

# 更新 feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 配置目标平台
make menuconfig
# 选择:
#   Target System: Atheros ATH79
#   Subtarget: Generic devices with NAND flash
#   Target Profile: GL.iNet GL-AR300M (NOR)

# 下载依赖
make download -j$(nproc)

# 编译工具链和内核
make -j$(nproc)
```

---

## 三、内核源码准备

### 3.1 获取内核头文件

如果使用 SDK，内核头文件位于：
```
openwrt-sdk/staging_dir/target-mips_24kc_musl/linux-ath79_nand/linux-5.10.x/
```

如果从源码编译，位于：
```
openwrt/build_dir/target-mips_24kc_musl/linux-ath79_nand/linux-5.10.x/
```

### 3.2 验证内核版本

```bash
# 查看内核版本
cat ~/openwrt-build/openwrt-sdk/staging_dir/target-mips_24kc_musl/linux-ath79_nand/linux-*/include/generated/utsrelease.h
```

---

## 四、配置交叉编译环境

### 4.1 设置环境变量

创建环境配置脚本 `env-openwrt.sh`：

```bash
#!/bin/bash

# OpenWrt SDK 根目录
export OPENWRT_SDK=~/openwrt-build/openwrt-sdk

# Staging 目录
export STAGING_DIR=$OPENWRT_SDK/staging_dir

# 工具链路径
export TOOLCHAIN_DIR=$STAGING_DIR/toolchain-mips_24kc_gcc-11.2.0_musl

# 目标平台路径
export TARGET_DIR=$STAGING_DIR/target-mips_24kc_musl

# 内核源码路径（根据实际内核版本调整）
export KERNEL_DIR=$TARGET_DIR/linux-ath79_nand/linux-5.10.176

# 添加工具链到 PATH
export PATH=$TOOLCHAIN_DIR/bin:$PATH

# 交叉编译器前缀
export CROSS_COMPILE=mips-openwrt-linux-musl-
export ARCH=mips

# 显示配置信息
echo "=== OpenWrt Cross-Compile Environment ==="
echo "OPENWRT_SDK: $OPENWRT_SDK"
echo "TOOLCHAIN_DIR: $TOOLCHAIN_DIR"
echo "KERNEL_DIR: $KERNEL_DIR"
echo "CROSS_COMPILE: $CROSS_COMPILE"
echo "========================================"
```

加载环境：
```bash
source env-openwrt.sh
```

### 4.2 验证交叉编译器

```bash
# 检查编译器版本
${CROSS_COMPILE}gcc --version

# 输出示例:
# mips-openwrt-linux-musl-gcc (OpenWrt GCC 11.2.0 r20123-38ccc47687) 11.2.0
```

---

## 五、编译 AODV-UU 内核模块

### 5.1 修改 Makefile

编辑 `aodv-uu-0.9.6/lnx/Makefile`，配置以下变量：

```makefile
# 交叉编译设置 - OpenWrt 22.03.4 on GL-AR300M (MIPS 24Kc)
ARCH = mips
CROSS_COMPILE = mips-openwrt-linux-musl-

# 内核目录 - 根据实际路径修改
KERNEL_DIR = $(HOME)/openwrt-build/openwrt-sdk/staging_dir/target-mips_24kc_musl/linux-ath79_nand/linux-5.10.176

# 内核头文件
KERNEL_INC = $(KERNEL_DIR)/include
```

### 5.2 编译内核模块

```bash
# 进入源码目录
cd /path/to/AODV-UU-0.9.6-on-kernel-4.14/aodv-uu-0.9.6/lnx

# 加载交叉编译环境
source ~/openwrt-build/env-openwrt.sh

# 清理旧的编译文件
make clean

# 编译内核模块
make ARCH=mips CROSS_COMPILE=mips-openwrt-linux-musl- \
     KERNEL_DIR=$KERNEL_DIR

# 或者使用完整命令
make -C $KERNEL_DIR M=$(pwd) \
     ARCH=mips CROSS_COMPILE=mips-openwrt-linux-musl- \
     modules
```

### 5.3 编译用户空间程序 (aodvd)

```bash
# 进入主源码目录
cd /path/to/AODV-UU-0.9.6-on-kernel-4.14/aodv-uu-0.9.6

# 编译 aodvd
make CC=mips-openwrt-linux-musl-gcc \
     CFLAGS="-I$STAGING_DIR/target-mips_24kc_musl/usr/include" \
     LDFLAGS="-L$STAGING_DIR/target-mips_24kc_musl/usr/lib"
```

### 5.4 验证编译结果

```bash
# 检查内核模块格式
file lnx/kaodv.ko
# 应显示: ELF 32-bit MSB relocatable, MIPS, MIPS32 rel2 version 1

# 检查 aodvd 格式
file aodvd
# 应显示: ELF 32-bit MSB executable, MIPS, MIPS32 rel2 version 1
```

---

## 六、部署到 GL-AR300M

### 6.1 传输文件到设备

```bash
# 使用 SCP 传输（假设设备 IP 为 192.168.8.1）
scp lnx/kaodv.ko root@192.168.8.1:/tmp/
scp aodvd root@192.168.8.1:/tmp/
```

### 6.2 在设备上安装

SSH 登录到设备：
```bash
ssh root@192.168.8.1
```

在设备上执行：
```bash
# 移动文件到合适位置
mv /tmp/kaodv.ko /lib/modules/$(uname -r)/
mv /tmp/aodvd /usr/sbin/

# 设置执行权限
chmod +x /usr/sbin/aodvd

# 更新模块依赖（可选）
depmod -a
```

### 6.3 加载内核模块

```bash
# 加载模块，指定网络接口
insmod /lib/modules/$(uname -r)/kaodv.ko ifname=wlan0

# 或者加载多个接口
insmod /lib/modules/$(uname -r)/kaodv.ko ifname=wlan0,eth0

# 验证模块加载
lsmod | grep kaodv

# 查看内核日志
dmesg | tail -20
```

### 6.4 运行 AODV 守护进程

```bash
# 启动 aodvd（使用无线接口）
aodvd -i wlan0

# 启动 aodvd（多接口模式）
aodvd -i wlan0 -i eth0

# 后台运行
aodvd -i wlan0 -d

# 查看 AODV 路由表
cat /proc/kaodv_expl
cat /proc/kaodv_queue
cat /proc/kaodv
```

---

## 七、开机自启动配置

### 7.1 创建 init 脚本

创建文件 `/etc/init.d/aodv`：

```bash
#!/bin/sh /etc/rc.common

START=99
STOP=10

USE_PROCD=1

PROG=/usr/sbin/aodvd
IFACE=wlan0

start_service() {
    # 加载内核模块
    insmod /lib/modules/$(uname -r)/kaodv.ko ifname=$IFACE 2>/dev/null
    
    procd_open_instance
    procd_set_param command $PROG -i $IFACE
    procd_set_param respawn
    procd_close_instance
}

stop_service() {
    # 卸载内核模块
    rmmod kaodv 2>/dev/null
}
```

### 7.2 启用自启动

```bash
chmod +x /etc/init.d/aodv
/etc/init.d/aodv enable
/etc/init.d/aodv start
```

---

## 八、故障排除

### 8.1 常见错误

#### 错误：模块版本不匹配
```
insmod: ERROR: could not insert module kaodv.ko: Invalid module format
```
**解决方案**：确保使用与设备完全相同的内核版本和配置编译模块。

```bash
# 在设备上检查内核版本
uname -r

# 确保编译时使用相同版本的内核源码
```

#### 错误：符号未找到
```
insmod: ERROR: could not insert module kaodv.ko: Unknown symbol in module
```
**解决方案**：检查内核配置是否启用了所需功能（如 netfilter）。

#### 错误：交叉编译器找不到
```
mips-openwrt-linux-musl-gcc: command not found
```
**解决方案**：确保已正确设置 PATH 环境变量。

### 8.2 调试命令

```bash
# 查看内核日志
dmesg | grep -i kaodv
logread | grep -i aodv

# 检查 proc 文件
ls -la /proc/net/ | grep kaodv

# 查看模块信息
modinfo /lib/modules/$(uname -r)/kaodv.ko

# 查看网络接口状态
ifconfig -a
ip addr show
```

### 8.3 获取设备内核配置

如果编译仍有问题，可以从设备获取准确的内核配置：

```bash
# 在设备上（如果启用了 /proc/config.gz）
zcat /proc/config.gz > /tmp/kernel.config
scp root@192.168.8.1:/tmp/kernel.config .
```

---

## 九、参考资源

- [OpenWrt 官方文档](https://openwrt.org/docs/start)
- [OpenWrt 22.03.4 发布说明](https://openwrt.org/releases/22.03/notes-22.03.4)
- [GL.iNet GL-AR300M 设备页面](https://openwrt.org/toh/gl.inet/gl-ar300m)
- [Linux 内核模块开发指南](https://www.kernel.org/doc/html/latest/kbuild/modules.html)
- [AODV-UU 项目](http://aodvuu.sourceforge.net/)

---

## 十、版本历史

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-01-29 | 1.0 | 初始版本，适配 Linux 5.10 内核 |

---

## 附录 A：快速编译脚本

创建 `build.sh`：

```bash
#!/bin/bash
set -e

# 配置
OPENWRT_SDK=~/openwrt-build/openwrt-sdk
STAGING_DIR=$OPENWRT_SDK/staging_dir
TOOLCHAIN_DIR=$STAGING_DIR/toolchain-mips_24kc_gcc-11.2.0_musl
TARGET_DIR=$STAGING_DIR/target-mips_24kc_musl
KERNEL_DIR=$TARGET_DIR/linux-ath79_nand/linux-5.10.176

export PATH=$TOOLCHAIN_DIR/bin:$PATH
export STAGING_DIR

ARCH=mips
CROSS_COMPILE=mips-openwrt-linux-musl-

echo "=== Building AODV-UU for OpenWrt ==="
echo "Kernel: $KERNEL_DIR"
echo "Cross-compiler: ${CROSS_COMPILE}gcc"

# 清理
echo "[1/3] Cleaning..."
make -C lnx clean 2>/dev/null || true

# 编译内核模块
echo "[2/3] Building kernel module..."
make -C $KERNEL_DIR M=$(pwd)/lnx \
    ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE modules

# 编译用户空间程序
echo "[3/3] Building aodvd..."
make CC=${CROSS_COMPILE}gcc aodvd

echo "=== Build Complete ==="
echo "Kernel module: lnx/kaodv.ko"
echo "Daemon: aodvd"

# 验证
file lnx/kaodv.ko
file aodvd
```

使用方法：
```bash
chmod +x build.sh
./build.sh
```
