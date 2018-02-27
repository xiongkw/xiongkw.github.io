---
layout: post
title: spring-boot中通过websocket实现客户端和服务端的通信
categories: [编程, java, spring]
tags: [spring-boot, websocket]
---

> 在`web`服务和客户端的实时通信通常会使用`TCP`连接来实现，但`TCP`连接需要开启额外的端口，而`WebSocket`可以和web服务共用相同的端口，本文通过`spring-boot`实现基于`WebSocket`的实时通信。

#### 1.简介
`WebSocket`: `WebSocket`是`HTML5`规范制定的一个新的协议，该协议允许浏览器与`web`服务器建立`TCP`连接，从而实现服务端与客户端的双向实时通信。

#### 2.服务端
`pom.xml`
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
        </dependency>
```

`WebSocketConfig.java`
```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new TextWebSocketHandler(){
            @Override
            protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
                System.out.println(message);
            }

            @Override
            public void afterConnectionEstablished(WebSocketSession session) throws Exception {
                session.sendMessage(new TextMessage("Hello: "+session.getRemoteAddress().getHostName()+":"+session.getRemoteAddress().getPort()));
            }
        }, "/ws");
    }

}
```

#### 3.客户端
`pom.xml`
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-websocket</artifactId>
        </dependency>

        <dependency>
            <groupId>org.eclipse.jetty.websocket</groupId>
            <artifactId>websocket-client</artifactId>
        </dependency>
```
> `spring-websocket`提供了`WebSocketClient`接口，其实现有多种，这里使用`jetty`的`websocket client`实现

`WebSocketClient.java`
```java
@Component
public class WebSocketClient {

    private WebSocketConnectionManager manager;

    @PostConstruct
    public void init() {
        JettyWebSocketClient webSocketClient = new JettyWebSocketClient();
        manager = new WebSocketConnectionManager(webSocketClient, new TextWebSocketHandler(){
            @Override
            protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
                System.out.println(message);
            }
        }, "ws://127.0.0.1:8080/ws");
        manager.start();
    }

    @PreDestroy
    public void destroy() {
        if (manager != null) {
            manager.stop();
        }
    }
}
```

#### 4. 参考文档
[https://docs.spring.io/spring/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#websocket](https://docs.spring.io/spring/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#websocket)
