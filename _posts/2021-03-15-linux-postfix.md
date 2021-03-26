---
layout: post
title: 使用postfix解决企业邮箱发件限制问题
categories: [编程, linux]
tags: [postfix]
---

> 免费的企业邮箱一般都会限制发件数和频率

#### 1. 安装postfix

yum安装即可
```
$ yum install postfix -y
```

#### 2. 配置postfix


```
$ vi /etc/postfix/main.cf

#服务器名
myhostname = mail.my.com
#根域名
mydomain = my.com
#监听网卡的所有IP
inet_interfaces = all
#设置信任本机，就设置为host
mynetworks_style = host
#配置哪些地址的邮件能够被Postfix转发，设置企业邮箱地址xxx.com
relay_domains = $mydomain,xxx.com
```

#### 3. 测试

```
$ systemctl restart postfix
# 使用企业邮箱abc@xxx.com发送邮件
$ echo "Hello world!" | mail -r abc@xxx.com -s "hello" xxxx@qq.com
```

登录xxxx@qq.com邮箱，查看刚刚发送的邮件

#### 4. 常用命令

```
# 查看当前配置信息
$ postconf -n
# 重新加载配置文件
$ postfix reload
# 查看队列（或 postqueue -p）
$ mailq
# 查看邮件内容
$ postcat -q Queue_ID
# 删除队列中指定邮件
$ postsuper -d Queue_ID
# 清空队列
$ postsuper -d ALL
# 删除队列中的异常邮件
$ postsuper -d ALL deferred
# 查看邮件详细日志
$ tail -f /var/log/maillog
```

#### 5. 参考

* [How To Install and Configure Postfix as a Send-Only SMTP Server](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-postfix-as-a-send-only-smtp-server-on-ubuntu-20-04)