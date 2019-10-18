---
layout: post
title: 使用Htmlunit实现网站数据的自动抓取
categories: [编程, linux, java]
tags: [爬虫, htmlunit]
---

> 我在某网站上有一些数据需要整理到本地，其实就是大量复制粘贴的事，为了不消耗键鼠的寿命，只好偷懒写个爬虫来帮忙

#### 1. 分析用户登录协议 

通过`Chrome-DevTools`查看网站登录的`HTTP`协议，发现登录成功后重定向了，并且`DevTools-Network`没有记录到登录协议，原来`Chrome-DevTools`中设置的是不保存重定向前的监听日志，于是勾选`Network-Preserve log`，再次登录才记录到了协议

```
POST https://xxx.com/login

username: fool
password: e10adc3949ba59abbe56e057f20f883e
token: 11854d2a6ec523a97dac010bb0316b015eb274aeb698249a5d42b3e3017c8bb2422e9a2624f6093b183bf58b44cc5879fe63e8cf58cafe51f2f2340d7a9378e5c17506f856df945f
signature: 4ca83e486e5780a9f1f03d871f578d51
```

说明：

* `username`: 账号
* `password`: 密码应该是被编码了，查看`js`源码(混淆过的，很难看懂)，发现是用了`md5`编码
* `token`: 不知怎么来的
* `signature`: 应该是对请求参数做了签名

#### 2. 使用Postman测试

通过`Postman`发送以上请求数据，响应`非法的请求`，看来使用常规手段会比较费时，果断知难而退

#### 3. 使用htmlunit模拟登录

输入用户密码，并触发登录按钮点击

```java
WebClient webClient = new WebClient();
WebClientOptions options = webClient.getOptions();
options.setRedirectEnabled(true);
options.setJavaScriptEnabled(true);
options.setCssEnabled(false);
options.setThrowExceptionOnScriptError(false);
options.setTimeout(10000);
webClient.getCookieManager().setCookiesEnabled(true);
HtmlPage page = webClient.getPage("https://xxx.com/login/");
HtmlListItem o = (HtmlListItem) page.getByXPath("//li[@name='login']").get(0);
o.click();
List<HtmlForm> forms = page.getForms();
HtmlForm form = forms.get(1);
form.getInputByName("username").type("fool");
form.getInputByName("password").type("123456");
HtmlPage p = form.getInputByName("submit").click();
System.out.println(p.getTitleText());
```

> 发现并没有跳转到重定向页面

#### 4. 处理重定向

添加监听器处理重定向，再调用接口获取需要的数据

```java

webClient.addWebWindowListener(new WebWindowAdapter() {
    @Override
    public void webWindowContentChanged(WebWindowEvent event) {
        Page p = event.getNewPage();
        if (p instanceof HtmlPage && "我的首页".equals(((HtmlPage) p).getTitleText())) {
            try {
                WebRequest webRequest = new WebRequest(new URL("https://xx.com/getmylist.json"), HttpMethod.POST);
                webRequest.setAdditionalHeader("X-Requested-With","XMLHttpRequest");
                Page page = webClient.getPage(webRequest);
                String content = page.getWebResponse().getContentAsString();
                System.out.println(content);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
});
```

> 接口调用成功，得到正确数据

#### 5. 参考

* [HtmlUnit](http://htmlunit.sourceforge.net/details.html)
* [Can HtmlUnit handle JavaScript redirects?](https://stackoverflow.com/questions/2173487/can-htmlunit-handle-javascript-redirects?r=SearchResults)
* [jsoup: Java HTML Parser](https://jsoup.org/)