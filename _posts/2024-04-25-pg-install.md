---
layout: post
title: Postgres使用
categories: [linux]
tags: []
---

> 

#### 1. 开始
`PostgreSQL`号称是世界上最先进最流行的开源数据库，本文使用容器来部署`postgres`和`pgadmin`

* `postgres`: PostgreSQL服务端
* `psql`: PostgreSQL的命令行客户端
* `pgadmin`: PostgreSQL Web管理页面


#### 2. 拉取镜像

```
# pg后端服务
$ docker pull postgres
# web管理页面
$ docker pull dpage/pgadmin4
```

#### 3. 运行

```
$ docker run --name postgres \
       	-p 15432:5432 \
	-e 'POSTGRES_PASSWORD=123456' \
	-v /data/postgres:/var/lib/postgresql/data \
	-d postgres
$ docker run --name pgadmin4 \
    -p 15433:80 \
    -v /data/pgadmin:/var/lib/pgadmin \
    -e 'PGADMIN_DEFAULT_EMAIL=postgres@postgres.com' \
    -e 'PGADMIN_DEFAULT_PASSWORD=123456' \
    -d dpage/pgadmin4

```

#### 4. psql使用
```
# 进入容器终端
$ docker exec -it postgres bash
# 切换到postgres用户
root@ea997adffada:/# su postgres
$ 执行psql
postgres@ea997adffada:/$ psql
psql (16.2 (Debian 16.2-1.pgdg120+2))
Type "help" for help.

postgres=#
```
#### 5. pgadmin使用

浏览器访问`http://127.0.0.1:15433`

#### 6. 参考

* [PostgreSQL](https://www.postgresql.org/download/)
* [pgadmin](https://www.pgadmin.org/docs/pgadmin4/latest/container_deployment.html)