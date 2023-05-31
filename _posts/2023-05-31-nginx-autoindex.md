---
layout: post
title: nginx文件下载自动索引
categories: [linux]
tags: [nginx]
---

> 如何用nginx搭建一个文件下载服务器

#### 1. nginx配置autoindex

```
location /download {
    root   /download/;
    autoindex on;
    autoindex_format html;
    autoindex_exact_size off;
    autoindex_localtime on;
}

location / {
    root   html;
    index  index.html index.htm;
}
```


访问/download，发现文件名过长，被截断成"..."了

```
../
2023-05-31-xxxxxxxxxxxxxxxxxxxxxxxxx... 31-May-2023 14:56     24M
```

#### 2. 格式美化

##### 2.1 安装ngx_http_addition_module

```
--with-http_addition_module
```

##### 2.2 编写autoindex.html文件

目标很明确：展示文件全名

```
<style>
pre a{
  display: inline-block;
  min-width: 600px;
}
</style>
<script>
(function (window) {
    let links = document.getElementsByTagName('a');
    if(links.length==0){
      return;
    }
    for(let a of links){
      a.innerText=a.getAttribute('href');
    }
}(window));
</script>
```

##### 2.3 修改nginx配置
```
location /download {
    root   /download/;
    autoindex on;
    autoindex_format html;
    autoindex_exact_size off;
    autoindex_localtime on;
    add_after_body /autoindex.html;
}
```

> 注意`/autoindex.html`的请求会匹配到`location /`，所以`autoindex.html`文件应该放到root对应的html目录下


#### 3. 参考

* [Module ngx_http_autoindex_module](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)
* [Module ngx_http_addition_module](https://nginx.org/en/docs/http/ngx_http_addition_module.html#add_after_body)