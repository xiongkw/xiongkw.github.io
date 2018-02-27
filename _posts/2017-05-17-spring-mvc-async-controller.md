---
layout: post
title: spring-mvc中的异步controller实现
categories: [编程, java, spring, web]
tags: [spring-mvc, async]
---

> `servlet 3.0`加入了异步支持

#### 1. 同步servlet

即普通`servlet`
```java
@WebServlet(urlPatterns = "/sync")
public class SyncServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        resp.getWriter().println("Hello world");
    }
}
```

#### 2. 异步servlet
```java
@WebServlet(urlPatterns = "/async", asyncSupported = true)
public class AsyncServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        AsyncContext asyncContext = req.startAsync();
        new Thread(()->{
            try {
                Thread.sleep(5000);
                PrintWriter writer = asyncContext.getResponse().getWriter();
                writer.println("Hello world");
                asyncContext.complete();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();

    }
}
```

> `asyncSupported = true`用于开启异步支持，另见`web.xml`配置`<async-supported>true</async-supported>`   
> `AsyncContext`,异步`servlet`是基于它来完成   
> `asyncContext.complete()`，完成异步操作

#### 3. spring中的异步controller
```java
@RestController
public class AsyncController {

    @RequestMapping(value = "/asyncController", method = RequestMethod.GET)
    public DeferredResult<String> test(){
        DeferredResult<String> result = new DeferredResult<>();
        new Thread(()->{
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            result.setResult("Hello world");
        }).start();
        return result;
    }

}
```

#### 4. 关于异步servlet

* 分别访问同步和异步`servlet`，发现二者响应时间都在5s-6s，说明`异步`并不是作用在`http`请求上
* `web容器`一般分为`accept线程`和`handler线程`(`java nio`)，大量的耗时请求会耗尽`handler线程`，从而阻塞后续请求
* `异步`其实是作用于`handler线程`上，使得一个处理线程可以立刻返回并接着处理下一个请求，等到耗时的请求操作完成以后，再通知处理线程响应结果
* `异步servlet`适用于有`高延迟处理`的情况，可以提高服务器的并发处理能力，但并不能提高吞吐量