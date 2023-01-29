---
layout: post
title: shell中使用envsubst处理模板文件
categories: [linux]
tags: [shell, envsubst]
---

>  

#### 1. envsubst的用法

> 用于替换字符串中的环境变量，例如${var}或$var

```
# 替换标准输入
$ echo "${HOME}" | envsubst
# 替换文件内容
$ envsubst < test.tpl
# 替换文件内容并输出到文件
$ envsubst < test.tpl > test.ini
```

#### 2. 一个例子

##### 2.1 模板文件

application.yml.tpl
```
spring:
  application:
    name: ${service_name}
server:
  port: ${server_port}
```

##### 2.2 变量配置文件

config.ini
```
service_name=test
server_port=8080
```

##### 2.3 处理脚本

```
# 把source读取的变量导出到envsubst命令
$ set -a 
$ source config.ini
$ set +a

# 替换模板文件中的变量，生成配置文件
$ envsubst < application.yml.tpl > application.yml
```