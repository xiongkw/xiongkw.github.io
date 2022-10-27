---
layout: post
title: jmeter压力测试中use_keepalive的坑
categories: [编程, web, 性能测试]
tags: [jmeter, keepalive]
---

> 使用`jmeter use_keepalive 2000`线程压测`jetty`，查看网络连接状况，发现网络连接不断重连

#### 1. jmeter配置
`test.jmx`
```xml
<elementProp name="ThreadGroup.main_controller" elementType="LoopController" guiclass="LoopControlPanel" testclass="LoopController" testname="循环控制器" enabled="true">
  <boolProp name="LoopController.continue_forever">false</boolProp>
  <stringProp name="LoopController.loops">-1</stringProp>
</elementProp>
<stringProp name="ThreadGroup.num_threads">2000</stringProp>
  

<boolProp name="HTTPSampler.use_keepalive">true</boolProp>
<stringProp name="HTTPSampler.implementation">HttpClient4</stringProp>

```

#### 2. 问题

运行`jmeter`
```
jmeter.sh -n -t test.jmx
```

查看网络连接
```
netstat -an| grep 8777|awk '/^tcp/ {state[$6]++} END {for(key in state) print key, state[key] }'

LAST_ACK 1
LISTEN 1
SYN_RECV 2
CLOSE_WAIT 19
ESTABLISHED 1766
```

> `tcp`不断重连，说明`keepalive`没起作用


#### 3. 使用ab测试

同样用`ab`压`2000`并发

```
ab -n 10000 -c 2000 -p input -T "application/json" -k -r http://192.168.1.100:8777/services/test
```

```
netstat -an| grep 8777|awk '/^tcp/ {state[$6]++} END {for(key in state) print key, state[key] }'
ESTABLISHED 2000
```

> `tcp`连接保持稳定

#### 4. 改正

说明问题在`jmeter`，于是查看`jmeter.properties`

```properties
# Idle connection timeout (Milliseconds) to apply if the server does not send
# Keep-Alive headers (default 0)
# Set this > 0 to compensate for servers that don't send a Keep-Alive header
# If <= 0, idle timeout will only apply if the server sends a Keep-Alive header
httpclient4.idletimeout=60000

# TTL (in Milliseconds) represents an absolute value.
# No matter what, the connection will not be re-used beyond its TTL.
httpclient4.time_to_live=60000
```

修改`httpclient4.time_to_live=60000`后，`keepalive`正常