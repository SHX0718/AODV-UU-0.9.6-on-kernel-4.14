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

## 二、获取编译环境

> **重要说明**：
> - **OpenWrt SDK** 仅用于编译**用户空间程序**，**不包含内核头文件**
> - 编译**内核模块 (kaodv.ko)** 必须使用 **OpenWrt 完整源码编译**
> - 这是因为内核模块需要与目标设备的内核版本和配置完全匹配

### 方式一：从 OpenWrt 源码完整编译（推荐，用于内核模块）

这是编译内核模块的**唯一可靠方式**。

#### 步骤 1：安装依赖

```bash
# Ubuntu/Debian 系统
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk \
    gcc-multilib g++-multilib gettext git libncurses-dev libssl-dev \
    python3-distutils rsync unzip zlib1g-dev file wget
```

#### 步骤 2：克隆 OpenWrt 源码

```bash
cd ~/openwrt-build

# 克隆 OpenWrt 源码（这会下载约 300MB）
git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt

# 切换到 22.03.4 版本（与您设备的 OpenWrt 版本匹配）
git checkout v22.03.4
```

#### 步骤 3：更新和安装 feeds

```bash
# 更新 feeds（软件包源）
./scripts/feeds update -a

# 安装所有 feeds
./scripts/feeds install -a
```

#### 步骤 4：配置目标平台

```bash
# 启动图形化配置界面
make menuconfig
```

**在 menuconfig 中进行以下选择**（使用方向键移动，Enter 进入，空格选择）：

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. Target System (第一个选项)                                         │
│    按 Enter 进入，选择: Atheros ATH79                                 │
│                                                                      │
│ 2. Subtarget                                                         │
│    按 Enter 进入，选择: Generic devices with NAND flash               │
│                                                                      │
│ 3. Target Profile                                                    │
│    按 Enter 进入，选择: GL.iNet GL-AR300M                             │
│    (如果找不到精确型号，选择 Multiple devices 或 Default)              │
└──────────────────────────────────────────────────────────────────────┘
```

**注意**：如果看不到 "Target System"，可能是终端窗口太小。请：
- 最大化终端窗口
- 或者设置终端至少 80x24 字符

保存配置并退出：
- 按 `Esc` 两次
- 选择 `Yes` 保存

#### 步骤 5：生成默认配置（可选但推荐）

```bash
# 如果 menuconfig 有问题，可以直接写入配置
cat > .config << 'EOF'
CONFIG_TARGET_ath79=y
CONFIG_TARGET_ath79_nand=y
CONFIG_TARGET_ath79_nand_DEVICE_glinet_gl-ar300m-nand=y
EOF

# 展开完整配置
make defconfig
```

#### 步骤 6：下载所需源码包

```bash
# 下载所有需要的源码包（需要一些时间，取决于网络）
make download -j$(nproc)

# 如果下载失败，可以单线程重试
make download -j1 V=s
```

#### 步骤 7：编译工具链和内核

```bash
# 首次编译建议单线程，便于发现错误
make -j1 V=s

# 或者多线程编译（编译成功后再用）
# make -j$(nproc)
```

> **编译时间**：首次编译可能需要 1-3 小时，取决于电脑性能。
> 
> **磁盘空间**：需要约 15-20GB 空闲空间。

#### 步骤 8：确认内核源码位置

编译完成后，内核源码位于：

```bash
# 查找内核源码目录
ls ~/openwrt-build/openwrt/build_dir/target-mips_24kc_musl/linux-ath79_nand/

# 应该看到类似：
# linux-5.10.176/
```

完整路径示例：
```
~/openwrt-build/openwrt/build_dir/target-mips_24kc_musl/linux-ath79_nand/linux-5.10.176/
```

---

### 方式二：仅下载 SDK（只能编译用户空间程序 aodvd）

> **警告**：此方式**无法编译内核模块 kaodv.ko**，只能编译 aodvd 守护进程。

```bash
cd ~/openwrt-build

# 下载 SDK
wget https://downloads.openwrt.org/releases/22.03.4/targets/ath79/nand/openwrt-sdk-22.03.4-ath79-nand_gcc-11.2.0_musl.Linux-x86_64.tar.xz

