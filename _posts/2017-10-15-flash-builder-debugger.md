---
layout: post
title: 使用FlashBuilder调试flex应用
categories: [编程, flash, ActionScript]
tags: [flex, FlashBuilder]
---

> 使用Flash Builder调试flex应用，需要安装flash player debugger版本

首先进官网找debugger版本flash player
[http://www.adobe.com/support/flashplayer/debug_downloads.html](http://www.adobe.com/support/flashplayer/debug_downloads.html)

下载安装多次却依然没能成功，ie右键总是没有出现`调试器`选项，严重怀疑flash官方提供的安装方式有问题

于是从Flash Builder安装包中查找`player\win\11.4`
```
InstallAX.exe
```
双击安装，却提示不是最新版本，无法安装

于是打开控制面板卸载flash player，完成后继续安装，仍然提示无法安装

网上搜索删除注册表信息，继续无法安装
```
HKEY_LOCAL_MACHINE\SOFTWARE\Macromedia\FlashPlayer\SafeVersions
```

继续搜索，进adobe官网下载卸载器
[https://helpx.adobe.com/flash-player/kb/uninstall-flash-player-windows.html](https://helpx.adobe.com/flash-player/kb/uninstall-flash-player-windows.html)

卸载后终于成功安装，Flash builder调试也成功运行

> 比较坑的flash，反复折腾了大半天
