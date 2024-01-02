---
layout: post
title: vnc连接docker容器桌面
categories: [linux]
tags: [nginx]
---

> 我们常用docker来运行linux容器，那么怎样才能进入容器桌面呢？

#### 1. 拉取镜像

Github上有很多可用的镜像，例如：[ubuntu vnc desktop](https://github.com/search?q=ubuntu+vnc++desktop&type=repositories&s=stars&o=desc)

本文使用[Headless Ubuntu/Xfce containers with VNC/noVNC](https://github.com/accetto/ubuntu-vnc-xfce-g3)

```
# 直接拉取latest镜像
$ docker pull accetto/ubuntu-vnc-xfce-chromium-g3
#运行容器
$ docker run --rm -it 2d3cd75e82e2 bash

#查看操作系统版本
$ cat /etc/lsb-release 
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=22.04
DISTRIB_CODENAME=jammy
DISTRIB_DESCRIPTION="Ubuntu 22.04.3 LTS"

```

#### 2. 使用vncviwer连接容器

```
# 开启vnc端口
$ docker run -p 15901:5901  -e VNC_PW=password -e VNC_COL_DEPTH=32 -e VNC_RESOLUTION=1920x1080 -v /dev/shm:/dev/shm -it 2d3cd75e82e2 bash
```

使用[vncviwer](https://www.realvnc.com/en/connect/download/viewer/)连接 127.0.0.1:15901

#### 3. 安装其它软件

使用vncviwer连接容器桌面，打开终端安装libreoffice

```
# 安装常用软件包
$ apt install curl apache2-utils vim less telnet net-tools
# 安装libreoffice
$ apt install libreoffice  pinta language-pack-zh-hant language-pack-gnome-zh-hant firefox-locale-zh-hant libreoffice-l10n-zh-tw
```

#### 4. 参考

* [Headless Ubuntu/Xfce containers with VNC/noVNC](https://github.com/accetto/ubuntu-vnc-xfce-g3)
* [using VNC viewer](https://accetto.github.io/user-guide-g3/using-vnc/)
* [Download RealVNC® Viewer](https://www.realvnc.com/en/connect/download/viewer/)