# 解压
tar -xJf openwrt-sdk-22.03.4-ath79-nand_gcc-11.2.0_musl.Linux-x86_64.tar.xz
mv openwrt-sdk-22.03.4-ath79-nand_gcc-11.2.0_musl.Linux-x86_64 openwrt-sdk

# SDK 中的工具链位置
ls openwrt-sdk/staging_dir/toolchain-mips_24kc_gcc-11.2.0_musl/bin/
```

---

### 常见问题排查

#### 问题 1：menuconfig 看不到 Target System

**原因**：终端窗口太小或 ncurses 库问题

**解决方案**：
```bash
# 确保安装了 ncurses
sudo apt install libncurses-dev libncursesw5-dev

# 设置终端大小
stty rows 40 cols 120

# 或者使用文本配置
make defconfig
```

#### 问题 2：feeds update 失败

**原因**：网络问题或 git 配置

**解决方案**：
```bash
# 使用国内镜像（如果在中国）
sed -i 's/git.openwrt.org/github.com\/openwrt/g' feeds.conf.default

# 或者手动设置代理
export http_proxy=http://your-proxy:port
export https_proxy=http://your-proxy:port
```

#### 问题 3：编译报错

**解决方案**：
```bash
# 清理后重新编译
make clean
make -j1 V=s   # 单线程编译，显示详细信息

# 如果还是失败，完全清理
make distclean
# 然后从 make menuconfig 重新开始
```

#### 问题 4：找不到 GL-AR300M 设备配置

GL-AR300M 有两个版本：
- **GL-AR300M (NOR)** - 使用 NOR flash
- **GL-AR300M-Nand** - 使用 NAND flash

在 menuconfig 中：
```
Target System    → Atheros ATH79
Subtarget        → Generic devices with NAND flash  (如果是 NAND 版本)
                 → Generic                          (如果是 NOR 版本)
Target Profile   → GL.iNet GL-AR300M 或 GL.iNet GL-AR300M (NOR)
```

如果找不到精确型号，选择 "Default Profile" 或 "Multiple devices" 也可以，只要 Target System 和 Subtarget 正确即可。

---

## 三、确认内核源码位置

> **前提**：您已经按照"方式一"完成了 OpenWrt 源码编译。

### 3.1 内核源码目录结构

OpenWrt 完整编译后，内核源码位于 `build_dir` 目录下：

```bash
# 进入 OpenWrt 目录
cd ~/openwrt-build/openwrt

# 查找内核源码目录
find build_dir -name "linux-5.10*" -type d 2>/dev/null

# 典型路径
ls -la build_dir/target-mips_24kc_musl/linux-ath79_nand/
```

**目录结构示例**：
```
~/openwrt-build/openwrt/
├── build_dir/
│   └── target-mips_24kc_musl/
│       └── linux-ath79_nand/
│           ├── linux-5.10.176/          ← 内核源码目录
│           │   ├── include/             ← 内核头文件
│           │   ├── arch/mips/           ← MIPS 架构相关
│           │   └── ...
│           └── ...
├── staging_dir/
│   ├── toolchain-mips_24kc_gcc-11.2.0_musl/
│   │   └── bin/                         ← 交叉编译工具
│   └── target-mips_24kc_musl/
│       └── usr/                         ← 目标系统库文件
└── ...
```

### 3.2 验证内核版本

```bash
# 方法1：查看内核版本头文件
cat ~/openwrt-build/openwrt/build_dir/target-mips_24kc_musl/linux-ath79_nand/linux-*/include/generated/utsrelease.h

# 输出示例：#define UTS_RELEASE "5.10.176"

# 方法2：查看内核配置
cat ~/openwrt-build/openwrt/build_dir/target-mips_24kc_musl/linux-ath79_nand/linux-*/.config | grep "CONFIG_LOCALVERSION"
```

### 3.3 确认内核已正确编译

```bash
# 检查是否存在编译后的内核模块
ls ~/openwrt-build/openwrt/build_dir/target-mips_24kc_musl/linux-ath79_nand/linux-*/modules.builtin

