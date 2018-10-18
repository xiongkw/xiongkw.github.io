---
layout: post
title: etcd入门
categories: [编程, linux]
tags: [etcd]
---


> `etcd`是一个用`Go`语言开发的高可用分布式键值(`key-value`)数据库

#### 1.下载安装

到官网下载指定版本[Etcd release](https://github.com/etcd-io/etcd/releases)

```
$ tar zxvf etcd-v3.3.10-linux-amd64.tar.gz -C /usr/local

$ export PATH=$PATH:/usr/local/etcd-v3.3.10-linux-amd64
```

#### 2. 配置

从官网源码中获取配置样例[etcd.conf.yml.sample](https://github.com/etcd-io/etcd/blob/master/etcd.conf.yml.sample)

```yaml
# This is the configuration file for the etcd server.

# Human-readable name for this member.
name: 'myetcd'

# Path to the data directory.
data-dir: /home/foo/data/myetdc.data

# List of comma separated URLs to listen on for peer traffic.
listen-peer-urls: http://localhost:2381

# List of comma separated URLs to listen on for client traffic.
listen-client-urls: http://localhost:2378
```

> 更多配置参考[Configuration flags](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/configuration.md)

#### 3. 启动

```
nohup etcd --config etcd.conf >etcd.log 2>&1 &
```

#### 4. etcdctl

简单的读写操作
```
# v3和v2版的api差别非常大，这里使用v3版
$ export ETCDCTL_API=3
$ etcdctl --endpoints=127.0.0.1:2378 get foo
$ etcdctl --endpoints=127.0.0.1:2378 put foo "Hello world"
OK
$ etcdctl --endpoints=127.0.0.1:2378 get foo
foo
Hello world
```

> 参考[Interacting with etcd](https://coreos.com/etcd/docs/latest/dev-guide/interacting_v3.html#read-keys)
> [Demo](https://coreos.com/etcd/docs/latest/demo.html)

#### 5. rest

`http post`写入
```
# k/v必须是Base64编码
$ echo -n foo|base64
Zm9v
$ echo -n "Hello world"|base64
SGVsbG8gd29ybGQ=
$ curl -L http://127.0.0.1:2378/v3beta/kv/put -X POST -d '{"key": "Zm9v", "value": "SGVsbG8gd29ybGQ="}'
{"header":{"cluster_id":"14841639068965178418","member_id":"10276657743932975437","revision":"4","raft_term":"2"}}
```

使用`etcdctl`查看
```
$ etcdctl --endpoints=127.0.0.1:2378 get foo
foo
Hello world
```

`http post`读取
```
$ curl -L http://127.0.0.1:2378/v3beta/kv/range -X POST -d '{"key": "Zm9v"}'
{"header":{"cluster_id":"14841639068965178418","member_id":"10276657743932975437","revision":"4","raft_term":"2"},"kvs":[{"key":"Zm9v","create_revision":"4","mod_revision":"4","version":"1","value":"SGVsbG8gd29ybGQ="}],"count":"1"}
# 解码base64
$ echo -n SGVsbG8gd29ybGQ= |base64 -d
Hello world
```

> 参考[HTTP JSON API through the gRPC gateway](https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/api_grpc_gateway.md)

#### 6. 集群

> TODO

#### 7. 参考
* [etcd Documentation](https://coreos.com/etcd/docs/latest/)
* [etcd](https://github.com/etcd-io/etcd/tree/master)
* [etcd.conf.yml.sample](https://github.com/etcd-io/etcd/blob/master/etcd.conf.yml.sample)
* [Documentation](https://github.com/etcd-io/etcd/blob/master/Documentation/docs.md)