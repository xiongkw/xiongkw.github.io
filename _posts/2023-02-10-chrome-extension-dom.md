---
layout: post
title: Chrome插件修改网页内容
categories: [编程]
tags: [javascript]
---

>  

#### 1. 一个例子

添加右键菜单，隐藏或显示网页图片元素

#### 2. 右键菜单

##### 2.1 manifest

清单文件添加contextMenus权限

```
	"content_scripts": [
        {
            "matches": ["<all_urls>"],
            "js": ["js/jquery-3.4.1.min.js", "js/content.js"],
            "run_at": "document_end"
        }
    ],
	"background": {
		"service_worker": "js/background.js"
	},
    "permissions":[
		"tabs",
		"contextMenus"
    ],
    "web_accessible_resources": [{
        "resources": [
        "js/jquery-3.4.1.min.js"
		],
        "matches": ["*://*/*"]
    }]
```

##### 2.2 创建右键菜单

在background.js中创建右键菜单

```
chrome.runtime.onInstalled.addListener(function() {
	  
	chrome.contextMenus.create({
		id: 'Hide',
		title: 'Hide',
		type: 'normal',
		contexts: ['all']
	}, function () {
		console.log('contextMenus are create.');
	});

	chrome.contextMenus.create({
		id: 'Show',
		title: 'Show',
		type: 'normal',
		contexts: ['all']
	}, function () {
		console.log('contextMenus are create.');
	});
	
	
});
```

#### 3.菜单事件

##### 3.1 菜单事件监听

在background.js中注册菜单事件监听器

```
chrome.runtime.onInstalled.addListener(function() {

    chrome.contextMenus.onClicked.addListener(function(info, tab){
        // 发送消息通知页面
		chrome.tabs.sendMessage(tab.id, {'contextMenuId': info.menuItemId}, function(response) {});
	});
	
});
```

##### 3.2 菜单事件响应实现

在content.js中编写菜单事件响应代码

```
$(function(){
	chrome.runtime.onMessage.addListener(async function (request, sender, sendResponse) {
		if(request.contextMenuId == "Hide"){
			$('image,img').hide();
		}else if(request.contextMenuId == "Show"){
			$('image,img').show();
		}
	});
});
```