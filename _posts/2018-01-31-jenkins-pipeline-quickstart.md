---
layout: post
title: Jenkins-Pipeline入门
categories: [编程, jenkins]
tags: [pipeline, CI, CD, groovy]
---

> `Jenkins Pipeline`, 流水线，是一套`jenkins`插件，用于提供持续交付(`CD`)的能力

#### 1. 简介

* `CI Continuous Integration`：持续集成，指的是开发阶段对代码提交的持续集成。随着项目和开发人员规模的扩大，团队成员之间的协作沟通更加频繁，每个人提交的代码都可能会影响到其它人甚至整个系统。持续集成提出早集成和常集成的要求，通常包括自动化的编译-测试-发布-验证等过程，旨在帮助项目尽早发现集成中的问题。
* `CD Continuous Delevery`：持续交付，是一系列的开发实践方法，用来确保代码能够快速、安全的部署到生产环境中，它通过将每一次改动都提交到一个模拟产品环境中，使用严格的自动化测试，确保业务应用和服务能符合预期。

`CI`和`CD`的区别：持续集成解决的是软件集成的问题，其包含代码提交到编译-测试-集成，其产出是代码。持续交付则包含从代码提交到集成-测试-生产这一系列过程，其产出是软件。可见`CD`了涵盖了`CI`的范围。

`Jenkins Pipeline`则是用于提供持续交付能力的一套插件，发布于`jenkins 2.0`版本，`jenkins`早期版本提供的是持续集成的能力。

#### 1. Jenkins安装
参考[Jenkins安装]({{ site.url}}/2018/01/29/jenkins-install/)

#### 2. hello pipeline
在`jenkins`管理页面新建一个`pipeline`任务，进入任务配置页面

在流水线脚本中输入

```groovy
node {
   echo 'Hello Pipeline'
}
```
保存，并点击`立即构建`按钮，查看构建结果
```
Started by user Administrator
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] node
Running on 192.168.1.100 in /home/fool/jenkins-2.89.3/workspace/workspace/hellopipeline
[Pipeline] {
[Pipeline] echo
Hello Pipeline
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

> `Pipeline`脚本使用的是`groovy`语言

#### 3. pipeline语法

以下是一段`pipeline`：

```groovy
pipeline {
    agent any
    tools {
        maven 'apache-maven-3.5.2'
        jdk 'jdk1.8.0_60'
    }
    stages {
        stage('Non-Parallel Stage') {
            steps {
                echo 'This stage will be executed first.'
            }
        }
        stage('Parallel Stage'){
            parallel {
                stage('Build') {
                    agent {
                        label 'java'
                    }
                    steps {
                        echo 'Building..'
                    }
                    post { 
                        success { 
                            echo pwd()
                        }
                    }
                }
                stage('Test') {
                    agent {
                        label 'java'
                    }
                    steps {
                        echo 'Testing..'
                    }
                }
                stage('Deploy') {
                    agent {
                        label 'java'
                    }
                    steps {
                        echo 'Deploying....'
                    }
                }
            }
        }
    }
    post { 
        always { 
            echo 'The end'
        }
    }
}
```

- `pipeline`：`pipeline`入口，`Jenkinsfile`的根节点
- `agent`：指定在`Jenkins`集群中某个节点上运行，可定义在`pipeline`或者`stage`中
- `tools`：指定执行`stage`的工具，例如`jdk、maven、gradle`，必须先安装在指定目录，并在`jenkins全局工具配置`中预先配置好
- `stages`：舞台集合，定义在`pipeline`内，且只能定义一个
- `stage`：舞台，可定义在`stages`或`parallel`内，可定义多个，是每个`Jenkins`节点执行任务的最小单元，即一个`state`会作为一个整体分发到一个`Jenkins`节点
- `steps`：阶段内的具体执行步骤
- `parallel`：并行，`pipeline`中的`stage`是按照严格的先后顺序串行执行的，`parallel`可以使多个`stage`并发运行
- `post`：后置动作，可定义在`pipeline`或`stage`内

更多语法请参考[Pipeline Syntax](https://jenkins.io/doc/book/pipeline/syntax/)

#### 4. 参考文档
* [Installing Jenkins](https://jenkins.io/doc/book/installing/)
* [Jenkins官方文档](https://www.w3cschool.cn/jenkins/)
* [Apache Groovy](http://groovy-lang.org/documentation.html)
* [Blue Ocean](https://jenkins.io/doc/book/blueocean/)
* [Declarative Pipeline for Maven Projects](https://jenkins.io/blog/2017/02/07/declarative-maven-project/)