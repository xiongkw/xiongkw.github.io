---
layout: post
title: linux中使用expect批量安装和配置rsyslog
categories: [编程, linux]
tags: [expect, rsyslog]
---

> 分布式集群环境下，使用`rsyslog`采集应用日志，需要在每台宿主机上安装和配置`rsyslog`，一共需要维护30多台服务器，于是想到用`expect`批量操作

#### 1. expect简介
> `Expect`是一个用来处理交互的命令。借助`Expect`，我们可以将交互过程写在一个脚本上，自动完成`ssh, telnet, ftp`等交互操作

#### 2. expect安装
```
yum install expect -y
```

#### 3. expect用法
```
send：发送字符串，用于发送命令
expect：期望字符串，用于等待交互命令的回显
spawn：在expect中执行shell命令
interact：允许用户交互
```

#### 4. 安装rsyslog的例子
```
#!/bin/expect

proc enterPassword {password} {
    expect {
        "(yes/no)?" {
              send "yes\n"
              expect "password:"
              send "$password\r"
        }
        "password:" {
              send "$password\r"
        }
    }
}

set ip [lindex $argv 0]
set password [lindex $argv 1]
set timeout 30

spawn scp /etc/yum.repos.d/rsyslog.repo docker@$ip:/home/docker
enterPassword $password
spawn echo 'File rsyslog.repo copyed'

spawn scp /etc/rsyslog.d/myapp.conf docker@$ip:/home/docker
enterPassword $password
spawn echo 'File myapp.conf copyed'

spawn ssh docker@$ip
enterPassword $password
expect "Authorized"

send "sudo mv -f /home/docker/rsyslog.repo /etc/yum.repos.d\r"
send "sudo yum update rsyslog -y\r"
send "sudo yum install rsyslog-kafka -y\r"

send "sudo mv -f /home/docker/myapp.conf /etc/rsyslog.d\r"
send "sudo service rsyslog restart\r"
send "service rsyslog status\r"

send "sudo echo '0  1  *  *  * root service rsyslog restart >/dev/null 2>&1' |sudo tee -a /etc/crontab\r"
send "sudo service crond restart\r"
send "service crond status\r"

interact
```

运行
```shell
rsyslog.sh 192.168.1.100 docker
```

说明：
* `expect`函数的格式为`proc func1 {param1 param2} {}`
* 更新`rsyslog`yum源，升级`rsyslog`到最新版，并安装`rsyslog-kafka`
* 由于`docker`用户没有写`/etc/`目录的权限，需要先`scp`到用户目录，再`sudo mv`
* 文件复制完成后`ssh`到目标主机，执行安装和配置操作
* `ssh`是一个交互操作，需要等待回显字符`Authorized`
* 重启并查看`rsyslog`服务状态
* 在`crontab`中配置`rsyslog`重启定时器

批量查看定时器配置的例子

```
#!/bin/bash
for ip in `awk '{print $0}' hosts.txt`
do
echo $ip
expect sh.exp $ip "tail -3 /etc/crontab"
done
```

`sh.exp`
```
#!/bin/expect
proc fn_expect {password} {
    expect {
        "(yes/no)?" {
              send "yes\n"
              expect "password:"
              send "$password\r"
        }
        "password:" {
              send "$password\r"
        }
    }
}

set username "docker"
set password "docker!123"
set ip [lindex $argv 0]

set timeout 3

spawn ssh $username@$ip
fn_expect $password
expect "Authorized"

send "[lindex $argv 1]\r"
expect "@"
expect "@"

#interact
```

`hosts.txt`
```
192.168.1.101
192.168.1.102
192.168.1.103
```

批量更新配置文件
```
#!/bin/bash
for ip in `awk '{print $0}' hosts.txt`
do
echo $ip
expect scp.exp $ip /etc/crontab
expect scp.exp $ip /etc/rsyslog.d/myapp.conf
expect sh.exp $ip "sudo service crond restart"
expect sh.exp $ip "sudo service rsyslog restart"
#expect sh.exp $ip "tail -3 /etc/crontab"
done
```

`scp.exp`
```
#!/bin/expect
proc fn_expect {password} {
    expect {
        "(yes/no)?" {
              send "yes\n"
              expect "password:"
              send "$password\r"
        }
        "password:" {
              send "$password\r"
        }
    }
}

set username "docker"
set password "docker!123"
set ip [lindex $argv 0]
set path [lindex $argv 1]


set timeout 3

spawn scp $path $username@$ip:/home/docker/.mytmp
fn_expect $password

spawn ssh $username@$ip
fn_expect $password
expect "Authorized"

send "cat /home/docker/.mytmp|sudo tee $path\r"
expect "@"
send "rm -f /home/docker/.mytmp\r"
expect "@"

#interact
```