# 检查 Module.symvers（编译内核模块必需）
ls ~/openwrt-build/openwrt/build_dir/target-mips_24kc_musl/linux-ath79_nand/linux-*/Module.symvers
```

如果这些文件不存在，说明内核没有正确编译，需要重新执行 `make -j1 V=s`。

---

## 四、配置交叉编译环境

### 4.1 设置环境变量

在 AODV-UU 源码目录下创建环境配置脚本 `env-openwrt.sh`：

```bash
#!/bin/bash

# ============================================================
# OpenWrt 交叉编译环境配置脚本
# 适用于 GL-AR300M (MIPS 24Kc, OpenWrt 22.03.4)
# ============================================================

# OpenWrt 源码根目录（请根据实际路径修改）
export OPENWRT_ROOT=~/openwrt-build/openwrt

# Staging 目录
export STAGING_DIR=$OPENWRT_ROOT/staging_dir

# 工具链路径
export TOOLCHAIN_DIR=$STAGING_DIR/toolchain-mips_24kc_gcc-11.2.0_musl

# 内核源码路径（请根据实际内核版本修改）
# 使用 ls 命令查找: ls $OPENWRT_ROOT/build_dir/target-mips_24kc_musl/linux-ath79_nand/
export KERNEL_DIR=$OPENWRT_ROOT/build_dir/target-mips_24kc_musl/linux-ath79_nand/linux-5.10.176

# 添加工具链到 PATH
export PATH=$TOOLCHAIN_DIR/bin:$PATH

# 交叉编译器前缀
export CROSS_COMPILE=mips-openwrt-linux-musl-
export ARCH=mips

# 验证配置
echo "============================================="
echo "OpenWrt Cross-Compile Environment"
echo "============================================="
echo "OPENWRT_ROOT:  $OPENWRT_ROOT"
echo "STAGING_DIR:   $STAGING_DIR"
echo "TOOLCHAIN_DIR: $TOOLCHAIN_DIR"
echo "KERNEL_DIR:    $KERNEL_DIR"
echo "CROSS_COMPILE: $CROSS_COMPILE"
echo "============================================="

# 检查关键路径是否存在
if [ ! -d "$TOOLCHAIN_DIR" ]; then
    echo "错误: 工具链目录不存在: $TOOLCHAIN_DIR"
    echo "请确认 OpenWrt 已正确编译"
    return 1
fi

if [ ! -d "$KERNEL_DIR" ]; then
    echo "错误: 内核源码目录不存在: $KERNEL_DIR"
    echo "请检查内核版本号是否正确"
    echo "可用的目录:"
    ls $OPENWRT_ROOT/build_dir/target-mips_24kc_musl/linux-ath79_nand/ 2>/dev/null
    return 1
fi

# 检查交叉编译器
if ! command -v ${CROSS_COMPILE}gcc &> /dev/null; then
    echo "错误: 找不到交叉编译器 ${CROSS_COMPILE}gcc"
    return 1
fi

echo ""
echo "交叉编译器版本:"
${CROSS_COMPILE}gcc --version | head -1
echo ""
echo "环境配置成功！"
```

### 4.2 加载环境变量

```bash
# 保存脚本后，加载环境
source env-openwrt.sh
```

**成功输出示例**：
```
=============================================
OpenWrt Cross-Compile Environment
=============================================
OPENWRT_ROOT:  /home/user/openwrt-build/openwrt
STAGING_DIR:   /home/user/openwrt-build/openwrt/staging_dir
TOOLCHAIN_DIR: /home/user/openwrt-build/openwrt/staging_dir/toolchain-mips_24kc_gcc-11.2.0_musl
KERNEL_DIR:    /home/user/openwrt-build/openwrt/build_dir/target-mips_24kc_musl/linux-ath79_nand/linux-5.10.176
CROSS_COMPILE: mips-openwrt-linux-musl-
=============================================

交叉编译器版本:
mips-openwrt-linux-musl-gcc (OpenWrt GCC 11.2.0 r20123-38ccc47687) 11.2.0

环境配置成功！
```

### 4.3 验证交叉编译器

```bash
# 检查编译器版本
${CROSS_COMPILE}gcc --version

