---
layout: post
title: 使用sed和find批量替换文本
categories: [编程, linux]
tags: []
---


> 公司的`nexus`私服改地址了，所有源码(数十个吧)都要相应的修改

#### 1. 使用sed命令替换nexus地址

```
$ sed -i 's/192.168.1.100/172.16.200.100/g' pom.xml
```

#### 2. 结合find命令批量替换

```
$ sed -i 's/192.168.1.100/172.16.200.100/g' $(find workspace -name pom.xml)
```

#### 3. 提交上传到gogs

```
for dir in $(ls)
do
        cd $dir
        git add **/*.pom.xml
        git commit -m "Update nexus address"
        git push origin master
done
```

#### 4. 参考

* [sed命令](https://man.linuxde.net/sed)
* [find命令](https://man.linuxde.net/find)