---
layout: post
title: Linux配置ssh免密码登录
categories: [编程, linux]
tags: [ssh]
---

> ssh 登录每次都要输入密码太烦人

#### 1. 客户机生成密钥对
登录客户机生成密钥对
```
ssh-keygen -t rsa
ll ~/.ssh
```

```
id_rsa
id_rsa.pub
```

#### 2. 复制客户机公钥到服务器
登录服务器
```
vi ~/.ssh/authorized_keys

```

复制客户机`id_rsa.pub`内容到服务器`authorized_keys`

#### 3. 客户机免密登录服务器

> 不烦人了
