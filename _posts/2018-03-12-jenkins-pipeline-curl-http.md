---
layout: post
title: jenkins pipeline中使用curl命令实现参数化的http调用
categories: [编程, linux]
tags: [curl, http, jenkins, pipeline]
---


> 

#### 1. 场景
使用`jenkins`调用`http restful`接口，以下仅作演示

#### 1. job创建
在`jenkins`中创建一个`pipeline job`，详情略

#### 2. 参数定义
在`1`中创建`job`中配置参数

|参数类型   |   参数名  |  说明 |
| -------- | -------------------| ------------------------------ |
|String    |   URL         |     http url         |
|String    |   TOKEN         |     http请求头，用于身份验证         |
|String    |   APP_NAME         |     app name         |
|String    |   APP_DESC         |     app description         |
|Multi-line String| CONFIG_FILES   | 配置文件(多个,形如"file1 content1 file2 content2")       |

#### 3. 定义Pipeline script

```
pipeline {
    agent any
    stages {
        stage('check app') {
            steps {
                echo 'Stage 1: Checking app...'
                sh '''
				set +x
                check_app(){
					echo "Checking app [$3]..."
					echo -e "Request:\ncurl -X HEAD -H \\"token:$2\\" -Iis $1/api/apps/$3"

					res=`curl -X HEAD -H "token:$2" -Iis $1/api/apps/$3`
					echo -e "Response:\n$res"

					status=`echo "$res" | awk '/^HTTP/  {print $2}'`
					echo "Status: $status"

					if [[ $status = 200 ]];then
							echo "App [$3] exists"
					elif [[ $status = 404 ]];then
							echo "App [$3] not exists, creating..."
							echo -e "Request\ncurl -X POST -d \'{\\"appName\\":\\"$3\\",\\"appDesc\\":\\"$4\\"}\' -H \\"token:$4\\" -H \\"Content-Type: application/json\\" -is $1/api/apps"

							res=`curl -X POST -d '{"appName":"'"$3"'","appDesc":"'"$4"'"}' -H "token:$4" -H "Content-Type: application/json" -is $1/api/apps`
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
                check_app "${URL}" "${TOKEN}" "${APP_NAME}" "${APP_DESC}"

                '''
            }
        }
        stage('upload config files') {
            steps {
                echo 'Stage 2: Uploading config files...'
                sh '''
				set +x
                upload_configFile(){
					echo "Checking config file [$3]..."
					echo -e "Request:\ncurl -X HEAD -H \\"token:$2\\" -Iis $1/api/files/$3"

					res=`curl -X HEAD -H "token:$2" -Iis $1/api/files/$3`
					echo -e "Response:\n$res"

					status=`echo "$res" | awk '/^HTTP/  {print $2}'`
					echo "Status: $status"

					if [[ $status = 200 ]];then
							echo "Config file [$3] exists, updating..."
							echo -e "Request\ncurl -X PUT -d @$3 -H \\"Content-Type: application/json\\" -is $1/api/files/$3"

							res=`curl -X PUT -d @$3 -H "token:$4" -H "Content-Type: application/json" -is $1/api/files/$3`
							echo -e "Response:\n$res"

							status=`echo $res | awk '/^HTTP/  {print $2}'`
							echo "Status: $status"

							if [[ $status = 200 ]]; then
									echo "Config file [$3] updated"
							else
									echo "Update config file [$3] failed"
									exit 1
							fi
					elif [[ $status = 404 ]];then
							echo "Config file [$3] not exists, creating..."
							echo -e "Request\ncurl -X POST -d @$3 -H \\"token:$4\\" -H \\"Content-Type: application/json\\" -is $1/api/files"

							res=`curl -X POST -d @$3 -H "token:$4" -H "Content-Type: application/json" -is $1/api/files`
							echo -e "Response:\n$res"

							status=`echo $res | awk '/^HTTP/  {print $2}'`
							echo "Status: $status"

							if [[ $status = 200 ]]; then
									echo "Config file [$3] created"
							else
									echo "Create config file [$3] failed"
									exit 1
							fi
					else
							echo "Check config file [$3] error"
							exit 1
					fi
				}
				
				fileArray=(${CONFIG_FILES})
				i=0
                while [ $i -lt ${#fileArray[@]} ]
                do
                  fileName=${fileArray[$i]}
                  echo ${fileArray[$i+1]} | base64 -d > $fileName
                  upload_configFile "${URL}" "${TOKEN}" "$fileName"
                  let i+=2
                done
                '''
            }
        }

    }
}
```

解释：

* 该`pipeline`分为`check app`和`upload files`两个`stage`, `check app`的逻辑是检查应用是否存在，如果不存在则创建，`upload files`的则是上传多个配置文件
* `${}`: `jenkins pipeline`中使用参数的语法，例如`${APP_NAME}`
* `check app`中首先使用`curl -X HEAD`请求检查应用是否存在，若不存在(`404`)，则使用`curl -X POST -d '{"appName":"'"$3"'","appDesc":"'"$4"'"}'`请求创建应用
* `files`接口接收的参数为`{"name":"a.txt","content":"xx"}`, 由于接口不支持批量文件上传，所以这里的思路是在后台把多个文件转换成`file1 content1 file2 content2`的格式，然后在`shell`中将其转换为字符串数组，再遍历处理每一个文件
* `fileArray=(${CONFIG_FILES})`: 把形如`file1 content1 file2 content2`的参数转换为数组
* `while [ $i -lt ${#fileArray[@]} ]`: 遍历数组，`${#fileArray[@]}`表示数组`size`
* `fileName=${fileArray[$i]}`: 解析出文件名
* `echo ${fileArray[$i+1]} | base64 -d > $fileName`: 使用`base64`解码文件内容，原因是文件可能是二进制格式，或者文件中包含空格等特殊字符
* `curl -X PUT -d @$3`: `@`的作用是从文件`$3`中加载`PUT`调用的`data`

#### 4. 参数化构建

```
curl -X POST -d 'DCOS_URL=http://127.0.0.1:8088&TOKEN=xxxx&APP_NAME=myapp&APP_DESC=my app&CONFIG_FILES=a.txt aGVsbG8K b.txt aGkK' http://127.0.0.1:8080/job/myJob/buildWithParameters
```

> `-d`参数默认会自动`url encode`

#### 5. 参考
* [Pipeline Syntax](https://jenkins.io/doc/book/pipeline/syntax/)
* [使用curl命令实现复杂的http调用]({{site.url}}/2018/03/08/curl-http-request)