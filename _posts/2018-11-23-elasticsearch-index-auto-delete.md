---
layout: post
title: ElasticSearch索引定时删除
categories: [编程, java, linux]
tags: [elasticsearch, crontab]
---

> 使用`ElasticSearch`存储业务日志，使用日期区分索引，因为磁盘不够用，所以要定时删除`7`天前的索引

#### 1. 编写删除脚本

编写脚本`clearEsIndices.sh`

```
#!/bin/bash

DATE=`date -d "-7 day" +%Y-%m-%d`
pattern=log-*-${DATE}

echo "Deleting indices: [${pattern}] ..."

res=`curl -X DELETE -is http://192.168.1.100:9200/${pattern}`
status=`echo "$res" | awk '/^HTTP/  {print $2}'`

if [[ $status = 200 ]];then
        echo "Delete $pattern success"
else
        echo "Delete $pattern error"
        echo -e "Request:\ncurl -X DELETE http://192.168.1.100:9200/${pattern}"
        echo -e "Response:\n$res"
fi
```

#### 2. 配置定时器

```
$ crontab -e
0  1  *  *  *  /apps/tasks/clearEsIndices.sh >> /apps/tasks/clearEsIndices.log 2>&1 
```

或者

```
$ echo "0  1  *  *  *  foo  /apps/tasks/clearEsIndices.sh >> /apps/tasks/clearEsIndices.log 2>&1" >> /etc/crontab
```