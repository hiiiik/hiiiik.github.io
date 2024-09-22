---
title: macOS交叉编译Raspberry Pi OS Linux内核
date: 2024-09-22
tags: [linux, kernel, raspberry-pi, macOS]
---

## 1 环境信息
- 开发板：Raspberry Pi 5 \| Raspberry Pi OS Lite x64
- 宿主机：MacBook Air M3 \| macOS 14.3

## 2 编译内核
### 2.1 准备交叉编译工具链
*macos-cross-toolchains*<sup id="ref1-inc">[[1]](#ref1)</sup>这个开源项目提供了预编译好的工具链，可以通过*homebrew*直接下载使用：
```shell
$ brew tap messense/macos-cross-toolchains
$ brew install aarch64-unknown-linux-gnu
```

### 2.2 准备文件系统
*Linux*内核通常只能在大小写敏感的文件系统上编译，但*macOS*默认的文件系统是不区分大小写的，因此需要提前准备好一个大小写敏感的存储空间用于编译内核。

这里使用自带的命令行工具创建一个容量为40GB、大小写敏感的*APFS*文件系统、占用空间动态增长的磁盘映像：
```shell
$ hdiutil create -size 40g -fs "Case-sensitive APFS" -type SPARSE -volname build_raspberrypi_kernel build_raspberrypi_kernel
```

挂载刚刚创建的磁盘映像：
```shell
$ export KERNEL_BUILD_PARTITION_CS_APFS=/path/to/mountpoint/dir
$ hdiutil attach build_raspberrypi_kernel.sparseimage -mountpoint $KERNEL_BUILD_PARTITION_CS_APFS
```
与内核构建相关的操作都将在`$KERNEL_BUILD_PARTITION_CS_APFS`目录下进行。

> 后续可使用如下命令卸载这个分区
>
> $ **hdiutil detach $KERNEL_BUILD_PARTITION_CS_APFS**


### 2.3 下载内核源码
这里直接浅克隆代码仓，以减少等待时间：
```shell
$ cd $KERNEL_BUILD_PARTITION_CS_APFS
$ git clone --depth=1 https://github.com/raspberrypi/linux
```

### 2.4 构建内核配置
不做额外调整，直接构建默认配置。根据树莓派型号和系统架构的不同，*make*目标亦有区别，树莓派官方文档有详细说明<sup id="ref2-inc">[[2]](#ref2)</sup>。
```shell
$ cd linux
$ KERNEL=kernel_2712
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2712_defconfig
```

构建完成后，内核源码目录下将会产生`.config`配置文件。修改`.config`文件中的**CONFIG_LOCALVERSION**配置项，可以自定义内核版本号。这一步不是必须的，但可以避免覆盖既有的同版本号内核文件，方便区分新旧内核。
```
CONFIG_LOCALVERSION="-v8-16k-ik"
```

### 2.5 编译内核源码
开始编译内核：
```shell
$ make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
```
上述命令需要执行较长时间，且可能会出现多种编译报错，具体的问题和解决方案见下节汇总。

### 2.6 编译问题汇总

#### 2.6.1 make版本过低
##### 错误信息
```
Makefile:15: *** GNU Make >= 3.82 is required. Your Make version is 3.81.  Stop.
```

##### 问题根因
*macOS*自带的`make`版本通常不高，可能无法达到构建内核所要求的版本。

##### 解决方案
升级*make*版本即可。直接通过*homebrew*安装：
```shell
$ brew install make
```

调整**PATH**环境变量，确保当前优先使用非系统自带的*make*：
```shell
$ export PATH="$(brew --prefix)/opt/make/libexec/gnubin:$PATH"
```

#### 2.6.2 找不到elf.h
##### 错误信息
```shell
  HOSTCC  scripts/sorttable
scripts/sorttable.c:27:10: fatal error: 'elf.h' file not found
#include <elf.h>
         ^~~~~~~
```

##### 问题根因
*macOS*标准库没有`elf.h`头文件，该头文件中定义有ELF相关的类型、结构体和宏。

##### 解决方案
从*Glibc*代码仓下载`elf.h`头文件<sup id="ref5-inc">[[5]](#ref5)</sup>，并将其置于`/usr/local/include/`目录下：
```shell
$ wget https://github.com/bminor/glibc/raw/refs/heads/master/elf/elf.h -O /usr/local/include/elf.h
```

#### 2.6.3 uuid_t重定义
##### 错误信息
```
  HOSTCC  scripts/mod/file2alias.o
scripts/mod/file2alias.c:45:3: error: typedef redefinition with different types ('struct uuid_t' vs '__darwin_uuid_t' (aka 'unsigned char[16]'))
} uuid_t;
  ^
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/sys/_types/_uuid_t.h:31:25: note: previous definition is here
typedef __darwin_uuid_t uuid_t;
                        ^
```

##### 问题根因
内核中定义了一个`uuid_t`结构体，但是*macOS*的SDK头文件中也定义了同名的类型，与内核的实现有细微的区别，导致出现重定义错误。

该结构体的定义位于`scripts/mod/file2alias.c`。相关的改动在内核5.1版本引入<sup id="ref3-inc">[[3]](#ref3)</sup>，用于支持TEE设备的总线驱动框架，其使用uuid来识别设备和服务。
```c
typedef struct {
	__u8 b[16];
} uuid_t;
```

而在*macOS*的SDK头文件中也存在类似的定义：
```c
typedef __darwin_uuid_t uuid_t;
typedef unsigned char   __darwin_uuid_t[16];
```

##### 解决方案
在内核代码定义`uuid_t`结构体之前，使用宏替换掉冲突的名称即可<sup id="ref4-inc">[[4]](#ref4)</sup>：
```c
// 必须置于uuid结构体定义之前
#ifdef __APPLE__
    #define uuid_t compat_uuid_t
#endif

typedef struct {
    __u8 b[16];
} uuid_t;
```

#### 2.6.4 找不到endian.h

##### 错误信息
```shell
  HOSTCC  arch/arm64/kvm/hyp/nvhe/gen-hyprel
arch/arm64/kvm/hyp/nvhe/gen-hyprel.c:28:10: fatal error: 'endian.h' file not found
#include <endian.h>
         ^~~~~~~~~~
```

##### 问题根因
*macOS*标准库没有可用的`endian.h`头文件，该文件用于处理字节序转换。

##### 解决方案
下载`endian.h`头文件<sup id="ref6-inc">[[6]](#ref6)</sup>，并将其置于`/usr/local/include/`目录下：
```shell
$ wget https://gist.github.com/panzi/6856583/raw/12f9f02f1298bb0bc054ba667bccc0cf032cdb03/portable_endian.h -O /usr/local/include/endian.h
```

#### 2.6.5 sed调用异常

##### 错误信息
```shell
  GEN     security/apparmor/capability_names.h
sed: 1: "include/uapi/linux/capa ...": command i expects \ followed by text
```
##### 问题根因
*macOS*自带的*sed*工具在语法上和内核构建使用的*GNU sed*有些区别，可能会造成兼容性问题。

##### 解决方案
直接通过*homebrew*安装GNU版本的*sed*：
```shell
$ brew install gsed
```

调整**PATH**环境变量，确保当前优先使用非系统自带的*sed*：
```shell
$ export PATH="$(brew --prefix)/opt/gnu-sed/libexec/gnubin:$PATH"
```

#### 2.6.6 找不到linux/version.h
##### 错误信息
```shell
  HOSTCC  drivers/accessibility/speakup/makemapdata.o
drivers/accessibility/speakup/makemapdata.c:13:10: fatal error: 'linux/version.h' file not found
#include <linux/version.h>
         ^~~~~~~~~~~~~~~~~
```
##### 问题根因
内核构建配置默认会启用*SPEAKUP*模块，该模块提供无障碍辅助功能，可以帮助视障用户使用计算机（例如屏幕内容朗读）。

该模块的生成代码包含了`linux/version.h`这个头文件，显然*macOS*上不会有这个头文件。

但实际上，对应的源码并没有使用到该头文件中的任何宏，所以在内核6.11之后的版本去除了冗余的包含代码<sup id="ref7-inc">[[7]](#ref7)</sup>。

待后续树莓派内核代码与上游版本同步，应该就不会再出现这个问题了。

##### 解决方案
可以直接在内核配置`.config`中取消启用该模块：
```
#
# Speakup console speech
#
# 将下述内容全部注释
# CONFIG_SPEAKUP=m
# CONFIG_SPEAKUP_SYNTH_ACNTSA is not set
# CONFIG_SPEAKUP_SYNTH_APOLLO is not set
# CONFIG_SPEAKUP_SYNTH_AUDPTR is not set
# CONFIG_SPEAKUP_SYNTH_BNS is not set
# CONFIG_SPEAKUP_SYNTH_DECTLK is not set
# CONFIG_SPEAKUP_SYNTH_DECEXT is not set
# CONFIG_SPEAKUP_SYNTH_LTLK is not set
# CONFIG_SPEAKUP_SYNTH_SOFT=m
# CONFIG_SPEAKUP_SYNTH_SPKOUT is not set
# CONFIG_SPEAKUP_SYNTH_TXPRT is not set
# CONFIG_SPEAKUP_SYNTH_DUMMY is not set
# end of Speakup console speech

```

或者删除*SPEAKUP*模块代码中的`#include <linux/version.h>`，涉及到的文件有：
```
drivers/accessibility/speakup/makemapdata.c
drivers/accessibility/speakup/genmap.c
```

## 3 安装内核

### 3.1 准备ext4挂载工具
*macOS*上无法直接挂载*ext4*文件系统，需要借助额外的工具，可以使用任意支持挂载*ext4*文件系统的工具（需支持读写）。

这里使用**fuse-ext2**，开启一个新的终端进行编译安装。首先安装依赖：
```shell
$ brew install --cask macfuse
$ brew install m4
$ brew install autoconf
$ brew install libtool
$ brew install e2fsprogs
```

下载源码并配置依赖环境变量：
```shell
$ git clone https://github.com/alperakcan/fuse-ext2.git
$ export PATH="$(brew --prefix)/opt/m4/bin:$PATH"
$ export PATH="$(brew --prefix)/opt/libtool/libexec/gnubin:$PATH"
$ export PKG_CONFIG_PATH="$(brew --prefix)/opt/e2fsprogs/lib/pkgconfig"
$ export LDFLAGS="-L$(brew --prefix)/opt/e2fsprogs/lib"
$ export CFLAGS="-I$(brew --prefix)/opt/e2fsprogs/include -Wno-incompatible-function-pointer-types"
```

如果只安装了*Xcode*命令工具，而没有安装完整的*Xcode*环境，则需要手动注释`tools/macosx/Makefile.am`文件中的所有内容，以避免编译时出现类似这样的报错：
```shell
Making all in macosx
( cd ../../tools/macosx/prefpane; \
	  xcodebuild; \
	)
xcode-select: error: tool 'xcodebuild' requires Xcode, but active developer directory '/Library/Developer/CommandLineTools' is a command line tools instance
```

编译安装：
```shell
$ cd fuse-ext2
$ ./autogen.sh
$ ./configure --prefix="/absolute/dir/for/fuse-ext2/installation/"
$ make -j$(nproc)
$ make install
```

可以建立软链接方便使用：
```shell
$ sudo ln -s /path/to/fuse-ext2/installation/bin/fuse-ext2 /usr/local/bin
```

### 3.2 挂载内核镜像
插入SD卡，执行下述命令找到SD卡对应的设备：
```shell
$ diskutil list
```

通常会有类似这样的输出：
```shell
/dev/disk8 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *63.9 GB    disk8
   1:             Windows_FAT_32 bootfs                  536.9 MB   disk8s1
   2:                      Linux                         63.3 GB    disk8s2
```
其中，`/dev/disk8s1`是内核启动引导分区（通常是*FAT32*格式），`/dev/disk8s2`是内核根文件系统分区（通常是*ext4*格式）。
根据实际情况导出对应的环境变量：
```shell
export KERNEL_PART_DEV=/dev/disk8
export KERNEL_BOOT_DEV=/dev/disk8s1
export KERNEL_ROOT_DEV=/dev/disk8s2
```

*macOS*通常会自动挂载*FAT32*格式的分区到`/Volumes/`目录下，因此需要先进行卸载。
执行如下命令进行检查：
```shell
$ mount
```

如果看到类似这样的输出：
```shell
/dev/disk8s1 on /Volumes/bootfs (msdos, local, nodev, nosuid, noowners, noatime)
```

则需要先进行卸载操作：
```shell
$ sudo diskutil unmount /Volumes/bootfs
```

接着就可以在内核源码根目录先创建好对应的挂载目录：
```shell
$ mkdir mnt
$ mkdir mnt/boot
$ mkdir mnt/root
```

执行挂载操作：
```shell
$ sudo mount -t msdos $KERNEL_BOOT_DEV mnt/boot
$ sudo fuse-ext2 $KERNEL_ROOT_DEV mnt/root -o rw+
```

### 3.3 执行安装

安装内核模块：
```shell
$ sudo env PATH=$PATH make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=mnt/root modules_install
```


安装内核和设备树：
```shell
$ sudo cp mnt/boot/$KERNEL.img mnt/boot/$KERNEL-backup.img
$ sudo cp arch/arm64/boot/Image mnt/boot/$KERNEL.img
$ sudo cp arch/arm64/boot/dts/broadcom/*.dtb mnt/boot/
$ sudo cp arch/arm64/boot/dts/overlays/*.dtb* mnt/boot/overlays/
$ sudo cp arch/arm64/boot/dts/overlays/README mnt/boot/overlays/
```

安装内核符号表，后续用于生成内核模块依赖关系和导出符号：
```shell
$ sudo mv mnt/root/boot/System.map mnt/root/boot/System.map.old
$ sudo cp System.map mnt/root/boot/System.map
```

当从*macOS*复制文件到外部卷时，会自动创建前缀为**\._**的同名文件(*Apple Double Format*)，这些文件用于记录*macOS*独有的附加信息，对于内核构建使用没有任何意义，反而可能导致生成内核模块依赖信息时出错，因此需要先清理干净：
```shell
$ sudo find mnt -type file -size 4k -name '._*' -delete
```

安装完成，卸载分区并弹出SD卡：
```
$ sudo diskutil unmount mnt/boot
$ sudo umount mnt/root
$ sudo diskutil eject $KERNEL_PART_DEV
```

此时，可以将SD卡重新装载到树莓派上并启动系统。

### 3.4 生成内核模块依赖文件
在前面安装内核时，可能会出现类似的警告信息：
```shell
  DEPMOD  mnt/root/lib/modules/6.6.51-v8-16k-ik+
Warning: 'make modules_install' requires depmod. Please install it.
This is probably in the kmod package.
```
这是因为*macOS*上没有**depmod**工具（属于**kmod**套件），无法分析内核模块的依赖信息和符号信息，可能会导致部分内核模块无法正常加载（例如无线网卡驱动）。

因此，在新的内核安装后，可以连接到树莓派，利用其自带的**depmod**重新处理内核模块的依赖关系：
```shell
$ sudo /sbin/depmod -ae -F /boot/System.map
```

重启系统使依赖文件生效：
```shell
$ sudo /sbin/reboot
```

重启后，可见新的内核版本已经生效：
```shell
$ uname -a
Linux raspberryki 6.6.51-v8-16k-ik+ #1 SMP PREEMPT Sat Sep 21 17:12:01 CST 2024 aarch64 GNU/Linux
```

至此，内核的编译、安装全部完成。

## 4 内核模块测试

既然已经成功编译了内核，不妨顺便用内核树跑个简单的内核模块hello world一下：

**Makefile：**
```makefile
obj-m := hello_world.o

KERNEL_DIR     ?= ~/dev/kernel/raspberrypi/build_raspberrypi_kernel/linux/
TARGET_ARCH 	= arm64
COMPILER_PREFIX = aarch64-linux-gnu-
PWD      		= $(shell pwd)

all:
	make -C ${KERNEL_DIR} M=$(PWD) ARCH=${TARGET_ARCH} CROSS_COMPILE=${COMPILER_PREFIX} modules

clean:
	make -C ${KERNEL_DIR} M=$(PWD) ARCH=${TARGET_ARCH} CROSS_COMPILE=${COMPILER_PREFIX} clean
```

**hello_world.c：**
```c
#include <linux/init.h>
#include <linux/module.h>

MODULE_LICENSE("Dual BSD/GPL");

static int __init hello_init(void)
{
	printk(KERN_INFO "hello world\n");
	return 0;
}

static void __exit hello_exit(void)
{
	printk(KERN_INFO "goodbye world\n");
}

module_init(hello_init);
module_exit(hello_exit);
```

将编译后的内核模块复制到树莓派上，然后装载/卸载模块，即可在内核日志看到相关的打印信息：
```shell
$ sudo /sbin/insmod hello_world.ko
$ sudo /sbin/rmmod hello_world
$ dmesg | tail -3
[ 1773.361977] hello_world: loading out-of-tree module taints kernel.
[ 1773.362134] hello world
[ 1777.345510] goodbye world
```

## 参考

<small id="ref1"><sup>[[1]](#ref1-inc)</sup> [macOS cross compiler toolchains](https://github.com/messense/homebrew-macos-cross-toolchains)</small>

<small id="ref2"><sup>[[2]](#ref2-inc)</sup> [Linux kernel cross compiled build configuration - Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/computers/linux_kernel.html#cross-compiled-build-configuration)</small>

<small id="ref3"><sup>[[3]](#ref3-inc)</sup> [tee: add bus driver framework for TEE based devices](https://github.com/torvalds/linux/commit/0fc1db9d105915021260eb241661b8e96f5c0f1a)</small>

<small id="ref4"><sup>[[4]](#ref4-inc)</sup> [kernel: 5.4 fix build on darwin](https://git.openwrt.org/?p=openwrt/openwrt.git;a=commitdiff;h=0b7ad6f7f061e0cd7a3f267b23d411cc2bd44e00)</small>

<small id="ref5"><sup>[[5]](#ref5-inc)</sup> [The GNU C Library](https://github.com/bminor/glibc)</small>

<small id="ref6"><sup>[[6]](#ref6-inc)</sup> [Endian conversion functions form endian.h](https://gist.github.com/panzi/6856583)</small>

<small id="ref7"><sup>[[7]](#ref7-inc)</sup> [treewide: remove unnecessary <linux/version.h> inclusion](https://github.com/torvalds/linux/commit/e332a5aba83500e8d422c90d2a84d8a5f888673e)</small>
