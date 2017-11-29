---
layout: post
title: 基于RTMFP和flash实现web版的p2p聊天
categories: [编程, flash, ActionScript]
tags: [rtmfp, p2p, MonoServer]
---

> 本文演示如何使用`ActionScript`编写基于`flash`的`p2p`聊天程序

#### 1. RTMFP
`Real Time Media Flow Protocol`,`Adobe`公司开发的一套新的通信协议,可以让Adobe Flash Player的客户端进行p2p通信。

#### 2. NetStream的publish和play
* publish 发布一个指定名称的流
* play 播放一个指定名称的流

> 可以理解为`MQ`中的发布和订阅，由于发布和订阅只能是单向的，所以建立一个p2p的读写连接需要同时建立发布和订阅两个流

#### 3. MonoServer和Cumulus和Cirrus
`adobe`官方提供的`rtmfp`服务器为`Cirrus`，不过这个需要注册，`Cumulus`是一个开源`rtmfp`服务器，现在已经停止开发了，而`MonoServer`是其衍生版，本文使用`MonaServer`

> CumulusServer is a complete open source and cross-platform RTMFP server extensible by way of scripting. CumulusServer has been developed under GPL license in keeping in mind the 4 following notions: speed, light weight, cross-platform and scalable.

> MonaServer is a new Web server born from the Cumulus project. In addition to RTMFP it includes RTMP/RTMPE, WebSocket, HTTP, a NoDB system and a lot of improvements.

#### 4. 代码

