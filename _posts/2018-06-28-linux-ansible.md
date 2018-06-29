---
layout: post
title: Ansible入门
categories: [编程, linux]
tags: [ansible]
---


> `Ansible`是一个简单的自动化运维管理工具，基于`Python`语言实现，由`Paramiko`和`PyYAML`两个关键模块构建，可用于自动化部署应用、配置、编排task(持续交付、无宕机更新等)

#### 1. 安装

`CentOS`下直接通过`yum`安装
```
$ yum install ansible -y
```

检查是否安装成功
```
$ ansible -version

ansible 2.5.5
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/devops/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Apr 11 2018, 07:36:10) [GCC 4.8.5 20150623 (Red Hat 4.8.5-28)]

```

> `Ansible` 依赖`ssh` 和 `python(2.6+ or 3.5+)`

#### 2. 快速开始

```
$ tree /etc/ansible

/etc/ansible
├── ansible.cfg
├── hosts
└── roles
```

编辑hosts文件，添加`test`组
```
$ vi /etc/ansible/hosts

# test组
[test]
192.168.1.100

# test组的参数
[test:var]
# ssh 用户
ansible_ssh_user=test
# ssh 密码
ansible_ssh_pass=test
```

测试
```
$ ansible test -m ping

192.168.1.100 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

```

> `ansible test -m ping`: 对`hosts`中定义的`test`组内的所有主机执行`ping`模块，`ping`模块是`ansible`的内置模块，用于检查主机是否可以正常`ssh`访问

使用sudo执行命令

```
$ echo "ansible_become_pass=test" >> hosts

$ ansible test -b -m shell -a 'ls /'
```

> `ansible_become_pass=test`: 配置`sudo`密码
> `ansible -b`: 使用`sudo`执行

#### 3. 常用模块

##### 3.1 command

> 在远程主机上执行`shell`命令，不支持管道符

```
$ ansible test -m command -a 'free'

192.168.1.100 | SUCCESS | rc=0 >>
              total        used        free      shared  buff/cache   available
Mem:      528273972    79506384    10236588     4319320   438531000   424742324
Swap:       4194300         128     4194172
```

##### 3.2 script

> 在远程主机上执行`本机`上的`shell`脚本，相当于`scp + shell`的组合

```
$ vi test.sh

#!/bin/bash

free

$ chmod +x test.sh

$ ansible test -m script -a 'test.sh'

192.168.1.100 | SUCCESS => {
    "changed": true,
    "rc": 0,
    "stderr": "Shared connection to 192.168.1.100 closed.\r\n",
    "stdout": "              total        used        free      shared  buff/cache   available\r\nMem:      528273972    79494548    10184296     4319320   438595128   424754156\r\nSwap:       4194300         128     4194172\r\n",
    "stdout_lines": [
        "              total        used        free      shared  buff/cache   available",
        "Mem:      528273972    79494548    10184296     4319320   438595128   424754156",
        "Swap:       4194300         128     4194172"
    ]
}
```

##### 3.3 copy

> 复制文件到指定主机，相当于`scp`

```
$ ansible test -m copy -a 'src=test.sh dest=~/ mode=755'

192.168.1.100 | SUCCESS => {
    "changed": true,
    "checksum": "d2df40b39009a8a93bf872509ff71ec268ee8cde",
    "dest": "/home/test/test.sh",
    "gid": 1002,
    "group": "test",
    "md5sum": "f6483fb37833d41ac151d471ca9cccee",
    "mode": "0755",
    "owner": "test",
    "size": 18,
    "src": "/home/test`/.ansible/tmp/ansible-tmp-1530153652.33-71523009530091/source",
    "state": "file",
    "uid": 1002
}
```

##### 3.4 shell

> 执行远程主机上的`shell`脚本，支持管道符

```
$ ansible test -m shell -a '~/test.sh'

192.168.1.100 | SUCCESS | rc=0 >>
            total        used        free      shared  buff/cache   available
Mem:      528273972    79518764    10155564     4319352   438599644   424729484
Swap:       4194300         128     4194172
```

##### 3.5 get_url

> 从指定url下载文件到目标主机

```
$ ansible test -m get_url -a 'url=http://192.168.1.101:8080/hello dest=~/test'

192.168.1.101 | SUCCESS => {
    "changed": true,
    "checksum_dest": null,
    "checksum_src": "6d01ee8a8f3e2e901435250a2b94cb184aa1dcc9",
    "dest": "/home/test/test",
    "gid": 1002,
    "group": "test",
    "md5sum": "1dfe6ee180e788dd129e56ea0c82cc5e",
    "mode": "0664",
    "msg": "OK (433 bytes)",
    "owner": "test",
    "size": 433,
    "src": "/tmp/tmpfxHxOs",
    "state": "file",
    "status_code": 200,
    "uid": 1002,
    "url": "http://192.168.1.101:8200"
}
```

##### 3.6 file

> 管理目标主机上文件

```
$ ansible test -m file -a 'path=~/test state=absent'

192.168.1.100 | SUCCESS => {
    "changed": true,
    "path": "/home/test/test",
    "state": "absent"
}
```

##### 3.7 yum

> 在目标主机上执行`yum`命令

```
$ ansible test -m yum -a 'name=curl state=latest'

