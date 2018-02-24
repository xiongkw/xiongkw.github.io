---
layout: post
title: Linux配置ssh免密码登录
categories: [编程, linux]
tags: [ssh]
---

> `ssh` 登录每次都要输入密码太烦人

#### 1. 客户机生成密钥对

登录客户机生成密钥对：

```
ssh-keygen -t rsa
ll ~/.ssh
```

```
id_rsa
id_rsa.pub
```

#### 2. 复制客户机公钥到服务器

```
ssh user@host 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```
解释：
* `ssh user@host`，表示登录远程主机
* `'mkdir .ssh && cat >> .ssh/authorized_keys'`，表示登录后在远程`shell`上执行的命令
* `mkdir -p .ssh`的作用是如果用户主目录中的`.ssh`目录不存在，就创建
* `cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub`的作用是，将本地的公钥文件`~/.ssh/id_rsa.pub`，重定向追加到远程文件`authorized_keys`的末尾

#### 3. 客户机免密登录服务器

> 不烦人了

#### 4. SSH免密登录原理

```
 +---------------+                                                   +-----------------+         
 |               | -------> C生成密钥对，发送公钥给S ----------------> |                 |
 |               |                                                   |                 |
 |               | <----- 把C的公钥加到~/.ssh/authorized_keys <------ |                 |
 |   客户机(C)    |                                                   |    服务机(S)    |
 |               | -----> C发送ssh连接请求 -------------------------> |                 |
 |               |                                                   |                 |
 |   密钥对       | <----- 使用C的公钥加密一个随机字符串发送给C <------  |    授权列表     |
 |  (公钥私钥)    |                                                   |(authorized_keys)|
 |               | -----> 用自己的私钥解密并返回给S -----------------> |                 |
 |               |                                                   |                 |
 |               | <----- 比对解密结果，认证完成  <------------------  |                 |
 |               |                                                   |                 |
 +---------------+                                                   +-----------------+
```