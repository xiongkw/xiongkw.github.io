---
layout: post
title: Linux服务和chkconfig
categories: [编程, linux]
tags: [service]
---

> 虽说`linux`服务器重启的机率非常小，但凡事总会有特例，如何让程序在服务器重启后自动启动，这就要用到`chkconfig`命令了

#### 1. chkconfig命令

查看帮助

```
$ chkconfig --help

usage:   chkconfig [--list] [--type <type>] [name]
         chkconfig --add <name>
         chkconfig --del <name>
         chkconfig --override <name>
         chkconfig [--level <levels>] [--type <type>] <name> <on|off|reset|resetpriorities>
```

参数说明：

* `--list`: 查看所有服务
* `--add`: 添加服务
* `--del`: 删除服务
* `--override`: 覆盖(即更新)服务
* `--level`: 修改服务的启动或关闭等级

等级说明：

* `0`：关机
* `1`：单用户模式
* `2`：无网络连接的多用户命令行模式
* `3`：有网络连接的多用户命令行模式
* `4`：不可用，例如休眠模式
* `5`：带图形界面的多用户模式
* `6`：重启

#### 2. 编写一个tomcat服务

编写服务脚本：`/etc/init.d/tomcatd`

```
#!/bin/bash

# chkconfig: 2345 10 90
# description: tomcat ...

prog="Tomcat"

start() {
    echo -n $"Starting $prog: "
    /app/tomcat/bin/startup.sh
}

stop() {
    echo -n $"Stopping $prog: "
    PID=`ps aux | grep tomcat | grep -v grep | awk '{print $2}'`
    if [[ -n "$PID" ]]; then
        kill $PID
    fi
}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    *)
        echo "Usage: $0 {start|stop}"
        RETVAL=1
esac

exit $RETVAL
```

> linux系统服务必须放在`/etc/init.d`目录
> `# chkconfig: 2345 100 90`: 指定在`2345`等级下开机启动，`100`表示启动优先级，`90`表示关闭优先级
> 别忘了给脚本加上执行权限`chmod 755 tomcatd`

这样便可以通过`service`命令来管理`tomcat`了

```
$ service tomcatd start

$ service tomcatd stop
```

#### 3. 使用chkconfig管理tomcat服务的启动等级

添加`nginxd`服务

```
$ chkconfig --add tomcatd

$ chkconfig --list tomcatd
tomcatd          0:off   1:off   2:on    3:on    4:on    5:on    6:off
```

> 这里的`2:on    3:on    4:on    5:on`对应tomcatd文件中的`# chkconfig: 2345`

修改tomcatd启动等级

```
$ chkconfig --level 24 tomcatd off

$ chkconfig --list tomcatd
tomcatd          0:off   1:off   2:off    3:on    4:off    5:on    6:off

$ chkconfig --level 24 tomcatd on
tomcatd          0:off   1:off   2:on    3:on    4:on    5:on    6:off
```
