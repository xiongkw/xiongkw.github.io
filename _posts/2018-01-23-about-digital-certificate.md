---
layout: post
title: 关于加密解密、消息摘要、数字签名和数字证书
categories: [编程]
tags: [对称加密, 非对称加密, 消息摘要, 数字签名, certificate, 数字证书]
---

#### 1 对称加密与非对称加密算法

* 对称加密，加密和解密使用同一个密钥，由于通信双方需要同时知道密钥，导致密钥在传输过程中容易泄露。常用的对称性加密算法有DES、AES、3DES、TDEA、Blowfish、RC5、IDEA等
* 非对称性加密，即使用两个不同的密钥分别进行加密和解密，这两个密钥称为私钥/公钥对。使用私钥加密的信息必须使用公钥解密，反之亦然。公钥可以公开发布，私钥由加密方保存，因此非对称性加密算法的安全性比对称性加密算法的安全性更高。常用的非对称性加密算法有DSA、RSA、Elgamal、背包算法、Rabin、D-H、ECC等

> 既然非对称性加密算法比对称性加密算法安全性更高，那对称性加密算法有什么存在的必要呢？这是因为对称性加密算法的运算速度更快。现实中，往往将对称性加密算法和非对称性加密算法结合使用，对于要传输的大块数据使用对称性加密算法加密，然后对加密使用的密钥使用非对称性加密算法进行加密，这样既可以获得更高的安全性，又可以获得更高的加解密运算速度。

#### 2 消息摘要算法
* 消息摘要算法的主要目的是生成消息的摘要，消息摘要一般是一段固定长度的hash值。只有对相同的消息运算才能得到相同的摘要，而且不可能从摘要反过来推算出数据。常用的消息摘要算法有MD5和SHA-1等

#### 3 数字签名
举个例子，A发给B一个消息，并且有A的`亲笔签名`，那么A将无法抵赖曾经发送过这段消息，因为我们假设所有`亲笔签名`是不可伪造

实际上签名往往会配合摘要一起使用，发送方先生成消息摘要，再使用自己的私钥对摘要加密，接收方则先使用发送方的公钥解密，再和生成的消息摘要比对。如果摘要相同，则说明消息没有被篡改。同时因为发送方的私钥只有他自己才有，所以可以确定消息确实由他所发出。

> 数字签名一般用于防篡改和防抵赖

#### 4 SSL/TLS和HTTPS
* `SSL`: Secure Sockets Layer，安全套接层，由网景公司设计，其采用密钥对对消息进行加密和解密，保证了消息的传输安全
* `TLS`: Transport Layer Security，传输层安全协议，由`IETF`在SSL3.0基础上制定了标准协议TLS1.0，最新版为TLS1.2
* `OpenSSL`: SSL的开源实现
* `HTTPS`: HTTP over SSL，即在SSL协议上的HTTP协议

SSL/TLS工作原理：
![]({{site.url}}/public/images/2018-01-23-about-digital-certificate.svg)

TLS协议的握手过程使用证书校验双方身份并协商一个对称加密密钥用于对消息加密解密

> 参考[The Transport Layer Security (TLS) Protocol](https://tools.ietf.org/html/rfc5246#section-7.4.3)

#### 5 数字证书
对于两个很熟识的人来说，交换公钥可以建立在相互信任的基础上。

而对于互不认识的人来说，如何交换公钥？如何保证公钥确实属于某人？这就要求有一个双方都信任的第三方机构来做认证。例如身份证，去银行办事得带身份证，因为银行信任身份证的颁发方(公安)，所以银行通过身份证就能确认客户确实是某个身份。

数字证书就是一个类似的`身份证`，由权威认证机构颁发，其内容包含证书所有者的标识和它的公钥，并由权威认证机构使用它的私钥进行签名。信息的发布者通过在网络上发布证书来公开它的公钥，该证书由权威认证机构进行签名，认证机构也是通过发布它的证书来公开该机构的公钥，认证机构的证书由更权威的认证机构进行签名，这样就形成了证书链。证书链最顶端的证书称为根证书，根证书就只有自签名了。

##### 5.1 数字证书的标准
`X.509`: 由国际电信联盟（ITU-T）制定的数字证书标准

##### 5.2 数字证书的编码格式

* `PEM`: Privacy Enhanced Mail,打开看文本格式,以"-----BEGIN..."开头, "-----END..."结尾,内容是BASE64编码
* `DER`: Distinguished Encoding Rules,打开看是二进制格式,不可读

##### 5.3 数字证书相关的扩展名

* `CRT`: `Certificate`的缩写，常见于`*NIX`系统
* `CER`: `Certificate`的缩写，常见于`WINDOWS`系统
* `KEY`: 通常用于存放私钥
* `CSR`: Certificate Signing Request，即证书签名请求
* `PFX/P12`: `WINDOWS`下的一种证书格式，包含了证书和私钥(`*NIX`下通常是分成两个文件)
* `JKS`: `Java key storage`，java存储证书的一种格式，可使用keytool生成

##### 5.4 数字证书的申请

数字证书可以通过权威认证机构申请，以下通过openssl生成一个证书申请文件

```
openssl req -new -nodes -days 999 -newkey rsa:2048 -keyout my.key -out my.csr
```
* -new: 生成证书申请
* -nodes: 忽略私钥密码
* -days 999: 证书有效期999天
* -newkey rsa:2048: 生成加密强度为RSA 2048的私钥
* -keyout: 生成私钥文件名
* -out: 生成证书申请请求文件名

##### 5.5 生成自签名证书

```
openssl req -x509 -nodes -days 999 -newkey rsa:2048 -keyout my.key -out my.crt

```

* -x509: 生成自签名证书
* -nodes: 忽略私钥密码
* -days 999: 证书有效期999天
* -newkey rsa:2048: 生成加密强度为RSA 2048的私钥
* -keyout: 生成私钥文件名
* -out: 生成证书文件名

#### 6. openssl req命令
openssl req命令格式
```
openssl req [-inform PEM|DER] [-outform PEM|DER] [-in filename] [-passin arg] [-out filename] [-passout arg] [-text] [-pubkey] [-noout] [-verify] [-modulus] [-new] [-rand file(s)] [-newkey rsa:bits][-newkey alg:file] [-nodes] [-key filename] [-keyform PEM|DER] [-keyout filename] [-keygen_engine id] [-[digest]] [-config filename] [-subj arg] [-multivalue-rdn] [-x509] [-days n] [-set_serial n][-asn1-kludge] [-no-asn1-kludge] [-newhdr] [-extensions section] [-reqexts section] [-utf8] [-nameopt] [-reqopt] [-subject] [-subj arg] [-batch] [-verbose] [-engine id]

```

* new/x509: 当使用-new选取的时候，说明是要生成证书请求，当使用x509选项的时候，说明是要生成自签名证书。

* /key/newkey/keyout: key和newkey是互斥的，key是指定已有的密钥文件，而newkey是指在生成证书请求或者自签名证书的时候自动生成密钥，然后生成的密钥名称有keyout参数指定。   
当指定newkey选项时，后面指定rsa:bits说明产生rsa密钥，位数由bits指定。指定dsa:file说明产生dsa密钥，file是指生成dsa密钥的参数文件(由dsaparam生成)

* in/out/inform/outform/keyform: in选项指定证书请求文件，当查看证书请求内容或者生成自签名证书的时候使用   
out选项指定证书请求或者自签名证书文件名，或者公钥文件名(当使用pubkey选项时用到)，以及其他一些输出信息。   
inform、outform、keyform分别指定了in、out、key选项指定的文件格式，默认是PEM格式。