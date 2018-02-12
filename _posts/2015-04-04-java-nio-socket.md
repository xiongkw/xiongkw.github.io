---
layout: post
title: java中的nio socket编程
categories: [编程, java]
tags: [nio]
---

> `java nio (jdk1.4 io)` 的`socket`编程

server
```java
public static void main(String[] args) throws IOException {
    ServerSocketChannel ssc = ServerSocketChannel.open();
    //设置非阻塞模式
    ssc.configureBlocking(false);
    ssc.socket().bind(new InetSocketAddress(8081), 1024);
    List<SocketChannel> list = new ArrayList<SocketChannel>();
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    while (true) {
        SocketChannel sc = ssc.accept();
        //非阻塞模式下 accept会立即返回，所以这里需要判断sc是否为null
        if (sc != null) {
            sc.configureBlocking(false);
            //非阻塞IO不会阻塞下一个连接，所以可以在一个线程中处理所有连接，这里在list中保持所有channel
            list.add(sc);
        }
        //同步IO需要自己去轮询消息
        Iterator<SocketChannel> iterator = list.iterator();
        while (iterator.hasNext()) {
            SocketChannel next = iterator.next();
            //非阻塞模式下accept完成不代表connect完成
            if (!next.finishConnect()) {
                continue;
            }
            //读取通道消息到缓冲区
            int len = next.read(buffer);
            // len == -1表示客户端关闭了连接
            if (len == -1) {
                iterator.remove();
                //被动关闭，不关闭会导致CLOSE_WAIT
                next.close();
                continue;
            }
            if (len > 0) {
                //转换缓冲区写到读状态
                buffer.flip();
                byte[] bytes = new byte[len];
                //读取缓冲区
                buffer.get(bytes);
                //清空缓冲区
                buffer.clear();
                String str = new String(bytes);
                System.out.println(str);
                if ("bye".equals(str)) {
                    next.write(ByteBuffer.wrap(bytes));
                }else {
                    next.write(ByteBuffer.wrap(("Echo: " + str).getBytes()));
                }
            }
        }
    }
}
```

client
```java
public static void main(String[] args) throws IOException {
    SocketChannel cs = SocketChannel.open();
    cs.connect(new InetSocketAddress("127.0.0.1", 8081));
    //启用新线程读取System.in
    new Thread(() -> {
        try {
            BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
            String str = null;
            while ((str = reader.readLine()) != null) {
                ByteBuffer buffer = ByteBuffer.wrap(str.getBytes());
                cs.write(buffer);
                if ("bye".equals(str)) {
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

    }).start();

    ByteBuffer buff = ByteBuffer.allocate(1024);
    while (true) {
        int len = cs.read(buff);
        // len == -1 表示服务端关闭了连接
        if(len == -1){
            cs.close();
            break;
        }
        //缓冲区从写转换为读状态
        buff.flip();
        byte[] bytes = new byte[len];
        //读取缓冲区数据
        buff.get(bytes);
        //清空缓冲区
        buff.clear();
        String str = new String(bytes, "UTF-8");
        System.out.println(str);
        if("bye".equals(str)){
            cs.close();
            break;
        }
    }

}
```

> 同步非阻塞的IO模式，因为不会阻塞其它请求，所以可以在一个线程中处理所有请求   
> 但因为是同步模式，所以需要自己不断询问消息(事件)是否到达，不断循环调用会消耗过多的CPU资源   

参考

* [java中的bio socket编程]({{ site.url}}/2015/04/03/java-bio-socket/)
* [java中的nio多路复用 socket编程]({{ site.url}}/2015/04/05/java-nio-socket-select/)
* [java中的aio socket编程]({{ site.url}}/2015/04/06/java-aio-socket/)
* [netty socket编程]({{ site.url}}/2015/04/10/netty-socket/)