# 检查编译器目标架构
${CROSS_COMPILE}gcc -dumpmachine
# 输出应为: mips-openwrt-linux-musl

# 测试编译一个简单程序
echo 'int main() { return 0; }' > /tmp/test.c
${CROSS_COMPILE}gcc -o /tmp/test /tmp/test.c
file /tmp/test
# 输出应包含: ELF 32-bit MSB executable, MIPS
```

---

## 五、编译 AODV-UU 内核模块

> **前提**：已完成 OpenWrt 源码编译，并正确配置了交叉编译环境。

### 5.1 准备 AODV-UU 源码

```bash
# 假设 AODV-UU 源码在此目录
cd ~/AODV-UU-0.9.6-on-kernel-4.14

# 确保已加载交叉编译环境
source env-openwrt.sh
```

### 5.2 编译内核模块 (kaodv.ko)

**方法 A：使用内核构建系统（推荐）**

```bash
# 进入内核模块源码目录
cd ~/AODV-UU-0.9.6-on-kernel-4.14/aodv-uu-0.9.6/lnx

# 清理旧的编译文件
make clean 2>/dev/null || true

# 使用内核构建系统编译模块
make -C $KERNEL_DIR M=$(pwd) \
    ARCH=mips \
    CROSS_COMPILE=mips-openwrt-linux-musl- \
    modules

# 查看编译结果
ls -la kaodv.ko
file kaodv.ko
```

**方法 B：修改 Makefile 后直接 make**

编辑 `aodv-uu-0.9.6/lnx/Makefile`，在文件开头添加或修改：

```makefile
# 交叉编译设置 - OpenWrt 22.03.4 on GL-AR300M (MIPS 24Kc)
ARCH ?= mips
CROSS_COMPILE ?= mips-openwrt-linux-musl-

# 内核源码目录 - 使用 OpenWrt 源码编译后的内核
# 请根据实际路径修改！
KERNEL_DIR ?= $(HOME)/openwrt-build/openwrt/build_dir/target-mips_24kc_musl/linux-ath79_nand/linux-5.10.176

KERNEL_INC = $(KERNEL_DIR)/include
```

然后编译：
```bash
cd ~/AODV-UU-0.9.6-on-kernel-4.14/aodv-uu-0.9.6/lnx
make clean
make
```

### 5.3 编译用户空间守护进程 (aodvd)

```bash
# 进入主源码目录
cd ~/AODV-UU-0.9.6-on-kernel-4.14/aodv-uu-0.9.6

# 清理旧文件
make clean 2>/dev/null || true

# 编译 aodvd（使用交叉编译器）
make CC=${CROSS_COMPILE}gcc \
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

在 AODV-UU 源码目录下创建 `build.sh`：

