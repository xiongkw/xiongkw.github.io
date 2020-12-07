---
layout: post
title: 通过Launchd实例mac下的开机自启动
categories: [编程, unix]
tags: [mac, launchd]
---

> `mac osx`通过`launchd`实现开机自启动

#### 1. OSX自启动

`mac osx`中有三种自启动方式

* `Login Items`：用户登录时启动
* `StartupItems`：系统启动时启动
* `Launchd`：内核加载完成后启动，类似`linux`中的`init`和`systemd`

> 本文以`Launchd`为例，因为习惯了`linux`下的`systemd`

#### 2. Launchd的例子

##### 2.1 编写plist

编写`/Library/LaunchDaemons/jenkins.plist`

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>KeepAlive</key>
        <true/>
        <key>Label</key>
        <string>jenkins</string>
        <key>Program</key>
        <string>/Users/fool/jenkins/startup.sh</string>
        <key>RunAtLoad</key>
        <true/>
        <key>WorkingDirectory</key>
        <string>/Users/fool/jenkins</string>
        <key>StandardOutPath</key>
        <string>/Users/fool/jenkins/agent.log</string>
        <key>StandardErrorPath</key>
        <string>/Users/fool/jenkins/agent.err</string>
</dict>
</plist>
```

##### 2.2 编写startup.sh

```
#!/bin/zsh

HOME=/Users/fool/jenkins

java -jar $HOME/agent.jar -jnlpUrl http://192.168.1.100:8000/computer/osx-agent/slave-agent.jnlp -secret xxxx -workDir "/Users/fool/jenkins"
~
```

##### 2.3 启动

```
$ launchctl load /Library/LaunchDaemons/jenkins.plist
```

##### 2.4 其它launchctl命令

```
$ launchctl list|grep jenkins
$ launchctl unload /Library/LaunchDaemons/jenkins.plist
```

#### 3. 参考

* [Mac里面如何设置自启动服务](https://www.cnblogs.com/zhepama/p/3853677.html)
