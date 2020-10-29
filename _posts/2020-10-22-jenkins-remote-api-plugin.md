---
layout: post
title: Jenkins Remote Api插件开发
categories: [编程, java]
tags: [jenkins]
---

> 如何通过Jenkins api调用远程Agent节点ssh到内网主机执行命令

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

##### 2.1 继承hudson.Plugin

```java
public class JdrpPlugin extends Plugin {

    public Api getApi() {
        return new RemoteApi(this);
    }
}

```

##### 2.2 编写Api代码

```java
public class RemoteApi extends Api {

    private static final Logger LOGGER = Logger.getLogger(RemoteApi.class.getName());
    
    public RemoteApi(Object bean) {
        super(bean);
    }

    @WebMethod(name = "ssh")
    @RequirePOST
    public HttpResponse ssh(@QueryParameter(value = "node", fixEmpty = true) String nodeName,
                            @QueryParameter(fixEmpty = true) String host,
                            @QueryParameter(fixEmpty = true) String username,
                            @QueryParameter(fixEmpty = true) String password,
                            @QueryParameter(fixEmpty = true) Integer port,
                            @QueryParameter(fixEmpty = true) String command) {
        String check = checkParameters(nodeName, host, username, command);
        if (check != null) {
            return HttpResponses.errorWithoutStack(400, check);
        }
        if (port == null) {
            port = 22;
        }
        Node node = Jenkins.getInstanceOrNull().getNode(nodeName);//getNode方法获取不到master，需要创建其它node
        if (node == null) {
            String msg = "Can't find node [" + nodeName + "]";
            LOGGER.warning(msg);
            return HttpResponses.errorJSON(msg);
        }
        VirtualChannel channel = node.getChannel();
        if (channel == null) {
            String msg = "Can't open channel for node [" + nodeName + "]";
            LOGGER.warning(msg);
            return HttpResponses.errorJSON(msg);
        }

        Map<String, Object> result = null;
        try {
            result = channel.call(new SshCommand(host, port, username, password, command));
            LOGGER.info("Execute ssh command: [" + command + "] with params: [host: " + host + ", port: " + port + ", username: " + username + ", password: " + password + "] succeed, result: \n" + result);
        } catch (Exception e) {
            LOGGER.log(Level.WARNING, "Execute ssh command: [" + command + "] with params: [host: " + host + ", port: " + port + ", username: " + username + ", password: " + password + "] failed", e);
            return HttpResponses.errorJSON(e.getMessage());
        }
        return HttpResponses.okJSON(result);
    }

    private String checkParameters(String node, String host, String username, String command) {
        if (node == null) {
            return "Required Query parameter node is missing";
        }
        if (host == null) {
            return "Required Query parameter host is missing";
        }
        if (username == null) {
            return "Required Query parameter username is missing";
        }
        if (command == null) {
            return "Required Query parameter command is missing";
        }
        return null;
    }

}
```

##### 2.4 编写Callable代码

```java
public class SshCommand extends MasterToSlaveCallable<Map<String, Object>, Exception> {
    private static final long serialVersionUID = 1L;

    private final String host;
    private final int port;
    private final String username;
    private final String password;
    private final String command;

    SshCommand(String host, int port, String username, String password, String command) {
        this.host = host;
        this.port = port;
        this.username = username;
        this.password = password;
        this.command = command;
    }

    @Override
    public Map<String, Object> call() throws Exception {
        Logger logger = Logger.getLogger(SshCommand.class.getName());
        SshClient client = null;
        ChannelExec ec = null;
        Map<String, Object> result = new HashMap<>();
        try {
            client = SshClient.setUpDefaultClient();
            client.start();
            ClientSession session = client.connect(username, host, port).verify().getSession();
            session.addPasswordIdentity(password);
            if (!session.auth().verify().isSuccess()) {
                logger.warning("Ssh auth failed, host: " + host + ", port: " + port + ", username: " + username + ", password: " + password);
                result.put("code", 5001);
                result.put("msg", "Authentication failed");
                return result;
            }
            ec = session.createExecChannel(command);
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            ByteArrayOutputStream err = new ByteArrayOutputStream();
            ec.setOut(out);
            ec.setErr(err);
            ec.open();
            ec.waitFor(EnumSet.of(ClientChannelEvent.CLOSED), 0L);
            Integer exitStatus = ec.getExitStatus();
            String msg = exitStatus == 0 ? out.toString("UTF-8") : err.toString("UTF-8");
            result.put("code", exitStatus);
            result.put("msg", msg);
            return result;
        } catch (Exception e) {
            logger.warning("SSH failed, host: " + host + ", port: " + port + ", username: " + username + ", password: " + password);
            throw e;
        } finally {
            if (ec != null) {
                ec.close();
            }
            if (client != null) {
                client.stop();
            }
        }
    }
}
```

> 需要引入`org.apache.sshd:sshd-core`依赖

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
http://localhost:8080/jenkins/plugin/jdrp-plugin/api/ssh?node=node1&host=192.168.1.100&username=root&password=123456&command=uname -a
```

```
{
  "status": "ok",
  "data": {
    "msg": "Linux eadp-dev-0001.novalocal 3.10.0-1127.8.2.el7.x86_64 #1 SMP Tue May 12 16:57:42 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux\n",
    "code": 0
  }
}
```

#### 5. 参考

* [Plugin tutorial](https://wiki.jenkins.io/display/JENKINS/Plugin+tutorial)
* [Handling Requests](https://www.jenkins.io/doc/developer/handling-requests/)
* [Web Method](https://wiki.jenkins.io/display/JENKINS/Web+Method)
* [Making your plugin behave in distributed Jenkins](https://wiki.jenkins.io/display/JENKINS/Making+your+plugin+behave+in+distributed+Jenkins)