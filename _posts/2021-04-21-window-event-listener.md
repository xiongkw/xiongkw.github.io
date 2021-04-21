---
layout: post
title: HTML中window事件监听的妙用
categories: [编程, javascript]
tags: []
---


> 一个页面打开多个tab时，tab之间如何通信

#### 1. iframe之间使用postMessage通信

A页面监听`message`事件
```
windowA.addEventListener("message", function(e) {
    console.log(e.data);
});
```

B页面发送消息
```
windowA.postMessage({"msg":"hello"});
```

> 前提条件是B页面能持有A页面`window`对象的引用，所以只能用于iframe之间的通信

#### 2. 不同tab之间借助localStorage通信

A页面监听`storage`事件
```
windowA.addEventListener("storage", function(e) {
    if(e.key == "msg"){
        console.log(e.newValue);
    }
});
```

B页面发起`storage`事件
```
localStorage.setItem("msg", "hello");
```

> 前提是两个页面必须同源