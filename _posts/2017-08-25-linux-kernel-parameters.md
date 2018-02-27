---
layout: post
title: linux服务器内核参数调优
categories: [编程, linux, web]
tags: [kernel]
---

> 修改`/etc/sysctl.conf`来调整`linux`内核参数，以提高`web`服务器性能

#### 1. 修改方法
```
vi /etc/sysctl.conf

sysctl -p
```

#### 2. 参数说明
```properties
#每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目
net.core.netdev_max_backlog = 32768
#接收套接字缓冲区大小的缺省值(字节)
net.core.rmem_default = 8388608
#接收套接字缓冲区大小的最大值(字节)
net.core.rmem_max = 16777216
#发送套接字缓冲区大小的缺省值(字节)
net.core.wmem_default = 8388608
#发送套接字缓冲区大小的最大值(字节)
net.core.wmem_max = 16777216
#表示socket监听（listen）的backlog上限，是socket的监听队列，当一个请求（request）尚未被处理或建立时，他会进入backlog
#而socket server可以一次性处理backlog中的所有请求，处理后的请求不再位于监听队列中。当server处理请求较慢，以至于监听队列被填满后，新来的请求会被拒绝。
net.core.somaxconn = 32768

#允许系统打开的端口范围，作为客户端连接其它服务时有用
net.ipv4.ip_local_port_range = 10000 65000
#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间，减小可以加快TIME_WAIT回收速度
net.ipv4.tcp_fin_timeout = 30
#TCP发送keepalive探测消息的间隔时间（秒），用于确认TCP连接是否有效，默认7200s
net.ipv4.tcp_keepalive_time = 1200
#在tcp_keepalive_time之后,没有接收到对方确认,继续发送保活探测包次数,默认值为9(次)
net.ipv4.tcp_keepalive_probes = 3 
#探测消息未获得响应时，重发该消息的间隔时间（秒）。默认值为75秒
net.ipv4.tcp_keepalive_intvl = 30 

#系统所能处理不属于任何进程的TCP sockets最大数量。假如超过这个数量﹐那么不属于任何进程的连接会被立即reset，并同时显示警告信息。
#之所以要设定这个限制﹐纯粹为了抵御那些简单的 DoS 攻击﹐千万不要依赖这个或是人为的降低这个限制。如果内存大更应该增加这个值。
net.ipv4.tcp_max_orphans = 3276800

#SYN队列的长度，默认为1024，增大后能容纳更多等待连接的网络连接数
net.ipv4.tcp_max_syn_backlog = 65536
#在内核放弃建立连接之前发送SYN包的数量
net.ipv4.tcp_syn_retries = 2
#为了打开对端的连接，内核需要发送一个SYN并附带一个回应前面一个SYN的ACK,tcp的第二次握手,这个设置决定了内核放弃连接之前发送SYN+ACK包的数量
net.ipv4.tcp_synack_retries = 2

#tcp_mem有3个INTEGER变量：low, pressure, high
#low：当TCP使用了低于该值的内存页面数时，TCP没有内存压力，TCP不会考虑释放内存。(理想情况下，这个值应与指定给tcp_wmem的第2个值相匹配。
#这第2个值表明，最大页面大小乘以最大并发请求数除以页大小 (131072*300/4096)
#pressure：当TCP使用了超过该值的内存页面数量时，TCP试图稳定其内存使用，进入pressure模式，当内存消耗低于low值时则退出pressure状态。
#(理想情况下这个值应该是TCP可以使用的总缓冲区大小的最大值(204800*300/4096)
#high：允许所有TCP Sockets用于排队缓冲数据报的页面量。如果超过这个值，TCP连接将被拒绝，这就是为什么不要令其过于保守(512000*300/4096)的原因了。
#在这种情况下，提供的价值很大，它能处理很多连接，是所预期的2.5倍；或者使现有连接能够传输2.5倍的数据。
net.ipv4.tcp_mem = 94500000 91500000 92700000

#开启SYN Cookies，当出现SYN等待队列溢出时，启用cookies来处理
net.ipv4.tcp_syncookies = 1
#时间戳可以避免序列号的卷绕。一个1Gbps的链路肯定会遇到以前用过的序列号。时间戳能够让内核接受这种“异常”的数据包。这里需要将其关掉
net.ipv4.tcp_timestamps = 0


#表示系统同时保持TIME_WAIT套接字的最大数量，如果超过这个数字，TIME_WAIT套接字将立刻被清除并打印警告信息, 默认为180000
net.ipv4.tcp_max_tw_buckets = 6000
#启用TIME-WAIT状态sockets快速回收功能
net.ipv4.tcp_tw_recycle = 1
#开启重用功能，允许将TIME-WAIT状态的sockets重新用于新的TCP连接
net.ipv4.tcp_tw_reuse = 1

```