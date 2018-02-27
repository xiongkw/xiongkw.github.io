---
layout: post
title: netty socket编程
categories: [编程, java, netty]
tags: [netty]
---

> 由于`java nio api`的复杂性，`netty`对其做了封装，提供简单强大的`api`

####  1. server
```java
public static void main(String[] args) {
    NioEventLoopGroup bossGroup = new NioEventLoopGroup();
    NioEventLoopGroup workerGroup = new NioEventLoopGroup();
    try {
        ServerBootstrap sb = new ServerBootstrap();
        sb.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 1024).childHandler(new ChannelInitializer<Channel>() {

            protected void initChannel(Channel ch) throws Exception {
                ch.pipeline().addLast(new ChannelHandlerAdapter() {

                    @Override
                    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                        ByteBuf buf = (ByteBuf) msg;
                        byte[] bytes = new byte[buf.readableBytes()];
                        buf.readBytes(bytes);
                        String str = new String(bytes, "UTF-8");
                        System.out.println(str);
                        if("bye".equals(str)){
                            ctx.writeAndFlush(Unpooled.copiedBuffer(bytes));
                            return;
                        }
                        ByteBuf b = Unpooled.copiedBuffer(("Echo: " + str).getBytes());
                        ctx.writeAndFlush(b);
                    }

                });
            }
        });
        ChannelFuture f = sb.bind(8081).sync();
        f.channel().closeFuture().sync();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }

}
```

####  2. client
```java
public static void main(String[] args) throws Exception {
    NioEventLoopGroup group = new NioEventLoopGroup();
    try {
        Bootstrap b = new Bootstrap();
        b.group(group).option(ChannelOption.TCP_NODELAY, true).channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<Channel>() {

                    @Override
                    protected void initChannel(Channel ch) throws Exception {
                        ch.pipeline().addLast(new ChannelHandlerAdapter() {

                            @Override
                            public void channelActive(ChannelHandlerContext ctx) throws Exception {
                                ByteBuf buffer = Unpooled.copiedBuffer("Hello".getBytes());
                                ctx.writeAndFlush(buffer);
                                new Thread(() -> {
                                    try {
                                        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
                                        String str = null;
                                        while ((str = reader.readLine()) != null) {
                                            ctx.writeAndFlush(Unpooled.copiedBuffer(str.getBytes()));
                                            if ("bye".equals(str)) {
                                                break;
                                            }
                                        }
                                    } catch (IOException e) {
                                        e.printStackTrace();
                                    }

                                }).start();
                            }

                            @Override
                            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                ByteBuf buf = (ByteBuf) msg;
                                byte[] bytes = new byte[buf.readableBytes()];
                                buf.readBytes(bytes);
                                String str = new String(bytes, "UTF-8");
                                System.out.println(str);
                                if("bye".equals(str)){
                                    ch.close();
                                }
                            }

                        });
                    }
                });
        ChannelFuture f = b.connect("localhost", 8081).sync();
        f.channel().closeFuture().sync();
    } finally {
        group.shutdownGracefully();
    }

}
```

#### 3. 参考

* [java中的bio socket编程]({{ site.url}}/2015/04/03/java-bio-socket/)
* [java中的nio socket编程]({{ site.url}}/2015/04/04/java-nio-socket/)
* [java中的nio多路复用 socket编程]({{ site.url}}/2015/04/05/java-nio-socket-select/)
* [java中的aio socket编程]({{ site.url}}/2015/04/06/java-aio-socket/)