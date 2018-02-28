---
layout: post
title: Three.js入门
categories: [编程, javascript]
tags: [3d, three.js]
---

> `HTML5`中制定了`WebGL`标准，使得在网页中通过`javascript`实现`3D`绘图成为了可能

#### 1. 概念
`WebGL`: `Web Graphics Library`是一种`javascript`绘图`api`，`WebGL`可以为`HTML5 Canvas`提供硬件`3D`加速渲染，这样`Web`开发人员就可以借助系统显卡来在浏览器里更流畅地展示`3D`场景和模型了，还能创建复杂的导航和数据视觉化

#### 2. 重要概念
* `渲染器(render)`: 我们可以把渲染器想想成为一个画布，我们需要在这个画布上去画出我们需要展示的东西
* `场景(scene)`: 相当于一个空间，我们需要将展示的东西放在这个空间里，然后再在画布上绘制出来
* `照相机(camera)`: 相当于眼睛，我们想要看到物体，就需要眼睛去看
* `光源(light)`: 物体需要光照才能看见，不然就是漆黑一片(但是在某些情况下展示物体不需要光源)
* `物体(object)`: 我们想要表现的内容，会有形状和材质属性

#### 3. 一个例子
```html
<html>
<head>
<title>My first three.js app</title>
<style>
body { margin: 0; }
canvas { width: 100%; height: 100% }
</style>
</head>
<body>
<script src="three.min.js"></script>
<script>
// 创建一个场景
var scene = new THREE.Scene();

// 创建一个相机(视角)
var camera = new THREE.PerspectiveCamera( 75, window.innerWidth/window.innerHeight, 0.1, 1000 );

// 渲染器
var renderer = new THREE.WebGLRenderer();
renderer.setSize( window.innerWidth, window.innerHeight );
document.body.appendChild( renderer.domElement );

// 创建一个立方体形状
var geometry = new THREE.BoxGeometry( 1, 1, 1 );
// 材质
var material = new THREE.MeshBasicMaterial( { color: 0x00ff00 } );
// 通过形状和材质创建一个物体
var cube = new THREE.Mesh( geometry, material );
// 把物体加入场景
scene.add( cube );

camera.position.z = 5;

// 添加动画效果
var animate = function () {
    requestAnimationFrame( animate );

    cube.rotation.x += 0.1;
    cube.rotation.y += 0.1;

    renderer.render(scene, camera);
};

animate();
</script>
</body>
</html>
```

> 来自`three.js`官方文档

#### 4. 参考

* [three.js](https://threejs.org/)
* [Three.js入门指南](http://www.ituring.com.cn/book/1272)