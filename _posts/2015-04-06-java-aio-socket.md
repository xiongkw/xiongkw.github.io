---
layout: post
title: java中的aio socket编程
categories: [编程, java]
tags: [aio]
---

> java aio (jdk1.7 io) 的socket编程

server
```java
public static void main(String[] args) throws IOException, InterruptedException {
    AsynchronousChannelGroup group = AsynchronousChannelGroup.withThreadPool(Executors.newFixedThreadPool(10));
    AsynchronousServerSocketChannel ssc = AsynchronousServerSocketChannel.open(group);
    ssc.bind(new InetSocketAddress(8081));
    ssc.accept(null, new CompletionHandler<AsynchronousSocketChannel, Server>() {
        @Override
        public void completed(AsynchronousSocketChannel asc, Server attachment) {
            ssc.accept(null, this);
            ByteBuffer buff = ByteBuffer.allocate(1024);
            asc.read(buff, buff, new CompletionHandler<Integer, ByteBuffer>() {
                @Override
                public void completed(Integer result, ByteBuffer attachment) {
                    if(result == -1){
                        try {
                            asc.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                        return;
                    }
                    attachment.flip();
                    byte[] bytes = new byte[result];
                    attachment.get(bytes);
                    attachment.clear();
                    String str = new String(bytes);
                    System.out.println(str);
                    if("bye".equals(str)){
                        asc.write(ByteBuffer.wrap(bytes));
                    }else {
                        asc.write(ByteBuffer.wrap(("Echo: "+str).getBytes()));
                    }
                    //写完后继续读
                    asc.read(buff, buff, this);
                }

                @Override
                public void failed(Throwable exc, ByteBuffer attachment) {
                    exc.printStackTrace();
                }
            });
        }

        @Override
        public void failed(Throwable exc, Server attachment) {
            exc.printStackTrace();
        }
    });

    group.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
}
```

client
```java
public static void main(String[] args) throws IOException, InterruptedException {
    CountDownLatch latch = new CountDownLatch(1);
    AsynchronousSocketChannel asc = AsynchronousSocketChannel.open();
    ByteBuffer buff = ByteBuffer.allocate(1024);
    
    //阻塞的写法
    // Future<Integer> read = asc.read(buff);
    //read.get();
    
    //非阻塞的写法
    asc.connect(new InetSocketAddress("127.0.0.1", 8081), buff, new CompletionHandler<Void, ByteBuffer>() {
        @Override
        public void completed(Void result, ByteBuffer attachment) {
            asc.read(attachment, attachment, new CompletionHandler<Integer, ByteBuffer>() {
                @Override
                public void completed(Integer result, ByteBuffer attachment) {
                    if (result == -1) {
                        close();
                        return;
                    }
                    attachment.flip();
                    byte[] bytes = new byte[result];
                    attachment.get(bytes);
                    attachment.clear();
                    String str = new String(bytes);
                    System.out.println(str);
                    if("bye".equals(str)){
                        close();
                        return;
                    }
                    //继续读
                    asc.read(buff, buff, this);
                }

                private void close() {
                    try {
                        asc.close();
                        latch.countDown();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }

                @Override
                public void failed(Throwable exc, ByteBuffer attachment) {
                    exc.printStackTrace();
                }
            });

            new Thread(() -> {
                try {
                    BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
                    String str = null;
                    while ((str = reader.readLine()) != null) {
                        ByteBuffer buffer = ByteBuffer.wrap(str.getBytes());
                        asc.write(buffer);
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
        public void failed(Throwable exc, ByteBuffer attachment) {
            exc.printStackTrace();
        }
    });
    //使用CountDownLatch来阻塞当前线程，直到socket关闭
    latch.await();
}
```

> 异步和同步的区别在于消息通知的方式，异步是被通知，表现为回调函数

参考
* [java中的bio socket编程]({{ site.url}}/2015/04/03/java-bio-socket/)
* [java中的nio socket编程]({{ site.url}}/2015/04/04/java-nio-socket/)
* [java中的nio多路复用 socket编程]({{ site.url}}/2015/04/05/java-nio-socket-select/)
* [netty socket编程]({{ site.url}}/2015/04/10/netty-socket/)