---
layout: post
title: Javascript中的线程和事件
categories: [编程, javascript]
tags: [线程, 事件]
---


> 在`javascript`中我们常常使用`setTimeout`和`setInterval`来模拟多线程

#### 1. 一个例子

```js
setTimeout(() => {
    console.log("Hello 1");
}, 1000);

setTimeout(() => {
    console.log("Hello 2");
}, 1000);
```

> 运行以上代码，发现`Hello 1`和`Hello 2`几乎同时打印，看起来两个定时器是异步执行的

#### 2. 为什么setTimeout延迟了

```js
function sayHello(){
    console.log("Hello world");
}

setTimeout(sayHello, 1000);

setTimeout(() => {
let beginTime = new Date().getTime();
while(true){
    if(new Date().getTime()-beginTime > 10000) break;
}
}, 0);
```

> 运行上面代码，发现`sayHello`方法大概延迟了`10s`才执行

#### 3. javascript主线程和事件循环

`js`引擎使用队列管理所有事件，使用单线程循环获取队列中事件并执行，所以两事件不可能并发执行

如上面例子，第二个`setTimeout`需要执行`10s`，所以第一个`setTimeout`事件会被阻塞`10s`

#### 4. javascript中的多线程

通常我们说的单线程仅仅是对主线程而言，其实`javascript`中还是有多线程

* `Ajax`: `Ajax`请求是另一个线程负责执行
* `Web Worker`: `HTML5`引入的，可以创建多个后台任务并发执行

#### 5. 一个web worker例子

worker1.js

```js
setInterval( ()=>{
    console.log("Hello worker1")
}, 1000);
```

worker2.js

```js
setTimeout(() => {
    let beginTime = new Date().getTime();
    while(true){
        if(new Date().getTime()-beginTime > 10000) break;
    }
    console.log("Hello worker2");
}, 0);
```

main.js

```js
new Worker("worker1.js");
new Worker("worker2.js");
```

> 运行`main.js`可以看到`worker2.js`并不会阻塞`worker1.js`

