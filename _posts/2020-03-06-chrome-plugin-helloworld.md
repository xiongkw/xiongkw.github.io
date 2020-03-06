---
layout: post
title: chrome插件Helloworld
categories: [编程]
tags: [chrome]
---


> 一个例子，抓取某网站的数据并发送到指定服务器

#### 1. 编写manifest

清单文件`manifest.json`

```json
{
    "manifest_version": 2,
    "name": "HelloWorld",
    "version": "1.0.0",
    "description": "HelloWorld",
	"browser_action": 
    {
        "default_icon": "img/icon.png",
        "default_title": "HelloWorld"
    },
    "content_scripts": 
    [
        {
            "matches": ["https://xxx.com/*"],
            "js": ["js/helloworld.js"],
            "run_at": "document_end"
        }
    ],
    "permissions":
    [
        "cookies",
        "https://192.168.1.100/*"
    ]
}
```

> `content_scripts`表示在`matches`的网页执行指定`js`

#### 2. helloworld.js

执行`ajax`请求获取数据

```js
axios.get("https://xxx.com/resources")
.then( res => {
    console.log(res.data);
});
```

> 因为涉及到会话，所以这里需要配置`permissions-cookies`权限

#### 3. 发送数据到本地服务器

```js
axios.get("https://xxx.com/resources")
.then( res => {
    console.log(res.data);
    axios.post("https://192.168.1.100/resources", res.data);
});
```

> 这里涉及到跨域，所以需要配置`permissions-https://192.168.1.100/*`权限

#### 4. 测试

打开`chrome://extensions/`，拖动`helloworld`文件夹到页面，再访问`https://xxx.com`，查看`Console`