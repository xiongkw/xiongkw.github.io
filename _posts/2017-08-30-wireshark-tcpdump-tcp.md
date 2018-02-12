---
layout: post
title: 使用wireshark和tcpdump分析tcp的三次握手与四次挥手
categories: [编程, tcp]
tags: [wireshark, tcpdump, tcp]
---

> 关于理论知识，这里有一篇牛人的文章 [简析TCP的三次握手与四次挥手](http://www.jellythink.com/archives/705)   

本文通过`wireshark`分析`tcp`的三次握手与四次挥手(所谓握手挥手即是报文的发送)，首先复习一下`tcp`协议

#### 1.TCP协议
![]({{site.url}}/public/images/2017-08-30-wireshark-tcpdump-tcp-protocol.png)

* `Source Port` 2字节16位 TCP连接是双向的，如果我要连接s，那我必须先准备一个端口给你连接 
* `Destination Port` 2字节16位 目标的端口(TCP层只有端口, 因为ip在IP层)
* `Sequence number` 4字节 每个TCP报文都有一个序列号，表示数据段第一个字节的序号
* `Acknowledgment number` 收到数据段的字节序号+1，返回给发送端，意思是：我已经收到序号x的数据了，你下一次要从x+1序号开始发送
* `Offset` TCP报文的头信息占用字节数，因为`选项`长度不固定
* `Flags` 报文标识，占用6位，依次为`URG，ACK，PSH，RST，SYN，FIN`
```
    URG：1表示TCP包的紧急指针域有效，用来保证TCP连接不被中断，并且督促中间层设备要尽快处理这些数据；   
    ACK：1表示应答域有效   
    PSH：表示Push操作。所谓Push操作就是指在数据包到达接收端以后，立即传送给应用程序，而不是在缓冲区中排队；
    RST：表示连接复位请求。用来复位那些产生错误的连接，也被用来拒绝错误和非法的数据包；
    SYN：表示同步序号，用来建立连接。SYN标志位和ACK标志位搭配使用，当连接请求的时候，SYN=1，ACK=0；连接被响应的时候，SYN=1，ACK=1；
    FIN： 表示发送端已经达到数据末尾，也就是说双方的数据传送完成，没有数据可以传送了，发送FIN标志位的TCP数据包后，连接将被断开。
```

#### 2.TCP状态转换
状态转换图

![]({{site.url}}/public/images/2017-08-30-wireshark-tcpdump-tcp-state.png)


#### 3.使用tcpdump抓包
环境
```
服务端: 192.168.1.100:8011 以下简称 s
客户端: 192.168.1.101      以下简称 c
```

使用`tcpdump`在客户端抓包(服务端也可)
```
tcpdump -i eth0 host 192.168.1.100 and port 8011 -w test.cap
```

#### 4.使用wireshark分析

![]({{site.url}}/public/images/2017-08-30-wireshark-tcpdump-tcp-wireshark.png)

#### 5.三次握手
握手一
```
Transmission Control Protocol, Src Port: 45982, Dst Port: 8011, Seq: 0, Len: 0
    Source Port: 45982
    Destination Port: 8011
    [Stream index: 0]
    [TCP Segment Len: 0]
    Sequence number: 0    (relative sequence number)
    Acknowledgment number: 0
    Header Length: 40 bytes
    Flags: 0x002 (SYN)
    Window size value: 14600
    [Calculated window size: 14600]
    Checksum: 0x35ce [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    Options: (20 bytes), Maximum segment size, SACK permitted, Timestamps, No-Operation (NOP), Window scale

```
> c: hi s, 我要连接你(当然s事先已经公布了自己的地址`ip+port`).   
> 准备我的 `Source Port: 45982`，其实是创建了一个`socket`   
> 准备`Sequence number: 0` ，这里是一个相对数值   
> `Flags: 0x002 (SYN)`，表示这是一个请求建立连接的报文   
> `Acknowledgment number: 0 `, 只有在`Flags-ACK`为1时才有效，这里无效   
> 此时 `socket c` 进入 `SYN_SEND`状态

握手二
```
Transmission Control Protocol, Src Port: 8011, Dst Port: 45982, Seq: 0, Ack: 1, Len: 0
    Source Port: 8011
    Destination Port: 45982
    [Stream index: 0]
    [TCP Segment Len: 0]
    Sequence number: 0    (relative sequence number)
    Acknowledgment number: 1    (relative ack number)
    Header Length: 40 bytes
    Flags: 0x012 (SYN, ACK)
    Window size value: 14480
    [Calculated window size: 14480]
    Checksum: 0xe67c [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    Options: (20 bytes), Maximum segment size, SACK permitted, Timestamps, No-Operation (NOP), Window scale
    [SEQ/ACK analysis]

```
> s: 你可以连接我，同时我也要连接你   
> `Destination Port: 45982`，从你的请求中获取   
> `Sequence number: 0`，我发送给你的数据的序号   
> `Acknowledgment number: 1`, 这是一个回复确认序号，表示我收到你发送的序号`握手一中Sequence number: 0`，你下一次可以用`0+1=1`发送   
> `Flags: 0x012 (SYN, ACK)'`   两个有效标志：发送建立连接请求和同意对方的连接请求，其实是两个动作放在一个报文里了，否则就变成四次握手了      
>  创建一个 `socket s`并进入 `SYN_SEND`状态

握手三
```
Transmission Control Protocol, Src Port: 45982, Dst Port: 8011, Seq: 1, Ack: 1, Len: 0
    Source Port: 45982
    Destination Port: 8011
    [Stream index: 0]
    [TCP Segment Len: 0]
    Sequence number: 1    (relative sequence number)
    Acknowledgment number: 1    (relative ack number)
    Header Length: 32 bytes
    Flags: 0x010 (ACK)
    Window size value: 29
    [Calculated window size: 14848]
    [Window size scaling factor: 512]
    Checksum: 0x4100 [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    Options: (12 bytes), No-Operation (NOP), No-Operation (NOP), Timestamps
    [SEQ/ACK analysis]

```
> `Sequence number: 1`, `握手二`中要求的序号   
> `Acknowledgment number: 1`,同意你的请求，要求你下一次用这个序号
> `Flags: 0x010 (ACK)`, 回复确认标志   
> `socket c` 进入 `ESTABLISHED`状态
> `socket s` 收到这段报文后进入 `ESTABLISHED`状态

#### 6.四次挥手
挥手一
```
Transmission Control Protocol, Src Port: 45982, Dst Port: 8011, Seq: 115, Ack: 918, Len: 0
    Source Port: 45982
    Destination Port: 8011
    [Stream index: 0]
    [TCP Segment Len: 0]
    Sequence number: 115    (relative sequence number)
    Acknowledgment number: 918    (relative ack number)
    Header Length: 32 bytes
    Flags: 0x011 (FIN, ACK)
    Window size value: 33
    [Calculated window size: 16896]
    [Window size scaling factor: 512]
    Checksum: 0x3cec [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    Options: (12 bytes), No-Operation (NOP), No-Operation (NOP), Timestamps

```
> `c`发送`FIN`报文给`s` 表示数据发送完毕，请求关闭，`c` 进入 `FIN_WAIT_1`状态   

挥手二和三
```
Transmission Control Protocol, Src Port: 8011, Dst Port: 45982, Seq: 918, Ack: 116, Len: 0
    Source Port: 8011
    Destination Port: 45982
    [Stream index: 0]
    [TCP Segment Len: 0]
    Sequence number: 918    (relative sequence number)
    Acknowledgment number: 116    (relative ack number)
    Header Length: 32 bytes
    Flags: 0x011 (FIN, ACK)
    Window size value: 29
    [Calculated window size: 14848]
    [Window size scaling factor: 512]
    Checksum: 0xe674 [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    Options: (12 bytes), No-Operation (NOP), No-Operation (NOP), Timestamps
    [SEQ/ACK analysis]

```
> 这里把挥手二和三放在一起，因为抓包发现两个报文经常会合并到一起，原因是`s`收到`c`的`FIN`报文后，如果数据发送未完成，则只发送`ACK`，如果数据发送已经完成则合并`ACK+FIN`一起发送   
> `s`收到`c`的`FIN`报文后进入`CLOSE_WAIT`状态   
> `s`回复一个`ACK`报文给`c`, 而`c`收到这个`ACK`后会进入`FIN_WAIT_2`状态         
> `s`发送`FIN`报文请求关闭, `s`进入`LAST_ACK`状态   

挥手四
```
Transmission Control Protocol, Src Port: 45982, Dst Port: 8011, Seq: 116, Ack: 919, Len: 0
    Source Port: 45982
    Destination Port: 8011
    [Stream index: 0]
    [TCP Segment Len: 0]
    Sequence number: 116    (relative sequence number)
    Acknowledgment number: 919    (relative ack number)
    Header Length: 32 bytes
    Flags: 0x010 (ACK)
    Window size value: 33
    [Calculated window size: 16896]
    [Window size scaling factor: 512]
    Checksum: 0x3cea [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    Options: (12 bytes), No-Operation (NOP), No-Operation (NOP), Timestamps
    [SEQ/ACK analysis]

```
> `c`收到`FIN`后进入`TIME_WAIT`状态   
> `c` 回复`ACK` 确认报文，`s`收到这个`ACK`后就直接`CLOSE`了   
> `c`继续等待`2MSL`后被`CLOSE`

#### 7.几个问题

* `wireshark`中`HTTP`请求和响应后都有一段`TCP`的`ACK`报文，原因在于TCP对于每个数据报文都会返回一个确认报文，而`HTTP`的响应显然也是一个`TCP`数据报文
* 挥手二和挥手三经常会合并到同一个报文，原因是被动关闭方数据也已经发送完毕了，既然可以一次发送为何还要分两次？
* 大量的`CLOSE_WAIT`是由于被动关闭引起，原因在于收到了`FIN`报文却不回复`FIN`报文，一定是程序引起的
* 大量的`TIME_WAIT`是由于主动关闭引起，可以通过服务器参数优化解决
```
#开启重用，允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
net.ipv4.tcp_tw_reuse = 1  
#开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
net.ipv4.tcp_tw_recycle = 1  
#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间。
net.ipv4.tcp_fin_timeout = 30  
```