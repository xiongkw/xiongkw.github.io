---
layout: post
title: Jenkins配置https访问
categories: [编程, linux]
tags: [jenkins]
---

> 

#### 1. 设置启动参数

查看官网文档[Configuring HTTP](https://www.jenkins.io/doc/book/installing/initial-settings/#configuring-http)，很简单，设置三个https参数即可，但是这里的证书是jks格式，难点在于制作jks证书

```
--httpPort=-1 \
--httpsPort=443 \
--httpsKeyStore=path/to/keystore \
--httpsKeyStorePassword=keystorePassword
```

#### 2. 生成证书

* Obtain SSL certificates
* Convert SSL keys to PKCS12 format
* Convert PKCS12 to JKS format

##### 2.1 生成自签名证书

```
$ openssl req -x509 -nodes -days 999 -newkey rsa:2048 -keyout server.key -out server.crt 
$ openssl req -x509 -new -nodes -key server.key -sha256 -days 999 -out ca.crt
```

##### 2.2 转换为PKCS12格式

```
$ openssl pkcs12 -export -out jenkins.p12 \
  -passout 'pass:123456' -inkey server.key \
  -in server.crt -certfile ca.crt -name fool.com
```

##### 2.3 转换为jks格式

```
$ keytool -importkeystore -srckeystore jenkins.p12 \
  -srcstorepass '123456' -srcstoretype PKCS12 \
  -srcalias fool.com -deststoretype JKS \
  -destkeystore jenkins.jks -deststorepass '123456' \
  -destalias fool.com
```

#### 3. 启动jenkins

```
$ java -DJENKINS_HOME=/app/jenkins -jar jenkins.war --httpPort=80 --httpsPort=8443 --httpsKeyStore=jenkins.jks --httpsKeyStorePassword=123456
```

#### 4. 关于nginx https代理

> 使用nginx https代理jenkins会出现agent-jnlp通过https连接不到master的情况，原因是通过jnlp接口获取到的jenkins地址为非https

#### 5. 参考

* [Configuring HTTP](https://www.jenkins.io/doc/book/installing/initial-settings/#configuring-http)
* [How to Configure SSL on Jenkins Server](https://devopscube.com/configure-ssl-jenkins/)
* [How To Create CA and Generate SSL/TLS Certificates & Keys](https://scriptcrunch.com/create-ca-tls-ssl-certificates-keys/)
