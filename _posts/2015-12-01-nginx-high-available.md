---
layout: post
title: 通过keepalived VIP转移实现nginx双机热备的高可用
categories: [编程, nginx, linux, web]
tags: [keepalived, vip, lvs]
---

#### 1. 概述
> 通过`keepalived VIP`转移，实现`nginx`双机热备的高可用

#### 2. 环境准备

```yaml
Keepalived主备:
    主机21: 10.142.90.21 网卡: bond0
    主机23: 10.142.90.23 网卡: bond0

VIP: 10.142.90.191 
```

> 注意：该环境作为示例用，具体以实际环境为准

#### 3. 安装keepalived和nginx
分别在21和23服务器上安装`keepalived`和`nginx`，省略

#### 4. 验证nginx安装
##### 4.1 配置nginx

在两个`nginx`实例中分别新建`$NGINX_HOME/html/test.html`

21:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Test nginx ha!</title>
</head>
<body>
    <h1>host: 10.142.90.21</h1>
</body>
</html>
```

23:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Test nginx ha!</title>
</head>
<body>
    <h1>host: 10.142.90.23</h1>
</body>
</html>
```

##### 4.2 验证nginx配置

分别访问:

`http://10.142.90.21/test.html`
```
host: 10.142.90.21
```

`http://10.142.90.23/test.html`
```
host: 10.142.90.23
```

验证页面中显示主机的ip

#### 5. 配置keepalived
##### 5.1 配置keepalived.conf
分别配置两台服务器`/etc/keepalived/keepalived.conf`

21:
```
global_defs {
   notification_email {
   }
   router_id LVS_DEVEL
}

vrrp_script chk_http_port {
    script "/opt/chk_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface bond0
    virtual_router_id 52
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        10.142.90.191
    }

    track_script {
        chk_http_port
    }
}
```

23:
```
global_defs {
   notification_email {
   }
   router_id LVS_DEVEL
}

vrrp_script chk_http_port {
    script "/opt/chk_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface bond0
    virtual_router_id 52
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        10.142.90.191
    }

    track_script {
        chk_http_port
    }
}
```

> 配置`vrrp_script`定时监控`nginx`进程


##### 5.2 配置nginx健康检查脚本

`/opt/chk_nginx.sh`

```shell
#!/bin/sh
# check nginx server status
NGINX=/usr/local/nginx-ha/sbin/nginx
PORT=1080

netstat -anlp|grep "$PORT.*LISTEN.*nginx"
if [ $? -ne 0 ];then
    $NGINX -s stop
    $NGINX
    sleep 3
    netstat -anlp|grep "$PORT.*LISTEN.*nginx"
    [ $? -ne 0 ] && service keepalived stop
fi
```
> 如果`nginx`进程挂了，则重新启动`nginx`

##### 5.3 查看VIP绑定结果

在两台服务器上执行命令
```
//启动keepalived服务
service keepalived start
//查看VIP绑定情况
ip addr |grep bond0
```
21: 
```
8: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    inet 10.142.90.21/24 brd 10.142.90.255 scope global bond0
```

23:
```
8: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    inet 10.142.90.23/24 brd 10.142.90.255 scope global bond0
    inet 10.142.90.191/32 scope global bond0
```
> 如上说明VIP被绑定到了主机23上

##### 5.4 验证VIP高可用

停止主机23上`keepalived`服务
```
service keepalived stop
```

再次查看VIP绑定情况：
```
ip addr |grep bond0
```

发现VIP漂移到了主机21上

#### 6. 验证nginx高可用
##### 6.1 服务器层(比如服务器或者keepalived挂了)
访问：`http://10.142.90.191/test.html`

```
host: 10.142.90.21
```
说明当前`nginx`实例在主机21上

停止主机21上的`keepalived`服务
```
service keepalived stop
```
再次访问：`http://10.142.90.191/test.html`
```
host: 10.142.90.23
```

说明`nginx`服务已经从主机21切换到了主机23

##### 6.2 应用层(比如nginx挂了)
访问：`http://10.142.90.191/test.html`
```
host: 10.142.90.23
```
说明当前`nginx`实例在主机23上

停止主机23上的`nginx`进程
```
nginx -s stop
```

间隔2s再次查看`nginx`进程
```
ps -ef|grep nginx
```
发现`nginx`进程重新启动了

访问：`http://10.142.90.191/test.html`
```
host: 10.142.90.23
```

说明`nginx`进程已经被`keepalived`重新启动了

#### 7.附 使用rsync同步nginx配置文件
使用`rsync`同步两台主机上的`nginx`配置文件，如下以主机21为服务端，23为客户端，通过`rsync`配置，使主机23自动同步主机21上的`nginx`配置

##### 7.1 环境安装
* 两台主机分别安装`rsync`

```
yum install rsync
```

* 配置主机23 ssh免密码登录主机21

> 详情略

##### 7.2 架设rsync服务端

编辑`/etc/rsyncd.conf`
```properties
uid = nobody
gid = nobody
use chroot = yes
max connections = 4
pid file = /var/run/rsyncd.pid
exclude = lost+found/ test.html
transfer logging = yes
timeout = 900
ignore nonreadable = yes
dont compress   = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2
host_allow=10.142.90.23

[nginx]
       path = /usr/local/nginx-ha/conf
       comment = nginx-ha
```

启动`rsyncd`服务：
```
rsync --daemon
```

##### 7.3 rsync客户端
同步命令：
```
rsync -avzP root@10.142.90.21::nginx /usr/local/nginx-ha/conf
```
##### 7.4 配置自动同步

###### 7.4.1 同步脚本编写
编写同步脚本`/opt/rsync-nginx.sh`
```
#!/bin/sh

rsync -avzP root@10.142.90.21::nginx /usr/local/nginx-ha/conf
```

###### 7.4.2 定时器配置
编辑`/etc/crontab`，增加如下一行
```
*  *  *  *  *  root  sh /opt/rsync-nginx.sh
```
> 表示每分钟同步一次

重启`crond`服务
```
service crond restart
```

