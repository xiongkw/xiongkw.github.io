---
layout: post
title: 使用FlashBuilder调试flex应用
categories: [编程, flash, ActionScript]
tags: [flex, FlashBuilder]
---

> `win7`下使用`Flash Builder`调试flex应用，需要安装`flash player debugger`版本

按提示进`adobe`官网找`debugger`版本`flash player`
[http://www.adobe.com/support/flashplayer/debug_downloads.html](http://www.adobe.com/support/flashplayer/debug_downloads.html)
```
DownloadDownload the Flash Player content debugger for Internet Explorer - ActiveX
DownloadDownload the Flash Player content debugger for Firefox - NPAPI
DownloadDownload the Flash Player projector content debugger
DownloadDownload the Flash Player projector
DownloadDownload the Flash Player content debugger for Opera and Chromium based applications – PPAPI

```

#### 坑IE
下载安装多次却依然没能成功，IE右键总是没有出现`调试器`选项

#### 坑Chrome
换`chrome`，同样不成功，有说法`chrome`内置了一个`flash player`，需要进`chrome://plugins`页面禁用，坑的是新版`chrome`已经禁用了`plugins`功能，于是继续IE

#### 继续坑IE
IE卸载安装多次依然失败，严重怀疑`adobe`官方提供的安装方式有问题，最坑的是`adobe`并不提供离线版的安装文件，每次都是在线安装，可能也下载到了本地临时文件，没细查

#### 坑FlashBuilder
`FlashBuilder`中会不会有离线版？于是在`Flash Builder`安装目录`player\win\11.4` 中找到了IE版
```
InstallAX.exe
```
双击安装，却提示不是最新版本，无法安装

#### 卸载坑
于是打开控制面板卸载`flash player`，完成后继续安装，仍然提示不是最新版本

#### 坑注册表
网上搜索删除注册表信息，继续无法安装
```
HKEY_LOCAL_MACHINE\SOFTWARE\Macromedia\FlashPlayer\SafeVersions
```

#### 终于不坑了
继续搜索，进`adobe`官网下载卸载器
[https://helpx.adobe.com/flash-player/kb/uninstall-flash-player-windows.html](https://helpx.adobe.com/flash-player/kb/uninstall-flash-player-windows.html)

卸载后终于成功安装，`Flash builder`调试也成功运行

> 比较坑的flash，反复折腾了大半天
