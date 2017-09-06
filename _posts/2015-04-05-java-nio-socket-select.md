---
layout: post
title: java中的nio多路复用 socket编程
categories: [编程, java]
tags: [IO, java]
---

> java nio (jdk1.4 io) 的socket编程

server
```java
public static void main(String[] args) throws IOException {
    //多路复用选择器
    Selector sel = Selector.open();
    ServerSocketChannel ssc = ServerSocketChannel.open();
    //设置为非阻塞模式
    ssc.configureBlocking(false);
    ssc.socket().bind(new InetSocketAddress(8081), 1024);
    //在选择器中注册ACCEPT事件
    ssc.register(sel, SelectionKey.OP_ACCEPT);
    new Thread(() -> {
        while (true) {
            try {
                //调用选择器，选择准备就绪的事件
                if (sel.select() == 0) {
                    continue;
                }
                Iterator<SelectionKey> iterator = sel.selectedKeys().iterator();
                while (iterator.hasNext()) {
                    SelectionKey sk = iterator.next();
                    iterator.remove();
                    if (sk.isValid()) {
                        handle(sel, sk);
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }).start();
}

protected static void handle(Selector sel, SelectionKey sk) throws IOException {
    if (sk.isAcceptable()) {
        ServerSocketChannel ssc = (ServerSocketChannel) sk.channel();
        //接受连接请求，打开通道
        SocketChannel sc = ssc.accept();
        //设置为非阻塞模式
        sc.configureBlocking(false);
        //监听读事件
        sc.register(sel, SelectionKey.OP_READ);
    }
    if (sk.isReadable()) {
        SocketChannel sc = (SocketChannel) sk.channel();
        //分配缓冲,如果缓冲分配过少，则一次无法读取完成，该事件会被调用多次
        ByteBuffer buff = ByteBuffer.allocate(1024);
        //从通道读取到缓冲区
        int len = sc.read(buff);
        //len == -1表示对方关闭了连接
        if (len == -1) {
            //被动关闭，不关闭会导致CLOSE_WAIT
            sc.close();
            sk.cancel();
            return;
        }
        if (len > 0) {
            //缓冲区从写转换为读状态
            buff.flip();
            byte[] bytes = new byte[len];
            //读取缓冲区到byte[]
            buff.get(bytes);
            //清空缓冲区以备下次重用(如果可能的话)
            buff.clear();
            String str = new String(bytes, "UTF-8");
            System.out.println(str);
            if ("bye".equals(str)) {
                sc.write(ByteBuffer.wrap(bytes));
            } else {
                //把缓冲区的数据写到通道里
                sc.write(ByteBuffer.wrap(("Echo: " + str).getBytes()));
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

> IO多路复用技术，是使用一个调用(select)去获取多个IO调用的状态(读，写等)，而这个调用(select)由操作系统支持，常用的linux复用方式有select、poll、epoll   
> 其原理同[java中的nio socket编程]({{ site.url}}/2015/04/04/java-nio-socket/)，不同的是select的轮询由操作系统完成 