---
layout: post
title: java中的bio socket编程
categories: [编程, java]
tags: [bio]
---

> `java bio (jdk1.0-1.3 io)` 的`socket`编程

#### 1. server
```java
public static void main(String[] args) throws IOException {
    //监听8081端口
    ServerSocket ss = new ServerSocket(8081);
    while (true) {
        final Socket s = ss.accept();
        //每一个socket连接都启用一个线程
        new Thread(() -> {
            try {
                BufferedReader reader = new BufferedReader(new InputStreamReader(s.getInputStream()));
                PrintWriter writer = new PrintWriter(s.getOutputStream(), true);
                while (true) {
                    String line = reader.readLine();
                    // line == null 说明对方关闭了socket
                    if (line == null) {
                        //不close的话会导致CLOSE_WAIT
                        s.close();
                        break;
                    }
                    System.out.println(line);
                    if("bye".equals(line)){
                        writer.println(line);
                    }else {
                        writer.println("Echo: " + line);
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

#### 2. client
```java
public static void main(String[] args) throws IOException {
    //建立socket连接
    final Socket s = new Socket("localhost", 8081);
    //从System.in读取输入是一个阻塞操作，所以需要新启一个线程
    new Thread(() -> {
        try {
            BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
            PrintWriter writer = new PrintWriter(s.getOutputStream(), true);
            String str = null;
            while ((str = reader.readLine()) != null) {
                writer.println(str);
                if ("bye".equals(str)) {
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

    }).start();

    final BufferedReader reader = new BufferedReader(new InputStreamReader(s.getInputStream()));
    // readLine是一个阻塞操作，读不到数据时会一直阻塞
    while (true) {
        String str = reader.readLine();
        //读取到null，说明服务端关闭了socket
        if (str == null || "bye".equals(str)) {
            s.close();
            break;
        }
        System.out.println(str);
    }
}
```

> 同步阻塞的`IO`模式，每个请求都会被阻塞，这里为每个请求启动一个线程，这样各请求之间相互不影响   
> 大量连接会耗尽系统资源，一种优化手段是使用线程池来处理

#### 3. 参考

* [java中的nio socket编程]({{ site.url}}/2015/04/04/java-nio-socket/)
* [java中的nio多路复用 socket编程]({{ site.url}}/2015/04/05/java-nio-socket-select/)
* [java中的aio socket编程]({{ site.url}}/2015/04/06/java-aio-socket/)
* [netty socket编程]({{ site.url}}/2015/04/10/netty-socket/)