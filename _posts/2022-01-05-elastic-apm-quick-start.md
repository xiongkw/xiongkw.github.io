---
layout: post
title: Elastic APM入门
categories: [编程, elastic]
tags: [apm]
---

> 

#### 概要说明

本例基于Elastic APM实现前后端服务的性能监控

* 快速定位软件的性能缺陷，指导研发优化代码

统计前端用户体验的三大核心指标LCP、FID、CLS   
统计和追踪慢接口，快速定位慢方法   
统计和分析前后端的运行时异常

* 统计服务的吞吐率和延迟，为部署扩容提供依据

统计服务的吞吐率和延迟（包括页面加载、路由变化、http请求、用户事件）

#### 名词解释

* Transaction：事务，一次完整的处理过程。例如从页面开始加载到渲染完成，或是从用户点击按钮开始到页面响应完成
* Span：事务内的单个事件，例如一次http请求、一次方法调用等

* FCP（First contentful paint）：从页面开始加载到页面内容的任何部分呈现在屏幕上的时间
* LCP（Largest Contentful Paint）：视窗最大可见图片或者文本块的渲染时间
* FID （first Input Delay）：从用户第一次与页面交互到浏览器实际能够响应的时间
* CLS（Cumulative Layout Shift）：累计位移偏移，页面渲染过程中突然出现元素偏移（例如图片）。计算方式为：位移影响的面积*位移距离。
* Long Task：用时超过50ms并且会阻塞UI线程的用户操作或浏览器任务，参考[Long_Tasks_API](https://developer.mozilla.org/zh-CN/docs/Web/API/Long_Tasks_API)
* TBT（Total blocking time）： 从FCP到事务完成之间发生的所有Long Task时长总和

* Throughput：吞吐率，服务器单位时间内处理的请求数（TPS/TPM）
* Latency：本例中指一个Transaction所用的时间（一般指服务器收到请求到响应的时间）

> 参考[Supported Technologies](https://www.elastic.co/guide/en/apm/agent/rum-js/5.x/supported-technologies.html)

#### 1. 安装ElasticSearch和Kibana

参考[Elastic APM Quick start](https://www.elastic.co/guide/en/apm/guide/current/apm-quick-start.html#set-up-fleet-traces)

##### 1.1 ElasticSearch

* 下载安装包并解压，[ElasticSearch](https://www.elastic.co/cn/downloads/elasticsearch)

```
$ tar zxvf elasticsearch-7.16.2-linux-x86_64.tar.gz
```

* 修改配置config/elasticsearch.yml

```
cluster.name: apm
node.name: node-1
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node

xpack.security.enabled: true 
xpack.security.authc.api_key.enabled: true
```

* 启动

```
$ export ES_JAVA_HOME=./jdk
$ nohup bin/elasticsearch >/dev/null 2>&1 &
```

* 设置用户密码

```
./bin/elasticsearch-setup-passwords interactive
```

* 访问ElasticSearch

浏览器访问http:127.0.0.1:9200，使用elastic账号：elastic/xxx

##### 1.2 Kibana

* 下载安装包并解压，[Kibana](https://www.elastic.co/cn/downloads/kibana)

```
$ tar zxvf kibana-7.16.2-linux-x86_64.tar.gz
```

* 修改配置config/kibana.yml

```
server.port: 9201
server.host: 0.0.0.0
elasticsearch.hosts: ["http://127.0.0.1:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "xxx"

xpack.security.enabled: true
xpack.encryptedSavedObjects.encryptionKey: "something_at_least_32_characters"
```

* 启动

```
$ nohup bin/kibana serve > std.log 2>&1 &
```

* 访问Kibana

浏览器访问http:127.0.0.1:9201，使用elastic账号：elastic/xxx

#### 2 安装Fleet Server

* 下载安装包[Elastic Agent](https://www.elastic.co/cn/downloads/elastic-agent)

```
$ tar zxvf elastic-agent-7.16.2-linux-x86_64.tar.gz
```

* 打开Kibana-Management-Fleet，初次打开有点慢，等几分钟就好

```
security mode选Quick Start

Genearte service token
```

* 复制install命令并执行

```
$ sudo ./elastic-agent install ...
```

* 在Fleet-UI Agents中查看

* 修改Fleet Server端口的方法

```
1. Kibana-Fleet选中Agent修改Fleet Server integration
2. 后台修改/opt/Elastic/Agent/fleet.yml
fleet:
  host: 10.3.0.84:9202
  server:
    host: 10.3.0.84
    port: 9202
3. 重启agent：systemctl restart elastic-agent
```

* 查看状态

```
$ systemctl status elastic-agent
$ elastic-agent status
Status: HEALTHY
Message: (no message)
Applications:
  * fleet-server           (HEALTHY)
                           Running on policy with Fleet Server integration: 499b5aa7-d214-5b5d-838b-3cd76469844e
  * filebeat_monitoring    (HEALTHY)
                           Running
  * metricbeat_monitoring  (HEALTHY)
                           Running
```

#### 3. 安装Elastic APM integration

* 打开Kibana-Management-Integrations，选择Elastic APM

* 选择Fleet - APM integration，Add Elastic APM

```
Host: 192.0.0.100:9203
URL: http://192.0.0.100:9203

Agent policy: Default Fleet Server Policy
```

注意：这里共用了`Fleet Server Agent`，所以选择`Default Fleet Server Policy`

* 保存继续：`Save and deploy changes`

* 查看elastic-agent状态，多了一个apm-server

```
$ elastic-agent status

Status: HEALTHY
Message: (no message)
Applications:
  * fleet-server           (HEALTHY)
                           Running on policy with Fleet Server integration: 499b5aa7-d214-5b5d-838b-3cd76469844e
  * filebeat_monitoring    (HEALTHY)
                           Running
  * metricbeat_monitoring  (HEALTHY)
                           Running
  * apm-server             (HEALTHY)
                           Running
```

* 访问apm server
```
$ curl http://192.18.1.100:9203
{
  "build_date": "2021-12-18T19:59:06Z",
  "build_sha": "24fe620eeff5a19e2133c940c7e5ce1ceddb1445",
  "publish_ready": true,
  "version": "7.16.2"
}
```

#### 4. 安装APM Agent

以前端(vue)和后端(java)为例

##### 4.1 后端

无侵入，使用javaagent方式启动即可，参考[APM Java Agent Reference](https://www.elastic.co/guide/en/apm/agent/java/1.x/index.html)

```
-javaagent:/home/apm/elastic-apm-agent-1.28.4.jar 
-Delastic.apm.service_name=backend 
-Delastic.apm.service_version=1.0
-Delastic.apm.environment=development 
-Delastic.apm.transaction_sample_rate=1 
-Delastic.apm.enabled=true 
-Delastic.apm.server_urls=http://192.168.1.100:9203 
-Delastic.apm.secret_token=apm-test  
-Delastic.apm.application_packages=com.mytest
```

##### 4.2 前端

需要编码，参考[Vue integration](https://www.elastic.co/guide/en/apm/agent/rum-js/5.x/vue-integration.html)

```
import { ApmVuePlugin } from '@elastic/apm-rum-vue';
Vue.use(ApmVuePlugin, {
  router,
  config: {
      serverUrl: 'http://192.168.1.100:9203', // apm server地址
      serviceName: 'frontend', // 服务名称
      serviceVersion: '1.0', // 服务版本
      environment: 'development', // 环境
      active: true, // 开关
      breakdownMetrics: true, // 是否采集breakdownMetrics
      transactionSampleRate: 1.0, // 采样率
    }
});
```

添加用户上下文，参考[Agent API](https://www.elastic.co/guide/en/apm/agent/rum-js/5.x/agent-api.html#apm-set-user-context)

```
this.$apm.setCustomContext(context)
```

#### 5. 统计分析

在Kibana-Observability-APM中查看

#### 6. 参考

* [Set up minimal security for Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/security-minimal-setup.html)
* [Fleet Server](https://www.elastic.co/guide/en/fleet/7.16/fleet-server.html#_learn_more)
* [Fleet UI settings](https://www.elastic.co/guide/en/fleet/7.16/fleet-settings.html)
* [Centrally manage Elastic Agents in Fleet](https://www.elastic.co/guide/en/fleet/7.16/manage-agents-in-fleet.html)
* [Elastic Agent command reference](https://www.elastic.co/guide/en/fleet/7.16/elastic-agent-cmd-options.html#elastic-agent-enroll-command)
* [Secure communication with APM agents](https://www.elastic.co/guide/en/apm/guide/current/secure-agent-communication.html)
* [How to apply source maps to error stack traces when using minified bundles](https://www.elastic.co/guide/en/apm/guide/7.16/sourcemaps.html#sourcemap-rum-upload)
* [Kibana Guide](https://www.elastic.co/guide/en/kibana/current/index.html)
* [APM Agent central configuration](https://www.elastic.co/guide/en/kibana/current/agent-configuration.html)
* [APM reader user](https://www.elastic.co/guide/en/kibana/current/apm-app-reader.html)