---
layout: post
title: 使用spring-boot搭建https服务
categories: [编程, java, spring]
tags: [spring-boot, https]
---

> 使用`spring-boot`开发的`web`项目，直接使用内嵌的`tomcat`运行

#### 1. keystore生成
```
keytool -genkeypair -keyalg RSA -keysize 2048 -sigalg SHA1withRSA -validity 3650 -alias myapp -keystore myapp.keystore

输入密钥库口令:
再次输入新口令:
您的名字与姓氏是什么?
  [Unknown]:  www.myapp.com
您的组织单位名称是什么?
  [Unknown]:  xx
您的组织名称是什么?
  [Unknown]:  xx
您所在的城市或区域名称是什么?
  [Unknown]:  xx
您所在的省/市/自治区名称是什么?
  [Unknown]:  xx
该单位的双字母国家/地区代码是什么?
  [Unknown]:  xx
CN=www.myapp.com, OU=xx, O=xx, L=xx, ST=xx, C=xx是否正确?
  [否]:  y

输入 <myapp> 的密钥口令
        (如果和密钥库口令相同, 按回车):
```

#### 2. spring-boot配置

`application.yml`

```yaml
server:
  port: 8443
  ssl:
    key-alias: myapp
    key-store: 'classpath:myapp.keystore'
    key-store-password: 123456
```

> 为方便，这里把`myapp.keystore`放到`resources`目录下

#### 3. 访问

运行`spring-boot`项目并访问

```
https://www.myapp.com:8443
```