```bash
#!/bin/bash
set -e

# ============================================================
# AODV-UU 快速编译脚本
# 适用于 OpenWrt 22.03.4 + GL-AR300M (MIPS 24Kc)
# ============================================================

# ===== 配置区域 - 请根据实际情况修改 =====

# OpenWrt 源码根目录
OPENWRT_ROOT=~/openwrt-build/openwrt

# 内核版本（ls $OPENWRT_ROOT/build_dir/target-mips_24kc_musl/linux-ath79_nand/ 查看）
KERNEL_VERSION=5.10.176

# ===== 以下内容通常不需要修改 =====

STAGING_DIR=$OPENWRT_ROOT/staging_dir
TOOLCHAIN_DIR=$STAGING_DIR/toolchain-mips_24kc_gcc-11.2.0_musl
KERNEL_DIR=$OPENWRT_ROOT/build_dir/target-mips_24kc_musl/linux-ath79_nand/linux-$KERNEL_VERSION

export PATH=$TOOLCHAIN_DIR/bin:$PATH
export STAGING_DIR

ARCH=mips
CROSS_COMPILE=mips-openwrt-linux-musl-

# ============================================================

echo "============================================="
echo "AODV-UU Cross-Compile for OpenWrt"
echo "============================================="
echo "OpenWrt:     $OPENWRT_ROOT"
echo "Kernel:      $KERNEL_DIR"
echo "Toolchain:   $TOOLCHAIN_DIR"
echo "============================================="

# 检查环境
if [ ! -d "$KERNEL_DIR" ]; then
    echo "错误: 内核源码目录不存在!"
    echo "请检查 OPENWRT_ROOT 和 KERNEL_VERSION 设置"
    echo ""
    echo "可用的内核目录:"
    ls $OPENWRT_ROOT/build_dir/target-mips_24kc_musl/linux-ath79_nand/ 2>/dev/null || echo "  (目录不存在)"
    exit 1
fi

if ! command -v ${CROSS_COMPILE}gcc &> /dev/null; then
    echo "错误: 找不到交叉编译器!"
    echo "请确认 OpenWrt 已正确编译"
    exit 1
fi

echo ""
echo "[1/4] 清理旧文件..."
make -C aodv-uu-0.9.6/lnx clean 2>/dev/null || true
make -C aodv-uu-0.9.6 clean 2>/dev/null || true

echo ""
echo "[2/4] 编译内核模块 (kaodv.ko)..."
make -C $KERNEL_DIR M=$(pwd)/aodv-uu-0.9.6/lnx \
    ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE modules

echo ""
echo "[3/4] 编译用户空间程序 (aodvd)..."
cd aodv-uu-0.9.6
make CC=${CROSS_COMPILE}gcc \
     CFLAGS="-I$STAGING_DIR/target-mips_24kc_musl/usr/include" \
     LDFLAGS="-L$STAGING_DIR/target-mips_24kc_musl/usr/lib" \
     aodvd 2>/dev/null || make CC=${CROSS_COMPILE}gcc aodvd
cd ..

echo ""
echo "[4/4] 复制编译结果..."
cp aodv-uu-0.9.6/lnx/kaodv.ko ./ 2>/dev/null || true
cp aodv-uu-0.9.6/aodvd ./ 2>/dev/null || true

echo ""
echo "============================================="
echo "编译完成!"
echo "============================================="
echo ""
echo "编译结果:"

if [ -f "kaodv.ko" ]; then
    echo "  内核模块: kaodv.ko"
    file kaodv.ko
else
    echo "  内核模块: 编译失败!"
fi

if [ -f "aodvd" ]; then
    echo "  守护进程: aodvd"
    file aodvd
else
    echo "  守护进程: 编译失败!"
fi

echo ""
echo "部署命令:"
echo "  scp kaodv.ko aodvd root@192.168.8.1:/tmp/"
```

使用方法：

```bash
# 添加执行权限
chmod +x build.sh

# 编辑脚本，修改 OPENWRT_ROOT 和 KERNEL_VERSION
nano build.sh

# 运行编译
./build.sh
```

---

## 附录 B：完整编译流程总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AODV-UU 交叉编译完整流程                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 准备 Ubuntu/Linux 主机                                          │
│     └── sudo apt install build-essential git ...                   │
│                                                                     │
│  2. 获取 OpenWrt 源码                                               │
│     └── git clone https://git.openwrt.org/openwrt/openwrt.git      │
│     └── git checkout v22.03.4                                      │
│                                                                     │
│  3. 配置目标平台                                                     │
│     └── make menuconfig                                            │
│         ├── Target System: Atheros ATH79                           │
│         ├── Subtarget: Generic devices with NAND flash             │
│         └── Target Profile: GL.iNet GL-AR300M                      │
│                                                                     │
│  4. 编译 OpenWrt（生成工具链和内核源码）                              │
│     └── make -j1 V=s                                               │
│         （需要 1-3 小时，15GB+ 磁盘空间）                            │
│                                                                     │
│  5. 设置交叉编译环境                                                 │
│     └── source env-openwrt.sh                                      │
│                                                                     │
│  6. 编译 AODV-UU                                                    │
│     └── ./build.sh                                                 │
│         ├── kaodv.ko (内核模块)                                     │
│         └── aodvd (用户空间守护进程)                                 │
│                                                                     │
│  7. 部署到设备                                                       │
│     └── scp kaodv.ko aodvd root@192.168.8.1:/tmp/                  │
│                                                                     │
│  8. 在设备上运行                                                     │
│     └── insmod kaodv.ko ifname=wlan0                               │
│     └── ./aodvd -i wlan0                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