二话不多说，上代码：
```xml
<?xml version="1.0"?>
<s:Application xmlns:fx="http://ns.adobe.com/mxml/2009"
			   xmlns:s="library://ns.adobe.com/flex/spark" >
	<fx:Script><![CDATA[
		import mx.controls.Alert;

		private const RTMFP_SERVER:String = "rtmfp://127.0.0.1:1935";

		[Bindable]
		private var username:String;
		[Bindable]
		private var remotePeerId:String;
		[Bindable]
		private var message:String;
		[Bindable]
		private var history:String = "";

		private var conn:NetConnection;

		private var inputStream:NetStream;
		private var outputStream:NetStream;

		private function connect():void {
			conn = new NetConnection();
			conn.addEventListener(NetStatusEvent.NET_STATUS, function (event:NetStatusEvent):void {
				switch (event.info.code) {
					case "NetConnection.Connect.Success":
						btn_connect.enabled = false;
						txt_username.enabled = false;
						onConnectedToServer();
						break;
					case "NetConnection.Connect.Failed":
					case "NetConnection.Connect.Rejected":
					case "NetConnection.Connect.Closed":
						onConnectionFailed();
						break;
				}
			});
			conn.connect(RTMFP_SERVER, username);
		}

		private function onConnectionFailed():void {
			Alert.show("Connection failed");
		}

		private function onConnectedToServer():void {
			var myPeerId:String = conn.nearID;
			history += "Connected to rtmfp server, peer id: \n" + myPeerId + "\n";
			var stream:NetStream = new NetStream(conn, NetStream.DIRECT_CONNECTIONS);
			stream.addEventListener(NetStatusEvent.NET_STATUS, publisherHandler);
			//1.A发布自己的id，公开告诉别人，我上线了，可以通过这个id找到我
			stream.publish(myPeerId);
			//1.1 注册回调对象，当有人连接时会触发onPeerConnect回调
			stream.client = {
				onPeerConnect: function (stream:NetStream):Boolean {
					var farId:String = stream.farID;
					outputStream = new NetStream(conn, NetStream.DIRECT_CONNECTIONS);
					outputStream.addEventListener(NetStatusEvent.NET_STATUS, calleeHandler);
					//4.发布B的id，作为对方连接我的私有线路,同2.1
					outputStream.publish(farId);

					//5.建立到B的input连接
					inputStream = new NetStream(conn, farId);
					//5.1播放自己的id，2.1中B发布的，触发对方的NetStream.Play.Start事件
					inputStream.play(myPeerId);
					inputStream.client = {
						username: function (username:String):void {
							//7.收到B的名字，A的input建立完成
							history += "Connected with: " + username + "\n";
							remotePeerId = farId;
							btn_call.enabled = false;
							txt_remote.enabled = false;
						},
						message: function (username:String, msg:String):void {
							history += username + ": " + msg + "\n";
						}
					}
					return true;
				}

			}
		}

		private function publisherHandler(event:NetStatusEvent):void {

		}

		private function call(peerId:String):void {
			//2.B建立output，用于给A连接(类似TCP中c端连接s端时也要开启一个本地端口给s端连接)
			outputStream = new NetStream(conn, NetStream.DIRECT_CONNECTIONS);
			outputStream.addEventListener(NetStatusEvent.NET_STATUS, callerHandler);
			//2.1 这里用A的id作为私有连接的凭据
			outputStream.publish(peerId);

			var stream:NetStream = new NetStream(conn, peerId);
			//3 播放A的公开流，触发对方的onPeerConnect回调
			stream.play(peerId);

			inputStream = new NetStream(conn, peerId);
			inputStream.client = {
				username: function (username:String):void {
					//10 A收到B的名字,input连接建立完成
					history += "Connected with: " + username + "\n";
				},
				message: function (username:String, msg:String):void {
					history += username + ": " + msg + "\n";
				}
			}
		}

		private function sendMsg(username:String, msg:String):void {
			history += username + ": " + msg + "\n";
			outputStream.send("message", username, msg);
			message = "";
		}

		protected function callerHandler(event:NetStatusEvent):void {
			if (event.info.code == "NetStream.Play.Start") {
				//6.A在播放我，B的output连接建立完成，这里发送B的名字给A
				outputStream.send("username", username);
				btn_call.enabled = false;
				txt_remote.enabled = false;
				//8 播放我的id, 4.1中A发布的我的id，触发A的NetStream.Play.Start事件
				inputStream.play(conn.nearID);
			}
		}

		protected function calleeHandler(event:NetStatusEvent):void {
			if (event.info.code == "NetStream.Play.Start") {
				//9 A收到B播放事件，于是发送自己的身份给B
				outputStream.send("username", username);
			}
		}
		]]></fx:Script>
	<s:layout>  
		<s:VerticalLayout horizontalAlign="center"/>  
	</s:layout>
	<s:Panel title="P2P Chat">
		<s:layout>
			<s:VerticalLayout paddingBottom="5" paddingLeft="5" paddingRight="5" paddingTop="5"/>
		</s:layout>
		<s:HGroup>
			<s:Label width="80" text="UserName:"/>
			<s:TextInput id="txt_username" width="300" text="@{username}"/>
			<s:Button id="btn_connect" label="Connect" click="connect()"/>
		</s:HGroup>
		<s:HGroup width="100%">
			<s:Label width="80" text="FarID:"/>
			<s:TextInput id="txt_remote" width="300" text="@{remotePeerId}"/>
			<s:Button id="btn_call" label="Call" click="call(this.remotePeerId)"/>
		</s:HGroup>
		<s:HGroup width="100%">
			<s:Label width="80" text="Message:"/>
			<s:TextInput width="300" text="@{message}"/>
			<s:Button label="Send" click="sendMsg(this.username, this.message)"/>
		</s:HGroup>
		<s:TextArea width="100%" text="@{history}"/>
	</s:Panel>
</s:Application>
```

#### 5. 运行
启动`MonaServer`, 在`FlashBuilder`中运行

省略...

#### 参考文档
* [Cirrus](https://labs.adobe.com/technologies/cirrus/)
* [ActionScript参考手册](https://help.adobe.com/zh_CN/FlashPlatform/reference/actionscript/3/index.html)
* [MonaServer](http://www.monaserver.ovh/index.html)