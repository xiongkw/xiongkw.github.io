---
layout: post
title: 使用curl命令实现复杂的http调用
categories: [编程, linux]
tags: [curl, http]
---


> `curl`是利用`URL`语法在命令行方式下工作的开源文件传输工具，在`*nix`下我们常常使用`curl`来发送请求`http`

#### 1. 一个例子

```
#!/bin/bash

check_app(){
        echo "Checking app [$3]..."
        echo -e "Request:\ncurl -X HEAD -H \"token:$2\" -Iis $1/api/apps/$3"

        res=`curl -X HEAD -H "token:$4" -Iis $1/api/apps/$3`
        echo -e "Response:\n$res"

        status=`echo "$res" | awk '/^HTTP/  {print $2}'`
        echo "Status: $status"

        if [[ $status = 200 ]];then
                echo "App [$3] exists"
        elif [[ $status = 404 ]];then
                echo "App [$3] not exists, creating..."
                echo -e "Request\ncurl -X POST -d '{\"appName\":\"$6\",\"appDesc\":\"$7\"}' -H \"token:$4\" -H \"Content-Type: application/json\" -is $1/api/apps"

                res=`curl -X POST -d '{"appName":"'$3'","appDesc":"'"$4"'"}' -H "token:$4" -H "Content-Type: application/json" -is $1/api/apps`
                echo -e "Response:\n$res"

                status=`echo $res | awk '/^HTTP/  {print $2}'`
                echo "Status: $status"

                if [[ $status = 200 ]]; then
                        echo "App [$3] created"
                else
                        echo "Create app [$3] failed"
                        exit 1
                fi
        else
                echo "Check app [$3] error"
                exit 1
        fi
}

check_app 'http://127.0.0.1:8080/' 'xxxxx' 'myapp' 'my app'
```

#### 2. 说明

* `check_app(){}`: 定义一个函数，`shell`中的函数不需要声明参数
* `echo "$3"`: `echo ""`和`echo ''`的区别是`双引号`内的`$xx`会被当作变量解析
* `echo -e`: `-e`参数的作用是打印`特殊字符`，例如`\n`会打印换行，`\"`会打印`"`
* `curl -X HEAD`: `-x`参数指定`http`请求方法
* `curl -H "userName:xxx"`: `-H`参数指定`http header`，注意多个`header`需要用多个`-H`指定
* `awk '/^HTTP/  {print $2}'`: 使用`awk`命令解析出`http`响应的`状态码`
* `curl -d '{"appName":"'$3'","appDesc":"'"$4"'"}'`: `"'$3'"`的作用是使用`'`把`$3`隔开，从而使得`$3`被当作变量解析；而`"'"$4"'"`的作用则是使用`"`把变量`$4`括起来，从而使得`$4`的值可以包含空格
* `check_app '' ''`: 函数调用