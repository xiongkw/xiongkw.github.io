---
layout: post
title: Ubuntu下tcpdump问题
categories: [编程, linux]
tags: [ubuntu, tcpdump]
---

> 

#### 1. 现象

```
$ tcpdump
tcpdump: error while loading shared libraries: libc.so.6: failed to map segment from shared object
```

#### 2. 查看内核日志
```
$ grep tcpdump /var/log/syslog
Nov 16 15:25:08 EmuBuilder kernel: [5458886.366507] audit: type=1400 audit(1605511508.321:1421): apparmor="DENIED" operation="file_mmap" profile="/usr/sbin/tcpdump" name="/root/vd-agent/vdagent-deps/lib/aarch64-linux-gnu/libc-2.27.so" pid=7606 comm="tcpdump" requested_mask="m" denied_mask="m" fsuid=0 ouid=0
```

#### 3. 查看tcpdump模式
```
$grep tcpdump /sys/kernel/security/apparmor/profiles
/usr/sbin/tcpdump ( enforce)
```

#### 4. 修改为complain模式
```
$ apt install apparmor-utils
$ aa-complain /usr/sbin/tcpdump
$ tcpdump
```

> 改为complain模式后tcpdump正常了

#### 5. 切换到enforce模式
```
$ aa-enforce /usr/sbin/tcpdump
```

