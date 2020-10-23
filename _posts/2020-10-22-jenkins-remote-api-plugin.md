---
layout: post
title: Jenkins Remote Api插件开发
categories: [编程, java]
tags: [jenkins]
---

> 如何通过Jenkins api调用远程Agent节点

#### 1. 创建插件工程

##### 1.1 修改maven settings.xml

```
<settings>
  <pluginGroups>
    <pluginGroup>org.jenkins-ci.tools</pluginGroup> 
  </pluginGroups>

  <profiles>
    <profile>
      <id>jenkins</id>
      <activation>
        <activeByDefault>true</activeByDefault> 
      </activation>
      <repositories> 
        <repository>
          <id>repo.jenkins-ci.org</id>
          <url>Index of public/</url>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>repo.jenkins-ci.org</id>
          <url>Index of public/</url>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
</settings>
```


##### 1.2 使用hpi创建插件工程

```
$ mvn archetype:generate -Dfilter=io.jenkins.archetypes:plugin
```

#### 2. 插件开发

```java
public class JdrpPlugin extends Plugin {

    public Api getApi() {
        return new SystemApi(this);
    }
}

```

```java
public class SystemApi extends Api {

    public SystemApi(Object bean) {
        super(bean);
    }

    @WebMethod(name = "system_info")
    public HttpResponse getSystemInfo(StaplerRequest req, StaplerResponse resp) throws IOException, InterruptedException {
        String name = req.getParameter("name");
        Node node = Jenkins.getInstanceOrNull().getNode(name);//getNode方法获取不到master，需要创建其它node
        if (node == null) {
            return HttpResponses.errorJSON("Can't find node by name [" + name + "]");
        }
        VirtualChannel channel = node.getChannel();
        if (channel == null) {
            return HttpResponses.errorJSON("Can't open channel for node[" + name + "]");
        }
        try {
            Properties props = channel.call(new GetSystemInfo());
            return HttpResponses.okJSON(props);
        } finally {
            channel.close();
        }
    }

    private static class GetSystemInfo extends MasterToSlaveCallable<Properties, RuntimeException> {
        public Properties call() {
            return System.getProperties();
        }

        private static final long serialVersionUID = 1L;
    }

}

```

#### 3. 运行

```
$ mvn hpi:run
```

也可使用mvnDebug开启远程debug，配合idea remote debug

```
$ mvnDebug hpi:run
```

#### 4. 访问api

需要在Manage Jenkins-Manage Nodes中添加一个node，使用添加的node名称访问api

```
http://localhost:8080/jenkins/plugin/jdrp-plugin/api/system_info?name=test
```

```
{"status":"ok","data":{"java.specification.name":"Java Platform API Specification","sun.cpu.endian":"little",
...
}}
```

#### 5. 参考

* [Plugin tutorial](https://wiki.jenkins.io/display/JENKINS/Plugin+tutorial)

* [Handling Requests](https://www.jenkins.io/doc/developer/handling-requests/)

* [Making your plugin behave in distributed Jenkins](https://wiki.jenkins.io/display/JENKINS/Making+your+plugin+behave+in+distributed+Jenkins)