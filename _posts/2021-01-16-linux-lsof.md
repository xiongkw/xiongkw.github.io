---
layout: post
title: Linux下lsof
categories: [编程, linux]
tags: [losf]
---

> 

#### 1. 关于lsof

```
$ lsof -h
lsof 4.87
 latest revision: ftp://lsof.itap.purdue.edu/pub/tools/unix/lsof/
 latest FAQ: ftp://lsof.itap.purdue.edu/pub/tools/unix/lsof/FAQ
 latest man page: ftp://lsof.itap.purdue.edu/pub/tools/unix/lsof/lsof_man
 usage: [-?abhKlnNoOPRtUvVX] [+|-c c] [+|-d s] [+D D] [+|-f[gG]] [+|-e s]
 [-F [f]] [-g [s]] [-i [i]] [+|-L [l]] [+m [m]] [+|-M] [-o [o]] [-p s]
[+|-r [t]] [-s [p:s]] [-S [t]] [-T [t]] [-u s] [+|-w] [-x [fl]] [--] [names]
Defaults in parentheses; comma-separated set (s) items; dash-separated ranges.
  -?|-h list help          -a AND selections (OR)     -b avoid kernel blocks
  -c c  cmd c ^c /c/[bix]  +c w  COMMAND width (9)    +d s  dir s files
  -d s  select by FD set   +D D  dir D tree *SLOW?*   +|-e s  exempt s *RISKY*
  -i select IPv[46] files  -K list tasKs (threads)    -l list UID numbers
  -n no host names         -N select NFS files        -o list file offset
  -O no overhead *RISKY*   -P no port names           -R list paRent PID
  -s list file size        -t terse listing           -T disable TCP/TPI info
  -U select Unix socket    -v list version info       -V verbose search
  +|-w  Warnings (+)       -X skip TCP&UDP* files     -Z Z  context [Z]
  -- end option scan     
  +f|-f  +filesystem or -file names     +|-f[gG] flaGs 
  -F [f] select fields; -F? for help  
  +|-L [l] list (+) suppress (-) link counts < l (0 = all; default = 0)
                                        +m [m] use|create mount supplement
  +|-M   portMap registration (-)       -o o   o 0t offset digits (8)
  -p s   exclude(^)|select PIDs         -S [t] t second stat timeout (15)
  -T qs TCP/TPI Q,St (s) info
  -g [s] exclude(^)|select and print process group IDs
  -i i   select by IPv[46] address: [46][proto][@host|addr][:svc_list|port_list]
  +|-r [t[m<fmt>]] repeat every t seconds (15);  + until no files, - forever.
       An optional suffix to t is m<fmt>; m must separate t from <fmt> and
      <fmt> is an strftime(3) format for the marker line.
  -s p:s  exclude(^)|select protocol (p = TCP|UDP) states by name(s).
  -u s   exclude(^)|select login|UID set s
  -x [fl] cross over +d|+D File systems or symbolic Links
  names  select named files or files on named file systems
Anyone can list all files; /dev warnings disabled; kernel ID check disabled.
```

* 默认 : 没有选项，lsof列出活跃进程的所有打开文件
* 组合 : 可以将选项组合到一起，如-abc，但要当心哪些选项需要参数
* -a : 结果进行“与”运算（而不是“或”）
* -l : 在输出显示用户ID而不是用户名
* -h : 获得帮助
* -t : 仅获取进程ID
* -U : 获取UNIX套接口地址
* -F : 格式化输出结果，用于其它命令。可以通过多种方式格式化，如-F pcfn（进程id、命令名、文件描述符、文件名，并以空终止）

#### 2. 查看文件被什么进程使用

```
# 查看打开catalina.out文件的进程
$ lsof catalina.out
# 查看tomcat目录下所有文件的打开进程
$ lsof +D /app/tomcat 
$ 查看已经被删除的文件
$ lsof |grep deleted
```

#### 3. 查看网络和端口

```
# 查看所有网络连接
$ lsof -nP -i
# 查看ipv4 TCP连接
$ lsof -nP -i 4 -a -i TCP
# 查看指定主机和端口
$ lsof -nP -i @127.0.0.1:8080
# 查看所有监听端口
$ lsof -nP -i -s TCP:LISTEN
```

#### 4. 用户和进程

```
# 根据进程打开文件句柄数排序
$ lsof -n|awk '{print $2}'|sort|uniq -c|sort -nr|more
# 查看用户打开文件
$ lsof -u foo
# 查看其它用户
$ sudo lsof -u ^foo
# 查看用户所有进程
$ lsof -t -u foo
# kill指定用户的所有进程
$ sudo kill -9 `lsof -t -u foo`
# 查看指定进程名打开的文件
$ lsof -c java
# 查看指定进程ID打开的文件
$ lsof –p PID
```

#### 5. 常见用法

```
# 查看指定进程打开的监听端口
$ lsof -p ${PID} -nP -a -i -s TCP:LISTEN
```
