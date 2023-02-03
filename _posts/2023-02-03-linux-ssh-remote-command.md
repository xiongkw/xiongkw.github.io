---
layout: post
title: linux中通过ssh执行远程命令
categories: [linux]
tags: [shell, ssh]
---

>  

#### 1. 基本用法

```
# 获取cpu信息
$ ssh user@host1 'cat /proc/cpuinfo'
# 获取内存信息
$ ssh user@host1 'cat /proc/meminfo'

```

#### 2. 执行多条命令

```
# 执行多条命令
$ ssh user@host1 'ls; pwd'
# 执行多行命令（命令行）
$ ssh user@host1 '
> ls
> pwd
> '
# 执行多行命令（脚本）
$ ssh user@host1 '\
ls;\
pwd'
```

#### 3. 执行本地脚本文件

```
# 不带参数
$ vi test.sh
#/bin/bash
ls
pwd
$ ssh user@host1 < test.sh
# 带参数
$ vi test.sh
#/bin/bash
echo $1
$ ssh user@host1 'bash -s' < test.sh Tomcat
Tomcat
```

#### 4. 执行远程脚本文件
```
$ ssh user@host1 '/root/test.sh Tomcat'
Tomcat
```

#### 5. 执行交互命令

```
$ ssh -t user@host1 'top'
```

#### 6. 自动登录

设置ssh免密登录，或借助sshpass、expect等工具

#### 7. 一个例子

批量对多个主机创建用户

```
#!/bin/bash

hosts="192.168.1.100,192.168.1.101,192.168.1.102"
for host in $hosts;
do
	sshpass -p "123456" ssh -o StrictHostKeyChecking=no root@$host '\
		useradd -d /home/fool -m fool; \
		echo "654321" | passwd --stdin fool; \
		echo "fool ALL=(ALL)       NOPASSWD:ALL" >> /etc/sudoers;'
done
```

#### 8. 参考

* [SSH 教程](https://wangdoc.com/ssh/)