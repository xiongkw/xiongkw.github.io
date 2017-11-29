---
layout: post
title: 基于socket.io-p2p实现web版p2p聊天
categories: [编程, javascript]
tags: [webrtc, p2p]
---

> 本文演示如何使用`socket.io-p2p`编写基于浏览器的`p2p`聊天程序

#### 1. WebRTC
关于`WebRTC`：

* `Web Real-Time Communication` 网页实时通信，是一个支持网页浏览器进行实时语音对话或视频对话的技术。Google 2010年收购`Global IP Solutions`公司而获得的一项技术。
* 正在被W3C标准化，即将成为下一代视频通信标准
* 正在蚕食`flash`，`adobe`宣布2020年停止`flash`开发多半是因为它

重要术语：

* `signaling` 翻译为`信令`,两个peer要建立p2p连接就需要交换一些信息，比如双方的ip端口、带宽、解码器等。所以需要提供一个信令服务器，用于交换双方信息。 

* `ICE`: `(Interactive Connectivity Establishment)`, A framework to allow your web browser to connect with peers. There are many reasons why a straight up connection from Peer A to Peer B simply won’t work. It needs to bypass firewalls that would prevent opening connections, give you a unique address if like most situations your device doesn’t have a public IP address, and relay data through a server if your router doesn’t allow you to directly connect with peers. ICE uses some of the following techniques described below to achieve this:
* `STUN` `(Session Traversal Utilities for NAT)`, A protocol to discover your public address and determine any restrictions in your router that would prevent a direct connection with a peer. The client will send a request to a STUN server on the Internet who will reply with the client’s public address and whether or not the client is accessible behind the router’s NAT.
* `TURN` `(Traversal Using Relays around NAT)`, TURN is meant to bypass the Symmetric NAT restriction by opening a connection with a TURN server and relaying all information through that server. You would create a connection with a TURN server and tell all peers to send packets to the server which will then be forwarded to you. This obviously comes with some overhead so it is only used if there are no other alternatives.
* `SDP` `Session Description Protocol`, A standard for describing the multimedia content of the connection such as resolution, formats, codecs, encryption, etc. so that both peers can understand each other once the data is transferring. This is, in essence, the metadata describing the content and not the media content itself. 
* `NAT`: `Network Address Translation`, Used to give your device a public IP address. A router will have a public IP address and every device connected to the router will have a private IP address. Requests will be translated from the device’s private IP to the router’s public IP with a unique port. That way you don’t need a unique public IP for each device but can still be discovered on the Internet.

> 参见[Introduction to WebRTC protocols](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols)

#### 2. simple-peer
> Simple WebRTC video/voice and data channels.

#### 3. socket.io-p2p
> This module provides an easy and reliable way to setup a WebRTC connection between peers and communicate using events (the socket.io-protocol).
  
> Socket.IO is used to transport signalling data and as a fallback for clients where WebRTC PeerConnection is not supported.

#### 4. 代码

二话不多说，上代码：
server.js
```javascript
var ecstatic = require('ecstatic')
var server = require('http').createServer(
  ecstatic({ root: __dirname, handleError: false })
)

var io = require('socket.io')(server);
var p2p = require('socket.io-p2p-server').Server;
server.listen(3030);
console.log('Listening on 3030');

io.on('connection', function(socket){
  
  socket.on('room', function(data){
	console.log(data.username + ' entering: ' + data.room)
	socket.join(data.room);
	
	p2p(socket, null);
  });
  
});
```
client.js
```javascript
var P2P = require('socket.io-p2p');
var io = require('socket.io-client');
var socket = io();

var p2p = new P2P(socket, null, function(){
	
  p2p.usePeerConnection = true;
  p2p.emit('peer-obj', { peerId: p2p.peerId });
  
  p2p.on('upgrade', function(username){
	var msg = {username: document.getElementById("user").value,text: 'Entered!'};
	onmessage(msg);
	disableUser();
	console.log(msg);
	p2p.emit('msg', msg);
  });

  p2p.on("msg", function(msg){
	console.log(msg);
	onmessage(msg);
  });

});

function disableUser(){
	document.getElementById('user').disabled = true;
	document.getElementById('room').disabled = true;
	document.getElementById('btn_enter').disabled = true;
}

function onmessage(msg){
	document.getElementById("messages").value += msg.username +": "+ msg.text + "\n";
}


document.getElementById("btn_enter").addEventListener('click', function(){
	p2p.emit('room', {username: document.getElementById("user").value, room: document.getElementById("room").value});
});

document.getElementById("btn_send").addEventListener('click', function(){
	var msg = {username: document.getElementById("user").value,text: document.getElementById("msg").value};
	onmessage(msg);
	p2p.emit("msg", msg);
	document.getElementById("msg").value  = '';
});

```
index.html
```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
  <title>MyChat</title>
</head>
<body>
<div>
  <label>姓名</label><input type="text" id="user"><label>房间: </label><input type="text" id="room">
  <input type="button" value="进入" id="btn_enter">
  </div>
  <hr>
  <div>
  <textarea readonly id="messages" rows="30" cols="80"></textarea>
  </div>
  <textarea rows="5" cols="60" id="msg"></textarea><input type="button" id="btn_send" value="发送">
  <script src="/bundle.js"></script>
</body>
</html>

```

#### 5. 运行
`browserify`打包
```
browserify client.js -o bundle.js
```

启动server
```
node server.js
```

#### 参考文档
* [Introduction to WebRTC protocols](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols)
* [simple-peer](https://github.com/feross/simple-peer)
* [socket.io-p2p](https://github.com/socketio/socket.io-p2p)
