---
layout: post
title: Javascript中的线程和事件
categories: [编程, javascript]
tags: [base64, img]
---


> 使用`javascript`把图片转为`base64`格式

#### 1. 一个例子

```js
async function base64Image(url){
    return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        xhr.onload = () => {
            var reader = new FileReader();
            reader.onloadend = function() {
                resolve(reader.result);
            }
            reader.readAsDataURL(xhr.response);
        };
        xhr.onerror = () => reject(xhr.statusText);
        xhr.open("GET", url);
        xhr.responseType = 'blob';
        xhr.send();
    });
}

base64Image("http://xxx/a.png").then((base64) => {
    console.log(base64);
});
```

#### 2. 参考

* [How to convert image into base64 string using javascript](https://stackoverflow.com/questions/6150289/how-to-convert-image-into-base64-string-using-javascript)