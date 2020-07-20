---
layout: post
title: CentOS内核升级
categories: [编程, linux]
tags: [kernel, docker]
---


> `CentOS`默认的内核版本为`3.10.x`，`docker-ce-19`依赖`4.x`版本内核

#### 1. 安装高版本内核
下载并安装`4.x`版本内核
```
sudo yum -y localinstall kernel-ml-4.16.13-1.el7.elrepo.x86_64.rpm
```
#### 2. 重新生成启动配置

```
$ vi /etc/default/grup

GRUB_DEFAULT=0
```
#### 3. 重新编译内核启动文件

```
$ grub2-mkconfig -o /boot/grub2/grub.cfg
```

> 如果系统是用`uefi`引导的，则编译命令为`sudo grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg`   
> 可通过`df -hT | grep /boot/efi`或`ls /sys/firmware/efi`命令查看系统是否由`uefi`引导

#### 4. 重启主机

```
$ reboot
$ uname -r
```
