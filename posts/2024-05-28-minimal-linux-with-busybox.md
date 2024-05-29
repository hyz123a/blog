---
title: 编译一个 AArch64 平台的最小 Linux 内核
categories: Dev
tags: [Linux, OS, 内核, BusyBox, QEMU, ARM, AArch64]
created: 2024-5-28 14:29:00
---

总结一下最近折腾的事情，方便以后查阅。

> 所有内容都假设已经安装了必须的构建工具链，如果没有装，可以在报错的时候再根据提示安装。

## 创建 Staging Directory

```bash
mkdir rootfs
```

接着创建一些空目录：

```bash
mkdir -pv {etc,proc,sys,usr/{bin,sbin}}
```

## 编译 BusyBox

需要先编译一个 BusyBox 作准备

在 [这里](https://busybox.net/downloads/) 下载适当版本的 BusyBox 源码并解压，然后运行：

```bash
cd busybox-1.32.0
mkdir build

make ARCH=arm64 defconfig # O=build
make ARCH=arm64 menuconfig
```

busybox 也使用 Kuild 进行构建，可以开启一个配置菜单。在「Settings」里面修改下面几项配置：

```
# 减少文件大小
[*] Don't use /usr
[*] Build static binary (no shared libs)
(aarch64-linux-gnu-) Cross compiler prefix

Settings  ---> 
  (../rootfs) Destination path for 'make install'
```

然后保存并退出。运行：

```bash
make # -j16
make install
cd ..
```

这会使用刚刚保存的配置进行编译，然后安装到 `rootfs` 目录，此时该目录如下：

```bash
$ tree -L 1 .
.
├── bin
├── etc
├── init
├── linuxrc -> bin/busybox
├── proc
├── sbin
├── sys
└── usr

6 directories, 2 files
```

然后在sbin目录下创建一个 `init` 文件，内容如下：

```bash
#!/bin/sh

# 为了使用 cat /proc/** /sys/**
mount -t proc none /proc # 这里的none表示这是一个虚拟的文件系统不对应任何块设备文件系统类型
mount -t sysfs none /sys

echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"

exec /bin/sh
```

修改 `init` 文件为可执行：

```bash
chmod +x init
```

set SUID bit, 可执行文件的 Permission 为 4xxx

``` bash
sudo chmod u+s bin linuxrc sbin usr` 
```

The `chmod u+s` command sets the SUID (Set User ID) bit on the specified files. When the SUID bit is set on an executable file, it allows users to execute the file with the permissions of the file owner, rather than with the permissions of the user who is running the file. (from gpt-4o)


把这些目录和文件打包：

```bash
cd rootfs
find . | cpio -H newc -ov --owner root:root > ../initramfs.cpio
cd ..
gzip initramfs.cpio
```

生成的 gzip 压缩后的 cpio 映像放在了 `build/initramfs.cpio.gz`，此时 BusyBox **ramdisk(initramfs)** 就做好了，保存备用。

## 编译最小配置的 Linux 内核

kernel官方手册也展示了如何使用 [llvm](https://docs.kernel.org/6.1/kbuild/llvm.html) 来进行交叉编译内核，这里我选择用clang来编译。(为了方便使用clangd来索引代码)

在 [这里](https://www.kernel.org/) 下载适当版本的内核源码并解压，然后运行：

```bash
cd linux-5.8.8
make ARCH=arm64 distclean # O=build 清除效果最强
mkdir build
make ARCH=arm64 LLVM=1 allnoconfig
make ARCH=arm64 LLVM=1 menuconfig
```

这会首先初始化一个最小的配置（`allnoconfig`），然后打开配置菜单。在配置菜单中做以下修改：

```
-> General setup
[*] Initial RAM filesystem and RAM disk (initramfs/initrd) support

-> General setup
  -> Configure standard kernel features
[*] Enable support for printk

-> Executable file formats / Emulations
[*] Kernel support for ELF binaries
[*] Kernel support for scripts starting with #!

-> Device Drivers
  -> Generic Driver Options
[*] Maintain a devtmpfs filesystem to mount at /dev
[*]   Automount devtmpfs at /dev, after the kernel mounted the rootfs

-> Device Drivers
  -> Character devices
[*] Enable TTY

-> Device Drivers
  -> Character devices
    -> Serial drivers
[*] ARM AMBA PL010 serial port support
[*]   Support for console on AMBA serial port
[*] ARM AMBA PL011 serial port support
[*]   Support for console on AMBA serial port

-> File systems
  -> Pseudo filesystems
[*] /proc file system support
[*] sysfs file system support
```

完成后保存并退出，再运行：

```bash
make O=build ARCH=arm64 LLVM=1 # -j16
```

即可编译 Linux 内核，编译出来的两个东西比较有用，一个是 `build/vmlinux`，另一个是 `build/arch/arm64/boot/Image`，前者是 ELF 格式的内核，可以用来在 GDB 中加载调试信息，后者是可启动的内核映像文件。

## 编译 qemu-system-aarch64

这一步是可选的，直接使用包管理器安装 QEMU 也可以。

在 [这里](https://download.qemu.org/) 下载适当版本的 QEMU 源码并解压，然后运行：

```bash
cd qemu-5.0.0

mkdir build
cd build

../configure --target-list=aarch64-softmmu
make # -j8
```

即可编译 AArch64 目标架构的 QEMU。

## 启动 Linux

为了清晰起见，回到上面三个源码目录的外层，即当前目录中内容如下：

```bash
$ tree -L 1 .
.
├── busybox-1.32.0
├── linux-5.8.8
└── qemu-5.0.0

3 directories, 0 files
```

然后使用 QEMU 启动刚刚编译的 Linux：

```bash
#!/bin/bash

qemu-system-aarch64 \
-machine virt \
-cpu cortex-a53 \
-smp 1 \
-m size=1G \
-nographic \
-kernel ./linux-5.15.160/arch/arm64/boot/Image \
-append "console=ttyAMA0 rdinit=/sbin/init" \
-initrd initramfs.cpio.gz \
# -s -S \
```

这里使用了 QEMU 的 [virt](https://www.qemu.org/docs/master/system/arm/virt.html) 平台。

## 如何启用 Kernel Module

> 因为是构建最小 linux，所以把加载内核模块给关闭了，如要打开，可以在 .config 文件中加上以下内容：
```
 CONFIG_MODULES=y
 CONFIG_MODULE_FORCE_LOAD=y
 CONFIG_MODULE_UNLOAD=y
 CONFIG_MODULE_FORCE_UNLOAD=y
```

```bash
make ARCH=arm64 LLVM=1 INSTALL_MOD_PATH=../rootfs/ modules_install 
# install 模块到 stagding directory
```  

在之前的init启动脚本中添加

```bash
mount -t devtmpfs devtmpfs /dev
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
```

来自动管理设备。

> Although both mdev and udev create the device nodes themselves, it is easier to just let devtmpfs do the job and use mdev/udev as a layer on top to implement the policy for setting ownership and permissions. The devtmpfs approach is the only maintainable way to generate device nodes prior to user space startup.


## 参考资料

- [《Mastering Embedded Linux Programming Third Edition》](https://www.amazon.com/Mastering-Embedded-Linux-Programming-potential/dp/1789530385)
- [How to Build A Custom Linux Kernel For Qemu](https://mgalgs.github.io/2015/05/16/how-to-build-a-custom-linux-kernel-for-qemu-2015-edition.html)
- [Busybox构建根文件系统和制作Ramdisk](https://www.cnblogs.com/lotgu/p/7020418.html)
- [Building a minimal AArch64 root filesystem for network booting](http://wiki.loverpi.com/faq:sbc:libre-aml-s805x-minimal-rootfs)
- [Build and run minimal Linux / Busybox systems in Qemu](https://gist.github.com/chrisdone/02e165a0004be33734ac2334f215380e)
- [Download QEMU](https://www.qemu.org/download/#source)