---
layout: post
title: 使用jquery-print插件实现网页区域打印
categories: [编程, javascript]
tags: [jquery, print, html]
---


> `javascript`中可以使用`window.print()`来打印整个`document`内容，但要指定打印某个区域，则相对比较复杂

#### 1. 介绍
`jQuery Print Plugin`:jQuery.print is a plugin for printing specific parts of a page

#### 2. 一个例子

以下是一个常见的左菜单右内容的布局，我们希望只打印内容区域

```html
<head>
<script type="text/javascript" src="/public/js/jquery-1.9.1.min.js"></script>
<script type="text/javascript" src="/public/js/jquery.print.min.js"></script>
<script type="text/javascript">
$(function(){
    $('#btn-print').click(function(){
        $('body div.content').print();
    });
});
</script>
</head>
<body>
<button id="btn-print">Print</button>
<div class="aside">
    <h1>Menu</h1>
</div>
<div class="content">
    <h1>Hello jquery print plugin</h1>
    <h1 class="no-print">This will not print</h1>
</div>

</body>
```

> `$('body div.content').print()`: 选择并打印指定的元素，默认会忽略`.no-print`元素

#### 更多细节

```javascript
$("#myElementId").print({
        globalStyles: true,
        mediaPrint: false,
        stylesheet: null,
        noPrintSelector: ".no-print",
        iframe: true,
        append: null,
        prepend: null,
        manuallyCopyFormValues: true,
        deferred: $.Deferred(),
        timeout: 750,
        title: null,
        doctype: '<!doctype html>'
});
```

参考[jQuery Print Plugin](https://github.com/DoersGuild/jQuery.print)