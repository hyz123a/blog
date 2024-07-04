---
title: 使用docker搭建交叉编译环境
categories: Dev
tags: [Linux, QEMU, ARM, AArch64, Docker, Cross-compile]
created: 2024-7-1 18:18:00
---

分享一种没有开发板的 SDK 时，一种可行的交叉编译方法。

## Docker 设置代理

创建一个 Docker 配置文件（如果不存在），并在其中添加代理设置。例如，如果使用的是 Linux 操作系统，则配置文件通常位于 /etc/docker/ 目录下，可以使用以下命令创建它：

```shell
sudo mkdir /etc/systemd/system/docker.service.d/ 
sudo vim /etc/systemd/system/docker.service.d/http-proxy.conf
```

添加以下内容：

```
[Service]
Environment="HTTP_PROXY=http://your_proxy_server:proxy_port"
Environment="HTTPS_PROXY=http://your_proxy_server:proxy_port"
Environment="NO_PROXY=localhost,127.0.0.1"
```

请注意，将 "your_proxy_server" 和 "proxy_port" 替换为实际的代理服务器和端口号。

保存并关闭文件后，重新加载 Docker 守护程序以使更改生效。使用以下命令重启 Docker：

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

验证代理设置是否生效。您可以使用以下命令来检查 Docker 是否正在使用代理：

docker info | grep -i proxy
如果代理设置正确，则应该看到类似以下内容的输出：

```
HTTP Proxy: http://your_proxy_server:proxy_port HTTPS Proxy: http://your_proxy_server:proxy_port
```

##  利用 qemu 运行非本机架构的容器

在 AMD64 (x86_64) 架构的主机上运行 ARM64 架构的容器：

1. 确保已安装 QEMU：

```shell
sudo apt-get install qemu-user-static
```

2. 启用 QEMU 模拟：

```shell
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

输出：

```
Unable to find image 'multiarch/qemu-user-static:latest' locally
latest: Pulling from multiarch/qemu-user-static
205dae5015e7: Pull complete
816739e52091: Pull complete
30abb83a18eb: Pull complete
0657daef200b: Pull complete
30c9c93f40b9: Pull complete
Digest: sha256:fe60359c92e86a43cc87b3d906006245f77bfc0565676b80004cc666e4feb9f0
Status: Downloaded newer image for multiarch/qemu-user-static:latest
Setting /usr/bin/qemu-alpha-static as binfmt interpreter for alpha
Setting /usr/bin/qemu-arm-static as binfmt interpreter for arm
Setting /usr/bin/qemu-armeb-static as binfmt interpreter for armeb
Setting /usr/bin/qemu-sparc-static as binfmt interpreter for sparc
Setting /usr/bin/qemu-sparc32plus-static as binfmt interpreter for sparc32plus
Setting /usr/bin/qemu-sparc64-static as binfmt interpreter for sparc64
Setting /usr/bin/qemu-ppc-static as binfmt interpreter for ppc
Setting /usr/bin/qemu-ppc64-static as binfmt interpreter for ppc64
Setting /usr/bin/qemu-ppc64le-static as binfmt interpreter for ppc64le
Setting /usr/bin/qemu-m68k-static as binfmt interpreter for m68k
Setting /usr/bin/qemu-mips-static as binfmt interpreter for mips
Setting /usr/bin/qemu-mipsel-static as binfmt interpreter for mipsel
Setting /usr/bin/qemu-mipsn32-static as binfmt interpreter for mipsn32
Setting /usr/bin/qemu-mipsn32el-static as binfmt interpreter for mipsn32el
Setting /usr/bin/qemu-mips64-static as binfmt interpreter for mips64
Setting /usr/bin/qemu-mips64el-static as binfmt interpreter for mips64el
Setting /usr/bin/qemu-sh4-static as binfmt interpreter for sh4
Setting /usr/bin/qemu-sh4eb-static as binfmt interpreter for sh4eb
Setting /usr/bin/qemu-s390x-static as binfmt interpreter for s390x
Setting /usr/bin/qemu-aarch64-static as binfmt interpreter for aarch64
Setting /usr/bin/qemu-aarch64_be-static as binfmt interpreter for aarch64_be
Setting /usr/bin/qemu-hppa-static as binfmt interpreter for hppa
Setting /usr/bin/qemu-riscv32-static as binfmt interpreter for riscv32
Setting /usr/bin/qemu-riscv64-static as binfmt interpreter for riscv64
Setting /usr/bin/qemu-xtensa-static as binfmt interpreter for xtensa
Setting /usr/bin/qemu-xtensaeb-static as binfmt interpreter for xtensaeb
Setting /usr/bin/qemu-microblaze-static as binfmt interpreter for microblaze
Setting /usr/bin/qemu-microblazeel-static as binfmt interpreter for microblazeel
Setting /usr/bin/qemu-or1k-static as binfmt interpreter for or1k
Setting /usr/bin/qemu-hexagon-static as binfmt interpreter for hexagon
```

3. 使用 `--platform` 标志明确指定平台：

```
sudo docker run --platform linux/arm64/v8 -it arm64v8/ubuntu:22.04 # 注意宿主机的版本一致
这里当然也可以直接在 docker 容器中编译，则不需要与宿主机版本一致
```

进入 docker 容器中。

## 使用 docker 获得 交叉编译环境所需要的 sysroot

在 docker 中：

```shell
apt update
apt install clang cmake build-essential \
exit
```

这里可以添加自己需要的库。

将 sysroot 从容器复制到宿主机：

```shell
$ sudo docker ps -a
CONTAINER ID   IMAGE                 COMMAND   CREATED       STATUS                   PORTS     NAMES
f367a6b0316f   arm64v8/ubuntu:22.04  "bash"    4 hours ago   Exited (0) 4 hours ago   interesting_knuth
$ mkdir ubuntu22-arm64-sysroot
$ sudo docker cp f367a6b0316f:/ ubuntu22-arm64-sysroot
```

## 使用 CMake 交叉编译

cross-toolchain-aarch64-template.cmake

```Cmake
cmake_minimum_required(VERSION 3.10)

set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

# Specify the sysroot.
# You need to modify <path_to_user_target_sysroot> appropriately for your environment.
set(target_sysroot <Absolute_path_to>/ubuntu22-arm64-sysroot)
set(CMAKE_SYSROOT ${target_sysroot})

# Specify the cross compiler.
set(triple aarch64-linux-gnu)
set(CMAKE_C_COMPILER_TARGET ${triple})
set(CMAKE_C_COMPILER clang)
set(CMAKE_CXX_COMPILER_TARGET ${triple})
set(CMAKE_CXX_COMPILER clang++)

set(CMAKE_FIND_ROOT_PATH ${target_sysroot})
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

编译时，指定 toolchain
```shell
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=./cmake/cross-toolchain-aarch64-template.cmake # 设置你的 toolchain file 所在地址
cmake --build build --parallel `nproc`
```
