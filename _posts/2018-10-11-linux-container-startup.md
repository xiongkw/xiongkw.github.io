---
layout: post
title: Linux下使用C语言编写一个容器
categories: [编程, docker, linux]
tags: [container, namespace, cgroups, rootfs]
---


> 容器的实现基于`linux`系统的`namespace cgroups rootfs`，本文演示如何使用`C语言`编写一个容器

#### 概念

* `rootfs`: 根文件系统，`Linux`中可以通过`chroot`命令改变当前根文件系统
* `Namespace`: `Linux`内核对资源的一种分组机制，例如把进程分成不同的组，那么在每个组中都有编号为`1`的进程
* `Cgroups`: `control groups`的缩写，是`Linux`内核提供的一种可以限制、记录、隔离进程组（`process groups`）所使用的物理资源（如：`cpu,memory,IO`等等）的机制

#### 1. 使用`clone`函数启动一个进程

编辑`container.c`

```c++
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>


/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char* const container_args[] = {
    "/bin/bash",
    NULL
};

int container_main(void* arg)
{
    printf("Container [%5d]- inside the container!\n", getpid());
    /* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    printf("Parent - start a container!\n");
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    int container_pid = clone(container_main, container_stack+STACK_SIZE, SIGCHLD, NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

编译运行
```
$ gcc -o container container.c
$ ./container
Parent - start a container!
Container [    4492]- inside the container!
```

在另一个终端通过`pstree`命令查看
```
├─sshd─┬─sshd───bash───pstree
          └─sshd───bash───containerd───bash
```

退出
```
$ exit
exit
Parent - container stopped!
```

#### 2. 制作一个镜像

##### 2.1 制作根文件系统

这里的镜像实际就是一个根文件系统，`copy`需要的文件即可

```
$ pstree image
image
├── bin
│   ├── bash
│   ├── ls
│   └── ps
├── etc
│   └── profile
├── lib64
│   ├── ld-linux-x86-64.so.2
│   ├── libacl.so.1
│   ├── libattr.so.1
│   ├── libbz2.so.1
│   ├── libcap.so.2
│   ├── libc.so.6
│   ├── libdl.so.2
│   ├── libdw.so.1
│   ├── libelf.so.1
│   ├── libgcc_s.so.1
│   ├── libgcrypt.so.11
│   ├── libgpg-error.so.0
│   ├── liblzma.so.5
│   ├── libm.so.6
│   ├── libpcre.so.1
│   ├── libprocps.so.4
│   ├── libpthread.so.0
│   ├── libresolv.so.2
│   ├── librt.so.1
│   ├── libselinux.so.1
│   ├── libsystemd.so.0
│   ├── libtinfo.so.5
│   └── libz.so.1
├── proc
├── root
└── usr
```

> 这里加入了`bash、ls、ps`三个命令，及其依赖库(`lib64/*.so.*`)
> 可通过命令`ldd [command]`查看命令依赖的库文件

##### 2.2 使用`chroot`命令测试镜像
```
$ chroot image
bash-4.2# pwd
/
bash-4.2# ls
bash: ls: command not found
bash-4.2# bin/ls
bin  etc  lib64  proc  root  usr
bash-4.2# source /etc/profile
bash-4.2# ls
bin  etc  lib64  proc  root  usr
bash-4.2# ps
Error, do this: mount -t proc proc /proc
```

> 可以看到根目录已经切换到了`image`目录

##### 2.3 编译运行

修改源码

```c++
int container_main(void* arg)
{
    printf("Container [%5d]- inside the container!\n", getpid());
    // 修改rootfs
    if(chroot("./image") !=0 || chdir("/root")!=0){
        printf("chroot error");
        return 1;
    }
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
```

编译运行
```
$ gcc -o container container.c
$ ./container
bash-4.2# /bin/ls /
bin  etc  lib64  proc  root  usr
bash-4.2# exit
```

> 可以看到根文件系统已经改变了

#### 3. 进程隔离

进程的隔离简单讲就是只能看见容器内的进程，继续修改源码

```c++
int container_main(void* arg)
{
    printf("Container [%5d]- inside the container!\n", getpid());
    // 通过挂载proc目录隔离进程命名空间
    mount("proc", "image/proc", "proc", 0, NULL);
    if(chroot("./image") !=0 || chdir("/root")!=0){
        printf("chroot error");
        return 1;
    }
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    printf("Parent - start a container!\n");
    int container_pid = clone(container_main, container_stack+STACK_SIZE,
        CLONE_NEWPID| SIGCHLD, NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    // 退出时要卸载proc
    umount("image/proc");
    printf("Parent - container stopped!\n");
    return 0;
}
```

编译运行
```
bash-4.2# /bin/ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
0            1     0  0 08:12 ?        00:00:00 /bin/bash
0            6     1  0 08:12 ?        00:00:00 /bin/ps -ef
```

> 只能看到两个进程，一个是`/bin/bash`，一个是`ps`
> 可通过`mount -t proc`和`umount proc`查看和卸载挂载目录

#### 4. 资源限制

运行`container`，并执行一个死循环

```
$ ./container

$ while : ; do : ; done &
```

在另一终端中查看进程，可以看到`cpu`使用率为`100%`
```
$ top
 PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 1090 root      20   0    9520    480    196 R 100.0  0.0   0:09.33 bash
```

创建`/sys/fs/cgroup/cpu/container`目录，发现`container`目录下自动生成了很多文件
```
$ mkdir /sys/fs/cgroup/cpu/container
$ ls /sys/fs/cgroup/cpu/container
cgroup.clone_children  cgroup.event_control  cgroup.procs  cpuacct.stat  cpuacct.usage  cpuacct.usage_percpu  cpu.cfs_period_us  cpu.cfs_quota_us  cpu.rt_period_us  cpu.rt_runtime_us  cpu.shares  cpu.stat  notify_on_release  tasks
```

修改`cpu`使用限制
```
# 修改cpu限制为20%
$ echo 20000 > cpu.cfs_quota_us
# 添加进程id
$ echo 1090 > tasks
```

再次查看进程状态，发现`cpu`使用率变成了`20%`
```
$ top
 PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 1090 root      20   0    9520    480    196 R  20   0.0   0:15.03 bash
```

#### 参考

* [DOCKER基础技术：LINUX NAMESPACE（上）](https://coolshell.cn/articles/17010.html)
* [DOCKER基础技术：LINUX NAMESPACE（下）](https://coolshell.cn/articles/17029.html)
* [Linux Programmer's Manual                 CLONE(2)](http://man7.org/linux/man-pages/man2/clone.2.html)
