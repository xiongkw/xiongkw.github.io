---
layout: post
title: Javascript中的Promise和async-await
categories: [编程, javascript]
tags: [promise, async, await]
---


> 

#### 1. 关于回调

`ES6`之前我们写异步`javascript`都是基于回调函数，例如：

```
function ajaxRequest(url, callback){
    var xhr = new XMLHttpRequest();
    xhr.onload = callback(xhr.response);
    xhr.open("GET", url);
    xhr.send();
}

ajaxRequest(url1, function(result1){
    console.log(result1);
    ajaxRequest(url2, function(result2){
        console.log(result2);
    })
});

```

> 上面是一个丑陋的多层回调例子

#### 2. 使用Promise改写

在`ES6`中，我们可以把上面例子改写成：

```js
function ajaxRequest(url){
    return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        xhr.onload = () => {
            resolve(reader.result);
        };
        xhr.onerror = () => reject(xhr.statusText);
        xhr.open("GET", url);
        xhr.send();
    });
}

ajaxRequest(url1).then( (result1) => {
    return ajaxRequest(url2);
}).then(result2 => {
    console.log(result2);
});

```

> 以上是`Promise`的链式写法，即当`then`中方法返回的也是`Promise`对象时，可以继续在外层写`then`


#### 3. Promise基本用法

```js
ajaxRequest(url1).then( (result1) => {
    console.log(result1);
}).catch(e => {
    console.log(e);
}).finally( => {
    
});
```

> 有点`java`中`try catch finally`的影子

#### 4. async和await

`Promise`虽然好用，但仍然是回调形式的，而有时候我们更喜欢简单粗暴的同步调用，于是`ES7`就提出了`async await`

```
async function ajaxRequest(url){
    return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        xhr.onload = () => {
            resolve(reader.result);
        };
        xhr.onerror = () => reject(xhr.statusText);
        xhr.open("GET", url);
        xhr.send();
    });
}

(async function(){
    let result1 = await ajaxRequest(url1);
    console.log(result1)
    let result2 = await ajaxRequest(url2);
    console.log(result2)
})();
```

> `await`用于等待`Promise`对象返回结果，并且可以直接获取结果   
> `async`用于修饰`function`，表示这个方法内部可以包含`await`   
> `await`只能用在`async`方法内部

#### 5. 附：一个sleep的例子

以前我们使用`setTimeout`来延迟执行

```
function doSomething(){
    //
};

setTimeout(doSomthing, 3000);
```

在`ES7`中可以定义一个同步的`sleep`方法

```
async function sleep(timeInMills){
    await new Promise(r => setTimeout(r, timeInMills));
}

sleep(3000);
doSomething();
```

#### 6. 参考

* [Promises, async/await](https://javascript.info/promise-basics)