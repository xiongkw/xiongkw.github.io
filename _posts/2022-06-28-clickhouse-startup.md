---
layout: post
title: ClickHouse试用
categories: [linux]
tags: []
---

>  

#### 1. 下载安装

> `ClickHouse`提供`DEB、RPM、Docker、Tgz`等多种格式安装包，本文使用[Tgz](https://packages.clickhouse.com/tgz/)安装包

```
export LATEST_VERSION=22.5.1.2079-amd64
curl -O "https://packages.clickhouse.com/tgz/stable/clickhouse-common-static-$LATEST_VERSION.tgz"
curl -O "https://packages.clickhouse.com/tgz/stable/clickhouse-common-static-dbg-$LATEST_VERSION.tgz"
curl -O "https://packages.clickhouse.com/tgz/stable/clickhouse-server-$LATEST_VERSION.tgz"
curl -O "https://packages.clickhouse.com/tgz/stable/clickhouse-client-$LATEST_VERSION.tgz"

tar -xzvf "clickhouse-common-static-$LATEST_VERSION.tgz"
sudo "clickhouse-common-static-$LATEST_VERSION/install/doinst.sh"

tar -xzvf "clickhouse-common-static-dbg-$LATEST_VERSION.tgz"
sudo "clickhouse-common-static-dbg-$LATEST_VERSION/install/doinst.sh"

tar -xzvf "clickhouse-server-$LATEST_VERSION.tgz"
sudo "clickhouse-server-$LATEST_VERSION/install/doinst.sh"

tar -xzvf "clickhouse-client-$LATEST_VERSION.tgz"
sudo "clickhouse-client-$LATEST_VERSION/install/doinst.sh"
```

#### 2. 配置

##### 2.1 设置server参数

修改/etc/clickhouse-server/config.xml

```
<http_port>8123</http_port>
<tcp_port>9000</tcp_port>
<mysql_port>9004</mysql_port>
<postgresql_port>9005</postgresql_port>
<interserver_http_port>9009</interserver_http_port>
<listen_host>::</listen_host>
<max_concurrent_queries>100</max_concurrent_queries>
<max_thread_pool_size>10000</max_thread_pool_size>
<max_server_memory_usage>0</max_server_memory_usage>
<max_server_memory_usage_to_ram_ratio>0.5</max_server_memory_usage_to_ram_ratio>
<max_open_files>262144</max_open_files>
<path>/var/lib/clickhouse/</path>
```

##### 2.1 设置用户密码

修改/etc/clickhouse-server/users.xml

```
<users>
	<default>
		<password>clickhouse@123</password>
	</default>	
</users>
```

#### 3. 启动server

使用systemd服务启动

```
[Unit]
Description=ClickHouse service
After=network.target
Wants=network.target

[Service]
LimitCORE=infinity
Type=forking
User=clickhouse
ExecStart=/bin/clickhouse-server -C /etc/clickhouse-server/config.xml --daemon
Restart=on-failure
TimeoutSec=300

[Install]
WantedBy=multi-user.target
```

#### 4. 客户端连接

#### 4.1 clickhouse-client

官方客户端，连接tcp端口

```
$ clickhouse-client -h 127.0.0.1 --port 9000 -u default --password 'clickhouse@123'
```

#### 4.2 使用DBeaver

新建ClickHouse连接，并使用默认驱动，连接报错`can't load driver class 'com.clickhouse.jdbc.clickhousedriver'`

猜测是maven方式的驱动依赖问题

解决方法：手工下载[clickhouse-jdbc及其依赖jar包](https://jar-download.com/artifacts/ru.yandex.clickhouse/clickhouse-jdbc)，并把所有依赖包加入驱动lib


#### 5. 参考

* [ClickHouse](https://clickhouse.com/docs/zh)