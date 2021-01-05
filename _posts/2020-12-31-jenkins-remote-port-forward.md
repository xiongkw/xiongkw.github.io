---
layout: post
title: 通过Jenkins-agent实现跨网络端口转发
categories: [编程, linux]
tags: [jenkins]
---

> 已有跨网络的分布式`jenkins`环境，要求实现从`master`端到`agent`端内网的端口访问

#### 1. 编写插件api

编写一个api用于开启端口转发

```java
@WebMethod(name = "port_forward")
@RequirePOST
public HttpResponse portForward(@QueryParameter(value = "label", fixEmpty = true) String label,
                                @QueryParameter(fixEmpty = true) Integer localPort,
                                @QueryParameter(fixEmpty = true) String remoteHost,
                                @QueryParameter(fixEmpty = true) int remotePort) {
    Node node = getNodeByLabel(label);
    Channel channel = (Channel) node.getChannel();

    new Thread(()->{
        // 创建一个ServerSocket用于监听本地端口
        ServerSocket serverSocket = new ServerSocket(localPort);
        while (true) {
            try {
                // 接收本地连接，创建master端socket
                Socket socket = serverSocket.accept();
                // 创建一个RemoteOutputStream代理到master端本地socket的输出流，供agent端写数据
                RemoteOutputStream masterOut = new RemoteOutputStream(socket.getOutputStream());
                // 调用call方法得到一个RemoteOutputStream，即agent端创建的目的地址socket输出流
                OutputStream agentOut = channel.call(new Callable(){

                    public OutputStream call() throws IOException {
                            // call方法会在agent端执行
                            // 创建一个socket连到目的地址
                            Socket s = new Socket(host, port);
                            // 启动一个线程读取数据并写到master端socket
                            new Thread(()->{
                                IOUtils.copy(s.getInputStream(), masterOut);
                            }).start();
                            // 返回socket输出流供master端写数据
                            return new RemoteOutputStream(s.getOutputStream());
                        }

                });
                // 从master端socket读取数据并写到agent端socket
                IOUtils.copy(socket.getInputStream(), agentOut);
            } catch (InterruptedException e) {
            }
        }
    }).start();

    return HttpResponses.okJSON();
}
```

#### 2. 实现原理

基于jenkins-remoting的RemoteOutputStream实现

#### 3. 参考

* [Jenkins分布式跨网络发布]({{site.url}}/2020/09/17/jenkins-remote-cd/)
* [Jenkins Remote Api插件开发]({{site.url}}/2020/10/22/enkins-remote-api-plugin/)
* [Plugin tutorial](https://wiki.jenkins.io/display/JENKINS/Plugin+tutorial)
* [Handling Requests](https://www.jenkins.io/doc/developer/handling-requests/)
* [Web Method](https://wiki.jenkins.io/display/JENKINS/Web+Method)
* [Making your plugin behave in distributed Jenkins](https://wiki.jenkins.io/display/JENKINS/Making+your+plugin+behave+in+distributed+Jenkins)