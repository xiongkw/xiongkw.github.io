---
layout: post
title: html响应式设计
categories: [编程, html]
tags: [响应式]
---

> `Write once, run anywhere`，早期应该是说java，后来才是html

#### 1. 设置viewport

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=0" />
```

手机上访问网页默认会缩放，以便展示更多内容，但缩放的后果是没法操作，该标签的作用是禁止网页缩放

* `width=device-width`: 视口为设备宽度（即用户设置的一个宽度）
* `initial-scale=1.0`: 初始化的视口大小是1.0倍
* `maximum-scale=1.0`: 最大的倍数是1.0倍
* `user-scalable=0`: 禁止缩放视口

#### 2. 使用css3的media

media有两种用法

##### 2.1 在link标签中根据不同media选择不同css文件

```html
<link rel="stylesheet" href="styleA.css" media="screen">  
<link rel="stylesheet" href="styleB.css" media="screen and (max-width: 800px)">  
<link rel="stylesheet" href="styleC.css" media="screen and (max-width: 600px)">
```

##### 2.2 在css文件中根据不同media选择不同css内容

```css
@media screen and (min-width: 1200px) {
    body {
        background-color: pink;
    }    
}
@media screen and(min-width: 960px) and (max-width: 1199px) {
    body {
        background-color: blue;
    }
}
@media screen and(min-width: 768px) and (max-width: 959px) {
    body {
        background-color: red;
    }
}
@media screen and(min-width: 480px) and (max-width: 767px) {
    body {
        background-color: green;
    }
}
@media screen and (max-width: 479px) {
    body {
        background-color: black;
    }
}
```

#### 3. 其它
##### 3.1 font-size

使用`em、rem`而不是`px`，例如：

```css
html {
    font-size: 100%; //默认16px
}

body {
    font-size: 1.25em; //16*1.25=20px
}

div {
    font-size: 2.5rem; //16*2.5=40px
}
```

* `em`: 相对于父元素`font-size`的倍数
* `em`: 相对于根元素`font-size`的倍数

##### 3.2 width height

> 使用百分比(相对于父元素)而不是`px`

##### 3.3 padding margin

> 使用百分比(相对于父元素width)而不是`px`