192.168.1.100 | SUCCESS => {
    "changed": false,
    "msg": "",
    "rc": 0,
    "results": [
        "All packages providing curl are up to date",
        ""
    ]
}
```

##### 3.8 cron

> 管理目标主机`crontab`

```
$ ansible test -m cron -a "name=test minute=* job='free >/dev/null'"

192.168.1.100 | SUCCESS => {
    "changed": true,
    "envs": [],
    "jobs": [
        "test"
    ]
}
```

查看目标主机crontab

```
$ crontab -l

#Ansible: test
* * * * * free >/dev/null
```

##### 3.9 service

> 管理目标主机服务

```
$ ansible test -m service -a 'name=test state=restarted'

192.168.1.100 | FAILED! => {
    "changed": false,
    "msg": "Could not find the requested service test: host"
}
```

##### 3.10 user

> 管理目标主机用户

```
$ ansible test -m user -a  'name=test password=test'

192.168.1.100 | SUCCESS => {
    "changed": true,
    "comment": "",
    "createhome": true,
    "group": 12,
    "home": "/home/test",
    "name": "test",
    "password": "NOT_LOGGING_PASSWORD",
    "shell": "/bin/bash",
    "state": "present",
    "uid": 12
}
```

> 更多模块参考[Module Index](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)

#### 4. playbook(剧本)

> `playbook`是一系列命令的集合，用于完成一组复杂的任务，以下是一个安装`nginx`的例子

##### 4.1 文件结构

```
$ tree /etc/ansible

├── ansible.cfg
├── hosts
├── install_nginx.yml
└── roles
    └── install_nginx
        ├── files
        │   └── nginx-1.13.8.tar.gz
        ├── tasks
        │   └── main.yml
        ├── templates
        │   └── nginxd
        └── vars
            └── main.yml
```

##### 4.2 hosts配置

```yaml

[test]
192.168.1.100

[test:vars]
ansible_ssh_user=test
ansible_ssh_pass=test
ansible_become_pass=test
```

##### 4.3 入口

`install_nginx.yml`

```yaml
---
- hosts: test
  roles:
  - install_nginx
```

##### 4.4 任务定义

`roles/install_nginx/tasks.yml`

```yaml
---
  - name: Install nginx
    become: true
    yum:
     name: "{{ item }}"
     state: present
    with_items:
    - openssl-devel
    - zlib-devel
  - name: Create nginx user
    become: true
    user:
      name: "{{ nginx_user }}"
      password: "{{ nginx_password }}"
      state: present
      group: root
  - name: Copy nginx source code
    become: true
    become_user: "{{ nginx_user }}"
    copy:
       src: nginx-{{ nginx_version }}.tar.gz
       dest: /home/nginx
       force: no
  - name: Unarchive nginx source code
    become: true
    become_user: "{{ nginx_user }}"
    shell: tar zxvf ~/nginx-{{ nginx_version }}.tar.gz -C ~/
  - name: Create nginx dir
    become: true
    file:
        path: "{{ nginx_dir }}"
        state: directory
        owner: "{{ nginx_user }}"
  - name: Configure nginx
    become: true
    become_user: "{{ nginx_user }}"
    shell: cd ~/nginx-{{ nginx_version }} && ./configure --prefix={{ nginx_dir }}
  - name: Install nginx
    become: true
    become_user: "{{ nginx_user }}"
    shell: cd /home/nginx/nginx-{{ nginx_version }} && make && make install
  - name: Clean nginx source code
    become: true
    shell: rm -rf /home/nginx/nginx-{{ nginx_version }}*
  - name: Copy nginxd script
    become: true
    template:
        src: nginxd
        dest: /etc/init.d
        mode: "0755"
        owner: "{{ nginx_user }}"
  - name: Start nginxd service
    become: true
    become_user: "{{ nginx_user }}"
    service:
        name: nginxd
        state: started
  - name: Add boot start nginxd service
    become: true
    shell: chkconfig --level 35 nginxd on
```

##### 4.5 模板文件

`roles/install_nginx/templates/nginxd`

```
#!/bin/bash

# chkconfig: 2345 10 90
# description: nginxd ...

prog="nginx"

start() {
    echo -n $"Starting $prog: "
    {{ nginx_dir }}/sbin/nginx
}

stop() {
    echo -n $"Stopping $prog: "
    PID=`cat {{ nginx_dir }}/logs/nginx.pid`
    if [[ -n "$PID" ]]; then
        kill $PID
    fi
}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    *)
        echo "Usage: $0 {start|stop}"
        RETVAL=1
esac

exit $RETVAL
```

##### 4.6 变量定义

`roles/install_nginx/vars/main.yml`

```yaml
nginx_user: nginx
nginx_password: nginx
nginx_dir: /apps/nginx
nginx_version: 1.13.8
```

##### 4.7 运行

```
$ ansible-playbook install_nginx.yml
```

#### 5. 参考

* [ANSIBLE PROJECT](https://docs.ansible.com)

* [Ansible 权威指南](http://www.ansible.com.cn/)