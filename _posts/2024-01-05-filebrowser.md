---
layout: post
title: File Browser
categories: [linux]
tags: []
---

> FileBrowser是一个开源的轻量级文件管理系统

#### 1. 下载安装

在GitHub下载linux版本安装包[v2.27.0](https://github.com/filebrowser/filebrowser/releases/tag/v2.27.0)

```
$ mkdir filebrowser
$ tar zxvf linux-amd64-filebrowser.tar.gz -C filebrowser
```

#### 2. 配置

```
$ mkdir /etc/filebrowser
$ vi /etc/filebrowser/.filebrowser.json
```

.filebrowser.json文件内容：
```json
{
  "port": 8001,
  "baseURL": "",
  "address": "",
  "log": "stdout",
  "database": "/etc/filebrowser/filebrowser.db",
  "root": "/data"
}
```


#### 3. 启动

```
$ nohup ./filebrowser >std.log 2>&1 &
```

#### 4. 访问
浏览器打开127.0.0.1:8001

初始账号admin/admin

#### 5. 参考

* [File Browser](https://github.com/filebrowser/filebrowser)
* [filebrowser](https://filebrowser.org/cli/filebrowser)