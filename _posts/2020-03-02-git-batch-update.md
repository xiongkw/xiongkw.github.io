---
layout: post
title: git仓库批量修改
categories: [编程, linux]
tags: [git]
---


> 由于更新了`nexus`私服地址，所以需要修改所有仓库所有分支的`pom.xml`

#### 1. 编写批量脚本

```
#!/bin/bash

DIR=$(cd $(dirname $0); pwd)

#把gogs账号密码写在URL中，跳过登录步骤
GOGS_ADDR=http://admin:123456@192.168.1.100:8000/

NEXUS_ADDR_OLD=192.168.1.100:8081
NEXUS_ADDR_NEW=172.26.1.202:8081

#每个仓库有多个不同分支
BRANCHES=(master develop dev)

#仓库列表
ARR_REPO=(test/web test/server myproject/myservice)

updateRepoBranch(){
	branch = $1
	git checkout $branch
	
	#使用sed 命令替换
	sed -i "s/$NEXUS_ADDR_OLD/$NEXUS_ADDR_NEW/g" pom.xml
	
	#提交并push
	git add pom.xml && git commit -m "refactor: Update nexus address by admin" && git push origin $branch
}	

updateRepo(){
	cd $DIR
	repo_name=$1
	dir_name=${repo_name##*/}
	git clone $GOGS_ADDR$repo_name
	cd $dir_name
	
	#遍历每个分支
	for branch in ${BRANCHES[@]}
	do
		updateRepoBranch $repo_name
	done
}

#遍历每个仓库
for repo in ${ARR_REPO[@]}
do
	update $repo
done

```