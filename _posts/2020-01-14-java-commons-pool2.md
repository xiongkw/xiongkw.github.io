---
layout: post
title: 使用commons-pool2创建webclient对象池
categories: [编程, java]
tags: [webclient, pool]
---

> 使用`HtmlUnit-WebClient`做爬虫，需要支持多用户并发，于是想到使用对象池

#### 1. 实现ObjectFactory

```java
@Component
public class WebClientFactory extends BasePooledObjectFactory<WebClient> {

    @Override
    public WebClient create() throws Exception {

        WebClient webClient = new WebClient();
        WebClientOptions options = webClient.getOptions();
        options.setRedirectEnabled(true);
        options.setJavaScriptEnabled(true);
        options.setCssEnabled(false);
        options.setThrowExceptionOnScriptError(false);
        options.setTimeout(10000);
        webClient.getCookieManager().setCookiesEnabled(true);
        return webClient;
    }

    @Override
    public PooledObject<WebClient> wrap(WebClient webClient) {
        return new DefaultPooledObject<>(webClient);
    }

    @Override
    public void passivateObject(PooledObject<WebClient> p) throws Exception {
        WebClient obj = p.getObject();
        if (obj == null) {
            return;
        }
        obj.getCookieManager().clearCookies();
        List<TopLevelWindow> topLevelWindows = obj.getTopLevelWindows();
        for (TopLevelWindow window : topLevelWindows) {
            window.close();
        }
    }

}
```

#### 2. 实现ObjectPool

```java
@Service
public class WebClientObjectPool extends GenericObjectPool<WebClient> {

    @Autowired
    public WebClientObjectPool(PooledObjectFactory<WebClient> factory) {
        super(factory);

    }

}
```

#### 3. 使用

```java
WebClient webClient = pool.borrowObject();
try {
    // do something
} finally {
    pool.returnObject(webClient);
}

```

#### 4. 总结

> 对于一些需要反复创建销毁，且代价比较大的对象，可以使用对象池技术