---
layout: post
title: 在x86主机上构建并运行arm架构的镜像
categories: [编程, docker]
tags: [x86, arm]
---

> 如何在x86主机上打出arm架构的镜像

#### 0. 概念

* `CPU架构`: 不同厂商生产的CPU规范，主要分为复杂指令集（例如Intel和AMD的x86架构）、精简指令集（例如各种手机CPU的ARM架构）。所以才有了x86程序和ARM程序的说法，即x86程序只能用x86架构的CPU执行，而ARM程序只能用ARM架构的CPU执行。
* `QEMU`: 一个开源的托管虚拟机，通过纯软件来实现虚拟化模拟器，几乎可以模拟任何硬件设备，它能在x86主机上运行arm程序 
* `binfmt_misc`: `Miscellaneous Binary Format`，可以通过要打开文件的特性(比如扩展名或文件的特殊字节标识)来选择使用哪个程序来打开
* `qemu-user-static`: 是一些`qemu`的静态二进制文件集合，我们可以通过`binfmt_misc`把它注册为指定程序的解释器

#### 1. arm基础镜像

使用`arm64v8/ubuntu`做测试

```
$ docker run --rm -t arm64v8/ubuntu uname -m
Unable to find image 'arm64v8/ubuntu:latest' locally
latest: Pulling from arm64v8/ubuntu
80bc30679ac1: Pull complete 
c937c19c2d76: Pull complete 
ba4ad2754376: Pull complete 
Digest: sha256:f796dba8ac91e7995df05e0184761061581953152a6107c0e0e0895e6bb44893
Status: Downloaded newer image for arm64v8/ubuntu:latest
standard_init_linux.go:207: exec user process caused "exec format error"
```

> 在x86下直接运行arm镜像会报错

#### 2. 下载并安装qemu-aarch64-static

```
$ wget https://github.com/multiarch/qemu-user-static/releases/download/v5.2.0-2/x86_64_qemu-aarch64-static.tar.gz
$ tar zxvf x86_64_qemu-aarch64-static.tar.gz -C /usr/bin/
qemu-aarch64-static
$ chmod 755 /usr/bin/qemu-aarch64-static 
```

#### 3. 注册binfmt_misc

`multiarch/qemu-user-static`容器实际是为`arm`程序注册了`binfmt_misc`，目的是自动使用`qemu-aarch64-static`执行`arm`程序

```
$ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
Unable to find image 'multiarch/qemu-user-static:latest' locally
latest: Pulling from multiarch/qemu-user-static
d60bca25ef07: Pull complete 
7045fbe28d35: Pull complete 
ed8a5179ae11: Pull complete 
1ec39da9c97d: Pull complete 
0421b3622b1a: Pull complete 
Digest: sha256:14ef836763dd8a1d69927699811f89338b129faa3bd9eb52cd696bc3d84aa81a
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
```

> 低版本的`Linux`内核可能会注册失败，本文使用`3.10.x`失败了，使用`4.16.x`成功

#### 4. 测试

```
$ docker run --rm -t arm64v8/ubuntu uname -m
aarch64
```

运行成功了，打一个arm镜像试试

```
$ docker build -t arm64v8_ubuntu_jdk8:1.0 .

```

#### 6.参考

* [qemu-user-static](https://github.com/multiarch/qemu-user-static)
* [Developers guide](https://github.com/multiarch/qemu-user-static/blob/master/docs/developers_guide.md)
* [QEMU](https://www.qemu.org/)
* [binfmt_misc](https://www.kernel.org/doc/html/latest/admin-guide/binfmt-misc.html)
