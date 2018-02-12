---
layout: post
title: netty 实现同步请求
categories: [编程, java, netty]
tags: [netty]
---

> `netty`使用异步的编码风格，即使用回调函数，但实际上，我们往往更习惯用同步(请求-响应)的方式来编写`rpc`.本文演示如何使用`netty`实现同步的`rpc`

如下是一个服务端`rpc`调用客户端的例子，注意以本文说的服务端和客户端是从`netty`角度，如果从`rpc`的角度看二者是颠倒的

#### 1. 客户端代码
```java
private static void connect(final String host, final int port) {
    NioEventLoopGroup workerGroup = new NioEventLoopGroup();
    Bootstrap b = new Bootstrap();
    b.group(workerGroup).channel(NioSocketChannel.class).option(ChannelOption.SO_KEEPALIVE, true)
            .option(ChannelOption.SO_REUSEADDR, true).handler(new ChannelInitializer<SocketChannel>() {
        @Override
        public void initChannel(SocketChannel ch) throws IOException {
            SerialMarshallerFactory factory = new SerialMarshallerFactory();
            MarshallingConfiguration config = new MarshallingConfiguration();
            ch.pipeline().addLast("MessageDecoder", new MarshallingDecoder(new DefaultUnmarshallerProvider(factory, config)))
                    .addLast("MessageEncoder", new MarshallingEncoder(new DefaultMarshallerProvider(factory, config)))
                    .addLast("ReadTimeoutHandler", new ReadTimeoutHandler(60))
                    .addLast("RequestHandler", new RequestHandler());
        }
    });
    try {
        ChannelFuture f = b.connect(host, port).sync();
        f.channel().closeFuture().sync();
    } catch (Exception e) {
        logger.error("Start agent client failed!", e);
    }
}

class RequestHandler extends ChannelHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        Message message = (Message) msg;
        if (MessageType.REQ_CMD.value() == message.getType()) {
            execCommand(ctx, message);
            return;
        }
        super.channelRead(ctx, msg);
    }

    private void execCommand(ChannelHandlerContext ctx, Message msg) {
        Request cmd = (Request) msg.getBody();
        Result res = cmd.execute();
        Message message = new Message(msg.getSessionId(), MessageType.RES_CMD.value(), res);
        ctx.writeAndFlush(message);
    }
}
```

> `netty`客户端实现非常简单，收到`netty`服务端请求后执行并返回结果

#### 2. netty服务端
```java
public void start() {
    bossGroup = new NioEventLoopGroup();
    workerGroup = new NioEventLoopGroup();
    ServerBootstrap bootstrap = new ServerBootstrap();
    bootstrap.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class).option(ChannelOption.SO_BACKLOG, 1024)
            .option(ChannelOption.SO_KEEPALIVE, true).option(ChannelOption.SO_REUSEADDR, true)
            .handler(new LoggingHandler(LogLevel.INFO)).childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        public void initChannel(SocketChannel ch) throws IOException {
            SerialMarshallerFactory factory = new SerialMarshallerFactory();
            MarshallingConfiguration config = new MarshallingConfiguration();
            ch.pipeline().addLast("MessageDecoder", new MarshallingDecoder(new DefaultUnmarshallerProvider(factory, config)))
                    .addLast("MessageEncoder", new MarshallingEncoder(new DefaultMarshallerProvider(factory, config)))
                    .addLast("ReadTimeoutHandler", new ReadTimeoutHandler(60))
                    .addLast("ResponseHandler", new ResponseHandler());
        }
    });
    try {
        bootstrap.bind(port).sync();
        logger.info("Agent server started, port: [{}]!", port);
    } catch (Exception e) {
        logger.error("Start agent server failed!", e);
    }
}
```

#### 3. netty服务端的异步请求方式

```java
public class ResponseHandler extends ChannelHandlerAdapter implements IRequestExecutor {
    private Map<Long, Callback> requestContext = new HashMap<Long, Callback>();
    
    private ChannelHandlerContextRegistry channelHandlerContextRegistry;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        Message message = (Message) msg;
        if (message.getType() == MessageType.RES_CMD.value()) {
            long sessionId = message.getSessionId();
            Callback callback = requestContext.get(sessionId);
            if(callback != null ){
                callback.onResponse((Result) message.getBody());
            }
            return;
        }
        super.channelRead(ctx, msg);
    }

    @Override
    public Result exec(Request request, String address, Callback callback) {
        ChannelHandlerContext ctx = channelHandlerContextRegistry.get(address);
        Long sessionId = Long.valueOf(request.hashCode());
        requestContext.put(sessionId, callback);
        ctx.writeAndFlush(new Message(sessionId, MessageType.REQ_CMD.value(), request));
    }
}
```

> 异步的请求方式一般需要传入一个`Callback`，等收到响应后再调用`Callback`内的业务逻辑，这在`servlet`编码中会非常复杂

#### 4. netty服务端同步请求的实现

`server ResponseHandler`
```java
public class ResponseHandler extends ChannelHandlerAdapter implements IRequestExecutor {

    private Map<Long, Result> requestContext = new ConcurrentHashMap<>();

    private ChannelHandlerContextRegistry channelHandlerContextRegistry;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        Message message = (Message) msg;
        if (message.getType() == MessageType.RES_CMD.value()) {
            long sessionId = message.getSessionId();
            onResponse(sessionId, (Result) message.getBody());
            return;
        }
        super.channelRead(ctx, msg);
    }

    private void onResponse(long sessionId, Result result) {
        String s = String.valueOf(sessionId).intern();
        synchronized (s) {
            requestContext.put(sessionId, result);
            s.notifyAll();
        }
    }

    @Override
    public Result exec(Request request, String address) {
        ChannelHandlerContext ctx = channelHandlerContextRegistry.get(address);
        if (ctx == null) {
            return new Result(Request.STATUS_FAIL, result);
        }
        Long sessionId = Long.valueOf(request.hashCode());
        String s = String.valueOf(sessionId).intern();
        synchronized (s) {
            ctx.writeAndFlush(new Message(sessionId, MessageType.REQ_CMD.value(), request));
            try {
                s.wait();
                Result result = requestContext.remove(sessionId);
                return result;
            } catch (InterruptedException e) {
                throw new RuntimeException("Request failed!", e);
            } finally {
                requestContext.remove(sessionId);
            }
        }
    }
}
```

> 同步请求基于`sessionId`和线程同步实现   
> `sessionId`: 会话`ID`，用于跟踪请求和响应   
> `synchronized`: 线程同步，这里使用`sessionId`作为线程同步锁